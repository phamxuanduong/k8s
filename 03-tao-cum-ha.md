# 03 - Tạo Cụm HA (High Availability Cluster)

Hướng dẫn triển khai Kubernetes cluster với 3 control plane nodes (master nodes) và load balancer để đảm bảo cluster vẫn hoạt động khi một hoặc nhiều master nodes gặp sự cố. Phù hợp cho môi trường production.

**Điều kiện tiên quyết**:
- ✅ Đã hoàn thành **File 01** trên tất cả nodes (3 master nodes + worker nodes + 1 load balancer)

---

## Kiến trúc HA Cluster

### Stacked etcd topology (Hướng dẫn này)
- etcd chạy cùng trên các control plane nodes
- Đơn giản hơn, ít tài nguyên hơn
- Yêu cầu ít nhất **3 control plane nodes**
- 1 load balancer để distribute traffic

### Yêu cầu

#### Hardware
- **3 Control Plane Nodes**: Mỗi node 2+ CPUs, 4+ GB RAM, 20+ GB disk
- **3+ Worker Nodes**: Mỗi node 1+ CPUs, 2+ GB RAM, 20+ GB disk
- **1 Load Balancer**: 1 CPU, 1 GB RAM (VM riêng hoặc cloud LB)

#### Network
- Tất cả nodes communicate được với nhau
- Load balancer có IP/DNS cố định
- Các ports giống như single master setup

---

## Bước 1: Chuẩn bị Load Balancer

### Cài đặt HAProxy

Trên một VM riêng dành cho load balancer:

```bash
sudo apt-get update
sudo apt-get install -y haproxy
```

### Cấu hình HAProxy

```bash
cat <<EOF | sudo tee /etc/haproxy/haproxy.cfg
global
    log /dev/log local0
    log /dev/log local1 notice
    chroot /var/lib/haproxy
    stats socket /run/haproxy/admin.sock mode 660 level admin
    stats timeout 30s
    user haproxy
    group haproxy
    daemon

defaults
    log     global
    mode    tcp
    option  tcplog
    option  dontlognull
    timeout connect 5000
    timeout client  50000
    timeout server  50000

frontend kubernetes-apiserver
    bind *:6443
    mode tcp
    option tcplog
    default_backend kubernetes-controlplane

backend kubernetes-controlplane
    mode tcp
    option tcp-check
    balance roundrobin
    # Thay <MASTER1_IP>, <MASTER2_IP>, <MASTER3_IP> bằng IP thực tế
    server master1 <MASTER1_IP>:6443 check fall 3 rise 2
    server master2 <MASTER2_IP>:6443 check fall 3 rise 2
    server master3 <MASTER3_IP>:6443 check fall 3 rise 2
EOF
```

**🔴 QUAN TRỌNG**: Thay `<MASTER1_IP>`, `<MASTER2_IP>`, `<MASTER3_IP>` bằng địa chỉ IP thực tế của các control plane nodes.

### Khởi động HAProxy

```bash
sudo systemctl restart haproxy
sudo systemctl enable haproxy
sudo systemctl status haproxy
```

### Verify HAProxy

```bash
# Kiểm tra HAProxy đang listen trên port 6443
sudo netstat -tlnp | grep 6443

# Kiểm tra logs
sudo tail -f /var/log/haproxy.log
```

### Lưu ý về DNS (Optional)

Nếu có DNS server, tạo DNS record:
```
k8s-api.yourdomain.com -> <LOAD_BALANCER_IP>
```

Nếu không, sử dụng trực tiếp IP của load balancer.

---

## Bước 2: Khởi tạo Master Node đầu tiên

**Thực hiện trên MASTER NODE 1.**

### Khởi tạo với control-plane-endpoint

```bash
# Định nghĩa endpoint của load balancer
LOAD_BALANCER_ENDPOINT="<LOAD_BALANCER_IP>:6443"
# Hoặc nếu có DNS: LOAD_BALANCER_ENDPOINT="k8s-api.yourdomain.com:6443"

POD_NETWORK_CIDR="192.168.0.0/16"

# Khởi tạo master đầu tiên
sudo kubeadm init \
  --control-plane-endpoint="${LOAD_BALANCER_ENDPOINT}" \
  --upload-certs \
  --pod-network-cidr="${POD_NETWORK_CIDR}"
```

**Giải thích tham số**:
- `--control-plane-endpoint`: Địa chỉ load balancer (DNS hoặc IP)
- `--upload-certs`: Upload certificates lên cluster để các master khác tải xuống
- `--pod-network-cidr`: Dải IP cho pods (192.168.0.0/16 cho Calico)

### Output quan trọng

Sau khi init thành công, output sẽ có **2 join commands**:

#### 1. Join command cho Control Plane Nodes

```bash
kubeadm join k8s-api.yourdomain.com:6443 --token abc123.xyz789 \
  --discovery-token-ca-cert-hash sha256:1234567890abcdef... \
  --control-plane --certificate-key fedcba0987654321...
```

**Đặc điểm**:
- Có flag `--control-plane`
- Có `--certificate-key`
- Dùng để join master nodes 2 và 3

#### 2. Join command cho Worker Nodes

```bash
kubeadm join k8s-api.yourdomain.com:6443 --token abc123.xyz789 \
  --discovery-token-ca-cert-hash sha256:1234567890abcdef...
```

**Đặc điểm**:
- Không có flag `--control-plane`
- Dùng để join worker nodes

**🔴 LƯU GIỮ CẢ HAI COMMANDS!**

### Cấu hình kubectl trên Master 1

```bash
mkdir -p $HOME/.kube
sudo cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
sudo chown $(id -u):$(id -g) $HOME/.kube/config
```

Verify:
```bash
kubectl get nodes
```

---

## Bước 3: Join Master Node thứ 2 và 3

### Join Master Node 2

**Trên MASTER NODE 2**, chạy join command cho control plane:

```bash
sudo kubeadm join <LOAD_BALANCER_ENDPOINT>:6443 --token abc123.xyz789 \
  --discovery-token-ca-cert-hash sha256:1234567890abcdef... \
  --control-plane --certificate-key fedcba0987654321...
```

Sau khi join thành công, cấu hình kubectl:

```bash
mkdir -p $HOME/.kube
sudo cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
sudo chown $(id -u):$(id -g) $HOME/.kube/config
```

### Join Master Node 3

Lặp lại tương tự trên **MASTER NODE 3**.

### Verify tất cả master nodes

Từ bất kỳ master node nào:

```bash
kubectl get nodes
```

Output:
```
NAME       STATUS     ROLES           AGE   VERSION
master-1   NotReady   control-plane   5m    v1.34.0
master-2   NotReady   control-plane   3m    v1.34.0
master-3   NotReady   control-plane   2m    v1.34.0
```

**Status NotReady là bình thường** - sẽ chuyển sang Ready sau khi cài CNI.

### Kiểm tra etcd members

```bash
kubectl get pods -n kube-system | grep etcd
```

Output:
```
etcd-master-1   1/1     Running   0   5m
etcd-master-2   1/1     Running   0   3m
etcd-master-3   1/1     Running   0   2m
```

### Xử lý khi certificate-key hết hạn

Certificate key chỉ có hiệu lực trong **2 giờ**. Nếu hết hạn, tạo mới từ master 1:

```bash
# Tạo certificate key mới
sudo kubeadm init phase upload-certs --upload-certs

# Tạo token mới với certificate key
kubeadm token create --print-join-command --certificate-key <NEW_CERT_KEY>
```

---

## Bước 4: Cài đặt CNI Network Plugin

**Chỉ cần chạy MỘT LẦN từ bất kỳ control plane node nào.**

### Calico (Khuyến nghị)

```bash
kubectl create -f https://raw.githubusercontent.com/projectcalico/calico/v3.28.0/manifests/tigera-operator.yaml
kubectl create -f https://raw.githubusercontent.com/projectcalico/calico/v3.28.0/manifests/custom-resources.yaml
```

**Hoặc cài trực tiếp:**

```bash
kubectl apply -f https://raw.githubusercontent.com/projectcalico/calico/v3.31.0/manifests/calico.yaml
```

### Flannel

```bash
kubectl apply -f https://raw.githubusercontent.com/flannel-io/flannel/master/Documentation/kube-flannel.yml
```

### Verify CNI

```bash
# Kiểm tra CNI pods
kubectl get pods -n calico-system
# Hoặc
kubectl get pods -n kube-system | grep calico
```

### Verify nodes Ready

Đợi vài phút và kiểm tra lại:

```bash
kubectl get nodes
```

Output:
```
NAME       STATUS   ROLES           AGE   VERSION
master-1   Ready    control-plane   10m   v1.34.0
master-2   Ready    control-plane   8m    v1.34.0
master-3   Ready    control-plane   6m    v1.34.0
```

**Tất cả nodes phải Ready!**

---

## Bước 5: Join Worker Nodes

**Trên MỖI WORKER NODE**, sử dụng worker join command:

```bash
sudo kubeadm join <LOAD_BALANCER_ENDPOINT>:6443 --token abc123.xyz789 \
  --discovery-token-ca-cert-hash sha256:1234567890abcdef...
```

**Lưu ý**: Worker nodes join vào load balancer endpoint, KHÔNG phải IP của một master cụ thể.

### Verify worker nodes

Từ control plane:

```bash
kubectl get nodes
```

Output:
```
NAME           STATUS   ROLES           AGE   VERSION
master-1       Ready    control-plane   15m   v1.34.0
master-2       Ready    control-plane   13m   v1.34.0
master-3       Ready    control-plane   11m   v1.34.0
k8s-worker-1   Ready    <none>          2m    v1.34.0
k8s-worker-2   Ready    <none>          1m    v1.34.0
```

### Label worker nodes (Optional)

```bash
kubectl label node k8s-worker-1 node-role.kubernetes.io/worker=worker
kubectl label node k8s-worker-2 node-role.kubernetes.io/worker=worker
```

---

## Bước 6: Verify HA Cluster

### Kiểm tra tất cả pods

```bash
kubectl get pods --all-namespaces
```

Tất cả pods phải Running.

### Test HA bằng cách tắt một master

```bash
# SSH vào master-2 và tắt
ssh master-2
sudo shutdown now

# Từ master-1 hoặc master-3, kiểm tra cluster vẫn hoạt động
kubectl get nodes
kubectl get pods --all-namespaces
```

Cluster vẫn phải hoạt động bình thường với 2 master còn lại.

### Test deploy application

```bash
kubectl create deployment nginx-ha-test --image=nginx --replicas=3
kubectl expose deployment nginx-ha-test --port=80 --type=NodePort
kubectl get pods -o wide
kubectl get svc nginx-ha-test
```

### Test etcd quorum

Etcd cần ít nhất **(N/2)+1** nodes để maintain quorum. Với 3 nodes, có thể chịu được 1 node failure.

```bash
# Kiểm tra etcd health
kubectl get pods -n kube-system -l component=etcd
```

### Clean up test

```bash
kubectl delete deployment nginx-ha-test
kubectl delete service nginx-ha-test
```

---

## Backup và Restore

### Backup etcd

Chạy trên bất kỳ control plane node nào:

```bash
sudo ETCDCTL_API=3 etcdctl snapshot save /tmp/etcd-backup.db \
  --endpoints=https://127.0.0.1:2379 \
  --cacert=/etc/kubernetes/pki/etcd/ca.crt \
  --cert=/etc/kubernetes/pki/etcd/server.crt \
  --key=/etc/kubernetes/pki/etcd/server.key

# Verify backup
sudo ETCDCTL_API=3 etcdctl snapshot status /tmp/etcd-backup.db
```

Copy backup ra ngoài:
```bash
scp /tmp/etcd-backup.db user@backup-server:/backups/
```

### Restore etcd

Nếu cần restore, thực hiện trên **TẤT CẢ control plane nodes**:

```bash
# Stop kubelet
sudo systemctl stop kubelet

# Backup current data
sudo mv /var/lib/etcd /var/lib/etcd.backup

# Restore snapshot
sudo ETCDCTL_API=3 etcdctl snapshot restore /tmp/etcd-backup.db \
  --data-dir=/var/lib/etcd

# Restart
sudo systemctl start kubelet
```

---

## Troubleshooting

### Master node không join được

```bash
# Reset và thử lại
sudo kubeadm reset -f
sudo rm -rf /etc/kubernetes/
# Chạy lại join command
```

### etcd unhealthy

```bash
# Kiểm tra logs
kubectl logs -n kube-system etcd-<master-name>

# Kiểm tra certificates
sudo kubeadm certs check-expiration
```

### Load balancer không hoạt động

```bash
# Test connectivity
curl -k https://<LOAD_BALANCER_IP>:6443/version

# Kiểm tra HAProxy logs
sudo tail -f /var/log/haproxy.log
```

---

## Script tự động tạo cụm HA

### Script cho Master Node đầu tiên

```bash
#!/bin/bash
# create-ha-master1.sh

set -e

LOAD_BALANCER_ENDPOINT="<LOAD_BALANCER_IP>:6443"
POD_NETWORK_CIDR="192.168.0.0/16"
CALICO_VERSION="v3.28.0"

echo "=========================================="
echo "HA Cluster - Initialize First Master"
echo "=========================================="
echo

# [1] Initialize first control plane
echo "[1/4] Initializing first master node..."
sudo kubeadm init \
  --control-plane-endpoint="${LOAD_BALANCER_ENDPOINT}" \
  --upload-certs \
  --pod-network-cidr="${POD_NETWORK_CIDR}"
echo "✓ First master initialized"
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
echo "First Master Initialized!"
echo "=========================================="
echo
echo "SAVE THE JOIN COMMANDS FROM ABOVE!"
echo
echo "Next steps:"
echo "1. Run join command with --control-plane on master-2 and master-3"
echo "2. Run join command without --control-plane on worker nodes"
echo
```

---

## Checklist tạo cụm HA

- [ ] Load balancer đã được cấu hình và chạy
- [ ] Master node 1 đã được khởi tạo
- [ ] Master node 2 đã join thành công
- [ ] Master node 3 đã join thành công
- [ ] CNI plugin đã được cài đặt
- [ ] Tất cả master nodes có status Ready
- [ ] etcd pods chạy trên cả 3 masters
- [ ] Worker nodes đã join thành công
- [ ] Test HA (tắt 1 master, cluster vẫn hoạt động)
- [ ] Backup etcd đã được thiết lập

---

## Best Practices cho HA

1. **Backup thường xuyên**: Tự động backup etcd hàng ngày
2. **Monitoring**: Setup Prometheus + Grafana
3. **Certificate rotation**: Auto-renew certificates
4. **Odd numbers**: Luôn dùng số lẻ masters (3, 5, 7)
5. **Load balancer redundancy**: Consider multiple LBs với keepalived
6. **Updates**: Upgrade masters từng cái một

---

## Tiếp theo

Sau khi tạo cụm HA thành công:
- **File 04**: Cài đặt Metrics Server (khuyến nghị)
- **File 06**: Kiểm tra và troubleshooting
- **File 07**: Best practices

**Nguồn tài liệu**: kubernetes.io/docs/setup
