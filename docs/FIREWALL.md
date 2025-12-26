# Firewall & Port Forwarding Guide

How to configure your network to allow Litecoin node traffic.

## Ports Overview

| Port | Protocol | Service | Required? |
|------|----------|---------|-----------|
| 9333 | TCP | P2P (peer network) | **Yes** - blockchain sync |
| 9332 | TCP | RPC (wallet/pool) | Depends on mode |
| 28332 | TCP | ZMQ hashblock (pool) | Optional |
| 28333 | TCP | ZMQ rawblock (pool) | Optional |

## What to Forward Based on Node Type

### Mining Pool Backend

**Forward only port 9333:**

| Port | Forward to VM? | Why |
|------|----------------|-----|
| **9333** | **YES** | Blockchain sync, peer connections |
| 9332 | NO | Localhost only (pool software runs locally) |
| 28332 | NO | Localhost only (if ZMQ enabled) |
| 28333 | NO | Localhost only (if ZMQ enabled) |

> **Important:** RPC and ZMQ ports should NEVER be exposed to the internet. Keep them on localhost (127.0.0.1) only.

### Standard Node (Personal Use)

**Minimum (local wallets only):**

| Port | Forward to VM? |
|------|----------------|
| **9333** | **YES** |
| 9332 | NO |

**With remote wallet access:**

| Port | Forward to VM? |
|------|----------------|
| **9333** | **YES** |
| **9332** | **YES** (with authentication!) |

> **Warning:** Only expose RPC (9332) if you understand the security implications. Always use strong RPC credentials.

## Router Configuration

### Generic Router Steps

1. Log into your router admin panel (usually `192.168.1.1` or `192.168.0.1`)
2. Find "Port Forwarding" or "NAT" settings
3. Add new rule:
   - **External Port:** 9333
   - **Internal IP:** Your VM's IP (e.g., `192.168.1.100`)
   - **Internal Port:** 9333
   - **Protocol:** TCP
4. Save and apply

### Ubiquiti UniFi

1. Open UniFi Controller
2. Go to **Settings → Routing & Firewall → Port Forwarding**
3. Click **Create New Port Forward**
4. Configure:
   ```
   Name: Litecoin P2P
   From: Any
   Port: 9333
   Forward IP: [Your VM IP]
   Forward Port: 9333
   Protocol: TCP
   ```
5. Click **Apply**

### Ubiquiti EdgeRouter

```bash
configure
set port-forward rule 1 description "Litecoin P2P"
set port-forward rule 1 forward-to address 192.168.1.100
set port-forward rule 1 forward-to port 9333
set port-forward rule 1 original-port 9333
set port-forward rule 1 protocol tcp
commit
save
```

### pfSense

1. Go to **Firewall → NAT → Port Forward**
2. Click **Add**
3. Configure:
   - Interface: WAN
   - Protocol: TCP
   - Destination port range: 9333-9333
   - Redirect target IP: [VM IP]
   - Redirect target port: 9333
4. Save and Apply

### OPNsense

1. Go to **Firewall → NAT → Port Forward**
2. Add rule:
   - Interface: WAN
   - TCP
   - Destination port: 9333
   - Redirect IP: [VM IP]
   - Redirect port: 9333
3. Apply

### MikroTik RouterOS

```bash
/ip firewall nat add chain=dstnat action=dst-nat \
  dst-port=9333 protocol=tcp to-addresses=192.168.1.100 to-ports=9333 \
  comment="Litecoin P2P"
```

## UFW (VM Firewall)

The setup script configures UFW automatically. To verify or modify:

```bash
# Check current rules
sudo ufw status verbose

# Manually allow ports
sudo ufw allow 9333/tcp comment 'Litecoin P2P'
sudo ufw allow 9332/tcp comment 'Litecoin RPC'  # Only if needed

# Remove a rule
sudo ufw delete allow 9332/tcp

# Reload
sudo ufw reload
```

### Recommended UFW Rules by Node Type

**Mining Pool Backend:**
```bash
sudo ufw allow 9333/tcp comment 'Litecoin P2P'
sudo ufw allow ssh comment 'SSH Access'
# RPC and ZMQ stay on localhost - no UFW rule needed
```

**Standard Node with Remote Wallet:**
```bash
sudo ufw allow 9333/tcp comment 'Litecoin P2P'
sudo ufw allow 9332/tcp comment 'Litecoin RPC'
sudo ufw allow ssh comment 'SSH Access'
```

## Verify Port is Open

### From the VM (internal)

```bash
# Check if litecoind is listening
sudo ss -tlnp | grep litecoind

# Expected output (mining pool mode):
# LISTEN  0  128  0.0.0.0:9333  *  users:(("litecoind",pid=1234,fd=10))
# LISTEN  0  128  127.0.0.1:9332  *  users:(("litecoind",pid=1234,fd=11))
```

### From Outside Your Network

**Using netcat:**
```bash
nc -zv YOUR_PUBLIC_IP 9333
# Success: "Connection to YOUR_PUBLIC_IP 9333 port [tcp/*] succeeded!"
```

**Using online tools:**
- https://www.yougetsignal.com/tools/open-ports/
- https://canyouseeme.org/
- https://portchecker.co/

### Check Peer Connections

A good test that port forwarding is working:

```bash
litecoin-cli -conf=/etc/litecoin/litecoin.conf getnetworkinfo | grep -E "connections|connections_in|connections_out"
```

- **connections_in: 0** - Port forwarding is NOT working
- **connections_in: 8+** - Good, you're accepting incoming connections
- **connections: 50+** - Excellent connectivity

## Common Issues

### Port Shows Closed But Service is Running

1. **Check UFW on VM:**
   ```bash
   sudo ufw status
   # Make sure 9333 is allowed
   ```

2. **Check router forwarding:**
   - Is the rule active?
   - Is the VM IP correct?
   - Did VM IP change? (DHCP lease expired)

3. **Check ISP blocking:**
   - Some ISPs block non-standard ports
   - Try from a mobile hotspot to test

4. **Check litecoind is binding correctly:**
   ```bash
   sudo ss -tlnp | grep 9333
   # Should show litecoind listening on 0.0.0.0:9333
   ```

### Double NAT

If your network setup is:
```
Internet → ISP Router → Your Router → VM
```

You need to forward ports on BOTH routers, or put ISP router in bridge mode.

**Detecting Double NAT:**
```bash
# On VM, check your gateway
ip route | grep default
# Note the gateway IP (e.g., 192.168.1.1)

# If your public IP doesn't match what you see at whatismyip.com,
# you have double NAT
```

### VM Network Mode

If using VirtualBox/VMware/Proxmox:

| Network Mode | Port Forward Needed? |
|--------------|---------------------|
| **Bridged** | Forward on router only |
| **NAT** | Forward on router AND VM host |
| **Host-only** | Can't access from internet |

**For Proxmox/Unraid VMs:** Use **bridged (br0/vmbr0)** mode for simplest setup.

### Firewall Blocking Outbound

If your node can't connect to peers, ensure outbound TCP is allowed:

```bash
# Check if you can reach other nodes
nc -zv 199.247.0.217 9333  # Example Litecoin seed node
```

## Security Notes

- **Never expose RPC (9332) without authentication** on a public IP
- Mining pool mode keeps RPC on localhost (127.0.0.1) for security
- P2P port (9333) is safe to expose - it's designed for public access
- ZMQ ports should always be localhost only
- Consider using a VPN if your ISP blocks crypto traffic
- Use strong, unique RPC credentials (the setup script generates these)

## Finding Your VM's IP

```bash
# On the VM
ip addr show | grep "inet " | grep -v 127.0.0.1

# Or simply
hostname -I

# Get just the primary IP
hostname -I | awk '{print $1}'
```

## Static IP Recommendation

To prevent port forwarding from breaking when DHCP lease changes:

### Option A: Static IP on VM

Edit `/etc/network/interfaces` (Debian):

```bash
auto eth0
iface eth0 inet static
    address 192.168.1.100
    netmask 255.255.255.0
    gateway 192.168.1.1
    dns-nameservers 1.1.1.1 8.8.8.8
```

Or with systemd-networkd:

```bash
# /etc/systemd/network/10-static.network
[Match]
Name=eth0

[Network]
Address=192.168.1.100/24
Gateway=192.168.1.1
DNS=1.1.1.1
DNS=8.8.8.8
```

### Option B: DHCP Reservation

Most routers allow you to "reserve" an IP for a specific MAC address:

1. Find your VM's MAC: `ip link show eth0 | grep ether`
2. In router admin, find DHCP reservation settings
3. Add entry: MAC → 192.168.1.100

## Testing Connectivity

### Full Connectivity Test

```bash
#!/bin/bash
echo "=== Litecoin Node Connectivity Test ==="

echo -e "\n1. Service Status:"
systemctl is-active litecoind

echo -e "\n2. Listening Ports:"
ss -tlnp | grep litecoind

echo -e "\n3. Peer Connections:"
litecoin-cli -conf=/etc/litecoin/litecoin.conf getnetworkinfo 2>/dev/null | grep -E '"connections"|connections_in|connections_out' || echo "RPC not responding"

echo -e "\n4. UFW Status:"
ufw status | head -10

echo -e "\n5. External Port Check (requires curl):"
PUBLIC_IP=$(curl -s ifconfig.me)
echo "Your public IP: $PUBLIC_IP"
echo "Test port 9333 at: https://canyouseeme.org/"
```

Save as `test-connectivity.sh` and run with `sudo bash test-connectivity.sh`.

