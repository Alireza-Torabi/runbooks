# Linux CLI Wi-Fi Connection Runbook

## 1. Identify the Wi-Fi Interface

List all network interfaces:

```bash
ip link
```

Or:

```bash
iw dev
```

Example:

```text
Interface wlx00113b2694a9
    type managed
```

Set the interface name for later commands:

```bash
WIFI_IF="wlx00113b2694a9"
```

> USB Wi-Fi adapters commonly use names such as `wlx<MAC>`. Internal adapters may use names such as `wlan0` or `wlp...`.

---

## 2. Check the Wi-Fi Driver

```bash
sudo ethtool -i "$WIFI_IF"
```

Example:

```text
driver: rt2800usb
firmware-version: ...
```

If `ethtool` is not installed:

```bash
sudo apt update
sudo apt install -y ethtool
```

---

## 3. Check NetworkManager

Check whether NetworkManager is running:

```bash
systemctl status NetworkManager
```

If it is not installed:

```bash
sudo apt update
sudo apt install -y network-manager
```

Enable and start it:

```bash
sudo systemctl enable --now NetworkManager
```

Verify:

```bash
nmcli general status
```

Expected example:

```text
STATE      CONNECTIVITY
connected  full
```

---

## 4. Enable Wi-Fi

```bash
nmcli radio wifi on
```

Check the device:

```bash
nmcli device status
```

Example:

```text
DEVICE            TYPE      STATE          CONNECTION
wlx00113b2694a9   wifi      disconnected   --
eth0              ethernet  connected      --
```

---

## 5. Scan for Available Wi-Fi Networks

Request a new scan:

```bash
nmcli device wifi rescan
```

List available networks:

```bash
nmcli device wifi list
```

Or:

```bash
nmcli dev wifi
```

Example:

```text
IN-USE  BSSID              SSID          MODE   CHAN  RATE        SIGNAL  SECURITY
        AA:BB:CC:DD:EE:FF  MyWiFi        Infra  6     270 Mbit/s  95      WPA2
        11:22:33:44:55:66  OfficeWiFi    Infra  11    260 Mbit/s  72      WPA2
```

---

## 6. Connect to a Wi-Fi Network

Use:

```bash
sudo nmcli device wifi connect "SSID" password 'PASSWORD'
```

Example:

```bash
sudo nmcli device wifi connect "OfficeWiFi" password 'MyPassword123'
```

A successful connection should return something similar to:

```text
Device 'wlx00113b2694a9' successfully activated ...
```

---

## 7. Connect Using a Specific Wi-Fi Interface

If the system has multiple Wi-Fi adapters:

```bash
sudo nmcli device wifi connect "OfficeWiFi" \
    password 'MyPassword123' \
    ifname "$WIFI_IF"
```

---

## 8. Connect to a Hidden SSID

Create the connection:

```bash
sudo nmcli connection add \
    type wifi \
    ifname "$WIFI_IF" \
    con-name "hidden-wifi" \
    ssid "MyHiddenWiFi"
```

Configure WPA-PSK:

```bash
sudo nmcli connection modify "hidden-wifi" \
    wifi-sec.key-mgmt wpa-psk \
    wifi-sec.psk 'MyPassword123'
```

Activate it:

```bash
sudo nmcli connection up "hidden-wifi"
```

---

## 9. Verify the Connection

Check device status:

```bash
nmcli device status
```

Check active connections:

```bash
nmcli connection show --active
```

Check the IP address:

```bash
ip addr show "$WIFI_IF"
```

Or:

```bash
nmcli device show "$WIFI_IF"
```

Example:

```text
IP4.ADDRESS[1]: 10.61.61.175/24
IP4.GATEWAY:     10.61.61.236
```

---

## 10. Check the Routing Table

```bash
ip route
```

Example:

```text
default via 10.61.61.236 dev wlx00113b2694a9
10.61.61.0/24 dev wlx00113b2694a9 proto kernel scope link src 10.61.61.175
```

> **Important:** On servers with Ethernet, Wi-Fi, VPN, and proxy interfaces, verify the routing table carefully. Connecting to Wi-Fi can change the default route and potentially interrupt remote access.

---

## 11. Test Network Connectivity

### 11.1 Test the Wi-Fi Gateway

```bash
ping -c 4 10.61.61.236
```

If the gateway is unknown:

```bash
ip route | grep default
```

---

### 11.2 Test Internet Without DNS

```bash
ping -c 4 1.1.1.1
```

---

### 11.3 Test DNS

```bash
ping -c 4 google.com
```

---

### 11.4 Test HTTPS

```bash
curl -I https://www.google.com
```

These tests help isolate the problem:

```text
Wi-Fi → Gateway → Internet → DNS → HTTPS
```

---

## 12. Check Wi-Fi Signal and Link Speed

The most useful command is:

```bash
iw dev "$WIFI_IF" link
```

Example:

```text
Connected to AA:BB:CC:DD:EE:FF
SSID: OfficeWiFi
freq: 2437
signal: -45 dBm
tx bitrate: 144.4 MBit/s
```

Approximate signal levels:

| Signal | Quality |
|---|---|
| -30 to -50 dBm | Excellent |
| -50 to -60 dBm | Good |
| -60 to -67 dBm | Acceptable |
| -67 to -75 dBm | Weak |
| Below -75 dBm | Very weak |

---

## 13. Check IP and DNS Information

```bash
nmcli device show "$WIFI_IF" | grep -E 'IP4|DNS'
```

Or:

```bash
resolvectl status
```

---

## 14. Create a Named Wi-Fi Connection

Create the connection with a specific name:

```bash
sudo nmcli device wifi connect "MyWiFi" \
    password 'MyPassword123' \
    ifname "$WIFI_IF" \
    name "wifi-office"
```

List saved connections:

```bash
nmcli connection show
```

Example:

```text
NAME          UUID                                  TYPE  DEVICE
wifi-office   xxxxxxxx-xxxx-xxxx-xxxx-xxxxxxxxxxxx  wifi  wlx00113b2694a9
```

---

## 15. Enable Automatic Reconnection

Enable autoconnect:

```bash
sudo nmcli connection modify "wifi-office" connection.autoconnect yes
```

Verify:

```bash
nmcli connection show "wifi-office" | grep autoconnect
```

This allows NetworkManager to reconnect the Wi-Fi automatically after reboot or temporary disconnection.

---

## 16. Disconnect and Reconnect

Disconnect the connection:

```bash
sudo nmcli connection down "wifi-office"
```

Reconnect:

```bash
sudo nmcli connection up "wifi-office"
```

Alternatively, disconnect the device:

```bash
sudo nmcli device disconnect "$WIFI_IF"
```

Reconnect it:

```bash
sudo nmcli device connect "$WIFI_IF"
```

---

## 17. Delete a Saved Wi-Fi Connection

List connections:

```bash
nmcli connection show
```

Delete the unwanted connection:

```bash
sudo nmcli connection delete "wifi-office"
```

---

# Troubleshooting

## 18. Wi-Fi Adapter Is Blocked

Check rfkill:

```bash
rfkill list
```

If you see:

```text
Soft blocked: yes
```

Unblock Wi-Fi:

```bash
sudo rfkill unblock wifi
```

Or:

```bash
sudo rfkill unblock all
```

Then:

```bash
nmcli radio wifi on
nmcli device wifi rescan
nmcli device wifi
```

---

## 19. Wi-Fi Interface Is Missing

Check interfaces:

```bash
ip link
```

Check kernel messages:

```bash
dmesg | grep -iE 'wifi|wlan|firmware|802.11'
```

For USB adapters:

```bash
lsusb
```

For PCI/PCIe adapters:

```bash
lspci -nnk | grep -A3 -i network
```

---

## 20. Check the Driver

```bash
sudo ethtool -i "$WIFI_IF"
```

Check loaded wireless modules:

```bash
lsmod | grep -E 'wifi|80211|rt2|rtl|ath|iwl'
```

---

## 21. Wi-Fi Connects but There Is No Internet

Check the IP address:

```bash
ip addr
```

Check routes:

```bash
ip route
```

Check DNS:

```bash
resolvectl status
```

Then run the tests in this order:

```bash
ping -c 4 <GATEWAY>
ping -c 4 1.1.1.1
ping -c 4 google.com
```

Interpretation:

| Test | Result | Likely Problem |
|---|---|---|
| Gateway | Fail | Wi-Fi/LAN problem |
| Gateway | OK | |
| 1.1.1.1 | Fail | Routing/Internet problem |
| 1.1.1.1 | OK | |
| google.com | Fail | DNS problem |
| google.com | OK | Internet/DNS working |

---

# Quick Reference

## Detect Wi-Fi

```bash
iw dev
```

## Check NetworkManager

```bash
systemctl is-active NetworkManager
```

## Enable Wi-Fi

```bash
nmcli radio wifi on
```

## Scan

```bash
nmcli dev wifi rescan
nmcli dev wifi
```

## Connect

```bash
sudo nmcli dev wifi connect "SSID" password 'PASSWORD'
```

## Check Status

```bash
nmcli dev status
```

## Check IP

```bash
ip addr
```

## Check Route

```bash
ip route
```

## Check Wi-Fi Link

```bash
iw dev <WIFI_INTERFACE> link
```

## Test Internet

```bash
ping -c 4 1.1.1.1
```

## Test DNS

```bash
ping -c 4 google.com
```

## Test HTTPS

```bash
curl -I https://www.google.com
```

---

# Production/Server Checklist

For a Linux server, use this sequence:

```bash
# 1. Identify the Wi-Fi interface
iw dev

# 2. Check the driver
sudo ethtool -i <WIFI_INTERFACE>

# 3. Check NetworkManager
systemctl is-active NetworkManager

# 4. Enable Wi-Fi
nmcli radio wifi on

# 5. Scan
nmcli dev wifi rescan
nmcli dev wifi

# 6. Connect
sudo nmcli dev wifi connect "SSID" password 'PASSWORD'

# 7. Verify
nmcli dev status
ip addr
ip route

# 8. Test gateway
ping -c 4 <GATEWAY>

# 9. Test Internet
ping -c 4 1.1.1.1

# 10. Test DNS
ping -c 4 google.com

# 11. Check Wi-Fi quality
iw dev <WIFI_INTERFACE> link

# 12. Enable persistent connection
nmcli connection modify "<CONNECTION>" connection.autoconnect yes
```

## Important Server Consideration

On servers with multiple network paths, always verify:

```bash
ip route
```

before and after connecting to Wi-Fi.

A Wi-Fi connection can become the system's **default route**, potentially changing the path used by:

- SSH
- VPN
- Proxy services
- Internet-bound traffic
- Management interfaces
- Application traffic

For advanced setups, use **route metrics, policy routing, or dedicated routing tables** instead of relying only on the default route.