# 13 - Scripts Tự Động Hoàn Chỉnh

Các scripts tự động hóa để triển khai và quản lý Kubernetes cluster.

## 1. Script cài đặt Control Plane hoàn chỉnh

```bash
#!/bin/bash
# install-control-plane.sh
# Script cài đặt Kubernetes Control Plane với Calico CNI

set -e

# Configuration
K8S_VERSION="v1.34"
POD_NETWORK_CIDR="192.168.0.0/16"
CALICO_VERSION="v3.28.0"

echo "=========================================="
echo "Kubernetes v1.34 Control Plane Installation"
echo "=========================================="
echo

# [1] Disable swap
echo "[1/10] Disabling swap..."
sudo swapoff -a
sudo sed -i '/ swap / s/^/#/' /etc/fstab
echo "✓ Swap disabled"
echo

# [2] Load kernel modules
echo "[2/10] Loading kernel modules..."
cat <<EOF | sudo tee /etc/modules-load.d/k8s.conf
overlay
br_netfilter
EOF

sudo modprobe overlay
sudo modprobe br_netfilter
echo "✓ Kernel modules loaded"
echo

# [3] Configure sysctl
echo "[3/10] Configuring sysctl..."
cat <<EOF | sudo tee /etc/sysctl.d/k8s.conf
net.ipv4.ip_forward = 1
EOF

sudo sysctl --system > /dev/null
echo "✓ Sysctl configured"
echo

# [4] Install containerd
echo "[4/10] Installing containerd..."
sudo apt-get update > /dev/null
sudo apt-get install -y apt-transport-https ca-certificates curl gnupg lsb-release > /dev/null

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

# [5] Configure containerd
echo "[5/10] Configuring containerd..."
sudo mkdir -p /etc/containerd
sudo containerd config default | sudo tee /etc/containerd/config.toml > /dev/null
sudo sed -i 's/SystemdCgroup = false/SystemdCgroup = true/' /etc/containerd/config.toml
sudo systemctl restart containerd
sudo systemctl enable containerd > /dev/null
echo "✓ Containerd configured"
echo

# [6] Install kubeadm, kubelet, kubectl
echo "[6/10] Installing kubeadm, kubelet, kubectl..."
sudo apt-get install -y apt-transport-https ca-certificates curl gpg > /dev/null
sudo mkdir -p -m 755 /etc/apt/keyrings
curl -fsSL https://pkgs.k8s.io/core:/stable:/${K8S_VERSION}/deb/Release.key | sudo gpg --dearmor -o /etc/apt/keyrings/kubernetes-apt-keyring.gpg

echo "deb [signed-by=/etc/apt/keyrings/kubernetes-apt-keyring.gpg] https://pkgs.k8s.io/core:/stable:/${K8S_VERSION}/deb/ /" | sudo tee /etc/apt/sources.list.d/kubernetes.list > /dev/null

sudo apt-get update > /dev/null
sudo apt-get install -y kubelet kubeadm kubectl > /dev/null
sudo apt-mark hold kubelet kubeadm kubectl > /dev/null
sudo systemctl enable --now kubelet > /dev/null
echo "✓ Kubernetes tools installed"
echo

# [7] Initialize control plane
echo "[7/10] Initializing control plane..."
echo "This may take a few minutes..."
sudo kubeadm init --pod-network-cidr=${POD_NETWORK_CIDR}
echo "✓ Control plane initialized"
echo

# [8] Configure kubectl
echo "[8/10] Configuring kubectl..."
mkdir -p $HOME/.kube
sudo cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
sudo chown $(id -u):$(id -g) $HOME/.kube/config
echo "✓ kubectl configured"
echo

# [9] Install Calico CNI
echo "[9/10] Installing Calico CNI..."
kubectl create -f https://raw.githubusercontent.com/projectcalico/calico/${CALICO_VERSION}/manifests/tigera-operator.yaml > /dev/null
kubectl create -f https://raw.githubusercontent.com/projectcalico/calico/${CALICO_VERSION}/manifests/custom-resources.yaml > /dev/null
echo "✓ Calico CNI installed"
echo

# [10] Install Metrics Server
echo "[10/10] Installing Metrics Server..."
kubectl apply -f https://github.com/kubernetes-sigs/metrics-server/releases/latest/download/components.yaml > /dev/null
sleep 5
kubectl patch deployment metrics-server -n kube-system --type='json' -p='[
  {
    "op": "add",
    "path": "/spec/template/spec/containers/0/args/-",
    "value": "--kubelet-insecure-tls"
  }
]' > /dev/null
echo "✓ Metrics Server installed"
echo

echo "=========================================="
echo "Installation Complete!"
echo "=========================================="
echo
echo "Cluster Status:"
kubectl get nodes
echo
echo "System Pods:"
kubectl get pods -n kube-system
echo
echo "To get worker node join command, run:"
echo "  kubeadm token create --print-join-command"
echo
```

## 2. Script cài đặt Worker Node

```bash
#!/bin/bash
# install-worker-node.sh
# Script cài đặt và chuẩn bị Worker Node

set -e

K8S_VERSION="v1.34"

echo "=========================================="
echo "Kubernetes v1.34 Worker Node Preparation"
echo "=========================================="
echo

# [1] Disable swap
echo "[1/6] Disabling swap..."
sudo swapoff -a
sudo sed -i '/ swap / s/^/#/' /etc/fstab
echo "✓ Swap disabled"
echo

# [2] Load kernel modules
echo "[2/6] Loading kernel modules..."
cat <<EOF | sudo tee /etc/modules-load.d/k8s.conf
overlay
br_netfilter
EOF

sudo modprobe overlay
sudo modprobe br_netfilter
echo "✓ Kernel modules loaded"
echo

# [3] Configure sysctl
echo "[3/6] Configuring sysctl..."
cat <<EOF | sudo tee /etc/sysctl.d/k8s.conf
net.ipv4.ip_forward = 1
EOF

sudo sysctl --system > /dev/null
echo "✓ Sysctl configured"
echo

# [4] Install containerd
echo "[4/6] Installing containerd..."
sudo apt-get update > /dev/null
sudo apt-get install -y apt-transport-https ca-certificates curl gnupg lsb-release > /dev/null

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

# [5] Configure containerd
echo "[5/6] Configuring containerd..."
sudo mkdir -p /etc/containerd
sudo containerd config default | sudo tee /etc/containerd/config.toml > /dev/null
sudo sed -i 's/SystemdCgroup = false/SystemdCgroup = true/' /etc/containerd/config.toml
sudo systemctl restart containerd
sudo systemctl enable containerd > /dev/null
echo "✓ Containerd configured"
echo

# [6] Install kubeadm, kubelet, kubectl
echo "[6/6] Installing kubeadm, kubelet, kubectl..."
sudo apt-get install -y apt-transport-https ca-certificates curl gpg > /dev/null
sudo mkdir -p -m 755 /etc/apt/keyrings
curl -fsSL https://pkgs.k8s.io/core:/stable:/${K8S_VERSION}/deb/Release.key | sudo gpg --dearmor -o /etc/apt/keyrings/kubernetes-apt-keyring.gpg

echo "deb [signed-by=/etc/apt/keyrings/kubernetes-apt-keyring.gpg] https://pkgs.k8s.io/core:/stable:/${K8S_VERSION}/deb/ /" | sudo tee /etc/apt/sources.list.d/kubernetes.list > /dev/null

sudo apt-get update > /dev/null
sudo apt-get install -y kubelet kubeadm kubectl > /dev/null
sudo apt-mark hold kubelet kubeadm kubectl > /dev/null
sudo systemctl enable --now kubelet > /dev/null
echo "✓ Kubernetes tools installed"
echo

echo "=========================================="
echo "Worker Node Ready!"
echo "=========================================="
echo
echo "Next steps:"
echo "1. Get join command from control plane:"
echo "   kubeadm token create --print-join-command"
echo
echo "2. Run the join command on this worker node:"
echo "   sudo kubeadm join <control-plane-ip>:6443 --token ... --discovery-token-ca-cert-hash sha256:..."
echo
```

## 3. Script Cluster Health Check

```bash
#!/bin/bash
# cluster-health-check.sh
# Comprehensive cluster health check

echo "=========================================="
echo "Kubernetes Cluster Health Check"
echo "=========================================="
echo

echo "=== [1] Cluster Info ==="
kubectl cluster-info
echo

echo "=== [2] Node Status ==="
kubectl get nodes -o wide
echo

echo "=== [3] System Pods Status ==="
kubectl get pods -n kube-system -o wide
echo

echo "=== [4] All Pods Status ==="
kubectl get pods -A
echo

echo "=== [5] Resource Usage ==="
echo "Node Resources:"
kubectl top nodes || echo "⚠ Metrics Server not available"
echo
echo "Top CPU Consuming Pods:"
kubectl top pods -A --sort-by=cpu | head -10 || echo "⚠ Metrics Server not available"
echo
echo "Top Memory Consuming Pods:"
kubectl top pods -A --sort-by=memory | head -10 || echo "⚠ Metrics Server not available"
echo

echo "=== [6] Recent Events (Last 20) ==="
kubectl get events --all-namespaces --sort-by='.lastTimestamp' | tail -20
echo

echo "=== [7] Component Health ==="
echo "API Server:"
kubectl get --raw /healthz
echo
echo "Control Plane Pods:"
kubectl get pods -n kube-system -l tier=control-plane
echo

echo "=== [8] CNI Status ==="
kubectl get pods -A | grep -E 'calico|flannel|cilium'
echo

echo "=== [9] DNS Status ==="
kubectl get pods -n kube-system -l k8s-app=kube-dns
echo

echo "=== [10] Certificate Expiration ==="
sudo kubeadm certs check-expiration 2>/dev/null || echo "⚠ Run on control plane node"
echo

echo "=========================================="
echo "Health Check Complete"
echo "=========================================="
```

## 4. Script Backup etcd

```bash
#!/bin/bash
# backup-etcd.sh
# Backup etcd data

set -e

BACKUP_DIR="/backups/etcd"
DATE=$(date +%Y%m%d_%H%M%S)
BACKUP_FILE="${BACKUP_DIR}/etcd-backup-${DATE}.db"
RETENTION_DAYS=7

echo "=========================================="
echo "etcd Backup Script"
echo "=========================================="
echo

# Create backup directory
mkdir -p ${BACKUP_DIR}

echo "[1/4] Creating etcd snapshot..."
sudo ETCDCTL_API=3 etcdctl snapshot save ${BACKUP_FILE} \
  --endpoints=https://127.0.0.1:2379 \
  --cacert=/etc/kubernetes/pki/etcd/ca.crt \
  --cert=/etc/kubernetes/pki/etcd/server.crt \
  --key=/etc/kubernetes/pki/etcd/server.key

echo "✓ Snapshot created: ${BACKUP_FILE}"
echo

echo "[2/4] Verifying snapshot..."
sudo ETCDCTL_API=3 etcdctl snapshot status ${BACKUP_FILE} -w table

echo "[3/4] Compressing backup..."
gzip ${BACKUP_FILE}
BACKUP_FILE="${BACKUP_FILE}.gz"
echo "✓ Backup compressed: ${BACKUP_FILE}"
echo

echo "[4/4] Cleaning old backups (older than ${RETENTION_DAYS} days)..."
find ${BACKUP_DIR} -name "etcd-backup-*.db.gz" -mtime +${RETENTION_DAYS} -delete
echo "✓ Old backups cleaned"
echo

echo "=========================================="
echo "Backup Complete!"
echo "=========================================="
echo "Backup file: ${BACKUP_FILE}"
echo "Size: $(du -h ${BACKUP_FILE} | cut -f1)"
echo

# Optional: Copy to remote backup server
# scp ${BACKUP_FILE} user@backup-server:/backups/
```

## 5. Script Deploy Test Application

```bash
#!/bin/bash
# deploy-test-app.sh
# Deploy a test nginx application

set -e

APP_NAME="test-nginx"
REPLICAS=3
NAMESPACE="default"

echo "=========================================="
echo "Deploy Test Application"
echo "=========================================="
echo

echo "[1/4] Creating deployment..."
kubectl create deployment ${APP_NAME} --image=nginx --replicas=${REPLICAS} -n ${NAMESPACE}
echo "✓ Deployment created"
echo

echo "[2/4] Waiting for pods to be ready..."
kubectl wait --for=condition=ready pod -l app=${APP_NAME} -n ${NAMESPACE} --timeout=60s
echo "✓ Pods ready"
echo

echo "[3/4] Exposing service..."
kubectl expose deployment ${APP_NAME} --port=80 --type=NodePort -n ${NAMESPACE}
echo "✓ Service exposed"
echo

echo "[4/4] Getting service details..."
kubectl get svc ${APP_NAME} -n ${NAMESPACE}
echo

NODE_PORT=$(kubectl get svc ${APP_NAME} -n ${NAMESPACE} -o jsonpath='{.spec.ports[0].nodePort}')
NODE_IP=$(kubectl get nodes -o jsonpath='{.items[0].status.addresses[?(@.type=="InternalIP")].address}')

echo "=========================================="
echo "Deployment Complete!"
echo "=========================================="
echo "Application: ${APP_NAME}"
echo "Replicas: ${REPLICAS}"
echo "Access URL: http://${NODE_IP}:${NODE_PORT}"
echo
echo "Test access:"
echo "  curl http://${NODE_IP}:${NODE_PORT}"
echo
echo "To delete:"
echo "  kubectl delete deployment ${APP_NAME} -n ${NAMESPACE}"
echo "  kubectl delete service ${APP_NAME} -n ${NAMESPACE}"
echo
```

## 6. Script Cleanup Cluster

```bash
#!/bin/bash
# cleanup-cluster.sh
# Clean up and reset Kubernetes cluster

set -e

echo "=========================================="
echo "Kubernetes Cluster Cleanup"
echo "=========================================="
echo
echo "⚠️  WARNING: This will completely reset the cluster!"
echo "⚠️  All pods, services, and data will be lost!"
echo
read -p "Are you sure you want to continue? (yes/no): " -r
echo

if [[ ! $REPLY =~ ^[Yy][Ee][Ss]$ ]]; then
    echo "Cleanup cancelled."
    exit 0
fi

echo "[1/5] Resetting kubeadm..."
sudo kubeadm reset -f
echo "✓ kubeadm reset"
echo

echo "[2/5] Cleaning iptables rules..."
sudo iptables -F
sudo iptables -t nat -F
sudo iptables -t mangle -F
sudo iptables -X
echo "✓ iptables cleaned"
echo

echo "[3/5] Removing CNI config..."
sudo rm -rf /etc/cni/net.d
sudo rm -rf /var/lib/cni/
echo "✓ CNI config removed"
echo

echo "[4/5] Removing kubelet data..."
sudo rm -rf /var/lib/kubelet
sudo rm -rf /etc/kubernetes
echo "✓ kubelet data removed"
echo

echo "[5/5] Restarting containerd..."
sudo systemctl restart containerd
echo "✓ containerd restarted"
echo

echo "=========================================="
echo "Cleanup Complete!"
echo "=========================================="
echo "The node is now ready for a fresh installation."
echo
```

## 7. Script Generate Join Command

```bash
#!/bin/bash
# generate-join-command.sh
# Generate fresh join command for worker nodes

echo "=========================================="
echo "Generate Join Command"
echo "=========================================="
echo

echo "Creating new token..."
JOIN_COMMAND=$(kubeadm token create --print-join-command)

echo
echo "=========================================="
echo "Worker Node Join Command"
echo "=========================================="
echo
echo "${JOIN_COMMAND}"
echo
echo "Run the above command on worker nodes to join the cluster."
echo
```

## 8. Script Monitor Cluster

```bash
#!/bin/bash
# monitor-cluster.sh
# Real-time cluster monitoring

watch -n 2 '
echo "=== Nodes ==="
kubectl get nodes

echo ""
echo "=== Pods (kube-system) ==="
kubectl get pods -n kube-system

echo ""
echo "=== Resource Usage ==="
kubectl top nodes 2>/dev/null || echo "Metrics Server not available"

echo ""
echo "=== Recent Events ==="
kubectl get events --all-namespaces --sort-by=.lastTimestamp | tail -5
'
```

## Hướng dẫn sử dụng Scripts

### 1. Cấp quyền thực thi

```bash
chmod +x *.sh
```

### 2. Cài đặt Control Plane

```bash
sudo ./install-control-plane.sh
```

### 3. Cài đặt Worker Node

```bash
# Trên worker node
sudo ./install-worker-node.sh

# Sau đó lấy join command từ control plane và chạy
```

### 4. Health Check

```bash
./cluster-health-check.sh
```

### 5. Backup

```bash
# Chạy manual
sudo ./backup-etcd.sh

# Hoặc setup cron job
sudo crontab -e
# Thêm dòng:
0 2 * * * /path/to/backup-etcd.sh
```

## Tạo Script Repository

```bash
# Tạo thư mục cho scripts
mkdir -p ~/k8s-scripts
cd ~/k8s-scripts

# Copy tất cả scripts vào đây
# Commit vào git để version control

git init
git add *.sh
git commit -m "Initial commit: Kubernetes management scripts"
```

## Checklist Scripts

- [ ] install-control-plane.sh - Cài đặt control plane
- [ ] install-worker-node.sh - Chuẩn bị worker node
- [ ] cluster-health-check.sh - Kiểm tra cluster health
- [ ] backup-etcd.sh - Backup etcd
- [ ] deploy-test-app.sh - Deploy test application
- [ ] cleanup-cluster.sh - Reset cluster
- [ ] generate-join-command.sh - Tạo join command
- [ ] monitor-cluster.sh - Monitor realtime

## Kết luận

Các scripts này giúp tự động hóa và đơn giản hóa việc quản lý Kubernetes cluster. Customize theo nhu cầu cụ thể của bạn.

**Nguồn tài liệu**: kubernetes.io/docs
