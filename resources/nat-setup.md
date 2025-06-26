```bash
nano /etc/sysctl.conf
net.ipv4.ip_forward=1

/sbin/sysctl -p

# Clear any existing rules (optional, but recommended for clean setup)
sudo iptables -F
sudo iptables -t nat -F

# Enable NAT masquerading on eth0 (external interface)
sudo iptables -t nat -A POSTROUTING -o eth0 -j MASQUERADE

# Allow forwarding from internal (enp7s0) to external (eth0)
sudo iptables -A FORWARD -i enp7s0 -o eth0 -j ACCEPT

# Allow established connections back
sudo iptables -A FORWARD -i eth0 -o enp7s0 -m state --state RELATED,ESTABLISHED -j ACCEPT

# Allow loopback traffic
sudo iptables -A INPUT -i lo -j ACCEPT

# Allow SSH access (important for remote management)
sudo iptables -A INPUT -p tcp --dport 22 -j ACCEPT

# Allow traffic on internal interface
sudo iptables -A INPUT -i enp7s0 -j ACCEPT


# Remove current IP and set it as gateway
sudo ip addr del 10.0.0.4/24 dev enp7s0
sudo ip addr add 10.0.0.1/24 dev enp7s0

# Ensure interface is up
sudo ip link set enp7s0 up

sudo nano /etc/network/interfaces
auto enp7s0
iface enp7s0 inet static
    address 10.0.0.1
    netmask 255.255.255.0
    
sudo netfilter-persistent save

# Enable the service to load rules automatically
sudo systemctl enable netfilter-persistent

# Check if it's enabled
sudo systemctl is-enabled netfilter-persistent

sudo netfilter-persistent reload


# Hetzner setup.
# Nat server
# Create NAT sudo user natadmin
echo 1 | sudo tee /proc/sys/net/ipv4/ip_forward

echo 1 | sudo tee /proc/sys/net/ipv4/ip_forward

sudo iptables -t nat -A POSTROUTING -s '10.0.0.0/16' -o eth0 -j MASQUERADE


# Hetzner automatically assigns 10.0.0.1 as the network gateway

# On all nodes in network

ip route

ip route add default via 10.0.0.1

ip route del default

ip route add default via 10.0.0.1


# NAT server

sudo vim /etc/network/interfaces

auto enp7s0
iface enp7s0 inet dhcp
    post-up echo 1 > /proc/sys/net/ipv4/ip_forward
    post-up iptables -t nat -A POSTROUTING -s '10.0.0.0/16' -o eth0 -j MASQUERADE
    post-up ip route add 10.1.0.0/16 via 10.0.0.1 2>/dev/null || true

# Add Route in gateway 10.1.0.0/16 (East) address via gateway and gateway via vpn hub. 

# Apply the new configuration or system reboot
sudo systemctl restart networking

# Verify the route was added
ip route show | grep "10.1.0.0/16"
    
# NAT Clients

sudo vim /etc/network/interfaces

auto enp7s0
iface enp7s0 inet dhcp
    post-up ip route add default via 10.0.0.1

# Manual add     
sudo ip route add default via 10.0.0.1 dev enp7s0

sudo vim /etc/systemd/system/default-route.service

[Unit]
Description=Add default route for enp7s0
After=network.target
Wants=network.target

[Service]
Type=oneshot
ExecStart=/bin/bash -c 'sleep 5 && /sbin/ip route add default via 10.0.0.1 dev enp7s0 || true'
RemainAfterExit=yes

[Install]
WantedBy=multi-user.target

sudo systemctl daemon-reload
sudo systemctl enable default-route.service
sudo systemctl start default-route.service

ping -c 3 8.8.8.8

```