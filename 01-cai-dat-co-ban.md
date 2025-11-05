# 01 - Cài đặt cơ bản và Chuẩn bị

Hướng dẫn này bao gồm tất cả các bước chuẩn bị cần thiết trước khi tạo Kubernetes cluster. Thực hiện các bước này trên **TẤT CẢ CÁC NODES** (cả control plane và worker nodes).

## Phần 1: Yêu cầu hệ thống

### Phiên bản Kubernetes

**Kubernetes v1.34** là phiên bản ổn định mới nhất tính đến tháng 10/2025, được phát hành với chu kỳ support 14 tháng.

### Yêu cầu phần cứng

#### Control Plane Node (Master)
- **CPU**: Tối thiểu 2 CPUs (khuyến nghị 4+ CPUs)
- **RAM**: Tối thiểu 2 GB (khuyến nghị 4 GB+)
- **Disk**: 20 GB trở lên

#### Worker Nodes
- **CPU**: Tối thiểu 1 CPU (khuyến nghị 2+ CPUs)
- **RAM**: Tối thiểu 2 GB
- **Disk**: 20 GB trở lên

### Hệ điều hành được hỗ trợ
- Ubuntu 18.04, 20.04, 22.04, hoặc 24.04 (amd64)
- Linux kernel >= 3.10

### Yêu cầu về mạng

#### Cổng (Ports) cần mở

**Control Plane Node:**
| Cổng | Mục đích |
|------|----------|
| 6443 | Kubernetes API server |
| 2379-2380 | etcd server client API |
| 10250 | Kubelet API |
| 10259 | kube-scheduler |
| 10257 | kube-controller-manager |

**Worker Nodes:**
| Cổng | Mục đích |
|------|----------|
| 10250 | Kubelet API |
| 30000-32767 | NodePort Services |

---

## Phần 2: Chuẩn bị hệ thống Ubuntu

**LƯU Ý**: Các bước sau phải được thực hiện trên **TẤT CẢ CÁC NODES**.

### Bước 1: Tắt Swap (Bắt buộc)

```bash
# Tắt swap ngay lập tức
sudo swapoff -a

# Tắt swap vĩnh viễn (sau khi reboot)
sudo sed -i '/ swap / s/^/#/' /etc/fstab

# Verify
free -h | grep Swap
```

### Bước 2: Load Kernel Modules

```bash
cat <<EOF | sudo tee /etc/modules-load.d/k8s.conf
overlay
br_netfilter
EOF

sudo modprobe overlay
sudo modprobe br_netfilter

# Verify
lsmod | grep br_netfilter
lsmod | grep overlay
```

### Bước 3: Cấu hình sysctl

```bash
cat <<EOF | sudo tee /etc/sysctl.d/k8s.conf
net.ipv4.ip_forward = 1
EOF

# Apply cấu hình
sudo sysctl --system

# Verify
sysctl net.ipv4.ip_forward
```

### Bước 4: Update hệ thống

```bash
sudo apt-get update
sudo apt-get upgrade -y
sudo apt-get install -y apt-transport-https ca-certificates curl gnupg lsb-release
```

### Bước 5: Cấu hình Hostname (Optional)

```bash
# Trên control plane node
sudo hostnamectl set-hostname k8s-master

# Trên worker node 1
sudo hostnamectl set-hostname k8s-worker-1

# Cập nhật /etc/hosts trên tất cả nodes
sudo tee -a /etc/hosts <<EOF
192.168.1.10 k8s-master
192.168.1.11 k8s-worker-1
192.168.1.12 k8s-worker-2
EOF
```

---

## Phần 3: Cài đặt Container Runtime (containerd)

### Bước 1: Thêm Docker repository

```bash
sudo install -m 0755 -d /etc/apt/keyrings
curl -fsSL https://download.docker.com/linux/ubuntu/gpg | sudo gpg --dearmor -o /etc/apt/keyrings/docker.gpg
sudo chmod a+r /etc/apt/keyrings/docker.gpg

echo \
  "deb [arch=$(dpkg --print-architecture) signed-by=/etc/apt/keyrings/docker.gpg] https://download.docker.com/linux/ubuntu \
  $(. /etc/os-release && echo "$VERSION_CODENAME") stable" | \
  sudo tee /etc/apt/sources.list.d/docker.list > /dev/null
```

### Bước 2: Cài đặt containerd

```bash
sudo apt-get update
sudo apt-get install -y containerd.io

# Verify
containerd --version
```

### Bước 3: Cấu hình containerd

```bash
# Tạo file cấu hình mặc định
sudo mkdir -p /etc/containerd
sudo containerd config default | sudo tee /etc/containerd/config.toml

# Cấu hình systemd cgroup driver
sudo sed -i 's/SystemdCgroup = false/SystemdCgroup = true/' /etc/containerd/config.toml

# Verify
grep SystemdCgroup /etc/containerd/config.toml
```

### Bước 4: Khởi động containerd

```bash
sudo systemctl enable containerd
sudo systemctl restart containerd
sudo systemctl status containerd
```

---

## Phần 4: Cài đặt kubeadm, kubelet và kubectl

### Bước 1: Thêm Kubernetes repository

```bash
sudo mkdir -p -m 755 /etc/apt/keyrings
curl -fsSL https://pkgs.k8s.io/core:/stable:/v1.34/deb/Release.key | sudo gpg --dearmor -o /etc/apt/keyrings/kubernetes-apt-keyring.gpg

echo 'deb [signed-by=/etc/apt/keyrings/kubernetes-apt-keyring.gpg] https://pkgs.k8s.io/core:/stable:/v1.34/deb/ /' | sudo tee /etc/apt/sources.list.d/kubernetes.list
```

### Bước 2: Cài đặt packages

```bash
sudo apt-get update
sudo apt-get install -y kubelet kubeadm kubectl
```

### Bước 3: Pin version

```bash
sudo apt-mark hold kubelet kubeadm kubectl
```

### Bước 4: Enable kubelet

```bash
sudo systemctl enable --now kubelet
```

### Verify cài đặt

```bash
kubeadm version
kubectl version --client
kubelet --version
```

---

## Script tự động cài đặt cơ bản

```bash
#!/bin/bash
# prepare-node.sh
# Script chuẩn bị node cho Kubernetes

set -e

K8S_VERSION="v1.34"

echo "=========================================="
echo "Kubernetes Node Preparation"
echo "=========================================="
echo

# [1] Disable swap
echo "[1/8] Disabling swap..."
sudo swapoff -a
sudo sed -i '/ swap / s/^/#/' /etc/fstab
echo "✓ Swap disabled"
echo

# [2] Load kernel modules
echo "[2/8] Loading kernel modules..."
cat <<EOF | sudo tee /etc/modules-load.d/k8s.conf
overlay
br_netfilter
EOF

sudo modprobe overlay
sudo modprobe br_netfilter
echo "✓ Kernel modules loaded"
echo

# [3] Configure sysctl
echo "[3/8] Configuring sysctl..."
cat <<EOF | sudo tee /etc/sysctl.d/k8s.conf
net.ipv4.ip_forward = 1
EOF

sudo sysctl --system > /dev/null
echo "✓ Sysctl configured"
echo

# [4] Update system
echo "[4/8] Updating system..."
sudo apt-get update > /dev/null
sudo apt-get upgrade -y > /dev/null
sudo apt-get install -y apt-transport-https ca-certificates curl gnupg lsb-release > /dev/null
echo "✓ System updated"
echo

# [5] Install containerd
echo "[5/8] Installing containerd..."
sudo install -m 0755 -d /etc/apt/keyrings
curl -fsSL https://download.docker.com/linux/ubuntu/gpg | sudo gpg --dearmor -o /etc/apt/keyrings/docker.gpg
sudo chmod a+r /etc/apt/keyrings/docker.gpg

echo \
  "deb [arch=$(dpkg --print-architecture) signed-by=/etc/apt/keyrings/docker.gpg] https://download.docker.com/linux/ubuntu \
  $(. /etc/os-release && echo "$VERSION_CODENAME") stable" | \
  sudo tee /etc/apt/sources.list.d/docker.list > /dev/null

sudo apt-get update > /dev/null
sudo apt-get install -y containerd.io > /dev/null
echo "✓ Containerd installed"
echo

# [6] Configure containerd
echo "[6/8] Configuring containerd..."
sudo mkdir -p /etc/containerd
sudo containerd config default | sudo tee /etc/containerd/config.toml > /dev/null
sudo sed -i 's/SystemdCgroup = false/SystemdCgroup = true/' /etc/containerd/config.toml
sudo systemctl restart containerd
sudo systemctl enable containerd > /dev/null
echo "✓ Containerd configured"
echo

# [7] Install kubeadm, kubelet, kubectl
echo "[7/8] Installing Kubernetes tools..."
sudo mkdir -p -m 755 /etc/apt/keyrings
curl -fsSL https://pkgs.k8s.io/core:/stable:/${K8S_VERSION}/deb/Release.key | sudo gpg --dearmor -o /etc/apt/keyrings/kubernetes-apt-keyring.gpg

echo "deb [signed-by=/etc/apt/keyrings/kubernetes-apt-keyring.gpg] https://pkgs.k8s.io/core:/stable:/${K8S_VERSION}/deb/ /" | sudo tee /etc/apt/sources.list.d/kubernetes.list > /dev/null

sudo apt-get update > /dev/null
sudo apt-get install -y kubelet kubeadm kubectl > /dev/null
sudo apt-mark hold kubelet kubeadm kubectl > /dev/null
sudo systemctl enable --now kubelet > /dev/null
echo "✓ Kubernetes tools installed"
echo

# [8] Verify installation
echo "[8/8] Verifying installation..."
echo "Containerd: $(containerd --version | awk '{print $3}')"
echo "kubeadm: $(kubeadm version -o short)"
echo "kubectl: $(kubectl version --client -o json 2>/dev/null | grep gitVersion | cut -d'"' -f4)"
echo "kubelet: $(kubelet --version | cut -d' ' -f2)"
echo

echo "=========================================="
echo "Node Preparation Complete!"
echo "=========================================="
echo
echo "Next steps:"
echo "- For control plane: Run file 02 (Single cluster) or 03 (HA cluster)"
echo "- For worker node: Wait for join command from control plane"
echo
```

Lưu script thành `prepare-node.sh` và chạy:
```bash
chmod +x prepare-node.sh
sudo ./prepare-node.sh
```

---

## Checklist chuẩn bị

Trên **MỖI NODE**, kiểm tra:

- [ ] Swap đã được tắt hoàn toàn
- [ ] Kernel modules (overlay, br_netfilter) đã được load
- [ ] IP forwarding đã được enable (net.ipv4.ip_forward = 1)
- [ ] Hệ thống đã được update
- [ ] containerd đã được cài đặt và chạy
- [ ] kubeadm, kubelet, kubectl đã được cài đặt
- [ ] Packages đã được hold (apt-mark hold)
- [ ] Hostname đã được cấu hình (optional)
- [ ] Firewall đã mở các ports cần thiết

---

## Tiếp theo

Sau khi hoàn thành cài đặt cơ bản trên tất cả nodes:

- **File 02**: Tạo cụm đơn (Single Master Cluster) - Phù hợp cho dev/test
- **File 03**: Tạo cụm HA (High Availability Cluster) - Phù hợp cho production

**Nguồn tài liệu**: kubernetes.io/docs/setup
