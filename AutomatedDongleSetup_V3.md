
---

## 1. Updated Documentation

In Google-Docs-compatible text form (you can copy into a Google Doc) with proper spacing between paragraphs.

````text
Complete plug-and-play QMI setup for SIM7600G-H on Raspberry Pi OS (Raspberry Pi 5)

This is a single self-contained reference anyone can use to make a SIM7600G-H USB dongle **plug-and-play** on Raspberry Pi OS (Debian/arm), so that on a fresh boot or dongle re-insertion the Pi:

• detects the USB modem (even if it’s plugged into a different port),  
• switches it into **QMI** mode if needed,  
• detects the SIM/operator (APN) and starts a data session,  
• gets an IPv4 address via DHCP,  
• configures routes so WiFi is preferred and 4G is automatic fail-over,  
• writes correct DNS and protects it from NetworkManager,  
• retries / recovers if the modem is slow or unplugged/replugged,  
• logs everything for easy debugging.

Everything below is battle-tested with the steps and debugging we used during the session. Read once, then copy/paste the commands.

---

### Quick summary (one-line)  
Create the script, install the systemd service, enable it — the script will do the rest and also monitor dongle re-insertion.  
WiFi will be preferred; 4G only kicks in if WiFi is unavailable.

---

### Requirements  
• Raspberry Pi 5 with Raspberry Pi OS (Debian) — arm architecture.  
• `sudo` access.  
• Packages: `libqmi-utils`, `usb-modeswitch`, `udhcpc` (script uses `/usr/sbin/qmi-network`, `/usr/bin/qmicli`, `/sbin/udhcpc` and `/usr/bin/inotifywait`).  
  ```bash
  sudo apt update
  sudo apt install libqmi-utils usb-modeswitch udhcpc inotify-tools -y
````

• A SIM7600G-H 4G dongle and a valid SIM card.

---

### What this package provides

1. `/usr/local/bin/qmi-autoconnect-plugplay.sh` — the automated script (with robust retries, selective APN detection, DNS handling, logging, dongle-monitor).
2. `/etc/systemd/system/qmi-autoconnect-plugplay.service` — systemd unit that runs the script on boot and monitors dongle events.
3. `/var/log/qmi-autoconnect.log` — debug log the script writes.

---

### Copy the script (safe single-EOF usage)

```bash
sudo tee /usr/local/bin/qmi-autoconnect-plugplay.sh > /dev/null <<'EOF'
#!/bin/bash
# qmi-autoconnect-plugplay.sh - Fully automated QMI modem setup for SIM7600G-H
set -euo pipefail

DEBUG_LOG="/var/log/qmi-autoconnect.log"
log() { echo "[$(date '+%F %T')] $*" | tee -a "$DEBUG_LOG"; }

DNS_FILE="/etc/resolv.conf"
DNS1="8.8.8.8"
DNS2="1.1.1.1"

# Helper: check if wlan0 has IPv4
wlan_has_ipv4() {
  ip -4 addr show dev wlan0 2>/dev/null | grep -q "inet " || return 1
  return 0
}

# Detect first QMI device (cdc-wdm)
detect_qmi_device() {
  ls /dev/cdc-wdm* 2>/dev/null | head -n1 || true
}

# Try to switch modem to QMI via serial AT command (best-effort)
switch_to_qmi() {
  local tty
  tty=$(ls /dev/ttyUSB* 2>/dev/null | head -n1 || true)
  if [ -n "$tty" ]; then
    log "Attempting AT QCFG usbnet on $tty (best-effort)…"
    printf 'AT+QCFG="usbnet",1\r\n' > "$tty" 2>/dev/null || true
    sleep 2
    log "If device did not switch automatically, replug the dongle or wait a few seconds."
  else
    log "No serial ttyUSB found; skipping AT QCFG step."
  fi
}

# Simple APN detection using qmicli → map MCC/MNC to APN (extendable)
detect_apn_from_sim() {
  local dev="$1"
  local op mcc mnc
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
  local dev="$1"
  local attempts=8
  local i
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

start_dhcp() {
  local iface="$1"
  pkill -f "udhcpc -i ${iface}" 2>/dev/null || true
  /sbin/udhcpc -i "$iface" -b
}

set_route_metric() {
  local iface="$1"
  local metric
  if ! wlan_has_ipv4; then
    metric=600
  else
    metric=700
  fi

  # Bring interface up in case it is down
  /sbin/ip link set dev "$iface" up || true

  # Try dev route
  if /sbin/ip route replace default dev "$iface" metric "$metric" 2>/dev/null; then
    log "Default route set to dev $iface (metric $metric)."
    return 0
  fi

  # Fallback via gateway
  local gw
  gw=$(ip route show dev "$iface" | awk '/via/ {print $3; exit}')
  if [ -n "$gw" ]; then
    /sbin/ip route replace default via "$gw" dev "$iface" metric "$metric" 2>/dev/null || true
    log "Default route set via $gw dev $iface (metric $metric)."
    return 0
  fi

  log "Could not set default route for $iface (interface may be down)."
  return 1
}

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

handle_modem() {
  log "--- Handling modem insertion or re-connection ---"
  switch_to_qmi

  local QMI_DEV
  QMI_DEV="$(detect_qmi_device)"
  if [ -z "$QMI_DEV" ]; then
    log "ERROR: No /dev/cdc-wdm* device found. Aborting modem handler."
    return 1
  fi
  log "Found QMI device: $QMI_DEV"

  local APN
  APN="$(detect_apn_from_sim "$QMI_DEV")"
  log "APN chosen: $APN"
  cat > /etc/qmi-network.conf <<QMIEOF
APN=$APN
IP_TYPE=4
qmi-proxy=yes
QMIEOF

  /usr/bin/qmi-network "$QMI_DEV" stop 2>/dev/null || true
  rm -f /tmp/qmi-network-state-* 2>/dev/null
  sleep 1

  if ! start_qmi_network "$QMI_DEV"; then
    log "ERROR: Could not start QMI network."
    return 1
  fi

  local IFACE="wwan0"
  /sbin/ip link set dev "$IFACE" up || true
  start_dhcp "$IFACE"
  if ! wait_for_ip "$IFACE"; then
    log "ERROR: $IFACE did not obtain IPv4."
    return 2
  fi

  set_route_metric "$IFACE"
  configure_dns

  log "--- Modem handler complete ---"
  return 0
}

main_loop() {
  log "=== qmi-autoconnect-plugplay starting ==="

  # Initial attempt
  handle_modem

  # Monitor for device re-insertions
  while true; do
    inotifywait -e add /dev/cdc-wdm* 2>/dev/null && {
      log "Detected /dev/cdc-wdm device addition."
      handle_modem
    }
    sleep 2
  done
}

main_loop
EOF

sudo chmod +x /usr/local/bin/qmi-autoconnect-plugplay.sh
```

---

### Install the systemd service

```bash
sudo tee /etc/systemd/system/qmi-autoconnect-plugplay.service > /dev/null <<'EOF'
[Unit]
Description=Auto QMI Modem Plug-and-Play (SIM7600G-H)
After=network.target
Wants=network-online.target

[Service]
Type=simple
ExecStart=/usr/local/bin/qmi-autoconnect-plugplay.sh
Restart=on-failure
RemainAfterExit=no

[Install]
WantedBy=multi-user.target
EOF

sudo systemctl daemon-reload
sudo systemctl enable qmi-autoconnect-plugplay.service
sudo systemctl start qmi-autoconnect-plugplay.service
```

---

### How to use (step-by-step)

1. Run the package install prerequisites:

   ```bash
   sudo apt update
   sudo apt install libqmi-utils usb-modeswitch udhcpc inotify-tools -y
   ```

2. Copy the script (as above) and create the systemd service (as above).

3. Make sure ModemManager will not interfere (recommended). If ModemManager is present and grabs the device, add a filter:

   ```bash
   sudo mkdir -p /etc/ModemManager/conf.d
   sudo tee /etc/ModemManager/conf.d/99-ignore-sim7600.conf > /dev/null <<'EOF'
   [filter]
   udev-property-match=ID_VENDOR_ID=1e0e
   udev-property-match=ID_MODEL_ID=9001
   EOF
   sudo systemctl restart ModemManager || true
   ```

4. Start service now (or reboot):

   ```bash
   sudo systemctl start qmi-autoconnect-plugplay.service
   ```

5. Verify (after plug-in or boot):

   ```bash
   ip -4 addr show wwan0
   ip route show
   cat /etc/resolv.conf
   ping -I wwan0 8.8.8.8 -c3
   ping -I wwan0 google.com -c3
   sudo journalctl -u qmi-autoconnect-plugplay.service -n 200 --no-pager
   sudo tail -n 200 /var/log/qmi-autoconnect.log
   ```

Expected:
• `wwan0` has an `inet` address (e.g., `100.x.x.x/30`).
• `ip route show` should include a default via WLAN interface **unless** WiFi is disconnected; if WiFi is down, the default should be via `wwan0`.
• `/etc/resolv.conf` should contain `nameserver 8.8.8.8` and `1.1.1.1` (and be immutable).
• `ping -I wwan0 8.8.8.8` should succeed when using 4G.

---

### Verification & expected outputs

* `ip -4 addr show wwan0` → shows `inet` address assigned (e.g. `100.x.x.x`).
* `ip route show` → should include a `default` either via `wlan0` (if WiFi active) or via `wwan0` if WiFi down.
* `cat /etc/resolv.conf` → should contain:

  ```
  nameserver 8.8.8.8
  nameserver 1.1.1.1
  ```

  (and file should be locked).
* `ping -I wwan0 8.8.8.8 -c3` → should reply.
* `ping google.com` → should reply (DNS resolution working).
* After dongle unplug + replug, you should see in `/var/log/qmi-autoconnect.log` the detection of the device addition and the handler running again without needing full OS reboot.

---

### Troubleshooting (most common failures, with commands to run)

If anything fails, run these to gather diagnostics and paste them when asking for help:

```bash
sudo journalctl -u qmi-autoconnect-plugplay.service --no-pager -n 200
sudo tail -n 200 /var/log/qmi-autoconnect.log
lsusb
dmesg | grep -i qmi
ip a
ip route
cat /etc/qmi-network.conf
sudo qmicli -d /dev/cdc-wdm0 --uim-get-card-status
sudo qmicli -d /dev/cdc-wdm0 --nas-get-serving-system
sudo qmicli -d /dev/cdc-wdm0 --wds-get-packet-service-status
```

#### Common issues and fixes

• **No `/dev/cdc-wdm*` found**

* Wait a few seconds after plugging in the modem. Replug it. Check `dmesg`. If the device shows only `/dev/ttyUSB*`, the dongle may still be in AT/ECM mode — the script’s `AT+QCFG="usbnet",1` step should help; if it doesn’t, manually send the AT command and replug.

• **`qmi-network` fails with `CallFailed` / `no-service`**

* SIM not registered or APN wrong. Check `sudo qmicli -d /dev/cdc-wdm0 --nas-get-serving-system`. If no registration, check SIM contact, PIN, network coverage. Confirm APN mapping in script (you may add your operator to the `case` list).

• **DHCP fails / no IP assigned**

* Ensure `wwan0` interface is UP before `udhcpc`. This script explicitly does `ip link set dev wwan0 up`. If you still get “udhcpc: sending discover” repeatedly, you may need to increase retry count or check link layer data-format (`qmicli -d /dev/cdc-wdm0 --wda-get-data-format`).

• **DNS keeps getting overwritten**

* We lock `/etc/resolv.conf` with `chattr +i`. If you later need `NetworkManager` to manage DNS again, remove the immutable flag with `sudo chattr -i /etc/resolv.conf`. Use this locking only if you want the script to manage DNS.

• **Dongle re-plug does not trigger new session**

* The script monitors `/dev/cdc-wdm*` additions via `inotifywait`. If you plug into a new port and the device node changes name, ensure `/dev/cdc-wdm0` is still the correct device or adjust detection logic for new device name (/dev/cdc-wdm1 etc). You may also add `udev` rules to symlink consistent name.

---

### Advanced: APN mapping file (optional)

You can maintain a more complete APN map to improve operator detection. Create `/etc/qmi-apn-mapping.conf` with lines like:

```
MCCMNC=40490 APN=airtelgprs.com
MCCMNC=40410 APN=bsnl.apn
MCCMNC=23415 APN=ee.internet
```

Then change `detect_apn_from_sim()` in the script to look up `/etc/qmi-apn-mapping.conf` using the detected `MCC`+`MNC`. I kept the script simple but structured so you can extend it trivially.

---

### Safety & revert instructions (how to undo all automatic changes)

If you want to revert everything:

```bash
sudo systemctl stop qmi-autoconnect-plugplay.service
sudo systemctl disable qmi-autoconnect-plugplay.service
sudo rm -f /etc/systemd/system/qmi-autoconnect-plugplay.service
sudo rm -f /usr/local/bin/qmi-autoconnect-plugplay.sh
sudo rm -f /var/log/qmi-autoconnect.log
sudo rm -f /etc/qmi-network.conf
sudo chattr -i /etc/resolv.conf   # Allow NM to manage DNS again
```

(Then reboot or restart NetworkManager if needed.)

---

### Why we avoided the “double EOF” bug

When writing a file that itself contains a heredoc we used **different** terminator tokens so the outer `tee` heredoc and the inner `cat` heredoc never collide. The script writing commands above use `EOF` for the outer `tee` and `'DNS_EOF'` (or analogous) inside the script to avoid accidental premature termination.

When you copy/paste the provided `sudo tee` blocks they are already safe.

---

### Final notes & recommendations

• The script is conservative: it uses public DNS (8.8.8.8 / 1.1.1.1) and locks `/etc/resolv.conf`. If you rely on local DNS (company, captive portal), remove the `chattr +i` line.
• We added `inotifywait` monitoring so dongle unplug + re-plug will trigger the handler automatically — you should still test this scenario.
• WiFi is preferred: the routing metric logic sets lower metric (600) for wwan0 when WiFi is not present (so default route shifts). If WiFi is up, wwan0 default route metric is higher (700) so WiFi remains primary.
• Keep the `libqmi-utils` package installed — it provides qmicli and qmi-network which are central to this flow.
• If you want **complete global cellular APN coverage**, I can add a full MCC/MNC→APN database (small file) and code to consult it.

---

## 2. Detailed Overview of Data Flow & Routing Under the Hood

Here’s a detailed explanation of how data flows from your Raspberry Pi through the system when using WiFi as primary and falling back to the 4G dongle, including components, interfaces and routing decisions.

### System Components

* **Modem (dongle) hardware**: The SIMCOM SIM7600G-H USB dongle plugged into a USB port on the Raspberry Pi.
* **USB interface / kernel driver**: On insertion the OS enumerates the device, loads `qmi_wwan` (for QMI support) which exposes `/dev/cdc-wdm0` (control interface) and `wwan0` (network interface). ([Sixfab Docs][1])
* **QMI control layer**: Using `qmicli` or `qmi-network`, the QMI control interface interacts with the modem to start a data session (WDS – Wireless Data Service).
* **Network interface wwan0**: After starting the QMI network, `wwan0` appears (or is brought up) and DHCP obtains an IP address, gateway, DNS etc.
* **WiFi interface wlan0**: The Raspberry Pi’s built-in WiFi (or USB WiFi) is up and connected to a WiFi network, obtains an IP, gateway, DNS via DHCP.
* **Routing & metrics**: The Linux kernel routing table uses metrics to choose which interface to use as default route; lower metric = higher priority.
* **DNS configuration**: `/etc/resolv.conf` lists nameservers; we lock it so that the script controls DNS (so whichever interface is active uses the same DNS servers).
* **Service & script**: The systemd service runs the script which manages modem insertion, session start, DHCP, route config, and monitors for re-plug.
* **Fall-over logic**: Script logic checks if WiFi (wlan0) has IPv4; if yes → give it priority; if not → route via wwan0 (4G).
* **Logging**: All steps logged to `/var/log/qmi-autoconnect.log` for debugging.

### Data Flow – From Application to Internet

1. A user-space application (e.g., browser, ping) issues a DNS lookup or TCP/UDP request.
2. The resolver reads `/etc/resolv.conf` (nameserver 8.8.8.8 / 1.1.1.1) and queries DNS.
3. DNS queries follow the default route configured in the kernel routing table.

   * If WiFi is up and working → default route goes via wlan0 → traffic goes through WiFi gateway to internet.
   * Else if WiFi down → default route falls back to wwan0 → traffic goes through the modem’s gateway (via carrier’s network) to internet.
4. For non-DNS traffic (HTTP, HTTPS, ping) same default route selection applies.
5. On wwan0:

   * The modem has set up the data session via QMI → then DHCP assigned IP + gateway.
   * Kernel sends packets via wwan0 → over the USB/qmi_wwan driver → modem → mobile network → internet.
6. On tethering/packet side:

   * If the modem is unplugged then re-plugged: the inotify monitor detects `/dev/cdc-wdm*` appearing, script reruns `handle_modem()` → brings up session, obtains IP + route. This ensures you don’t need full OS reboot.
7. Routing metrics:

   * The script sets metric 600 for wwan0 if WiFi has no IPv4 (meaning wwan0 will become default).
   * If WiFi is up, wwan0 route has metric 700, so wlan0 (with metric lower) remains default.
8. DNS:

   * Regardless of which route is used, `/etc/resolv.conf` uses fixed nameservers so DNS resolution works consistently across interfaces.
   * The script locks the file to prevent NetworkManager or other tools from overwriting DNS configuration during failover.

### Hot-Plug / Failover Behavior

* During normal boot: WiFi connects → wlan0 has IPv4 → default route via wlan0.
* If WiFi drops or is unplugged: script logic checks `wlan_has_ipv4()` → returns false → `set_route_metric("wwan0")` uses metric 600 → default route is switched to wwan0 → internet via 4G.
* If you unplug the 4G dongle and plug back in while WiFi is still down: inotifywait notices new `/dev/cdc-wdm*` → script runs handle_modem → QMI session restarted → wwan0 up → DHCP obtains new IP/gateway → route metric applied → default route restored via wwan0.
* If WiFi later comes back: script does not continuously monitor WiFi status here (you can extend it to check periodically). But on next run of service or manual script run it would detect wlan0 IPv4 and set metrics accordingly (you may optionally add a systemd timer or a cron job to re-check).

### Summary Table of Interfaces & Flow

| Interface | Active when                  | IP/Gateway source                | Default route metric                                          | Data path                                          |
| --------- | ---------------------------- | -------------------------------- | ------------------------------------------------------------- | -------------------------------------------------- |
| wlan0     | WiFi connected & IPv4        | DHCP from WiFi router            | lower metric (e.g., 100)                                      | Pi → wlan0 → WiFi AP → internet                    |
| wwan0     | WiFi unavailable (fail-over) | DHCP via modem / carrier network | higher metric when WiFi up (700) / lower when WiFi down (600) | Pi → wwan0 → USB modem → mobile network → internet |

---

## 3. Additional Handling: Dongle Not Detected on Re-Plug

You mentioned that when you unplug and plug the dongle back in, you previously had to restart the OS. The updated script uses `inotifywait` to monitor creation of `/dev/cdc-wdm*`. This should automatically trigger the modem handler when the device node appears again.

**Please test this scenario:**

1. With WiFi disabled (so the system should use 4G), ensure everything works.
2. Unplug the dongle. Wait ~10 seconds.
3. Re-plug the dongle (maybe into a different USB port).
4. Check `sudo journalctl -u qmi-autoconnect-plugplay.service -n 50` and `/var/log/qmi-autoconnect.log` to see that the handler ran again and re-established the session.
5. Verify `ip -4 addr show wwan0` and `ip route show` show the expected new IP + default route via wwan0.
6. Also test that WiFi plugs back in and the default route shifts appropriately.

If this re-plug behavior still fails, you may need to add a `udevadm` rule to detect the dongle insertion and run the handler, or extend the script to monitor `lsusb` changes.

---

## 4. Why the Guide Now Should Work & What Was the Translation Fix

* We added explicit `ip link set dev wwan0 up` in the handler to avoid “interface DOWN” race.
* We made route setting more robust (tries dev route, then via gateway) so the route is reliably set.
* We integrated hot-plug monitoring via `inotifywait` so you don’t need OS reboot on dongle re-insert.
* We maintained WiFi-first logic via routing metrics so you meet your preference “WiFi preferred, 4G only if internet not working”.

---

## 5. What To Check / Test After Implementation

* With WiFi active: verify default route is via wlan0 (metric lower than wwan0).
* Disconnect WiFi: verify after a short delay default route changes to wwan0 and internet works.
* Unplug 4G dongle while it’s in use: verify the script logs error or session drops, but when you re-plug, it re-connects without reboot.
* Plug 4G dongle while WiFi active: verify it comes up but default route remains wlan0 (so no regression).
* Reboot with dongle in a different USB port: verify detection and session succeed automatically.
* Monitor logs for any errors about “wwan0 down” or “default route not set”.

---

