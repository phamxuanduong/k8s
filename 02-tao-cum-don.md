# 02 - Tạo Cụm Đơn (Single Master Cluster)

Hướng dẫn này giúp bạn tạo một Kubernetes cluster với 1 control plane node (master) và có thể có nhiều worker nodes HOẶC chỉ 1 node duy nhất (all-in-one). Phù hợp cho môi trường development, testing, hoặc production nhỏ.

**Điều kiện tiên quyết**:
- ✅ Đã hoàn thành **File 01** trên tất cả nodes

---

## Bước 1: Khởi tạo Control Plane

**Thực hiện trên CONTROL PLANE NODE.**

### Khởi tạo cluster

```bash
sudo kubeadm init --pod-network-cidr=192.168.0.0/16
```

**Giải thích tham số**:
- `--pod-network-cidr=192.168.0.0/16`: Dải IP cho pods (phù hợp cho Calico CNI)
- Nếu dùng Flannel CNI, thay bằng `10.244.0.0/16`

### Các options nâng cao (Optional)

```bash
sudo kubeadm init \
  --pod-network-cidr=192.168.0.0/16 \
  --apiserver-advertise-address=<CONTROL_PLANE_IP> \
  --kubernetes-version=v1.34.0
```

**Tham số bổ sung**:
- `--apiserver-advertise-address`: IP của control plane (hữu ích nếu có nhiều network interfaces)
- `--kubernetes-version`: Chỉ định version cụ thể

### Output quan trọng

Sau khi khởi tạo thành công, bạn sẽ thấy:

```
Your Kubernetes control-plane has initialized successfully!

To start using your cluster, you need to run the following as a regular user:

  mkdir -p $HOME/.kube
  sudo cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
  sudo chown $(id -u):$(id -g) $HOME/.kube/config

You can now join any number of machines by running the following on each node as root:

  kubeadm join <control-plane-ip>:6443 --token abcdef.1234567890abcdef \
    --discovery-token-ca-cert-hash sha256:1234567890abcdef...
```

**🔴 QUAN TRỌNG**: LƯU GIỮ LỆNH JOIN - bạn sẽ cần nó để join worker nodes!

---

## Bước 2: Cấu hình kubectl

**Trên control plane node:**

```bash
mkdir -p $HOME/.kube
sudo cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
sudo chown $(id -u):$(id -g) $HOME/.kube/config
```

### Verify kubectl hoạt động

```bash
kubectl cluster-info
kubectl get nodes
```

Output:
```
NAME         STATUS     ROLES           AGE   VERSION
k8s-master   NotReady   control-plane   1m    v1.34.0
```

**Status "NotReady" là bình thường** - node sẽ chuyển sang "Ready" sau khi cài CNI.

---

## Bước 3: Cài đặt CNI Network Plugin

**Chọn MỘT trong các CNI plugins sau:**

### Option 1: Calico (Khuyến nghị cho Production)

Calico cung cấp network policy support và hiệu năng cao.

```bash
# Cài đặt Calico với Tigera Operator
kubectl create -f https://raw.githubusercontent.com/projectcalico/calico/v3.28.0/manifests/tigera-operator.yaml
kubectl create -f https://raw.githubusercontent.com/projectcalico/calico/v3.28.0/manifests/custom-resources.yaml
```

**Hoặc cài đặt trực tiếp không dùng operator:**

```bash
kubectl apply -f https://raw.githubusercontent.com/projectcalico/calico/v3.31.0/manifests/calico.yaml
```

**Verify Calico:**

```bash
# Nếu dùng operator
kubectl get pods -n calico-system

# Nếu cài trực tiếp
kubectl get pods -n kube-system | grep calico
```

### Option 2: Flannel (Đơn giản cho Dev/Test)

```bash
kubectl apply -f https://raw.githubusercontent.com/flannel-io/flannel/master/Documentation/kube-flannel.yml
```

**Lưu ý**: Khi dùng Flannel, phải khởi tạo kubeadm với `--pod-network-cidr=10.244.0.0/16`.

**Verify Flannel:**

```bash
kubectl get pods -n kube-flannel
```

### Option 3: Cilium (Advanced eBPF)

```bash
# Cài đặt Cilium CLI
CILIUM_CLI_VERSION=$(curl -s https://raw.githubusercontent.com/cilium/cilium-cli/main/stable.txt)
CLI_ARCH=amd64
curl -L --fail --remote-name-all https://github.com/cilium/cilium-cli/releases/download/${CILIUM_CLI_VERSION}/cilium-linux-${CLI_ARCH}.tar.gz{,.sha256sum}
sha256sum --check cilium-linux-${CLI_ARCH}.tar.gz.sha256sum
sudo tar xzvfC cilium-linux-${CLI_ARCH}.tar.gz /usr/local/bin
rm cilium-linux-${CLI_ARCH}.tar.gz{,.sha256sum}

# Cài đặt Cilium
cilium install --version 1.14.0
```

**Verify Cilium:**

```bash
cilium status --wait
```

### Xác minh CNI đã hoạt động

Sau khi cài CNI, kiểm tra nodes:

```bash
kubectl get nodes
```

Output:
```
NAME         STATUS   ROLES           AGE   VERSION
k8s-master   Ready    control-plane   5m    v1.34.0
```

**Status phải là Ready!**

Kiểm tra CoreDNS:

```bash
kubectl get pods -n kube-system -l k8s-app=kube-dns
```

CoreDNS pods phải ở trạng thái **Running**.

---

## Bước 4: Chọn loại cluster

Chọn MỘT trong hai options sau:

### Option A: Multi-Node Cluster (có worker nodes riêng)

Chuyển sang [Bước 4A: Join Worker Nodes](#bước-4a-join-worker-nodes-multi-node)

### Option B: Single-Node Cluster (all-in-one)

Chuyển sang [Bước 4B: Cấu hình Single-Node](#bước-4b-cấu-hình-single-node-all-in-one)

---

## Bước 4A: Join Worker Nodes (Multi-Node)

**Dành cho cluster có worker nodes riêng.**

### Lấy join command

Nếu bạn đã lưu join command từ output của `kubeadm init`, sử dụng nó.

Nếu mất, tạo lại từ control plane:

```bash
kubeadm token create --print-join-command
```

### Thực thi join command trên worker nodes

**Trên MỖI WORKER NODE**, chạy join command:

```bash
sudo kubeadm join <control-plane-ip>:6443 \
  --token <token> \
  --discovery-token-ca-cert-hash sha256:<hash>
```

### Verify worker nodes đã join

Từ control plane:

```bash
kubectl get nodes
```

Output:
```
NAME           STATUS   ROLES           AGE   VERSION
k8s-master     Ready    control-plane   10m   v1.34.0
k8s-worker-1   Ready    <none>          2m    v1.34.0
k8s-worker-2   Ready    <none>          1m    v1.34.0
```

**Tất cả nodes phải có status Ready.**

### Label worker nodes (Optional)

```bash
kubectl label node k8s-worker-1 node-role.kubernetes.io/worker=worker
kubectl label node k8s-worker-2 node-role.kubernetes.io/worker=worker
```

Sau khi label:
```
NAME           STATUS   ROLES           AGE   VERSION
k8s-master     Ready    control-plane   10m   v1.34.0
k8s-worker-1   Ready    worker          2m    v1.34.0
k8s-worker-2   Ready    worker          1m    v1.34.0
```

**Sau khi hoàn thành Option A, chuyển sang [Bước 5: Verify Cluster](#bước-5-verify-cluster)**

---

## Bước 4B: Cấu hình Single-Node (All-in-One)

**Dành cho cluster chỉ có 1 node duy nhất.**

### Tại sao cần cấu hình thêm?

Mặc định, control plane node có **taint** ngăn scheduler đặt pods lên đó:

```
node-role.kubernetes.io/control-plane:NoSchedule
```

Với single-node cluster, chúng ta cần chạy workload pods trên control plane node vì không có worker nodes.

### Kiểm tra taint hiện tại

```bash
kubectl describe node <control-plane-node-name> | grep Taint
```

Output mẫu:
```
Taints:             node-role.kubernetes.io/control-plane:NoSchedule
```

### Remove control plane taint

```bash
kubectl taint nodes --all node-role.kubernetes.io/control-plane-
```

**Giải thích**:
- `--all`: Apply cho tất cả nodes
- `node-role.kubernetes.io/control-plane-`: Dấu `-` ở cuối có nghĩa là **remove** taint này

### Verify taint đã được remove

```bash
kubectl describe node <control-plane-node-name> | grep Taint
```

Output mong đợi:
```
Taints:             <none>
```

### Tối ưu cho single-node (Optional)

Scale down các components không cần nhiều replicas:

```bash
# Giảm CoreDNS replicas
kubectl scale deployment coredns -n kube-system --replicas=1
```

### Set resource limits (Khuyến nghị)

Tạo LimitRange để tránh pods chiếm quá nhiều resources:

```bash
cat <<EOF | kubectl apply -f -
apiVersion: v1
kind: LimitRange
metadata:
  name: default-limit-range
  namespace: default
spec:
  limits:
  - default:
      cpu: 500m
      memory: 512Mi
    defaultRequest:
      cpu: 100m
      memory: 128Mi
    type: Container
EOF
```

### Hardware khuyến nghị cho Single-Node

- **CPU**: 4+ cores
- **RAM**: 8+ GB
- **Disk**: 50+ GB

Vì tất cả components (control plane + workloads) chạy trên 1 node.

### Use Cases phù hợp

- ✅ Local development
- ✅ Learning Kubernetes
- ✅ Testing applications
- ✅ CI/CD testing environments
- ❌ **KHÔNG** dùng cho production

**Sau khi hoàn thành Option B, chuyển sang [Bước 5: Verify Cluster](#bước-5-verify-cluster)**

---

## Bước 5: Verify Cluster

### Kiểm tra tất cả pods hệ thống

```bash
kubectl get pods --all-namespaces
```

Tất cả pods phải ở trạng thái **Running** hoặc **Completed**.

### Test deploy application

```bash
# Deploy nginx
kubectl create deployment nginx-test --image=nginx --replicas=3

# Verify pods
kubectl get pods -o wide

# Expose service
kubectl expose deployment nginx-test --port=80 --type=NodePort

# Get service details
kubectl get svc nginx-test
```

### Test service

```bash
# Lấy NodePort
NODE_PORT=$(kubectl get svc nginx-test -o jsonpath='{.spec.ports[0].nodePort}')

# Lấy node IP
NODE_IP=$(kubectl get nodes -o jsonpath='{.items[0].status.addresses[?(@.type=="InternalIP")].address}')

# Test
curl http://${NODE_IP}:${NODE_PORT}
```

### Test DNS

```bash
kubectl run test-dns --image=busybox --rm -it -- nslookup kubernetes.default
```

Output phải hiển thị DNS resolution thành công.

### Clean up test

```bash
kubectl delete deployment nginx-test
kubectl delete service nginx-test
```

---

## Troubleshooting

### Nodes vẫn NotReady sau khi cài CNI

```bash
# Kiểm tra CNI pods
kubectl get pods --all-namespaces | grep -E 'calico|flannel|cilium'

# Xem logs CNI pods
kubectl logs -n kube-system <cni-pod-name>

# Kiểm tra kubelet logs
sudo journalctl -u kubelet -f
```

### CoreDNS pending hoặc CrashLoopBackOff

```bash
# Xem logs
kubectl logs -n kube-system <coredns-pod>

# Kiểm tra CNI config
ls -la /etc/cni/net.d/
```

### Worker node không join được

```bash
# Kiểm tra connectivity đến control plane
ping <control-plane-ip>
telnet <control-plane-ip> 6443

# Kiểm tra token còn hạn không
kubeadm token list

# Tạo token mới nếu cần
kubeadm token create --print-join-command
```

### Token expired

Tokens hết hạn sau 24 giờ. Tạo mới:

```bash
kubeadm token create --print-join-command
```

### Single-node: Pods không schedule

```bash
# Verify taint đã remove
kubectl describe node | grep Taint

# Nếu vẫn có taint, remove lại
kubectl taint nodes --all node-role.kubernetes.io/control-plane-
```

---

## Scripts tự động

### Script tạo multi-node cluster

```bash
#!/bin/bash
# create-cluster.sh - Multi-node cluster

set -e

POD_NETWORK_CIDR="192.168.0.0/16"
CALICO_VERSION="v3.28.0"

echo "=========================================="
echo "Creating Kubernetes Cluster"
echo "=========================================="
echo

# [1] Initialize control plane
echo "[1/4] Initializing control plane..."
sudo kubeadm init --pod-network-cidr=${POD_NETWORK_CIDR}
echo "✓ Control plane initialized"
echo

# [2] Configure kubectl
echo "[2/4] Configuring kubectl..."
mkdir -p $HOME/.kube
sudo cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
sudo chown $(id -u):$(id -g) $HOME/.kube/config
echo "✓ kubectl configured"
echo

# [3] Install Calico CNI
echo "[3/4] Installing Calico CNI..."
kubectl create -f https://raw.githubusercontent.com/projectcalico/calico/${CALICO_VERSION}/manifests/tigera-operator.yaml > /dev/null
sleep 5
kubectl create -f https://raw.githubusercontent.com/projectcalico/calico/${CALICO_VERSION}/manifests/custom-resources.yaml > /dev/null
echo "✓ Calico CNI installed"
echo

# [4] Wait for node ready
echo "[4/4] Waiting for node ready..."
kubectl wait --for=condition=ready node --all --timeout=300s
echo "✓ Node ready"
echo

echo "=========================================="
echo "Cluster Created Successfully!"
echo "=========================================="
echo
echo "To join worker nodes:"
echo "  kubeadm token create --print-join-command"
echo
```

### Script tạo single-node cluster

```bash
#!/bin/bash
# create-single-node-cluster.sh

set -e

POD_NETWORK_CIDR="192.168.0.0/16"
CALICO_VERSION="v3.28.0"

echo "=========================================="
echo "Creating Single-Node Kubernetes Cluster"
echo "=========================================="
echo

# [1] Initialize control plane
echo "[1/5] Initializing control plane..."
sudo kubeadm init --pod-network-cidr=${POD_NETWORK_CIDR}
echo "✓ Control plane initialized"
echo

# [2] Configure kubectl
echo "[2/5] Configuring kubectl..."
mkdir -p $HOME/.kube
sudo cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
sudo chown $(id -u):$(id -g) $HOME/.kube/config
echo "✓ kubectl configured"
echo

# [3] Install Calico CNI
echo "[3/5] Installing Calico CNI..."
kubectl create -f https://raw.githubusercontent.com/projectcalico/calico/${CALICO_VERSION}/manifests/tigera-operator.yaml > /dev/null
sleep 5
kubectl create -f https://raw.githubusercontent.com/projectcalico/calico/${CALICO_VERSION}/manifests/custom-resources.yaml > /dev/null
echo "✓ Calico CNI installed"
echo

# [4] Wait for node ready
echo "[4/5] Waiting for node ready..."
kubectl wait --for=condition=ready node --all --timeout=300s
echo "✓ Node ready"
echo

# [5] Configure for single-node
echo "[5/5] Configuring single-node..."
kubectl taint nodes --all node-role.kubernetes.io/control-plane- || true
kubectl scale deployment coredns -n kube-system --replicas=1

cat <<EOF | kubectl apply -f - > /dev/null
apiVersion: v1
kind: LimitRange
metadata:
  name: default-limit-range
  namespace: default
spec:
  limits:
  - default:
      cpu: 500m
      memory: 512Mi
    defaultRequest:
      cpu: 100m
      memory: 128Mi
    type: Container
EOF

echo "✓ Single-node configured"
echo

echo "=========================================="
echo "Single-Node Cluster Ready!"
echo "=========================================="
echo "Test: kubectl create deployment nginx --image=nginx"
echo
```

---

## Checklist

### Multi-Node Cluster
- [ ] Control plane đã được khởi tạo
- [ ] kubectl đã được cấu hình
- [ ] CNI plugin đã cài đặt và chạy
- [ ] Control plane node Ready
- [ ] CoreDNS pods Running
- [ ] Worker nodes đã join thành công
- [ ] Tất cả nodes Ready
- [ ] Test deployment hoạt động
- [ ] DNS resolution hoạt động

### Single-Node Cluster
- [ ] Control plane đã được khởi tạo
- [ ] kubectl đã được cấu hình
- [ ] CNI plugin đã cài đặt và chạy
- [ ] Node Ready
- [ ] CoreDNS pods Running
- [ ] Taint đã được remove
- [ ] Có thể deploy pods lên control plane
- [ ] Resource limits đã được set
- [ ] Test deployment hoạt động

---

## Tiếp theo

Sau khi tạo cụm thành công:
- **File 03**: Tạo cụm HA (nếu cần production với 3 masters)
- **File 04**: Cài đặt Metrics Server (khuyến nghị)
- **File 05**: Best Practices và vận hành cluster

**Nguồn tài liệu**: kubernetes.io/docs/setup
