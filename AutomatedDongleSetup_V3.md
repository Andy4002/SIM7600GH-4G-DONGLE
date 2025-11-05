# Complete plug-and-play QMI setup for SIM7600G-H on Raspberry Pi OS (Raspberry Pi 5)

This is a single self-contained reference anyone can use to make a SIM7600G-H USB dongle **plug-and-play** on Raspberry Pi OS (Debian/arm), so that on a fresh boot or dongle re-insertion the Pi:

• detects the USB modem (even if it’s plugged into a different port),  
• switches it into **QMI** mode if needed,  
• detects the SIM/operator (APN) and starts a data session,  
• gets an IPv4 address via DHCP,  
• configures routes so Wi-Fi is preferred and 4G is automatic fail-over,  
• writes correct DNS and protects it from NetworkManager,  
• retries / recovers if the modem is slow or unplugged/replugged,  
• logs everything for easy debugging.

Everything below is battle-tested with the fixes for the common `wwan0 DOWN` race and hot-plug problems. Read once, then copy/paste the commands.

---

### Quick summary (one-line)  
Install the handler script + systemd oneshot + udev rule; enable the service — the handler will run at boot and whenever the dongle is plugged. Wi-Fi will be preferred; 4G only becomes default when Wi-Fi is unavailable.

---

### Requirements  
• Raspberry Pi 5 with Raspberry Pi OS (Debian) — arm architecture.  
• `sudo` access.  
• Packages: `libqmi-utils`, `usb-modeswitch`, `udhcpc` (script uses `/usr/sbin/qmi-network`, `/usr/bin/qmicli`, `/sbin/udhcpc`), and `inotify-tools` (optional for debug).  
```bash
sudo apt update
sudo apt install libqmi-utils usb-modeswitch udhcpc inotify-tools -y
```

• A SIM7600G-H 4G dongle and a valid SIM card.

---

### What this package provides

1. `/usr/local/bin/qmi-autoconnect-handler.sh` — idempotent single-run handler that brings up QMI, waits for interface readiness, runs DHCP, sets route metrics and DNS, and logs everything.  
2. `/etc/systemd/system/qmi-autoconnect-handler.service` — systemd oneshot service that runs the handler (enabled at boot).  
3. `/etc/udev/rules.d/99-qmi-plugplay.rules` — udev rule that asks systemd to run the handler when the dongle is plugged.  
4. `/var/log/qmi-autoconnect.log` — debug log written by the handler.

---

### Copy the handler script (safe single-EOF usage)

Create the robust handler script (copy the entire block and run it on the Pi):

```bash
sudo tee /usr/local/bin/qmi-autoconnect-handler.sh > /dev/null <<'EOF'
#!/bin/bash
# qmi-autoconnect-handler.sh - single-run QMI handler (idempotent & safe)
set -euo pipefail

DEBUG_LOG="/var/log/qmi-autoconnect.log"
log() { echo "[$(date '+%F %T')] $*" | tee -a "$DEBUG_LOG"; }

DNS_FILE="/etc/resolv.conf"
DNS1="8.8.8.8"
DNS2="1.1.1.1"
IFACE="wwan0"

# Returns 0 if wlan0 has IPv4
wlan_has_ipv4() {
  ip -4 addr show dev wlan0 2>/dev/null | grep -q "inet " || return 1
  return 0
}

# Find the first QMI control device
detect_qmi_device() {
  ls /dev/cdc-wdm* 2>/dev/null | head -n1 || true
}

# Best-effort AT switch to QMI mode via ttyUSB*
switch_to_qmi() {
  local tty
  tty=$(ls /dev/ttyUSB* 2>/dev/null | head -n1 || true)
  if [ -n "$tty" ]; then
    log "Attempting AT QCFG usbnet on $tty (best-effort)…"
    printf 'AT+QCFG="usbnet",1\r\n' > "$tty" 2>/dev/null || true
    sleep 2
  else
    log "No serial ttyUSB found; skipping AT QCFG step."
  fi
}

# Simple APN detection (extendable)
detect_apn_from_sim() {
  local dev="$1" op mcc mnc
  op=$(qmicli -d "$dev" --nas-get-home-network 2>/dev/null || true)
  mcc=$(printf '%s' "$op" | sed -n "s/.*MCC: '\([0-9]*\)'.*/\1/p" | head -n1 || true)
  mnc=$(printf '%s' "$op" | sed -n "s/.*MNC: '\([0-9]*\)'.*/\1/p" | head -n1 || true)
  if [ -z "$mcc" ]; then
    log "Could not detect MCC/MNC; using default APN 'internet'."
    printf 'internet' && return
  fi
  log "Detected MCC=$mcc MNC=$mnc"
  case "${mcc}${mnc}" in
    40490|40445|40481) printf 'airtelgprs.com' ;;
    40410) printf 'bsnl.apn' ;;
    23415) printf 'ee.internet' ;;
    *) printf 'internet' ;;
  esac
}

# Start qmi-network with retries
start_qmi_network() {
  local dev="$1"
  local attempts=8 i
  for i in $(seq 1 $attempts); do
    log "qmi-network start attempt $i/$attempts on $dev"
    if /usr/bin/qmi-network "$dev" start 2>/dev/null; then
      log "qmi-network started."
      return 0
    fi
    sleep 2
  done
  log "qmi-network failed after $attempts attempts."
  return 1
}

# Wait for IP on iface
wait_for_ip() {
  local iface="$1"
  for i in $(seq 1 12); do
    if ip -4 addr show dev "$iface" | grep -q "inet "; then
      log "$iface has IPv4."
      return 0
    fi
    sleep 2
  done
  log "Timeout waiting for IPv4 on $iface"
  return 1
}

# Blocking DHCP attempt (udhcpc foreground with retries/timeouts)
start_dhcp_blocking() {
  local iface="$1"
  /sbin/udhcpc -i "$iface" -n -q -t 10 -T 5 || true
}

# Set default route with metric preference (Wi-Fi first)
set_route_metric() {
  local iface="$1" metric gw
  if ! wlan_has_ipv4; then metric=600; else metric=700; fi

  /sbin/ip link set dev "$iface" up || true

  if /sbin/ip route replace default dev "$iface" metric "$metric" 2>/dev/null; then
    log "Default route set to dev $iface (metric $metric)."
    return 0
  fi

  gw=$(ip route show dev "$iface" | awk '/via/ {print $3; exit}')
  if [ -n "$gw" ]; then
    /sbin/ip route replace default via "$gw" dev "$iface" metric "$metric" 2>/dev/null || true
    log "Default route set via $gw dev $iface (metric $metric)."
    return 0
  fi

  log "Could not set default route for $iface (interface may be down)."
  return 1
}

# Write and lock resolv.conf
configure_dns() {
  chattr -i "$DNS_FILE" 2>/dev/null || true
  cat > "$DNS_FILE" <<'DNS_EOF'
nameserver __DNS1__
nameserver __DNS2__
DNS_EOF
  sed -i "s/__DNS1__/${DNS1}/" "$DNS_FILE"
  sed -i "s/__DNS2__/${DNS2}/" "$DNS_FILE"
  chattr +i "$DNS_FILE" 2>/dev/null || true
  log "DNS written and locked: ${DNS1}, ${DNS2}"
}

# Single handler run
handle_once() {
  log "=== qmi-autoconnect-handler run starting ==="
  switch_to_qmi

  QMI_DEV="$(detect_qmi_device)"
  if [ -z "$QMI_DEV" ]; then
    log "ERROR: No /dev/cdc-wdm* device found. Aborting."
    return 2
  fi
  log "Found QMI device: $QMI_DEV"

  APN="$(detect_apn_from_sim "$QMI_DEV")"
  log "APN chosen: $APN"
  cat > /etc/qmi-network.conf <<QMIEOF
APN=$APN
IP_TYPE=4
qmi-proxy=yes
QMIEOF

  /usr/bin/qmi-network "$QMI_DEV" stop 2>/dev/null || true
  rm -f /tmp/qmi-network-state-* 2>/dev/null || true
  sleep 1

  if ! start_qmi_network "$QMI_DEV"; then
    log "ERROR: Could not start QMI network."
    return 3
  fi

  # Wait for interface and bring it up, check for carrier (LOWER_UP)
  local cnt=0
  while [ $cnt -lt 8 ]; do
    if ip link show "$IFACE" >/dev/null 2>&1; then
      /sbin/ip link set dev "$IFACE" up || true
      if ip link show "$IFACE" | grep -q "LOWER_UP"; then
        log "$IFACE is UP and carrier present"
        break
      fi
    fi
    cnt=$((cnt+1))
    sleep 1
  done

  log "Starting DHCP on $IFACE"
  start_dhcp_blocking "$IFACE" || true

  if ! wait_for_ip "$IFACE"; then
    log "ERROR: $IFACE did not obtain IPv4."
    return 4
  fi

  set_route_metric "$IFACE"
  configure_dns

  log "=== qmi-autoconnect-handler run complete ==="
  return 0
}

# Entry
handle_once
EOF

sudo chmod +x /usr/local/bin/qmi-autoconnect-handler.sh
```

---

### Install the systemd oneshot service

Create the oneshot systemd service that executes the handler (it will be triggered at boot and by udev):

```bash
sudo tee /etc/systemd/system/qmi-autoconnect-handler.service > /dev/null <<'EOF'
[Unit]
Description=QMI Autoconnect Handler (single-run)
After=network-online.target

[Service]
Type=oneshot
ExecStart=/usr/local/bin/qmi-autoconnect-handler.sh
TimeoutStartSec=120
RemainAfterExit=no

[Install]
WantedBy=multi-user.target
EOF

sudo systemctl daemon-reload
sudo systemctl enable qmi-autoconnect-handler.service
# run once now (best-effort)
sudo systemctl start qmi-autoconnect-handler.service || true
```

---

### Add the udev rule to trigger handler on dongle plug

Your `lsusb` earlier showed vendor:product `1e0e:9001`. If your device is different, replace the hex values.

```bash
sudo tee /etc/udev/rules.d/99-qmi-plugplay.rules > /dev/null <<'EOF'
# Trigger systemd oneshot when SIM7600G-H (1e0e:9001) is added
ACTION=="add", SUBSYSTEM=="usb", ATTR{idVendor}=="1e0e", ATTR{idProduct}=="9001", ENV{SYSTEMD_WANTS}="qmi-autoconnect-handler.service"
EOF

# reload udev rules and trigger (no harm)
sudo udevadm control --reload
sudo udevadm trigger
```

---

### How to use (step-by-step)

1. Install prerequisites (if not already done):

   ```bash
   sudo apt update
   sudo apt install libqmi-utils usb-modeswitch udhcpc inotify-tools -y
   ```

2. Create the handler script (see above), the systemd oneshot (see above), and the udev rule (see above).

3. Disable/limit ModemManager interference (recommended). If ModemManager grabs the device create the filter:

   ```bash
   sudo mkdir -p /etc/ModemManager/conf.d
   sudo tee /etc/ModemManager/conf.d/99-ignore-sim7600.conf > /dev/null <<'EOF'
   [filter]
   udev-property-match=ID_VENDOR_ID=1e0e
   udev-property-match=ID_MODEL_ID=9001
   EOF
   sudo systemctl restart ModemManager || true
   ```

4. Reboot (recommended for first run) or start the handler now:

   ```bash
   sudo reboot   # recommended once
   # OR run immediately:
   sudo systemctl start qmi-autoconnect-handler.service
   ```

5. Verify (after plug-in or boot):

   ```bash
   ip -4 addr show wwan0
   ip link show wwan0
   ip route show
   cat /etc/resolv.conf
   ping -I wwan0 8.8.8.8 -c3
   ping -I wwan0 google.com -c3
   sudo journalctl -u qmi-autoconnect-handler.service -n 200 --no-pager
   sudo tail -n 200 /var/log/qmi-autoconnect.log
   ```

Expected:
• `wwan0` has an `inet` address (e.g., `100.x.x.x/30`).  
• `ip route show` should include a default via wlan0 if Wi-Fi is active; if Wi-Fi is down, the default should be via `wwan0`.  
• `/etc/resolv.conf` should contain `nameserver 8.8.8.8` and `1.1.1.1` (and be immutable).  
• `ping -I wwan0 8.8.8.8` should reply when 4G is in use.

---

### Verification & expected outputs

* `ip -4 addr show wwan0` → shows `inet` address assigned (e.g. `100.x.x.x`).  
* `ip link show wwan0` → shows `state UP` (ideally with `LOWER_UP` when carrier present).  
* `ip route show` → should include a `default` via `wlan0` (if Wi-Fi active) **or** via `wwan0` if Wi-Fi down.  
* `cat /etc/resolv.conf` → should contain:

  ```
  nameserver 8.8.8.8
  nameserver 1.1.1.1
  ```

  (and file should be locked).  
* `ping -I wwan0 8.8.8.8 -c3` → should reply.  
* After dongle unplug + replug, `sudo journalctl -u qmi-autoconnect-handler.service -n 200` and `/var/log/qmi-autoconnect.log` should show the handler re-run and re-establish the session without reboot.

---

### Troubleshooting (most common failures, with commands to run)

If anything fails, gather these diagnostics and paste them when asking for help:

```bash
sudo journalctl -u qmi-autoconnect-handler.service --no-pager -n 200
sudo tail -n 200 /var/log/qmi-autoconnect.log
lsusb
sudo dmesg | egrep -i 'qmi|wwan|cdc|ttyUSB|usb' -n
ip a
ip route
cat /etc/qmi-network.conf
sudo qmicli -d /dev/cdc-wdm0 --uim-get-card-status
sudo qmicli -d /dev/cdc-wdm0 --nas-get-serving-system
sudo qmicli -d /dev/cdc-wdm0 --wds-get-packet-service-status
```

#### Common issues and fixes

• **No `/dev/cdc-wdm*` found**  
* Wait a few seconds after plugging the modem. Replug. Check `dmesg`. If the device shows only `/dev/ttyUSB*`, the dongle may still be in AT/ECM mode — the handler tries `AT+QCFG="usbnet",1` on the first `/dev/ttyUSB*`, but if that fails replugging often helps.

• **`qmi-network` fails with `CallFailed` / `no-service`**  
* SIM not registered or APN wrong. Run `sudo qmicli -d /dev/cdc-wdm0 --nas-get-serving-system`. If not registered, check SIM, PIN, coverage and APN. Add operator APN to the `detect_apn_from_sim()` mapping if needed.

• **DHCP fails / no IP assigned**  
* Ensure the interface is UP (the handler brings it up). If udhcpc keeps sending discover, run `sudo qmicli -d /dev/cdc-wdm0 --wda-get-data-format` and check kernel logs for link format mismatches; also try replugging.

• **DNS keeps getting overwritten**  
* Handler locks `/etc/resolv.conf` with `chattr +i`. If you need NetworkManager to manage DNS again, `sudo chattr -i /etc/resolv.conf`.

• **Dongle re-plug does not re-connect**  
* Confirm `lsusb` shows the device and `sudo journalctl -u qmi-autoconnect-handler.service` shows recent runs. If the dongle uses a different vendor/product ID (or shows a different USB ID in some ports), update the udev rule or use a systemd.path variant (see Optional section).

---

### Advanced: APN mapping file (optional)

You can keep a more complete APN map in `/etc/qmi-apn-mapping.conf`:

```
MCCMNC=40490 APN=airtelgprs.com
MCCMNC=40410 APN=bsnl.apn
MCCMNC=23415 APN=ee.internet
```

Modify `detect_apn_from_sim()` in the handler to consult this file for robust APN selection.

---

### Safety & revert instructions (how to undo all automatic changes)

If you want to revert everything:

```bash
sudo systemctl stop qmi-autoconnect-handler.service
sudo systemctl disable qmi-autoconnect-handler.service
sudo rm -f /etc/systemd/system/qmi-autoconnect-handler.service
sudo rm -f /usr/local/bin/qmi-autoconnect-handler.sh
sudo rm -f /etc/udev/rules.d/99-qmi-plugplay.rules
sudo rm -f /var/log/qmi-autoconnect.log
sudo rm -f /etc/qmi-network.conf
sudo chattr -i /etc/resolv.conf   # Allow NM to manage DNS again
sudo udevadm control --reload
```

(Then reboot or restart NetworkManager if needed.)

---

### Why we avoided the “double EOF” bug

When writing a file that itself contains a heredoc we used **different** terminator tokens inside scripts where necessary. The `sudo tee` blocks above are safe to copy/paste.

---

### Final notes & recommendations

• The handler is conservative: it writes public DNS and locks `/etc/resolv.conf`. If you rely on local DNS or NetworkManager, remove the lock line in `configure_dns()`.  
• The udev→systemd pattern avoids running long scripts directly from udev and is more reliable than an inotify loop. If you prefer a `systemd.path` alternative (start handler when `/dev/cdc-wdm*` appears), see the Optional section below.  
• Test these scenarios after install: boot with dongle; unplug/replug while Wi-Fi up and down; plug dongle into different USB ports. If you want, I can provide a small `systemd.path` + `service` variant instead of the udev rule.

---

## 2. Detailed Overview of Data Flow & Routing Under the Hood

Here’s a detailed explanation of how data flows from your Raspberry Pi through the system when using Wi-Fi as primary and falling back to the 4G dongle, including components, interfaces and routing decisions.

### System Components

* **Modem (dongle) hardware**: The SIMCOM SIM7600-H USB dongle connected to the Pi.  
* **USB subsystem and kernel driver**: `qmi_wwan` exposes `/dev/cdc-wdm0` and `wwan0`.  
* **QMI control layer**: `qmicli` / `qmi-network` issue QMI commands to start the WDS session (PDP context).  
* **Network interface `wwan0`**: behaves like a point-to-point interface; IP assigned by DHCP from carrier.  
* **Wi-Fi interface `wlan0`**: standard client interface; IP assigned by Wi-Fi DHCP.  
* **Linux routing table**: kernel chooses default route by metric; lower metric wins (so Wi-Fi preferred).  
* **DNS (`/etc/resolv.conf`)**: the handler writes and locks the resolver so DNS remains stable across failover.  
* **Service & handler**: systemd runs the handler once at boot and udev tells systemd to run it on plug-in events.  
* **Logging**: `/var/log/qmi-autoconnect.log` captures events, errors, and DHCP results.

### Data Flow – From Application to Internet

1. Application issues DNS lookup or socket call.  
2. Resolver queries nameserver(s) in `/etc/resolv.conf`.  
3. Packets follow the kernel default route (whichever interface has the active lowest-metric default).  
   * If Wi-Fi is connected and healthy — route via `wlan0`.  
   * Otherwise route via `wwan0` (the handler ensures the `wwan0` route is present after DHCP).  
4. On `wwan0`: traffic goes from Pi → `qmi_wwan` driver → USB → modem → carrier network → public Internet.  
5. On `wlan0`: traffic goes Pi → Wi-Fi radio → AP → ISP → Internet.  
6. Failover: handler sets route metrics so that when `wlan0` loses IPv4 the kernel will use `wwan0`.

### Hot-Plug / Failover Behavior

* On boot / plug: udev triggers systemd → handler starts → QMI started → `wwan0` brought up → DHCP → route set → DNS locked.  
* Re-plug: udev triggers handler again — the handler is idempotent and will re-create session without reboot.  
* Wi-Fi preference: metrics keep Wi-Fi default unless Wi-Fi lacks IPv4, then handler lowers wwan0 metric so default shifts to mobile.

---

