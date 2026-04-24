# USB Internet → Ethernet Guide

> All commands run on the Raspberry Pi terminal or over SSH.
>
> Tested on: Raspberry Pi OS (Bullseye / Bookworm). Uses `dhcpcd` — if your OS uses NetworkManager, adjust accordingly.

## Step 1 — Identify interface names

```bash
ifconfig
```

Note your USB interface name (typically `usb0`) and Ethernet interface name (typically `eth0`).

## Step 2 — Install dnsmasq

```bash
sudo apt install dnsmasq
sudo systemctl stop dnsmasq
```

## Step 3 — Set a static IP for eth0

```bash
sudo nano /etc/dhcpcd.conf
```

Append to the end of the file:

```
interface eth0
    static ip_address=192.168.10.1/24
```

## Step 4 — Configure dnsmasq

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

## Step 5 — Enable IP forwarding

```bash
sudo nano /etc/sysctl.conf
```

Uncomment this line:

```
net.ipv4.ip_forward=1
```

Apply immediately (no reboot needed):

```bash
sudo sysctl -w net.ipv4.ip_forward=1
```

## Step 6 — Configure NAT masquerading

```bash
sudo iptables -t nat -A POSTROUTING -o usb0 -j MASQUERADE
```

Persist rules across reboots using `iptables-persistent`:

```bash
sudo apt install iptables-persistent
sudo netfilter-persistent save
```

When prompted during install, choose **Yes** to save current IPv4 rules.

## Step 7 — Reboot

```bash
sudo reboot
```

Devices plugged into the Pi's Ethernet port will receive IPs in the `192.168.10.x` range and route through the USB internet connection.

---

## Troubleshooting

| Issue | Check |
|---|---|
| No DHCP lease on client | `sudo systemctl status dnsmasq` |
| USB interface not showing | Ensure the device has USB tethering enabled |
| No internet on client | `cat /proc/sys/net/ipv4/ip_forward` — should be `1` |
| iptables rules lost after reboot | `sudo netfilter-persistent reload` |
