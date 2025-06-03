# VNC Fedora 42 setup

GNOME desktop
```bash
sudo dnf install @gnome-desktop
```

```bash
sudo dnf install gdm
sudo systemctl enable gdm
```

```bash 
sudo dnf install tigervnc-server
```

VNC
```bash
# Set VNC password
vncpasswd

# Create VNC startup script
mkdir -p ~/.vnc
```

Need to restart server in case bastion restart. 
```bash
vncserver :1 -geometry 1920x1080 -depth 24
```

Firewall
```bash
sudo dnf install firewalld
sudo systemctl enable firewalld
sudo systemctl start firewalld

sudo firewall-cmd --permanent --add-port=5901/tcp
sudo firewall-cmd --reload
```

```bash
useradd -m -s /bin/bash proxyusr
usermod -aG wheel proxyusr
passwd proxyusr

sudo dnf install dante-server

sudo tee /etc/sockd.conf << 'EOF'
# Basic logging
logoutput: /var/log/sockd.log

# Listen on all interfaces, port 1080
internal: 0.0.0.0 port = 1080

# Use the interface connected to internet
external: eth0

# No authentication required
socksmethod: none

# Allow connections from Ubuntu server
client pass {
    from: 0.0.0.0/0 to: 0.0.0.0/0
    log: error connect disconnect
}

# Allow SOCKS requests to anywhere
socks pass {
    from: 0.0.0.0/0 to: 0.0.0.0/0
    protocol: tcp udp
    log: error connect disconnect
}
EOF


# Start the service
sudo systemctl start sockd
sudo systemctl enable sockd

# Check status
sudo systemctl status sockd

# Check if listening
# netstat is in net-tools package
sudo dnf install net-tools

# Now this will work:
sudo netstat -tlnp | grep 1080

# Open SOCKS port
sudo dnf install firewalld
sudo firewall-cmd --add-port=1080/tcp --permanent
sudo firewall-cmd --reload

# Verify
sudo firewall-cmd --list-ports

# Test
curl --proxy socks5://10.0.0.2:1080 https://httpbin.org/ip


# On Ubuntu
export ALL_PROXY=socks5://10.0.0.2:1080
```