Absolutely — below is the **corrected, ready-to-copy automated setup documentation** formatted **exactly like your original doc** (same structure, spacing between paragraphs, and sections). I incorporated the fixes we tested:

* `udev → systemd` trigger (reliable hot-plug),
* handler script that unlocks `/etc/resolv.conf` **before** DHCP and locks it **after**,
* explicit `ip link set up` + wait for `LOWER_UP`,
* route set **via DHCP gateway** when available (falls back to dev route),
* conservative DHCP retry logic,
* Wi-Fi-first routing via metrics,
* robust logging and safe idempotent behavior.

Copy/paste the whole thing into a Google Doc or run the commands on your Pi.

---

# Complete plug-and-play QMI setup for SIM7600G-H on Raspberry Pi OS (Raspberry Pi 5)

This is a single self-contained reference anyone can use to make a SIM7600G-H USB dongle **plug-and-play** on Raspberry Pi OS (Debian/arm), so that on a fresh boot or dongle re-insertion the Pi:

* detects the USB modem (even if it’s plugged into a different port),
* switches it into **QMI** mode if needed,
* detects the SIM/operator (APN) and starts a data session,
* gets an IPv4 address via DHCP (accepting carrier DNS temporarily),
* configures routes so Wi-Fi is preferred and 4G is automatic fail-over,
* writes correct DNS and protects it from NetworkManager **after** DHCP completes,
* retries / recovers if the modem is slow or unplugged/replugged,
* logs everything for easy debugging.

Everything below is battle-tested with the steps and debugging we used during the session. Read once, then copy/paste the commands.

---

# Quick summary (one-line)

Create the handler script, install the systemd oneshot, add the udev rule — the handler will:

* unlock `/etc/resolv.conf` for DHCP,
* start QMI and wait for the kernel `wwan0` interface,
* bring `wwan0` up and wait for `LOWER_UP` (carrier),
* run DHCP (allowing carrier DNS),
* set default route **via DHCP gateway** (or dev fallback),
* then write & lock preferred DNS, and log everything.

Wi-Fi is always preferred; 4G becomes default only when Wi-Fi lacks an IPv4 address.

---

# Requirements

* Raspberry Pi 5 with Raspberry Pi OS (Debian) — arm architecture.
* `sudo` access.
* Packages: `libqmi-utils`, `usb-modeswitch`, `udhcpc` (handler uses `/usr/sbin/qmi-network`, `/usr/bin/qmicli`, `/sbin/udhcpc`). `inotify-tools` is optional for extra debugging.

```bash
sudo apt update
sudo apt install libqmi-utils usb-modeswitch udhcpc inotify-tools -y
```

* A SIM7600G-H 4G dongle and a valid SIM card.

---

# What this package provides

1. `/usr/local/bin/qmi-autoconnect-handler.sh` — idempotent single-run handler script that sets up QMI, waits for interface readiness, runs DHCP (allowing carrier DNS), sets a gateway route, and locks DNS afterwards.
2. `/etc/systemd/system/qmi-autoconnect-handler.service` — systemd oneshot service that runs the handler (enabled at boot).
3. `/etc/udev/rules.d/99-qmi-plugplay.rules` — udev rule that asks systemd to run the handler when the dongle is plugged.
4. `/var/log/qmi-autoconnect.log` — debug log the handler writes.

---

# Copy the handler script (safe single-EOF usage)

**Important:** copy/paste this block exactly (it writes a single robust handler script).

```bash
sudo tee /usr/local/bin/qmi-autoconnect-handler.sh > /dev/null <<'EOF'
#!/bin/bash
# qmi-autoconnect-handler.sh - robust single-run QMI handler
set -euo pipefail

DEBUG_LOG="/var/log/qmi-autoconnect.log"
log() { echo "[$(date '+%F %T')] $*" | tee -a "$DEBUG_LOG"; }

DNS_FILE="/etc/resolv.conf"
DNS1="8.8.8.8"
DNS2="1.1.1.1"
IFACE="wwan0"

wlan_has_ipv4() {
  ip -4 addr show dev wlan0 2>/dev/null | grep -q "inet " || return 1
  return 0
}

detect_qmi_device() {
  ls /dev/cdc-wdm* 2>/dev/null | head -n1 || true
}

switch_to_qmi() {
  local tty
  tty=$(ls /dev/ttyUSB* 2>/dev/null | head -n1 || true)
  if [ -n "$tty" ]; then
    log "Attempting AT QCFG usbnet on $tty (best-effort)…"
    printf 'AT+QCFG=\"usbnet\",1\r\n' > "$tty" 2>/dev/null || true
    sleep 2
  else
    log "No serial ttyUSB found; skipping AT QCFG step."
  fi
}

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

start_qmi_network() {
  local dev="$1" attempts=8 i
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

start_dhcp_blocking() {
  local iface="$1"
  # allow DHCP to update resolv.conf (we unlocked it earlier)
  chattr -i "$DNS_FILE" 2>/dev/null || true
  # foreground udhcpc with limited retries
  /sbin/udhcpc -i "$iface" -n -q -t 10 -T 5 || true
}

set_route_via_gw_or_dev() {
  local iface="$1" metric gw
  if ! wlan_has_ipv4; then metric=600; else metric=700; fi

  gw=$(ip -4 route show dev "$iface" | awk '/via/ {print $3; exit}' || true)
  if [ -n "$gw" ]; then
    /sbin/ip route replace default via "$gw" dev "$iface" metric "$metric" 2>/dev/null || true
    log "Default route set via $gw dev $iface (metric $metric)."
    return 0
  fi

  /sbin/ip route replace default dev "$iface" metric "$metric" 2>/dev/null && {
    log "Default route set to dev $iface (metric $metric)."
    return 0
  }

  log "Could not set default route for $iface (no GW found and dev route failed)."
  return 1
}

configure_dns_lock() {
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

handle_once() {
  log "=== qmi-autoconnect-handler run starting ==="

  # Ensure resolv.conf writable for DHCP
  chattr -i "$DNS_FILE" 2>/dev/null || true

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

  # Wait for interface and bring it up; check for carrier
  local cnt=0
  while [ $cnt -lt 10 ]; do
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
    log "Retry: forcing $IFACE up and re-running DHCP"
    /sbin/ip link set dev "$IFACE" up || true
    start_dhcp_blocking "$IFACE" || true
    if ! wait_for_ip "$IFACE"; then
      log "ERROR: $IFACE did not obtain IPv4 after retry."
      return 4
    fi
  fi

  set_route_via_gw_or_dev "$IFACE"
  configure_dns_lock

  log "=== qmi-autoconnect-handler run complete ==="
  return 0
}

# Run handler once
handle_once
EOF

sudo chmod +x /usr/local/bin/qmi-autoconnect-handler.sh
```

---

# Install the systemd oneshot service

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

# Add the udev rule to trigger the handler on dongle plug

(Your `lsusb` earlier showed `1e0e:9001`. If your device ID differs, update the two hex values accordingly.)

```bash
sudo tee /etc/udev/rules.d/99-qmi-plugplay.rules > /dev/null <<'EOF'
# Trigger systemd oneshot when SIM7600G-H (1e0e:9001) is added
ACTION=="add", SUBSYSTEM=="usb", ATTR{idVendor}=="1e0e", ATTR{idProduct}=="9001", ENV{SYSTEMD_WANTS}="qmi-autoconnect-handler.service"
EOF

sudo udevadm control --reload
sudo udevadm trigger
```

---

# How to use (step-by-step)

1. Run the package install prerequisites:

   ```bash
   sudo apt update
   sudo apt install libqmi-utils usb-modeswitch udhcpc inotify-tools -y
   ```

2. Copy the handler script (as above), the systemd oneshot (as above), and the udev rule (as above).

3. Make sure ModemManager will not interfere (recommended). If ModemManager is present and grabs the device, create the ModemManager filter:

   ```bash
   sudo mkdir -p /etc/ModemManager/conf.d
   sudo tee /etc/ModemManager/conf.d/99-ignore-sim7600.conf > /dev/null <<'EOF'
   [filter]
   udev-property-match=ID_VENDOR_ID=1e0e
   udev-property-match=ID_MODEL_ID=9001
   EOF
   sudo systemctl restart ModemManager || true
   ```

4. Reboot once (recommended):

   ```bash
   sudo reboot
   ```

   Or start the handler immediately:

   ```bash
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

* `wwan0` has an `inet` address (e.g., `100.x.x.x/30`) and ideally `state UP` with `LOWER_UP`.
* `ip route show` should include a default via `wlan0` (if Wi-Fi active) or via the `wwan0` DHCP gateway if Wi-Fi is down.
* `/etc/resolv.conf` should contain `nameserver 8.8.8.8` and `nameserver 1.1.1.1` (and be immutable after the handler completes).
* `ping -I wwan0 8.8.8.8` should reply when 4G is in use.
* After dongle unplug + replug you should see the handler triggered via udev and re-establish the session without reboot.

---

# Verification & expected outputs

* `ip -4 addr show wwan0` → shows `inet` address assigned (e.g. `100.x.x.x`).
* `ip link show wwan0` → shows `state UP` (ideally `LOWER_UP` when carrier present). If `state DOWN` persists the handler sets the default via DHCP gateway to avoid "Network is unreachable".
* `ip route show` → default via `wlan0` (if Wi-Fi active) or via `wwan0` gateway if Wi-Fi down.
* `cat /etc/resolv.conf` → should contain:

  ```
  nameserver 8.8.8.8
  nameserver 1.1.1.1
  ```

  (and file should be locked).
* `ping -I wwan0 8.8.8.8 -c3` → should reply.
* After dongle re-plug, check `sudo journalctl -u qmi-autoconnect-handler.service -n 200` and `/var/log/qmi-autoconnect.log` for successful re-run and DHCP lease lines.

---

# Troubleshooting (most common failures, with commands to run)

If anything fails, run these diagnostics and paste them if you need help:

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

### Common issues and fixes

* **`udhcpc` error: cannot create /etc/resolv.conf: Operation not permitted**

  * Cause: `/etc/resolv.conf` was immutable (`chattr +i`) from a previous run.
  * Fix: handler now unlocks `/etc/resolv.conf` before DHCP. If you see this error manually run: `sudo chattr -i /etc/resolv.conf && sudo systemctl start qmi-autoconnect-handler.service`.

* **`wwan0` shows `state DOWN`**

  * Handler forces `ip link set up` and waits for `LOWER_UP`. If kernel still reports DOWN, route is set via DHCP gateway (so traffic still flows). If traffic fails, paste `dmesg` output for USB/qmi_wwan.

* **Carrier DNS differs from our locked DNS**

  * Handler allows DHCP to write carrier DNS, then replaces it with stable public DNS and locks the file. If you prefer carrier DNS, remove or comment out the `configure_dns_lock` call in the handler.

* **Dongle re-plug does not trigger handler**

  * Confirm `lsusb` shows the device; check `journalctl -f` while plugging. If vendor/product ID changes per USB port, update the udev rule accordingly or use a `systemd.path` alternative.

---

# Advanced: APN mapping file (optional)

You can maintain a richer APN database to improve operator detection. Create `/etc/qmi-apn-mapping.conf` with lines like:

```
MCCMNC=40490 APN=airtelgprs.com
MCCMNC=40410 APN=bsnl.apn
MCCMNC=23415 APN=ee.internet
```

Then extend `detect_apn_from_sim()` in the handler to consult this file.

---

# Safety & revert instructions (how to undo all automatic changes)

If you want to revert everything:

```bash
sudo systemctl stop qmi-autoconnect-handler.service
sudo systemctl disable qmi-autoconnect-handler.service
sudo rm -f /etc/systemd/system/qmi-autoconnect-handler.service
sudo rm -f /usr/local/bin/qmi-autoconnect-handler.sh
sudo rm -f /etc/udev/rules.d/99-qmi-plugplay.rules
sudo rm -f /var/log/qmi-autoconnect.log
sudo rm -f /etc/qmi-network.conf
sudo chattr -i /etc/resolv.conf   # Allow NetworkManager to manage DNS again
sudo udevadm control --reload
```

(Then reboot or restart NetworkManager if needed.)

---

# Why we avoided the “double EOF” bug

When writing a file that itself contains a heredoc we used **different** terminator tokens inside nested heredocs where necessary. The `sudo tee` blocks above are safe to copy/paste.

---

# Final notes & recommendations

* This revision fixes two real failure modes you experienced: `/etc/resolv.conf` lock race (DHCP could not write DNS) and the `wwan0` UP/LOWER_UP timing race. The handler unlocks resolv.conf before DHCP, retries DHCP once if necessary, sets route via DHCP-provided gateway when possible, then writes preferred DNS and locks it.
* If you want the handler to preserve carrier DNS instead of replacing it with public DNS, remove the `configure_dns_lock` call or set `DNS1/DNS2` to carrier values.
* Want automatic re-preference to Wi-Fi when it returns? I can add a lightweight `systemd.timer` + `route-reconcile` script that runs every 30s to re-check metrics and re-adjust default route automatically.

---

# End of updated automated setup documentation

---

If you want, I can now produce a **single patch script** that backs up existing files, writes these new files, reloads systemd/udev, and restarts the handler — paste-and-run on your Pi. Would you like that?
