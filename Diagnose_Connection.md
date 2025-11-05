
---

### Create the diagnostic script and run it

Nice — here’s a single, **copy-pasteable diagnostic script** that runs a full set of checks for the whole stack (USB → QMI → wwan0 → DHCP → routing → DNS → application connectivity). It prints clear human-readable diagnostics and short “what to try next” hints for each failing check. Save it and run with `sudo`.

Copy/paste the whole block below into your Pi (it writes `/usr/local/bin/qmi-diagnose.sh` and makes it executable):

```bash
sudo tee /usr/local/bin/qmi-diagnose.sh > /dev/null <<'EOF'
#!/usr/bin/env bash
# qmi-diagnose.sh - Comprehensive network + QMI modem diagnostic for Raspberry Pi
# Usage: sudo /usr/local/bin/qmi-diagnose.sh

set -euo pipefail
DATE="$(date '+%F %T')"
LOGFILE="/var/log/qmi-diagnose.log"

# simple color helpers (if terminal supports)
RED='\033[0;31m'; GREEN='\033[0;32m'; YELLOW='\033[1;33m'; BLUE='\033[0;34m'; NC='\033[0m'

echo -e "${BLUE}=== QMI / Network Diagnostic ===${NC}  ($DATE)"
echo "Log: $LOGFILE"
echo

# Function for header
h() { echo -e "${BLUE}== $* ==${NC}"; }
ok() { echo -e "${GREEN}[ OK ]${NC} $*"; }
warn() { echo -e "${YELLOW}[WARN]${NC} $*"; }
err() { echo -e "${RED}[ERR ]${NC} $*"; }

echo "Writing full output to $LOGFILE"
: > "$LOGFILE"

run_and_log() {
  echo -e "\$ $*" | tee -a "$LOGFILE"
  if ! bash -c "$*" 2>&1 | tee -a "$LOGFILE"; then
    echo >> "$LOGFILE"
  fi
  echo >> "$LOGFILE"
}

# Ensure script is run as root (some checks require root)
if [ "$EUID" -ne 0 ]; then
  err "This script should be run with sudo/root. Re-run: sudo $0"
  exit 2
fi

h "Environment & dependencies"
echo "Checking required commands..."
CMDS=(lsusb dmesg ip qmicli qmi-network udhcpc pgrep ping inotifywait awk sed)
for c in "${CMDS[@]}"; do
  if command -v "$c" >/dev/null 2>&1; then
    ok "$c found: $(command -v $c)"
  else
    warn "$c not found"
  fi
done
echo
echo "Hint: install missing packages: sudo apt update && sudo apt install libqmi-utils usb-modeswitch udhcpc inotify-tools -y"
echo >> "$LOGFILE"

h "USB enumeration (physical layer)"
run_and_log "lsusb"
run_and_log "dmesg | tail -n 80 | sed -n '1,200p' | egrep -i 'qmi|cdc|ttyUSB|wwan|usb|simcom|1e0e' || true"

h "/dev nodes"
run_and_log "ls -l /dev/cdc-wdm* 2>/dev/null || true"
run_and_log "ls -l /dev/ttyUSB* 2>/dev/null || true"

h "Kernel interfaces and link state"
run_and_log "ip link show | sed -n '1,200p'"
run_and_log "ip -4 addr show | sed -n '1,200p'"

# check for wwan0 and wlan0
WWAN_PRESENT=0
if ip link show wwan0 >/dev/null 2>&1; then
  WWAN_PRESENT=1
  ok "Interface wwan0 present"
else
  warn "Interface wwan0 NOT present"
fi

if ip link show wlan0 >/dev/null 2>&1; then
  ok "Interface wlan0 present"
else
  warn "Interface wlan0 NOT present"
fi
echo

h "QMI / modem status (if /dev/cdc-wdm* exists and qmicli installed)"
if ls /dev/cdc-wdm* 1>/dev/null 2>&1 && command -v qmicli >/dev/null 2>&1; then
  run_and_log "for d in /dev/cdc-wdm*; do echo '--- $d ---'; sudo qmicli -d \$d --device-open-proxy --get-ids || true; echo; done"
  run_and_log "for d in /dev/cdc-wdm*; do echo '--- $d uim-get-card-status ---'; sudo qmicli -d \$d --uim-get-card-status || true; echo; done"
  run_and_log "for d in /dev/cdc-wdm*; do echo '--- $d nas-get-serving-system ---'; sudo qmicli -d \$d --nas-get-serving-system || true; echo; done"
  run_and_log "for d in /dev/cdc-wdm*; do echo '--- $d nas-get-signal-strength ---'; sudo qmicli -d \$d --nas-get-signal-strength || true; echo; done"
  run_and_log "for d in /dev/cdc-wdm*; do echo '--- $d wds-get-packet-service-status ---'; sudo qmicli -d \$d --wds-get-packet-service-status || true; echo; done"
else
  warn "No /dev/cdc-wdm* or qmicli not installed — QMI checks skipped"
fi

h "DHCP, IP address and route table"
run_and_log "ip -4 addr show wwan0 || true"
run_and_log "ip -4 addr show wlan0 || true"
run_and_log "ip route show || true"
run_and_log "ip route get 8.8.8.8 || true"

h "DNS resolver"
run_and_log "cat /etc/resolv.conf || true"

h "DHCP client / lease / udhcpc status"
if pgrep -a udhcpc >/dev/null 2>&1; then
  ok "udhcpc is running:"
  run_and_log "pgrep -a udhcpc || true"
else
  warn "udhcpc not running (may still have assigned address if used previously)"
fi

h "Service logs (qmi-autoconnect-plugplay.service if present)"
if systemctl status qmi-autoconnect-plugplay.service >/dev/null 2>&1; then
  run_and_log "sudo journalctl -u qmi-autoconnect-plugplay.service -n 200 --no-pager || true"
else
  warn "qmi-autoconnect-plugplay.service not installed or not running"
fi

h "Live connectivity tests (non-invasive)"
echo "Note: these try to use the current default route. They do NOT change routes."
run_and_log "ping -c 3 -w 6 8.8.8.8 || true"
run_and_log "ping -c 3 -w 6 google.com || true"

h "Optional interface-specific quick checks"
if ip link show wwan0 >/dev/null 2>&1; then
  echo "wwan0 quick test (show link state, try to ping from wwan0 if route exists)"
  run_and_log "ip link show wwan0 || true"
  run_and_log "ip -4 addr show wwan0 || true"
  # try route-get via wwan0
  run_and_log "ip route get 8.8.8.8 from $(ip -4 addr show wwan0 2>/dev/null | awk '/inet /{print \$2}' | cut -d/ -f1 || true) || true"
fi

h "Quick suggested fixes (based on checks)"
# heuristic suggestions appended to logfile and printed
echo >> "$LOGFILE"
{
  echo "=== Suggestions (automatic check results) ==="
  # suggest based on presence
  if ! lsusb | egrep -i '1e0e|simcom' >/dev/null 2>&1; then
    echo "• USB modem not visible via lsusb. Try: unplug/replug, try different USB port, verify power."
  fi
  if [ "$WWAN_PRESENT" -eq 0 ]; then
    echo "• wwan0 not present. If you see /dev/cdc-wdm*, try: sudo modprobe qmi_wwan; ensure usb_modeswitch has switched the device; run sudo qmi-network /dev/cdc-wdm0 start"
  fi
  # route/dns heuristics
  if ! ip route show | egrep -q '(default' ; then
    echo "• No default route found. If you have an interface with IP, add default route manually: sudo ip route replace default via <GW> dev <iface>"
  else
    echo "• Default route present. Use 'ip route get 8.8.8.8' to see which iface is used."
  fi
  echo "• If DNS fails but raw ping to 8.8.8.8 works: check /etc/resolv.conf and nameservers."
  echo "• If modem works only after reboot: confirm udev/systemd rules or use the qmi-autoconnect service and check its journal."
} | tee -a "$LOGFILE"

echo
ok "Diagnostic complete. Full verbatim output saved to $LOGFILE"
echo "Read the logfile, and pick the first failing layer from top to bottom in this output to start troubleshooting."
echo -e "${BLUE}Quick next steps (common):${NC}"
echo " - If wwan0 exists but state is DOWN: sudo ip link set wwan0 up"
echo " - If udhcpc hasn't given IP: sudo udhcpc -i wwan0 -n -q"
echo " - If no default route: sudo ip route replace default via <GW> dev wwan0"
echo " - If qmicli errors: check SIM, APN and signal: sudo qmicli -d /dev/cdc-wdm0 --nas-get-serving-system"
echo
EOF

sudo chmod +x /usr/local/bin/qmi-diagnose.sh

echo "Script written to /usr/local/bin/qmi-diagnose.sh. Run it with:"
echo "  sudo /usr/local/bin/qmi-diagnose.sh"
```

---

### How the script helps (what it checks)

The script runs checks in this order (matching the theory we discussed), and saves a full copy of all outputs to `/var/log/qmi-diagnose.log` for sharing or future reference:

1. Dependency check (qmicli, qmi-network, udhcpc, inotifywait, etc.)
2. USB detection (`lsusb`, `dmesg` filtering)
3. Device nodes `/dev/cdc-wdm*`, `/dev/ttyUSB*`
4. Kernel interfaces and addresses (`ip link`, `ip addr`)
5. QMI modem health and status (`qmicli` checks) — only if `/dev/cdc-wdm*` exists and `qmicli` installed
6. DHCP / udhcpc presence and IP assignment
7. Routing table and “which interface will be used” using `ip route get`
8. DNS contents of `/etc/resolv.conf`
9. Live pings to 8.8.8.8 and google.com (non-invasive)
10. A short set of human-readable suggestions appended to the logfile

---

### Interpreting the output (quick guide)

After the run, open `/var/log/qmi-diagnose.log` or read console output.

* If `lsusb` shows nothing for the modem → hardware/USB layer issue → try replug/port/power.
* If `/dev/cdc-wdm*` missing but `/dev/ttyUSB*` present → device likely in AT/ECM mode (run the AT QCFG, replug, or use usb_modeswitch).
* If `qmicli --wds-get-packet-service-status` shows `disconnected` or errors → SIM/APN/signal problem. Check `qmicli --nas-get-serving-system`.
* If `ip -4 addr show wwan0` shows no `inet` → DHCP failure: run `sudo udhcpc -i wwan0 -n -q`.
* If `ip route show` contains a default via `wlan0` but you expect 4G → metrics are favoring WiFi (by design). If no default route via wwan0 and WiFi down → set route manually.
* If `ping 8.8.8.8` works but `ping google.com` fails → DNS issue; check `/etc/resolv.conf`.

---
