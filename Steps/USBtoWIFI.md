# USB Internet → Wi-Fi Access Point Guide

> All commands run on the Raspberry Pi terminal or over SSH.
>
> Tested on: Raspberry Pi OS (Bullseye / Bookworm). Uses `dhcpcd` — if your OS uses NetworkManager, adjust accordingly.

## Step 1 — Identify interface names

```bash
ifconfig
```

Note your USB interface name (typically `usb0`) and Wi-Fi interface name (typically `wlan0`).

## Step 2 — Install dnsmasq and hostapd

```bash
sudo apt install dnsmasq hostapd
sudo systemctl stop dnsmasq
sudo systemctl stop hostapd
```

## Step 3 — Set a static IP for wlan0

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

## Step 4 — Configure dnsmasq

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

## Step 5 — Configure hostapd

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

> **Security:** Replace `YOUR_STRONG_PASSWORD` with a strong, unique passphrase before use.

`hw_mode` options: `g` = 2.4 GHz, `a` = 5 GHz, `b` = 2.4 GHz (legacy).

Tell the system where to find the config file:

```bash
sudo nano /etc/default/hostapd
```

Find the line with `#DAEMON_CONF` and replace it with:

```
DAEMON_CONF="/etc/hostapd/hostapd.conf"
```

## Step 6 — Enable and start hostapd

```bash
sudo systemctl unmask hostapd
sudo systemctl enable hostapd
sudo systemctl start hostapd
```

## Step 7 — Enable IP forwarding and NAT

```bash
sudo nano /etc/sysctl.conf
```

Uncomment:

```
net.ipv4.ip_forward=1
```

Apply immediately:

```bash
sudo sysctl -w net.ipv4.ip_forward=1
```

Add NAT masquerade rule and persist it:

```bash
sudo iptables -t nat -A POSTROUTING -o usb0 -j MASQUERADE
sudo apt install iptables-persistent
sudo netfilter-persistent save
```

When prompted during install, choose **Yes** to save current IPv4 rules.

## Step 8 — Reboot

```bash
sudo reboot
```

The Pi will broadcast an SSID of `RaspberryPi`. Clients connecting to it will receive IPs in the `192.168.20.x` range and route through the USB internet connection.

---

## Troubleshooting

| Issue | Check |
|---|---|
| Clients can't connect to Wi-Fi | `sudo journalctl -u hostapd` — often a channel/hw_mode mismatch |
| No DHCP lease on client | `sudo systemctl status dnsmasq` |
| No internet on client | `cat /proc/sys/net/ipv4/ip_forward` — should be `1` |
| iptables rules lost after reboot | `sudo netfilter-persistent reload` |
