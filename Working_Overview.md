
---

# 🌐 The Complete Flow of Internet Connectivity (Raspberry Pi + SIM7600G-H)

**Theory + Practical** of how your Raspberry Pi (with Wi-Fi + SIM7600G-H 4G dongle) gets internet access.
You’ll learn **what happens under the hood** — step-by-step from physical hardware to IP packets — and which **commands** reveal each layer so you can troubleshoot confidently.

We’ll follow your data as it moves through **7 layers**:

1. **Physical + Link (USB, drivers, interfaces)**
2. **QMI control (modem setup)**
3. **Network interface activation (wwan0)**
4. **IP addressing (DHCP or static)**
5. **Routing (kernel forwarding table)**
6. **DNS resolution**
7. **User-space networking (apps, sockets, connections)**

---

## 1️⃣ Physical + Link Layer — *“the connection exists”*

### 🧠 Concept

When you plug in the SIM7600G-H USB dongle, the **USB subsystem** detects a new device.
The kernel loads the **usbserial** and **qmi_wwan** drivers.
They expose:

* `/dev/ttyUSB0..3` → serial AT ports
* `/dev/cdc-wdm0` → QMI control interface
* `wwan0` → network interface (like eth0 or wlan0)

### ⚙️ What to check

```bash
lsusb                     # list USB devices
dmesg | grep -i usb       # show kernel messages when plugged in
ls /dev/ttyUSB* /dev/cdc-wdm* 2>/dev/null
ip link show              # show all interfaces
```

**Expected:**
`wwan0` appears in `ip link show`; `/dev/cdc-wdm0` exists.

If not, the dongle isn’t in QMI mode — use:

```bash
echo 'AT+QCFG="usbnet",1' > /dev/ttyUSB2
```

and replug.

---

## 2️⃣ QMI Control Layer — *“the modem starts a data session”*

### 🧠 Concept

The **QMI protocol** (Qualcomm MSM Interface) lets Linux control the modem using `/dev/cdc-wdm0`.

The command-line tools `qmicli` or `qmi-network` send QMI messages to:

* register the SIM on a cellular network,
* activate a PDP (Packet Data Protocol) context (APN),
* start a **WDS (Wireless Data Service)** network session.

When this session starts, the modem allocates an IP to the host (via DHCP later).

### ⚙️ What to check

```bash
sudo qmicli -d /dev/cdc-wdm0 --device-open-proxy --get-ids
sudo qmicli -d /dev/cdc-wdm0 --uim-get-card-status
sudo qmicli -d /dev/cdc-wdm0 --nas-get-signal-strength
sudo qmicli -d /dev/cdc-wdm0 --nas-get-serving-system
sudo qmicli -d /dev/cdc-wdm0 --wds-get-packet-service-status
```

**Expected:**
`wds-get-packet-service-status` shows `Connected`.

If not:

* SIM may not be registered.
* APN may be wrong.
* Weak signal.

---

## 3️⃣ Network Interface Activation — *“Linux brings up wwan0”*

### 🧠 Concept

Once the modem starts a data session, the `wwan0` interface becomes active.
It behaves like a point-to-point Ethernet device.
Linux assigns it a link-layer type (often **raw-IP**) and you must bring it up:

```bash
sudo ip link set dev wwan0 up
```

### ⚙️ What to check

```bash
ip link show wwan0
```

You should see:
`state UP` and `LOWER_UP` once connected.

If it’s DOWN, no data can flow.

---

## 4️⃣ IP Addressing — *“who am I on this network?”*

### 🧠 Concept

Now Linux uses **DHCP** to get:

* your **IPv4 address**,
* the **gateway** (router inside carrier’s network),
* **DNS servers**.

On your system, `udhcpc` (BusyBox DHCP client) performs this step.

### ⚙️ What to check

```bash
sudo udhcpc -i wwan0 -n -q
ip -4 addr show wwan0
```

**Expected output example:**

```
3: wwan0: <POINTOPOINT,MULTICAST,NOARP,UP> mtu 1500
    inet 100.67.169.178/30 brd 100.67.169.179 scope global wwan0
```

Here:

* `100.67.169.178` → your assigned IP
* `/30` → small subnet for the modem connection
* Gateway (in your log): `100.67.169.177`

If DHCP fails, no IP = no route = no internet.

---

## 5️⃣ Routing — *“how packets know where to go”*

### 🧠 Concept

Linux maintains a **routing table** — rules for how to reach networks.

The command `ip route show` displays it.

Typical output:

```
default via 10.41.108.1 dev wlan0 proto dhcp metric 100
default via 100.67.169.177 dev wwan0 metric 700
10.41.108.0/24 dev wlan0 proto kernel scope link src 10.41.108.30 metric 100
100.67.169.176/30 dev wwan0 proto kernel scope link src 100.67.169.178 metric 700
```

The kernel uses the **lowest metric** route as default.

* Wi-Fi metric = 100 (preferred)
* 4G metric = 700 (fallback)

If Wi-Fi disconnects, the default route via wlan0 disappears → kernel automatically uses wwan0.

### ⚙️ What to check

```bash
ip route show
ip route get 8.8.8.8
```

If `ip route get 8.8.8.8` shows:

```
8.8.8.8 via 100.67.169.177 dev wwan0
```

→ 4G is in use.

If it shows `via 10.41.108.1 dev wlan0`
→ Wi-Fi is being used.

---

## 6️⃣ DNS Resolution — *“how names become IPs”*

### 🧠 Concept

When you type `ping google.com`, Linux first asks the DNS resolver in `/etc/resolv.conf` to translate `google.com` → `142.250.x.x`.

Your script writes:

```
nameserver 8.8.8.8
nameserver 1.1.1.1
```

and locks the file with `chattr +i` so NetworkManager doesn’t overwrite it.

### ⚙️ What to check

```bash
cat /etc/resolv.conf
dig google.com
nslookup google.com
```

If these fail but `ping 8.8.8.8` works → DNS problem.

Fix by unlocking and rewriting:

```bash
sudo chattr -i /etc/resolv.conf
echo -e "nameserver 8.8.8.8\nnameserver 1.1.1.1" | sudo tee /etc/resolv.conf
sudo chattr +i /etc/resolv.conf
```

---

## 7️⃣ User-Space Networking — *“apps finally send data”*

### 🧠 Concept

Applications (browser, apt, etc.) use sockets.
The kernel decides which interface to send data through (based on routing table).
Once the packet exits the correct interface, it’s encapsulated, modulated, and sent over Wi-Fi or cellular.

At the far end (ISP or carrier), packets are routed to the Internet backbone and to your destination server.

---

# 🔍 Troubleshooting by Layer

| Layer                | Problem Symptom                                  | Commands to Check                        | Likely Fix                                                            |                                                                       |
| -------------------- | ------------------------------------------------ | ---------------------------------------- | --------------------------------------------------------------------- | --------------------------------------------------------------------- |
| **1. Hardware/Link** | Dongle not visible                               | `lsusb`, `dmesg                          | grep usb`, `ip link`                                                  | Replug, check power, ensure drivers loaded (`sudo modprobe qmi_wwan`) |
| **2. QMI Session**   | `/dev/cdc-wdm0` exists but no session            | `qmicli --wds-get-packet-service-status` | Run `qmi-network start`, check SIM & signal                           |                                                                       |
| **3. Interface**     | `state DOWN`                                     | `ip link show wwan0`                     | `sudo ip link set wwan0 up`                                           |                                                                       |
| **4. IP Address**    | `no inet` on wwan0                               | `ip -4 addr show wwan0`                  | `sudo udhcpc -i wwan0`                                                |                                                                       |
| **5. Routing**       | “Network is unreachable”                         | `ip route show`, `ip route get 8.8.8.8`  | Add default route: `sudo ip route replace default via <GW> dev wwan0` |                                                                       |
| **6. DNS**           | `ping 8.8.8.8` works but `ping google.com` fails | `cat /etc/resolv.conf`                   | Fix nameserver entries                                                |                                                                       |
| **7. App**           | Only one app fails                               | Depends on app                           | Check proxy/firewall settings                                         |                                                                       |

---

# 📈 End-to-End Example Command Flow

Assume Wi-Fi disconnected and 4G dongle plugged in:

```bash
# 1. Check device presence
lsusb | grep -i simcom

# 2. Confirm /dev/cdc-wdm0 exists
ls /dev/cdc-wdm*

# 3. Start QMI data session manually (optional test)
sudo qmi-network /dev/cdc-wdm0 start

# 4. Bring interface up
sudo ip link set wwan0 up

# 5. Get IP
sudo udhcpc -i wwan0 -n -q
ip -4 addr show wwan0

# 6. Set route
sudo ip route replace default via 100.67.169.177 dev wwan0

# 7. Verify connectivity
ping -I wwan0 8.8.8.8 -c3
ping google.com -c3

# 8. Check route and DNS
ip route get 8.8.8.8
cat /etc/resolv.conf
```

If all succeed, you’re fully online through 4G.

---

# 🧩 Bonus: How Linux Chooses Between Wi-Fi and 4G

Linux uses the **routing metric** to prioritize interfaces.

Example:

```
default via 10.41.108.1 dev wlan0 metric 100
default via 100.67.169.177 dev wwan0 metric 700
```

* Lower metric (100) → higher priority → **Wi-Fi used first**
* If Wi-Fi disconnects → route disappears → 4G route (700) becomes active.

Check active route anytime:

```bash
ip route get 1.1.1.1
```

---

# 🧠 Concept Recap

| Layer     | Component               | Responsible Process/Command | What Happens                     |
| --------- | ----------------------- | --------------------------- | -------------------------------- |
| Physical  | USB subsystem           | `lsusb`, `dmesg`            | Dongle enumerated                |
| Driver    | `qmi_wwan`              | kernel module               | Creates `/dev/cdc-wdm0`, `wwan0` |
| QMI       | `qmicli`, `qmi-network` | QMI messages                | Modem registers on network       |
| IP config | `udhcpc`                | DHCP client                 | Gets IP/gateway/DNS              |
| Routing   | kernel                  | `ip route`                  | Selects outbound interface       |
| DNS       | resolver                | `/etc/resolv.conf`          | Maps hostnames to IPs            |
| App       | user-space              | curl, ping                  | Sends packets using socket API   |

---

# 🛠️ Essential Troubleshooting Toolkit

| Purpose                   | Command                                                        | Notes                         |
| ------------------------- | -------------------------------------------------------------- | ----------------------------- |
| List network interfaces   | `ip link`                                                      | Check UP/DOWN state           |
| Show IP addresses         | `ip addr`                                                      | For wlan0/wwan0               |
| Show routes               | `ip route show`                                                | Verify default route          |
| Check which route is used | `ip route get 8.8.8.8`                                         | Shows interface used          |
| DNS lookup                | `dig google.com` or `nslookup`                                 | Checks resolver               |
| Check cellular signal     | `sudo qmicli -d /dev/cdc-wdm0 --nas-get-signal-strength`       | dBm values                    |
| Check registration status | `sudo qmicli -d /dev/cdc-wdm0 --nas-get-serving-system`        | Shows network type            |
| Check data session status | `sudo qmicli -d /dev/cdc-wdm0 --wds-get-packet-service-status` | Connected?                    |
| Check logs                | `sudo journalctl -u qmi-autoconnect-plugplay.service -n 100`   | Service activity              |
| Monitor connectivity live | `watch -n2 "ping -c1 8.8.8.8"`                                 | Continuous reachability check |

---

✅ **In short:**

1. Hardware → `/dev/cdc-wdm0` & `wwan0` appear.
2. QMI control → session started (`qmi-network`).
3. DHCP → gets IP, gateway, DNS.
4. Routing → default via Wi-Fi or 4G.
5. DNS → names resolved via `/etc/resolv.conf`.
6. App → sends data through correct interface.

When you understand these layers, you can isolate *any* “no internet” issue to a single stage and fix it with a single command.

---

