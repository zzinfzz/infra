```bash
useradd -m -s /bin/bash apiuser
usermod -aG sudo apiuser

# Set password
passwd apiuser

# Update system
sudo apt update && sudo apt upgrade -y

# Add proxy to system environment (persists across reboots)
sudo tee -a /etc/environment << 'EOF'
HTTP_PROXY=socks5://10.0.0.2:1080
HTTPS_PROXY=socks5://10.0.0.2:1080
NO_PROXY=localhost,127.0.0.1,10.0.0.0/16
EOF

# Add to bashrc for future sessions
tee -a ~/.bashrc << 'EOF'
export HTTP_PROXY=socks5://10.0.0.2:1080
export HTTPS_PROXY=socks5://10.0.0.2:1080
export NO_PROXY=localhost,127.0.0.1,10.0.0.0/16
EOF



# Install dependencies
sudo apt install -y curl wget git build-essential

sudo dnf group install development-tools

# Install Erlang/OTP 28.0.x
sudo apt install build-essential autoconf m4 libncurses-dev libssl-dev openjdk-11-jdk unixodbc-dev libgl1-mesa-dev libglu1-mesa-dev libwxgtk3.2-dev xsltproc fop libxml2-utils
sudo apt install libwebkit2gtk-4.1-dev libwebkitgtk-6.0-dev

sudo dnf install java-11-openjdk-devel unixODBC-devel mesa-libGL-devel mesa-libGLU-devel fop wxGTK-devel
sudo dnf install autoconf automake libtool flex bison

# Install ASDF
git clone https://github.com/asdf-vm/asdf.git ~/.asdf --branch v0.14.0
echo '. "$HOME/.asdf/asdf.sh"' >> ~/.bashrc
echo '. "$HOME/.asdf/completions/asdf.bash"' >> ~/.bashrc
source ~/.bashrc

# Add Erlang plugin and install
asdf plugin add erlang https://github.com/asdf-vm/asdf-erlang.git
asdf install erlang 28.0
asdf global erlang 28.0

# Verify installation
erl -version
# Should show: Erlang (SMP,ASYNC_THREADS) (BEAM) emulator version 14.0.1

# Create user and directories
sudo mkdir -p /opt/api/

sudo systemctl stop explorer-api-server || true

rm -rf /opt/api/*

sudo tar -xzf /tmp/explorer_api_server-0.1.0.tar.gz -C /opt/api
sudo mkdir -p /opt/api/log
sudo chown -R apiuser:apiuser /opt/api

sudo tee /etc/systemd/system/explorer-api-server.service << 'EOF'
[Unit]
Description=Explorer API Server
After=network.target

[Service]
Type=forking
User=apiuser
Group=apiuser
WorkingDirectory=/opt/api
ExecStart=/opt/api/bin/explorer_api_server daemon
ExecStop=/opt/api/bin/explorer_api_server stop
ExecReload=/opt/api/bin/explorer_api_server restart
Restart=on-failure
RestartSec=5
Environment=HOME=/home/apiuser
TimeoutStartSec=300
TimeoutStopSec=30

[Install]
WantedBy=multi-user.target
EOF

# Reload systemd configuration
sudo systemctl daemon-reload

# Enable the service to start on boot
sudo systemctl enable explorer-api-server

# Start the service
sudo systemctl start explorer-api-server
```