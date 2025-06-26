```bash
useradd -m -s /bin/bash vpn
usermod -aG sudo vpn

# Set password
passwd vpn

sudo apt update

sudo apt install wireguard wireguard-tools resolvconf

wg genkey | sudo tee /etc/wireguard/private.key

sudo chmod 600 /etc/wireguard/private.key

sudo cat /etc/wireguard/private.key | wg pubkey | sudo tee /etc/wireguard/public.key
```
# Complete WireGuard Cross-Region Setup Guide

## Overview
- **West Hub**: 172.16.0.1 (manages 10.0.0.0/16 network)
- **East Hub**: 172.16.0.2 (manages 10.1.0.0/16 network)
- **Goal**: Allow nodes like 10.0.102.1 to communicate with 10.1.102.1

---

## STEP 1: Configure UFW on Both Hubs

# Run these commands on **BOTH** west and east hubs:


# Allow WireGuard traffic through UFW
sudo ufw route allow in on wg0 out on wg0
sudo ufw route allow in on wg0 out on eth0
sudo ufw route allow in on eth0 out on wg0
sudo ufw route allow in on wg0 from 172.16.0.0/24
sudo ufw route allow in on wg0 from 10.0.0.0/16  
sudo ufw route allow in on wg0 from 10.1.0.0/16

# Reload UFW
sudo ufw reload

---

## STEP 2: Create WireGuard Scripts

### On West Hub (172.16.0.1):

# Create startup script
sudo tee /etc/wireguard/wg0-up.sh > /dev/null << 'EOF'
#!/bin/bash
# Enable IP forwarding
echo 1 > /proc/sys/net/ipv4/ip_forward

# Route to east network
ip route add 10.1.0.0/16 dev wg0 2>/dev/null || true

logger "WireGuard West Hub: Started"
EOF

# Create shutdown script
sudo tee /etc/wireguard/wg0-down.sh > /dev/null << 'EOF'
#!/bin/bash
# Remove route
ip route del 10.1.0.0/16 dev wg0 2>/dev/null || true

logger "WireGuard West Hub: Stopped"
EOF

# Make scripts executable
sudo chmod +x /etc/wireguard/wg0-*.sh

### On East Hub (172.16.0.2):
# Create startup script
sudo tee /etc/wireguard/wg0-up.sh > /dev/null << 'EOF'
#!/bin/bash
# Enable IP forwarding
echo 1 > /proc/sys/net/ipv4/ip_forward

# Route to west network
ip route add 10.0.0.0/16 dev wg0 2>/dev/null || true

logger "WireGuard East Hub: Started"
EOF

# Create shutdown script
sudo tee /etc/wireguard/wg0-down.sh > /dev/null << 'EOF'
#!/bin/bash
# Remove route
ip route del 10.0.0.0/16 dev wg0 2>/dev/null || true

logger "WireGuard East Hub: Stopped"
EOF

# Make scripts executable
sudo chmod +x /etc/wireguard/wg0-*.sh
```
---

## STEP 3: Update WireGuard Configuration

### West Hub (/etc/wireguard/wg0.conf):

```ini
[Interface]
PrivateKey = <YOUR_WEST_PRIVATE_KEY>
Address = 172.16.0.1/24
ListenPort = 51820
PostUp = /etc/wireguard/wg0-up.sh
PostDown = /etc/wireguard/wg0-down.sh

[Peer]
PublicKey = <YOUR_EAST_PUBLIC_KEY>
Endpoint = 5.161.222.248:51820
AllowedIPs = 172.16.0.2/32, 10.1.0.0/16
PersistentKeepalive = 25
```

### East Hub (/etc/wireguard/wg0.conf):

```ini
[Interface]
PrivateKey = <YOUR_EAST_PRIVATE_KEY>
Address = 172.16.0.2/24
ListenPort = 51820
PostUp = /etc/wireguard/wg0-up.sh
PostDown = /etc/wireguard/wg0-down.sh

[Peer]
PublicKey = <YOUR_WEST_PUBLIC_KEY>
Endpoint = 5.78.124.58:51820
AllowedIPs = 172.16.0.1/32, 10.0.0.0/16
PersistentKeepalive = 25
```

---

## STEP 4: Make IP Forwarding Permanent

Run on **BOTH** hubs:

```bash
# Make IP forwarding permanent
echo 'net.ipv4.ip_forward = 1' | sudo tee -a /etc/sysctl.conf
sudo sysctl -p
```

---

## STEP 5: Restart WireGuard

Run on **BOTH** hubs:

```bash
# Stop WireGuard
sudo wg-quick down wg0

# Start with new configuration
sudo wg-quick up wg0

# Enable auto-start on boot
sudo systemctl enable wg-quick@wg0

# Check status
sudo wg show
```
