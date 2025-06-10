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
Environment="NO_PROXY=localhost,127.0.0.1,10.0.0.0/16,192.168.0.0/16,172.20.0.0/16"
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
sudo mkdir -p /mnt/HC_Volume_102701581/etcd


# Create etcd data directory on the separate disk
sudo mkdir -p /mnt/HC_Volume_102701581/etcd
sudo chown -R root:root /mnt/HC_Volume_102701581/etcd
sudo chmod 700 /mnt/HC_Volume_102701581/etcd

cat <<EOF > kubeadm-config.yaml
apiVersion: kubeadm.k8s.io/v1beta3
kind: InitConfiguration
localAPIEndpoint:
  advertiseAddress: 10.0.1.1
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
    dataDir: "/mnt/HC_Volume_102701581/etcd"
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
    advertiseAddress: 10.0.1.1
    bindPort: 6443
  certificateKey: ""
EOF

sudo kubeadm init --config=kubeadm-config.yaml --upload-certs

# Configure kubectl for your user
mkdir -p $HOME/.kube
sudo cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
sudo chown $(id -u):$(id -g) $HOME/.kube/config


sudo kubeadm init phase upload-certs --upload-certs
sudo kubeadm token create --print-join-command --certificate-key 14f903854ddacd54978a05eb7acf443fab20b7ecfabb2ff9ae8b2928c47e2421

# Master II
sudo mkdir -p /mnt/HC_Volume_102701584/etcd
sudo chown -R root:root /mnt/HC_Volume_102701584/etcd
sudo chmod 700 /mnt/HC_Volume_102701584/etcd

# Create join configuration for Master2
cat <<EOF > kubeadm-join-master2.yaml
apiVersion: kubeadm.k8s.io/v1beta3
kind: JoinConfiguration
discovery:
  bootstrapToken:
    token: "ujizu7.7rqb1ianj2cbqrz9"
    apiServerEndpoint: "10.0.0.3:6443"
    caCertHashes: ["sha256:32a91f1c059e00e47f06aca2077aed43d5f76ced0763b491dd211ed8bbb1b48b"]
controlPlane:
  localAPIEndpoint:
    advertiseAddress: 10.0.1.2
    bindPort: 6443
  certificateKey: "28e5fcd4b4a9ff6551c23d2501ac4f97fc655b83df6113066090d6fbfcef66f0"
---
apiVersion: kubeadm.k8s.io/v1beta3
kind: ClusterConfiguration
etcd:
  local:
    dataDir: "/mnt/HC_Volume_102701584/etcd"
EOF

# Join using the configuration file
sudo kubeadm join --config=kubeadm-join-master2.yaml

# Master III
df -h
/mnt/HC_Volume_102701588/etcd

sudo mkdir -p /mnt/HC_Volume_102701588/etcd
sudo chown -R root:root /mnt/HC_Volume_102701588/etcd
sudo chmod 700 /mnt/HC_Volume_102701588/etcd

# Create join configuration for Master2

cat <<EOF > kubeadm-join-master3.yaml
apiVersion: kubeadm.k8s.io/v1beta3
kind: JoinConfiguration
discovery:
  bootstrapToken:
    token: "ujizu7.7rqb1ianj2cbqrz9"
    apiServerEndpoint: "10.0.0.3:6443"
    caCertHashes: ["sha256:32a91f1c059e00e47f06aca2077aed43d5f76ced0763b491dd211ed8bbb1b48b"]
controlPlane:
  localAPIEndpoint:
    advertiseAddress: 10.0.1.3
    bindPort: 6443
  certificateKey: "28e5fcd4b4a9ff6551c23d2501ac4f97fc655b83df6113066090d6fbfcef66f0"
---
apiVersion: kubeadm.k8s.io/v1beta3
kind: ClusterConfiguration
etcd:
  local:
    dataDir: "/mnt/HC_Volume_102701588/etcd"
EOF

sudo kubeadm join --config=kubeadm-join-master3.yaml

# worker node

useradd -m -s /bin/bash k8sworker
usermod -aG sudo k8sworker
passwd k8sworker


sudo kubeadm join 10.0.0.3:6443 \
  --token ujizu7.7rqb1ianj2cbqrz9 \
  --discovery-token-ca-cert-hash sha256:32a91f1c059e00e47f06aca2077aed43d5f76ced0763b491dd211ed8bbb1b48b



#CNI

# Create bridge CNI configuration on all nodes
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

CILIUM_CLI_VERSION=$(curl -s https://raw.githubusercontent.com/cilium/cilium-cli/main/stable.txt)
CLI_ARCH=amd64
if [ "$(uname -m)" = "aarch64" ]; then CLI_ARCH=arm64; fi
curl -L --fail --remote-name-all https://github.com/cilium/cilium-cli/releases/download/${CILIUM_CLI_VERSION}/cilium-linux-${CLI_ARCH}.tar.gz{,.sha256sum}
sha256sum --check cilium-linux-${CLI_ARCH}.tar.gz.sha256sum
sudo tar xzvfC cilium-linux-${CLI_ARCH}.tar.gz /usr/local/bin
rm cilium-linux-${CLI_ARCH}.tar.gz{,.sha256sum}

cilium install \
  --set ipam.mode=kubernetes \
  --set routingMode=tunnel \
  --set tunnelProtocol=vxlan \
  --set kubeProxyReplacement=true \
  --set hubble.relay.enabled=true \
  --set hubble.ui.enabled=true \
  --set prometheus.enabled=true \
  --set hubble.metrics.enabled="{dns,drop,tcp,flow,port-distribution,icmp,http}"
  
kubectl edit configmap coredns -n kube-system

apiVersion: v1
kind: ConfigMap
metadata:
  name: coredns
  namespace: kube-system
data:
  Corefile: |
    .:53 {
        template ANY AAAA { rcode NXDOMAIN }
        errors
        health {
           lameduck 5s
        }
        ready
        kubernetes cluster.local in-addr.arpa ip6.arpa {
           pods insecure
           fallthrough in-addr.arpa ip6.arpa
           ttl 30
        }
        prometheus :9153
        # Change this line from: forward . /etc/resolv.conf
        forward . 8.8.8.8 8.8.4.4 {
           max_concurrent 1000
        }
        cache 30
        loop
        reload
        loadbalance
    }

# Restart CoreDNS to apply changes
kubectl rollout restart deployment/coredns -n kube-system

# Wait for rollout to complete
kubectl rollout status deployment/coredns -n kube-system

# Recommended: Include both ranges on all nodes NAT -> NAT.
iptables -t nat -A POSTROUTING -s 192.168.0.0/16 ! -d 192.168.0.0/16 -j MASQUERADE
iptables -t nat -A POSTROUTING -s 172.20.0.0/16 ! -d 172.20.0.0/16 -j MASQUERADE

# Testing from Pod
kubectl run nat-test --image=busybox:1.28 --rm -it --restart=Never -- sh

cilium connectivity test --single-node

# DNS IPv6
resolvectl status

kubectl run test-dns --image=busybox --rm -it --restart=Never -- nslookup api.hetzner.cloud

kubectl logs -n kube-system <pod-name>

kubectl label node kworker-1 node-role.kubernetes.io/worker=worker
kubectl label node kworker-2 node-role.kubernetes.io/worker=worker
```