# 🚀 Kubernetes Cluster with Calico Traffic Shaping

This guide sets up a multi-node Kubernetes cluster using [`kubeadm`](https://kubernetes.io/docs/setup/production-environment/tools/kubeadm/install-kubeadm/), based on the [DevOps Pro's tutorial](https://github.com/devopsproin/certified-kubernetes-administrator/tree/main/Cluster%20Setup), with modifications for using **Calico** instead of Flannel to enable **pod-level traffic shaping** (bandwidth control).

---

## 📦 Cluster Setup Overview

- **Master Node:** Ubuntu laptop (Intel Core i7, 8GB RAM)
- **Worker Node:** Raspberry Pi 4 (2GB+ RAM)
- **CNI Plugin:** [Calico](https://www.tigera.io/project-calico/)
- **Traffic Testing:** Using `iperf3` between pods

---

## ⚙️ 1. Prerequisites

Run these on both master and worker nodes:

### 🔧 OS Setup

```
sudo apt-get update
sudo apt install apt-transport-https curl -y

```

## 🐳 Install and Configure Containerd

```
# Add Docker repo
sudo mkdir -p /etc/apt/keyrings
curl -fsSL https://download.docker.com/linux/ubuntu/gpg | sudo gpg --dearmor -o /etc/apt/keyrings/docker.gpg
echo "deb [arch=$(dpkg --print-architecture) signed-by=/etc/apt/keyrings/docker.gpg] https://download.docker.com/linux/ubuntu $(lsb_release -cs) stable" | sudo tee /etc/apt/sources.list.d/docker.list > /dev/null
sudo apt-get update
sudo apt-get install containerd.io -y

# Configure SystemdCgroup
sudo mkdir -p /etc/containerd
containerd config default | sudo tee /etc/containerd/config.toml
sudo sed -i 's/SystemdCgroup = false/SystemdCgroup = true/' /etc/containerd/config.toml
sudo systemctl restart containerd

```

## ⛓️ Install Kubernetes Tools

```
curl -fsSL https://pkgs.k8s.io/core:/stable:/v1.30/deb/Release.key | sudo gpg --dearmor -o /etc/apt/keyrings/kubernetes-apt-keyring.gpg
echo "deb [signed-by=/etc/apt/keyrings/kubernetes-apt-keyring.gpg] https://pkgs.k8s.io/core:/stable:/v1.30/deb/ /" | sudo tee /etc/apt/sources.list.d/kubernetes.list
sudo apt-get update
sudo apt-get install -y kubelet kubeadm kubectl
sudo apt-mark hold kubelet kubeadm kubectl

```

## 🧠 Kernel Config & Swap

```
sudo swapoff -a
sudo sed -i '/ swap / s/^/#/' /etc/fstab

sudo modprobe br_netfilter
echo 'br_netfilter' | sudo tee /etc/modules-load.d/k8s.conf
sudo sysctl -w net.ipv4.ip_forward=1

```
# 🚀 2. Cluster Initialization (Master Only)

```
sudo kubeadm init --pod-network-cidr=10.244.0.0/16
mkdir -p $HOME/.kube
sudo cp /etc/kubernetes/admin.conf $HOME/.kube/config
sudo chown $(id -u):$(id -g) $HOME/.kube/config

```
# 🔁 3. Install Calico

```
# On master node
curl -O https://raw.githubusercontent.com/projectcalico/calico/v3.27.0/manifests/calico.yaml
kubectl apply -f calico.yaml

```



# 🤝 4. Join Worker Node

###📛 Rename worker node if hostname conflicts with master:

```
sudo hostnamectl set-hostname worker-pi

```
Then run the kubeadm join ... command provided by the master output.


# 📡 5. Pod Traffic Shaping (Testing Calico)

## 🧪 Bandwidth-Limited Pod on Worker

```
# shaped-ubuntu.yaml
apiVersion: v1
kind: Pod
metadata:
  name: shaped-ubuntu
  annotations:
    kubernetes.io/egress-bandwidth: 100M
spec:
  nodeSelector:
    kubernetes.io/hostname: worker-pi
  containers:
  - name: ubuntu
    image: ubuntu
    command: ["sleep", "3600"]

```

Apply:


```
kubectl apply -f shaped-ubuntu.yaml

```

📡 Iperf3 Server Pod on Master

```
# iperf-server.yaml
apiVersion: v1
kind: Pod
metadata:
  name: iperf-server
spec:
  nodeSelector:
    kubernetes.io/hostname: master-node
  tolerations:
  - key: "node-role.kubernetes.io/control-plane"
    operator: "Exists"
    effect: "NoSchedule"
  containers:
  - name: iperf3
    image: networkstatic/iperf3
    command: ["iperf3", "-s"]
    ports:
    - containerPort: 5201

```

Apply:

```
kubectl apply -f iperf-server.yaml

```

📈 Run the Test

```
kubectl exec -it shaped-ubuntu -- bash
apt update && apt install -y iperf3
iperf3 -c <IP-OF-IPERF-SERVER>

```

#✅ 6. Success Indicators

You should see output similar to:

```
[  5]   7.00-8.00   sec  11.1 MBytes  93.3 Mbits/sec
[ ID]   0.00-10.00  sec  114 MBytes  95.7 Mbits/sec

```

This shows bandwidth shaping is in effect.


# ⚠️ 7. Common Pitfalls & Fixes

Issue		
Same hostname on nodes		
Flannel crashes post-reboot		
Calico pods missing		
No network post reboot		
Low bandwidth at 1M		


Cause
Name conflict when joining cluster
Missing br_netfilter module
Manual .conflist added incorrectly
Kernel modules not reloaded
Token bucket too small


Fix
Use hostnamectl to rename worker
Run modprobe and persist in /etc/modules-load.d/
Always use calico.yaml manifest
Use persistent sysctl and module configs
Start with 100M for testing



CILIUM_CLI_VERSION=$(curl -s https://raw.githubusercontent.com/cilium/cilium-cli/main/stable.txt)
CLI_ARCH=amd64; [ "$(uname -m)" = "aarch64" ] && CLI_ARCH=arm64
curl -L --fail --remote-name-all \
  https://github.com/cilium/cilium-cli/releases/download/${CILIUM_CLI_VERSION}/cilium-linux-${CLI_ARCH}.tar.gz{,.sha256sum}
sha256sum --check cilium-linux-${CLI_ARCH}.tar.gz.sha256sum
sudo tar xzvfC cilium-linux-${CLI_ARCH}.tar.gz /usr/local/bin
rm cilium-linux-${CLI_ARCH}.tar.gz{,.sha256sum}


# Pin to the current stable if you want reproducibility:
cilium install --version 1.18.1
# (Or just: cilium install)


cilium status --wait
# Optional, but great sanity test:
cilium connectivity test



kubectl get pods --all-namespaces -o wide
# You should see cilium-* pods in kube-system and CoreDNS running/Ready.
