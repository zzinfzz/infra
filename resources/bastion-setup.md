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

# On proxy server (10.0.0.2), update the configuration
sudo tee /etc/sockd.conf > /dev/null << 'EOF'
# Basic logging
logoutput: /var/log/sockd.log

# Listen on both IPv4 and IPv6
internal: 10.0.0.2 port = 1080        # IPv4 internal interface
internal: :: port = 1080               # IPv6 - listen on all IPv6 addresses

# External interfaces
external: eth0

# No authentication required
method: none

# Resource limits
child.maxrequests: 100
timeout.connect: 30
timeout.io: 300

# Client access rules - allow internal network to reach anywhere
client pass {
    from: 10.0.0.0/16 to: 0.0.0.0/0          # IPv4 clients to IPv4 destinations
}
client pass {
    from: 10.0.0.0/16 to: ::/0               # IPv4 clients to IPv6 destinations
}

# SOCKS forwarding rules - allow connections to both IPv4 and IPv6
pass {
    from: 10.0.0.0/16 to: 0.0.0.0/0          # IPv4 to IPv4
    command: connect
}
pass {
    from: 10.0.0.0/16 to: ::/0               # IPv4 to IPv6
    command: connect
}
EOF

sudo mkdir -p /etc/systemd/system/sockd.service.d
echo '[Service]
MemoryMax=512M
MemoryHigh=400M
TasksMax=30' | sudo tee /etc/systemd/system/sockd.service.d/override.conf

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

# Create kubelet proxy configuration directory
sudo mkdir -p /etc/systemd/system/kubelet.service.d/

# Create proxy configuration file for kubelet
sudo tee /etc/systemd/system/kubelet.service.d/proxy.conf << 'EOF'
[Service]
Environment="HTTP_PROXY=socks5://10.0.0.2:1080"
Environment="HTTPS_PROXY=socks5://10.0.0.2:1080"
Environment="NO_PROXY=localhost,127.0.0.1,10.0.0.0/16,192.168.0.0/16,172.20.0.0/16"
EOF

# Reload systemd configuration
sudo systemctl daemon-reload

# Restart kubelet to apply new settings
sudo systemctl restart kubelet

# Add proxy settings to system environment
sudo tee -a /etc/environment << 'EOF'
HTTP_PROXY=socks5://10.0.0.2:1080
HTTPS_PROXY=socks5://10.0.0.2:1080
NO_PROXY=localhost,127.0.0.1,10.0.0.0/16,192.168.0.0/16,172.20.0.0/16
EOF

# bashrc file.
export HTTP_PROXY=socks5://10.0.0.2:1080
export HTTPS_PROXY=socks5://10.0.0.2:1080
export NO_PROXY=localhost,127.0.0.1,10.0.0.0/16,192.168.0.0/16,172.20.0.0/16

# Test direct connection to Kubernetes API server once k8s is running
curl -k https://172.20.0.1:443 --connect-timeout 5

ls -la  /etc/resolv.conf 
lrwxrwxrwx. 1 root root 39 Apr 23 06:16 /etc/resolv.conf -> ../run/systemd/resolve/stub-resolv.conf

# DNS
cat /etc/resolv.conf 
nameserver 8.8.8.8
nameserver 8.8.4.4
nameserver 1.1.1.1
search .


```