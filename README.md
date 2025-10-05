# Raspberry Pi Zero 2W USB Gadget Ethernet Setup

This guide documents the configuration for connecting to a Raspberry Pi Zero 2W via USB gadget mode (USB Ethernet).

## Overview

The Pi Zero 2W is configured to act as a USB Ethernet gadget, allowing SSH access over USB without needing WiFi or a physical Ethernet connection. When plugged into a host computer via the USB port (not the power port), it appears as a network device.

## Pi Configuration

### Static IP Configuration

The Pi is configured with a static IP address for the USB interface in `/etc/dhcpcd.conf`:

```bash
# USB gadget static IP
interface usb0
static ip_address=10.55.0.2/24
```

This configuration:
- Assigns `10.55.0.2/24` to the `usb0` interface on the Pi
- Does not interfere with WiFi (wlan0) DHCP configuration
- Persists across reboots

### Applying Configuration

If you modify `/etc/dhcpcd.conf`, restart the service:

```bash
sudo systemctl restart dhcpcd
```

## Host Computer Connection

### Automatic Connection (One-Liner)

Use this command to automatically detect the USB interface and configure networking:

```bash
USBIF=$(ip -o link show | awk -F': ' '$2 ~ /enx[0-9a-f]{12}|u[0-9]i[0-9]/ {print $2}' | head -1) && sudo ip addr add 10.55.0.1/24 dev $USBIF && echo "Connected via $USBIF, SSH to 10.55.0.2"
```

Then connect via SSH:

```bash
ssh anthony@10.55.0.2
```

### Manual Connection

1. **Find the USB interface:**
   ```bash
   ip link show
   ```
   Look for an interface like `enx` followed by 12 hex characters (e.g., `enxee54cd375af5`) or `enp*u*i*` pattern.

2. **Assign IP address to the host interface:**
   ```bash
   sudo ip addr add 10.55.0.1/24 dev <interface_name>
   ```

   Example:
   ```bash
   sudo ip addr add 10.55.0.1/24 dev enxee54cd375af5
   ```

3. **Connect via SSH:**
   ```bash
   ssh anthony@10.55.0.2
   ```

## Network Details

- **Pi USB IP:** `10.55.0.2/24`
- **Host USB IP:** `10.55.0.1/24` (configured manually)
- **Subnet:** `10.55.0.0/24`

## Troubleshooting

### USB interface not appearing

1. Verify the Pi is connected via the **USB port**, not the power port
2. Check if the USB gadget is detected:
   ```bash
   lsusb | grep -i "linux-usb\|gadget\|rndis"
   ```
   You should see: `Linux-USB Ethernet/RNDIS Gadget`

### Cannot ping or SSH to the Pi

1. Verify the host interface has the correct IP:
   ```bash
   ip addr show <interface_name>
   ```
   Should show `10.55.0.1/24`

2. Try pinging the Pi:
   ```bash
   ping 10.55.0.2
   ```

3. Check if the Pi's USB interface is up:
   ```bash
   ssh anthony@<wifi-ip> "ip addr show usb0"
   ```

### WiFi stops working after USB configuration

If WiFi loses its IPv4 address after configuring USB gadget mode:

```bash
sudo systemctl restart dhcpcd
```

This should restore WiFi DHCP without affecting the USB static IP.

### Interface name keeps changing

The USB Ethernet interface name is derived from the MAC address and may change. The one-liner command automatically detects it by pattern matching for:
- `enx` followed by 12 hex digits (MAC-based naming)
- `enp*u*i*` pattern (USB bus-based naming)

## Notes

- The USB gadget configuration requires the Pi to be in gadget mode (default on Pi Zero/Zero 2W)
- The host IP assignment is temporary and will be lost on reboot or if the Pi is unplugged
- For persistent configuration on the host, consider creating a udev rule or NetworkManager connection profile
- This setup allows simultaneous WiFi and USB connectivity
