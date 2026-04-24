# Objective, Requirements & Assumptions

## Objective

Share a USB internet device (JioFi3, Airtel MyWifi, or any USB tethering-capable device) over a Raspberry Pi's Ethernet port or Wi-Fi interface — making the USB internet connection available to other devices on the network.

**Personal use case:** An unstable broadband connection at home, supplemented by a mobile data connection via JioFi3, fed into a TP-Link TL-R470T+ load-balancing router (1 dedicated WAN, 3 configurable WAN/LAN, 1 dedicated LAN) for automatic failover. Since the router requires WAN connections over Ethernet, the Pi bridges the USB device to an Ethernet WAN port.

## Requirements

| Item | Notes |
|---|---|
| Raspberry Pi | Any model with at least one USB port and one Ethernet port |
| MicroSD Card | 8 GB or larger |
| Power Adapter | 5V 2A recommended |
| USB Internet Device | JioFi3, Airtel MyWifi, or any device supporting USB tethering |
| Micro USB Cable | Compatible with your USB internet device |
| Ethernet Cable | Required for the USB → Ethernet guide |

## Assumptions

- The USB internet device supports USB tethering.
- Raspberry Pi OS (Raspbian) is installed on the Pi.
- The Pi has internet access and is reachable over SSH or VNC.

## Interface Reference

| Interface | Represents |
|---|---|
| `usb0` | USB internet device (tethered mobile/hotspot) |
| `eth0` | Pi's Ethernet port (or USB-to-Ethernet adapter) |
| `wlan0` | Pi's Wi-Fi interface |
