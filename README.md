# RaspberryPi USB to Ethernet / Wi-Fi Bridge

Turn a Raspberry Pi into a network bridge that shares a USB internet device (JioFi3, Airtel MyWifi, or any USB tethering-capable device) over Ethernet or Wi-Fi.

## Use Case

If you have a mobile broadband device (USB dongle or hotspot) and want to plug it into a load-balancing router or share it over a wired/wireless network, this guide walks you through configuring a Raspberry Pi as the bridge. The Pi takes the USB internet connection and redistributes it over its Ethernet port or as a Wi-Fi access point.

**Example scenario:** Using a TP-Link TL-R470T+ load-balance router that requires WAN connections over Ethernet — the Pi bridges a JioFi3 (USB) to an Ethernet WAN port for failover/load-balancing.

---

## Hardware Requirements

| Component | Notes |
|---|---|
| Raspberry Pi | Any model with at least one USB port and one Ethernet port |
| MicroSD Card | 8 GB or larger |
| Power Adapter | 5V 2A recommended |
| USB Internet Device | JioFi3, Airtel MyWifi, or any device supporting USB tethering |
| Micro USB Cable | Compatible with your USB internet device |
| Ethernet Cable | For the USB → Ethernet guide |

## Software Requirements

- Raspberry Pi OS — Bullseye or Bookworm (uses `dhcpcd`; NetworkManager-based installs require different steps)
- SSH or VNC access to the Pi
- Pi connected to the internet (for installing packages)

---

## Overview of Guides

| Guide | What it does |
|---|---|
| [USB → Ethernet](Steps/USBtoETH.md) | Shares the USB internet connection over the Pi's Ethernet port |
| [USB → Wi-Fi](Steps/USBtoWIFI.md) | Shares the USB internet connection as a Wi-Fi access point (hotspot) |

---

## Guide 1: USB Internet → Ethernet

> All commands run on the Raspberry Pi terminal or over SSH.

### Step 1 — Identify interface names

```bash
ifconfig
```

Note your USB interface name (typically `usb0`) and Ethernet interface name (typically `eth0`).

### Step 2 — Install dnsmasq

```bash
sudo apt install dnsmasq
sudo systemctl stop dnsmasq
```

### Step 3 — Set a static IP for eth0

Edit `/etc/dhcpcd.conf`:

```bash
sudo nano /etc/dhcpcd.conf
```

Append to the end of the file:

```
interface eth0
    static ip_address=192.168.10.1/24
```

### Step 4 — Configure dnsmasq

Back up the existing config and create a new one:

```bash
sudo mv /etc/dnsmasq.conf /etc/dnsmasq.conf.bkp
sudo nano /etc/dnsmasq.conf
```

Add:

```
interface=eth0
dhcp-range=192.168.10.150,192.168.10.170,255.255.255.0,24h
```

Start dnsmasq:

```bash
sudo systemctl start dnsmasq
```

### Step 5 — Enable IP forwarding

Edit `/etc/sysctl.conf`:

```bash
sudo nano /etc/sysctl.conf
```

Uncomment this line:

```
net.ipv4.ip_forward=1
```

Apply immediately:

```bash
sudo sysctl -w net.ipv4.ip_forward=1
```

### Step 6 — Configure NAT masquerading

```bash
sudo iptables -t nat -A POSTROUTING -o usb0 -j MASQUERADE
```

Persist rules across reboots using `iptables-persistent`:

```bash
sudo apt install iptables-persistent
sudo netfilter-persistent save
```

When prompted during install, choose **Yes** to save current IPv4 rules.

### Step 7 — Reboot

```bash
sudo reboot
```

Devices plugged into the Pi's Ethernet port will now receive IPs in the `192.168.10.x` range and route through the USB internet connection.

---

## Guide 2: USB Internet → Wi-Fi Access Point

> All commands run on the Raspberry Pi terminal or over SSH.

### Step 1 — Identify interface names

```bash
ifconfig
```

Note your USB interface (`usb0`) and Wi-Fi interface (`wlan0`).

### Step 2 — Install dnsmasq and hostapd

```bash
sudo apt install dnsmasq hostapd
sudo systemctl stop dnsmasq
sudo systemctl stop hostapd
```

### Step 3 — Set a static IP for wlan0

Edit `/etc/dhcpcd.conf`:

```bash
sudo nano /etc/dhcpcd.conf
```

Append:

```
interface wlan0
    static ip_address=192.168.20.1/24
    nohook wpa_supplicant
```

Restart the DHCP client:

```bash
sudo service dhcpcd restart
```

### Step 4 — Configure dnsmasq

```bash
sudo nano /etc/dnsmasq.conf
```

Add:

```
interface=wlan0
dhcp-range=192.168.20.150,192.168.20.200,255.255.255.0,24h
```

Reload:

```bash
sudo systemctl reload dnsmasq
```

### Step 5 — Configure hostapd

```bash
sudo nano /etc/hostapd/hostapd.conf
```

Add:

```
interface=wlan0
driver=nl80211
ssid=RaspberryPi
hw_mode=g
channel=7
wmm_enabled=0
macaddr_acl=0
auth_algs=1
ignore_broadcast_ssid=0
wpa=2
wpa_passphrase=YOUR_STRONG_PASSWORD
wpa_key_mgmt=WPA-PSK
wpa_pairwise=TKIP
rsn_pairwise=CCMP
```

> **Security note:** Change `wpa_passphrase` to a strong, unique password before use.

`hw_mode` options: `g` = 2.4 GHz, `a` = 5 GHz, `b` = 2.4 GHz (legacy).

Tell the system where to find the config file:

```bash
sudo nano /etc/default/hostapd
```

Find `#DAEMON_CONF` and replace it with:

```
DAEMON_CONF="/etc/hostapd/hostapd.conf"
```

### Step 6 — Enable and start hostapd

```bash
sudo systemctl unmask hostapd
sudo systemctl enable hostapd
sudo systemctl start hostapd
```

### Step 7 — Enable IP forwarding and NAT (same as Ethernet guide)

```bash
sudo nano /etc/sysctl.conf   # uncomment net.ipv4.ip_forward=1
sudo sysctl -w net.ipv4.ip_forward=1
sudo iptables -t nat -A POSTROUTING -o usb0 -j MASQUERADE
sudo apt install iptables-persistent
sudo netfilter-persistent save
```

### Step 8 — Reboot

```bash
sudo reboot
```

The Pi will broadcast an SSID of `RaspberryPi`. Clients connecting to it will receive IPs in the `192.168.20.x` range and route through the USB internet connection.

---

## Troubleshooting

| Issue | Check |
|---|---|
| No DHCP lease on client | `sudo systemctl status dnsmasq` — look for errors |
| USB interface not showing | Ensure the device supports USB tethering and is enabled on the device |
| No internet on client | Verify `net.ipv4.ip_forward=1` is active: `cat /proc/sys/net/ipv4/ip_forward` |
| iptables rules lost after reboot | Run `sudo netfilter-persistent reload` |
| hostapd fails to start | Check `sudo journalctl -u hostapd` — often a channel/hw_mode mismatch |

---

## Disclaimer

Tested with JioFi3 and Airtel MyWifi (Android-based USB routers). Expected to work with any device that supports USB tethering. Compatibility with older internet devices may vary.
