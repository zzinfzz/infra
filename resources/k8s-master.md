```bash
# As root, create a new user (run this on all nodes)
useradd -m -s /bin/bash k8sadmin
usermod -aG sudo k8sadmin

# Set password
passwd k8sadmin

sudo apt install -y apt-transport-https ca-certificates curl gpg software-properties-common

# Disable swap (required for Kubernetes)
sudo swapoff -a
sudo sed -i '/ swap / s/^\(.*\)$/#\1/g' /etc/fstab

swapon --show
# If swap is disabled, this should return no output

# Load required kernel modules
cat <<EOF | sudo tee /etc/modules-load.d/k8s.conf
overlay
br_netfilter
EOF

sudo modprobe overlay
sudo modprobe br_netfilter

# Configure sysctl for Kubernetes
cat <<EOF | sudo tee /etc/sysctl.d/k8s.conf
net.bridge.bridge-nf-call-iptables  = 1
net.bridge.bridge-nf-call-ip6tables = 1
net.ipv4.ip_forward                 = 1
EOF

sudo sysctl --system

# Add Docker repository (for containerd)
curl -fsSL https://download.docker.com/linux/ubuntu/gpg | sudo gpg --dearmor -o /usr/share/keyrings/docker-archive-keyring.gpg

echo "deb [arch=$(dpkg --print-architecture) signed-by=/usr/share/keyrings/docker-archive-keyring.gpg] https://download.docker.com/linux/ubuntu $(lsb_release -cs) stable" | sudo tee /etc/apt/sources.list.d/docker.list > /dev/null

# Install containerd
sudo apt update
sudo apt install -y containerd.io

# Configure containerd
sudo mkdir -p /etc/containerd
containerd config default | sudo tee /etc/containerd/config.toml

# Enable SystemdCgroup in containerd config
sudo sed -i 's/SystemdCgroup \= false/SystemdCgroup \= true/g' /etc/containerd/config.toml

# Start and enable containerd
sudo systemctl restart containerd
sudo systemctl enable containerd
sudo systemctl status containerd
containerd --version

# Configure containerd to use SOCKS proxy
sudo mkdir -p /etc/systemd/system/containerd.service.d

sudo tee /etc/systemd/system/containerd.service.d/proxy.conf > /dev/null << EOF
[Service]
Environment="HTTP_PROXY=socks5://10.0.0.2:1080"
Environment="HTTPS_PROXY=socks5://10.0.0.2:1080"
Environment="NO_PROXY=localhost,127.0.0.1,10.0.0.0/8,192.168.0.0/16,172.20.0.0/16"
EOF

# Restart containerd
sudo systemctl daemon-reload
sudo systemctl restart containerd

curl -fsSL https://pkgs.k8s.io/core:/stable:/v1.30/deb/Release.key | sudo gpg --dearmor -o /etc/apt/keyrings/kubernetes-apt-keyring.gpg

echo 'deb [signed-by=/etc/apt/keyrings/kubernetes-apt-keyring.gpg] https://pkgs.k8s.io/core:/stable:/v1.30/deb/ /' | sudo tee /etc/apt/sources.list.d/kubernetes.list
sudo apt update
sudo apt install -y kubelet kubeadm kubectl
sudo apt-mark hold kubelet kubeadm kubectl

# etcd in SSD
sudo mkdir -p /mnt/HC_Volume_102640304/etcd


# Create etcd data directory on the separate disk
sudo mkdir -p /mnt/HC_Volume_102640304/etcd
sudo chown -R root:root /mnt/HC_Volume_102640304/etcd
sudo chmod 700 /mnt/HC_Volume_102640304/etcd

cat <<EOF > kubeadm-config.yaml
apiVersion: kubeadm.k8s.io/v1beta3
kind: InitConfiguration
localAPIEndpoint:
  advertiseAddress: 10.0.1.3
  bindPort: 6443
---
apiVersion: kubeadm.k8s.io/v1beta3
kind: ClusterConfiguration
kubernetesVersion: v1.30.0
controlPlaneEndpoint: "10.0.0.3:6443"
networking:
  podSubnet: "192.168.0.0/16"
  serviceSubnet: "172.20.0.0/16"
etcd:
  local:
    dataDir: "/mnt/HC_Volume_102640304/etcd"
certificatesDir: "/etc/kubernetes/pki"
---
apiVersion: kubeadm.k8s.io/v1beta3
kind: JoinConfiguration
discovery:
  bootstrapToken:
    token: ""
    apiServerEndpoint: "10.0.0.3:6443"
    caCertHashes: []
controlPlane:
  localAPIEndpoint:
    advertiseAddress: 10.0.1.3
    bindPort: 6443
  certificateKey: ""
EOF

# Configure kubectl for your user
mkdir -p $HOME/.kube
sudo cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
sudo chown $(id -u):$(id -g) $HOME/.kube/config


sudo kubeadm init phase upload-certs --upload-certs
sudo kubeadm token create --print-join-command --certificate-key 14f903854ddacd54978a05eb7acf443fab20b7ecfabb2ff9ae8b2928c47e2421

# Master II
sudo mkdir -p /mnt/HC_Volume_102640419/etcd
sudo chown -R root:root /mnt/HC_Volume_102640419/etcd
sudo chmod 700 /mnt/HC_Volume_102640419/etcd

# Create join configuration for Master2
cat <<EOF > kubeadm-join-master2.yaml
apiVersion: kubeadm.k8s.io/v1beta3
kind: JoinConfiguration
discovery:
  bootstrapToken:
    token: "hc1kgf.rlkhkgubkiki8ulu"
    apiServerEndpoint: "10.0.0.3:6443"
    caCertHashes: ["sha256:c19c0545cf7ab0839fd4dd95205adb17c1826c2920d582b5238bad4405d11bb3"]
controlPlane:
  localAPIEndpoint:
    advertiseAddress: 10.0.1.1
    bindPort: 6443
  certificateKey: "b8c52a352529de818c9914aa12e5c3912e284f3d852f977293e1e4a07ade10a9"
---
apiVersion: kubeadm.k8s.io/v1beta3
kind: ClusterConfiguration
etcd:
  local:
    dataDir: "/mnt/HC_Volume_102640419/etcd"
EOF

# Join using the configuration file
sudo kubeadm join --config=kubeadm-join-master2.yaml

# Master III
df -h
/mnt/HC_Volume_102640442

sudo mkdir -p /mnt/HC_Volume_102640442/etcd
sudo chown -R root:root /mnt/HC_Volume_102640442/etcd
sudo chmod 700 /mnt/HC_Volume_102640442/etcd

# Create join configuration for Master2

cat <<EOF > kubeadm-join-master3.yaml
apiVersion: kubeadm.k8s.io/v1beta3
kind: JoinConfiguration
discovery:
  bootstrapToken:
    token: "hc1kgf.rlkhkgubkiki8ulu"
    apiServerEndpoint: "10.0.0.3:6443"
    caCertHashes: ["sha256:c19c0545cf7ab0839fd4dd95205adb17c1826c2920d582b5238bad4405d11bb3"]
controlPlane:
  localAPIEndpoint:
    advertiseAddress: 10.0.1.2
    bindPort: 6443
  certificateKey: "b8c52a352529de818c9914aa12e5c3912e284f3d852f977293e1e4a07ade10a9"
---
apiVersion: kubeadm.k8s.io/v1beta3
kind: ClusterConfiguration
etcd:
  local:
    dataDir: "/mnt/HC_Volume_102640442/etcd"
EOF

sudo kubeadm join --config=kubeadm-join-master3.yaml



useradd -m -s /bin/bash k8sworker
usermod -aG sudo k8sworker
passwd k8sworker


sudo kubeadm join 10.0.0.3:6443 \
  --token hc1kgf.rlkhkgubkiki8ulu \
  --discovery-token-ca-cert-hash sha256:c19c0545cf7ab0839fd4dd95205adb17c1826c2920d582b5238bad4405d11bb3



#CNI

# Create bridge CNI configuration
sudo tee /etc/cni/net.d/10-bridge.conf > /dev/null << EOF
{
    "cniVersion": "0.3.1",
    "name": "mynet",
    "type": "bridge",
    "bridge": "cni0",
    "isGateway": true,
    "ipMasq": true,
    "hairpinMode": true,
    "ipam": {
        "type": "host-local",
        "subnet": "192.168.0.0/16",
        "routes": [
            { "dst": "0.0.0.0/0" }
        ]
    }
}
EOF

sudo tee /etc/cni/net.d/99-loopback.conf > /dev/null << EOF
{
    "cniVersion": "0.3.1",
    "name": "lo",
    "type": "loopback"
}
EOF

# Restart kubelet to pick up CNI
sudo systemctl restart kubelet

# Cilium
# Remove bridge CNI configuration
sudo rm -f /etc/cni/net.d/10-bridge.conf
sudo rm -f /etc/cni/net.d/99-loopback.conf

# Clean up bridge interfaces
sudo ip link delete cni0 2>/dev/null || true

# Restart kubelet to clean up
sudo systemctl restart kubelet

cilium install \
  --set ipam.mode=kubernetes \
  --set routingMode=native \
  --set autoDirectNodeRoutes=true \
  --set ipv4NativeRoutingCIDR=192.168.0.0/16 \
  --set enableIPv4Masquerade=true \
  --set enableIPMasqAgent=false \
  --set hubble.relay.enabled=false \
  --set hubble.ui.enabled=false \
  --set prometheus.enabled=false


cilium upgrade \
  --set hubble.relay.enabled=true \
  --set hubble.ui.enabled=true \
  --set prometheus.enabled=true \
  --set hubble.metrics.enabled="{dns,drop,tcp,flow,port-distribution,icmp,http}"



```