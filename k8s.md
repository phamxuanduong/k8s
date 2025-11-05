# Hướng dẫn cài đặt Kubernetes với kubeadm trên Ubuntu

Kubernetes phiên bản 1.34 (tháng 10/2025) mang đến khả năng orchestration container mạnh mẽ cho cả môi trường phát triển và production. Hướng dẫn này tổng hợp tài liệu chính thức từ kubernetes.io với các bước chi tiết để triển khai cluster Kubernetes trên Ubuntu bằng kubeadm, bao gồm cả cấu hình single-node và multi-node.

**Nguồn tài liệu:** Toàn bộ hướng dẫn dựa trên tài liệu chính thức từ kubernetes.io/docs/setup và các nguồn được xác thực, đảm bảo tính chính xác và cập nhật.

## Phiên bản Kubernetes mới nhất và yêu cầu hệ thống

**Kubernetes v1.34** là phiên bản ổn định mới nhất tính đến tháng 10/2025, được phát hành với chu kỳ support 14 tháng. Kubernetes hiện hỗ trợ ba phiên bản gần nhất (1.34, 1.33, 1.32) theo chính sách N-2.

### Yêu cầu phần cứng tối thiểu

Đối với **control plane node**, cần tối thiểu **2 GB RAM** (khuyến nghị 4 GB+), **2 CPUs**, và **20 GB** dung lượng đĩa. Worker nodes yêu cầu 2 GB RAM, 1+ CPUs, và 20 GB dung lượng đĩa. Các yêu cầu này đảm bảo cluster hoạt động ổn định trong cả môi trường development và production nhỏ.

### Hệ điều hành và kernel

Ubuntu **18.04, 20.04, 22.04, hoặc 24.04** (amd64) đều được hỗ trợ với Linux kernel phiên bản 3.10 trở lên. Kiểm tra phiên bản kernel hiện tại bằng lệnh `uname -r`.

### Yêu cầu về mạng

Mọi node trong cluster cần có **kết nối mạng đầy đủ** với nhau, hostname duy nhất, và MAC address khác biệt. Các cổng quan trọng cần mở bao gồm **6443** (Kubernetes API server), **2379-2380** (etcd), **10250** (kubelet API) cho control plane, và **30000-32767** (NodePort Services) cho worker nodes.

## Chuẩn bị hệ thống Ubuntu

Trước khi cài đặt Kubernetes, cần thực hiện một số cấu hình hệ thống quan trọng để đảm bảo cluster hoạt động chính xác.

### Tắt swap (bắt buộc)

Kubernetes yêu cầu swap phải được tắt hoàn toàn. Điều này đảm bảo kubelet hoạt động đúng cách và tránh các vấn đề về performance:

```bash
sudo swapoff -a
sudo sed -i '/ swap / s/^/#/' /etc/fstab
```

### Kích hoạt IP forwarding và load kernel modules

Cấu hình sysctl để enable IPv4 packet forwarding, cần thiết cho pod networking:

```bash
cat <<EOF | sudo tee /etc/sysctl.d/k8s.conf
net.ipv4.ip_forward = 1
EOF

sudo sysctl --system
sysctl net.ipv4.ip_forward
```

Load các kernel modules cần thiết cho container runtime:

```bash
cat <<EOF | sudo tee /etc/modules-load.d/k8s.conf
overlay
br_netfilter
EOF

sudo modprobe overlay
sudo modprobe br_netfilter
```

Các bước chuẩn bị này phải được thực hiện trên **tất cả các nodes** trong cluster.

## Cài đặt Container Runtime (containerd)

Kubernetes yêu cầu một container runtime tương thích CRI. **containerd** là lựa chọn được khuyến nghị cho production environments vì tính ổn định và hiệu năng cao.

### Cài đặt containerd từ Docker repository

```bash
sudo apt-get update
sudo apt-get install -y apt-transport-https ca-certificates curl gnupg lsb-release

sudo install -m 0755 -d /etc/apt/keyrings
curl -fsSL https://download.docker.com/linux/ubuntu/gpg | sudo gpg --dearmor -o /etc/apt/keyrings/docker.gpg
sudo chmod a+r /etc/apt/keyrings/docker.gpg

echo \
  "deb [arch=$(dpkg --print-architecture) signed-by=/etc/apt/keyrings/docker.gpg] https://download.docker.com/linux/ubuntu \
  $(. /etc/os-release && echo "$VERSION_CODENAME") stable" | \
  sudo tee /etc/apt/sources.list.d/docker.list > /dev/null

sudo apt-get update
sudo apt-get install -y containerd.io
```

### Cấu hình systemd cgroup driver

Cấu hình containerd để sử dụng systemd làm cgroup driver, đảm bảo tương thích với kubelet:

```bash
sudo mkdir -p /etc/containerd
sudo containerd config default | sudo tee /etc/containerd/config.toml
sudo sed -i 's/SystemdCgroup = false/SystemdCgroup = true/' /etc/containerd/config.toml
```

Đối với containerd 1.x, cần chỉnh sửa file `/etc/containerd/config.toml` và set `SystemdCgroup = true` trong section `[plugins."io.containerd.grpc.v1.cri".containerd.runtimes.runc.options]`. Đảm bảo rằng `cri` KHÔNG nằm trong danh sách `disabled_plugins`.

Khởi động lại containerd:

```bash
sudo systemctl restart containerd
sudo systemctl enable containerd
sudo systemctl status containerd
```

## Cài đặt kubeadm, kubelet và kubectl

Các công cụ cốt lõi của Kubernetes cần được cài đặt trên tất cả nodes. **kubeadm** bootstrap cluster, **kubelet** quản lý containers trên mỗi node, và **kubectl** là CLI tool để tương tác với cluster.

### Thêm Kubernetes package repository

```bash
sudo apt-get update
sudo apt-get install -y apt-transport-https ca-certificates curl gpg

sudo mkdir -p -m 755 /etc/apt/keyrings
curl -fsSL https://pkgs.k8s.io/core:/stable:/v1.34/deb/Release.key | sudo gpg --dearmor -o /etc/apt/keyrings/kubernetes-apt-keyring.gpg

echo 'deb [signed-by=/etc/apt/keyrings/kubernetes-apt-keyring.gpg] https://pkgs.k8s.io/core:/stable:/v1.34/deb/ /' | sudo tee /etc/apt/sources.list.d/kubernetes.list
```

**Lưu ý quan trọng:** Đường dẫn repository bao gồm `/v1.34/` cho phiên bản mới nhất. Để cài đặt phiên bản khác (ví dụ: 1.33, 1.32), thay thế số phiên bản tương ứng trong URL.

### Cài đặt và pin version

```bash
sudo apt-get update
sudo apt-get install -y kubelet kubeadm kubectl
sudo apt-mark hold kubelet kubeadm kubectl
```

Lệnh `apt-mark hold` ngăn chặn việc tự động upgrade các packages này, tránh các vấn đề version incompatibility.

Xác minh cài đặt:

```bash
kubeadm version
kubectl version --client
kubelet --version
```

Enable kubelet service:

```bash
sudo systemctl enable --now kubelet
```

Kubelet sẽ crashloop cho đến khi `kubeadm init` được thực thi - đây là hành vi bình thường.

## Khởi tạo cluster với kubeadm init

Bước này chỉ thực hiện trên **control plane node** (master node) để khởi tạo Kubernetes cluster.

### Khởi tạo control plane

```bash
sudo kubeadm init --pod-network-cidr=192.168.0.0/16
```

Tham số `--pod-network-cidr` chỉ định dải IP cho pods. Giá trị **192.168.0.0/16** là default cho Calico CNI. Nếu sử dụng Flannel, thay bằng **10.244.0.0/16**.

### Các options nâng cao cho production

Đối với setup có khả năng mở rộng thành High Availability sau này:

```bash
sudo kubeadm init \
  --pod-network-cidr=192.168.0.0/16 \
  --control-plane-endpoint=<LOAD_BALANCER_DNS_OR_IP> \
  --apiserver-advertise-address=<CONTROL_PLANE_IP>
```

Tham số `--control-plane-endpoint` xác định shared endpoint cho multiple control planes, còn `--apiserver-advertise-address` chỉ định IP mà API server sẽ advertise.

### Cấu hình kubectl access

Sau khi khởi tạo thành công, cấu hình kubectl để tương tác với cluster:

```bash
mkdir -p $HOME/.kube
sudo cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
sudo chown $(id -u):$(id -g) $HOME/.kube/config
```

**Lưu giữ join command** xuất hiện sau khi init thành công - bạn sẽ cần nó để join worker nodes!

## Triển khai High Availability với 3 Master Nodes

Đối với môi trường production, việc có nhiều control plane nodes (master nodes) đảm bảo cluster vẫn hoạt động khi một hoặc nhiều master nodes gặp sự cố. Phần này hướng dẫn chi tiết cách setup HA cluster với 3 master nodes.

### Kiến trúc High Availability

Có hai topology chính cho HA Kubernetes cluster:

**1. Stacked etcd topology** (Khuyến nghị cho hầu hết trường hợp):
- etcd chạy cùng trên các control plane nodes
- Đơn giản hơn, ít tài nguyên hơn
- Yêu cầu ít nhất **3 control plane nodes**

**2. External etcd topology**:
- etcd cluster riêng biệt
- Phức tạp hơn nhưng etcd có thể scale độc lập
- Yêu cầu **3+ etcd nodes** và **2+ control plane nodes**

Hướng dẫn này tập trung vào **stacked etcd topology** - phổ biến và đơn giản hơn.

### Yêu cầu cho HA Cluster

**Hardware requirements:**
- **3 control plane nodes**: Mỗi node 2+ CPUs, 4+ GB RAM, 20+ GB disk
- **3+ worker nodes**: Mỗi node 1+ CPUs, 2+ GB RAM, 20+ GB disk
- **1 load balancer**: Có thể là VM riêng hoặc dùng cloud load balancer

**Network requirements:**
- Tất cả nodes phải communicate được với nhau
- Load balancer phải có IP/DNS cố định
- Các ports cần mở giống như single master setup

### Chuẩn bị Load Balancer

Load balancer là thành phần quan trọng để distribute traffic đến các control plane nodes. Có thể sử dụng HAProxy, nginx, hoặc cloud provider load balancer.

#### Option 1: HAProxy (Khuyến nghị)

Trên một VM riêng dành cho load balancer, cài đặt HAProxy:

```bash
sudo apt-get update
sudo apt-get install -y haproxy
```

Cấu hình HAProxy (`/etc/haproxy/haproxy.cfg`):

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

**Quan trọng:** Thay `<MASTER1_IP>`, `<MASTER2_IP>`, `<MASTER3_IP>` bằng địa chỉ IP thực tế của các control plane nodes.

Restart và enable HAProxy:

```bash
sudo systemctl restart haproxy
sudo systemctl enable haproxy
sudo systemctl status haproxy
```

Verify HAProxy đang listen trên port 6443:

```bash
sudo netstat -tlnp | grep 6443
```

#### Option 2: nginx

```bash
sudo apt-get update
sudo apt-get install -y nginx

cat <<EOF | sudo tee /etc/nginx/nginx.conf
events {}

stream {
    upstream kubernetes {
        server <MASTER1_IP>:6443;
        server <MASTER2_IP>:6443;
        server <MASTER3_IP>:6443;
    }
    
    server {
        listen 6443;
        proxy_pass kubernetes;
    }
}
EOF

sudo systemctl restart nginx
sudo systemctl enable nginx
```

### Chuẩn bị các Control Plane Nodes

Trên **TẤT CẢ 3 control plane nodes**, thực hiện đầy đủ các bước:
1. Chuẩn bị hệ thống (tắt swap, load kernel modules, cấu hình sysctl)
2. Cài đặt containerd
3. Cài đặt kubeadm, kubelet, kubectl

Tham khảo các sections 2, 3, 4 trong tài liệu này.

### Khởi tạo Master Node đầu tiên

Trên **control plane node đầu tiên**, khởi tạo cluster với `--control-plane-endpoint`:

```bash
# Định nghĩa biến
LOAD_BALANCER_DNS="k8s-api.yourdomain.com"  # Hoặc dùng IP của load balancer
LOAD_BALANCER_IP="<LOAD_BALANCER_IP>"

# Khởi tạo master đầu tiên
sudo kubeadm init \
  --control-plane-endpoint="${LOAD_BALANCER_DNS}:6443" \
  --upload-certs \
  --pod-network-cidr=192.168.0.0/16
```

**Giải thích các tham số quan trọng:**
- `--control-plane-endpoint`: Địa chỉ của load balancer (DNS hoặc IP). Đây sẽ là endpoint chung cho tất cả control planes
- `--upload-certs`: Upload certificates lên cluster để các control plane nodes khác có thể tải xuống
- `--pod-network-cidr`: Dải IP cho pods (192.168.0.0/16 cho Calico)

**LƯU Ý QUAN TRỌNG:** Output của lệnh này sẽ có **2 join commands**:
1. Join command cho **control plane nodes** (có flag `--control-plane`)
2. Join command cho **worker nodes**

**LƯU GIỮ CẢ HAI COMMANDS!**

Ví dụ output:

```
You can now join any number of control-plane nodes by running the following command on each as root:

  kubeadm join k8s-api.yourdomain.com:6443 --token abc123.xyz789 \
    --discovery-token-ca-cert-hash sha256:1234567890abcdef... \
    --control-plane --certificate-key fedcba0987654321...

Then you can join any number of worker nodes by running the following on each as root:

  kubeadm join k8s-api.yourdomain.com:6443 --token abc123.xyz789 \
    --discovery-token-ca-cert-hash sha256:1234567890abcdef...
```

Cấu hình kubectl trên master đầu tiên:

```bash
mkdir -p $HOME/.kube
sudo cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
sudo chown $(id -u):$(id -g) $HOME/.kube/config
```

### Join Control Plane Nodes thứ 2 và 3

Trên **master node thứ 2**, chạy join command cho control plane (từ output của kubeadm init):

```bash
sudo kubeadm join k8s-api.yourdomain.com:6443 --token abc123.xyz789 \
  --discovery-token-ca-cert-hash sha256:1234567890abcdef... \
  --control-plane --certificate-key fedcba0987654321...
```

Sau khi join thành công, cấu hình kubectl:

```bash
mkdir -p $HOME/.kube
sudo cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
sudo chown $(id -u):$(id -g) $HOME/.kube/config
```

Lặp lại tương tự trên **master node thứ 3**.

### Xử lý khi certificate-key hết hạn

Certificate key chỉ có hiệu lực trong **2 giờ**. Nếu hết hạn, tạo mới từ master đầu tiên:

```bash
sudo kubeadm init phase upload-certs --upload-certs
```

Output sẽ có certificate key mới. Sau đó tạo token mới:

```bash
kubeadm token create --print-join-command --certificate-key <NEW_CERT_KEY>
```

### Verify các Control Plane Nodes

Từ bất kỳ control plane node nào, kiểm tra:

```bash
# Kiểm tra tất cả nodes
kubectl get nodes

# Kiểm tra etcd members (trong stacked etcd)
kubectl get pods -n kube-system | grep etcd

# Kiểm tra chi tiết etcd cluster
sudo ETCDCTL_API=3 etcdctl \
  --endpoints=https://127.0.0.1:2379 \
  --cacert=/etc/kubernetes/pki/etcd/ca.crt \
  --cert=/etc/kubernetes/pki/etcd/server.crt \
  --key=/etc/kubernetes/pki/etcd/server.key \
  member list

# Kiểm tra API server endpoints
kubectl cluster-info
```

Output mong đợi cho `kubectl get nodes`:

```
NAME       STATUS     ROLES           AGE   VERSION
master-1   NotReady   control-plane   5m    v1.34.0
master-2   NotReady   control-plane   3m    v1.34.0
master-3   NotReady   control-plane   2m    v1.34.0
```

Status **NotReady** là bình thường vì chưa cài CNI plugin.

### Cài đặt CNI trên HA Cluster

Chỉ cần chạy **một lần** từ bất kỳ control plane node nào:

```bash
# Calico
kubectl create -f https://raw.githubusercontent.com/projectcalico/calico/v3.28.0/manifests/tigera-operator.yaml
kubectl create -f https://raw.githubusercontent.com/projectcalico/calico/v3.28.0/manifests/custom-resources.yaml
```

Đợi vài phút và kiểm tra lại nodes - tất cả sẽ chuyển sang **Ready**:

```bash
kubectl get nodes
```

### Join Worker Nodes vào HA Cluster

Trên mỗi worker node, sử dụng worker join command từ output ban đầu:

```bash
sudo kubeadm join k8s-api.yourdomain.com:6443 --token abc123.xyz789 \
  --discovery-token-ca-cert-hash sha256:1234567890abcdef...
```

**Lưu ý:** Worker nodes join vào load balancer endpoint, KHÔNG phải trực tiếp vào một master node cụ thể.

### Test High Availability

Để verify HA đang hoạt động, thử các test sau:

#### Test 1: Tắt một master node

```bash
# Từ master node bất kỳ, tắt một master khác
ssh master-2
sudo shutdown now

# Từ master-1 hoặc master-3, kiểm tra cluster vẫn hoạt động
kubectl get nodes
kubectl get pods --all-namespaces
```

Cluster vẫn phải hoạt động bình thường với 2 master nodes còn lại.

#### Test 2: Deploy test application

```bash
kubectl create deployment nginx-ha-test --image=nginx --replicas=3
kubectl expose deployment nginx-ha-test --port=80 --type=NodePort
kubectl get pods -o wide
kubectl get svc nginx-ha-test
```

#### Test 3: Kiểm tra etcd quorum

Etcd cần ít nhất **(N/2)+1** nodes để maintain quorum. Với 3 nodes, có thể chịu được 1 node failure.

```bash
# Kiểm tra etcd health
kubectl get pods -n kube-system -l component=etcd
```

### Backup và Restore cho HA Cluster

#### Backup etcd

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

Copy backup file ra ngoài cluster để lưu trữ an toàn:

```bash
scp /tmp/etcd-backup.db user@backup-server:/backups/
```

#### Restore etcd (Disaster Recovery)

Nếu cần restore, phải thực hiện trên **TẤT CẢ control plane nodes**:

```bash
# Stop kubelet và etcd
sudo systemctl stop kubelet
sudo mv /var/lib/etcd /var/lib/etcd.backup

# Restore snapshot
sudo ETCDCTL_API=3 etcdctl snapshot restore /tmp/etcd-backup.db \
  --data-dir=/var/lib/etcd

# Restart
sudo systemctl start kubelet
```

### Troubleshooting HA Cluster

#### Master node không join được

**Lỗi:** "error execution phase preflight: [preflight] Some fatal errors occurred: [ERROR FileAvailable--etc-kubernetes-manifests-kube-apiserver.yaml]"

**Giải pháp:** Reset node và thử lại:
```bash
sudo kubeadm reset -f
sudo rm -rf /etc/kubernetes/
# Chạy lại join command
```

#### etcd unhealthy

Kiểm tra etcd logs:
```bash
kubectl logs -n kube-system etcd-<master-name>
```

Kiểm tra certificates còn hạn:
```bash
sudo kubeadm certs check-expiration
```

#### Load balancer không hoạt động

Test kết nối đến load balancer:
```bash
curl -k https://<LOAD_BALANCER_IP>:6443/version
```

Kiểm tra HAProxy logs:
```bash
sudo tail -f /var/log/haproxy.log
```

### Best Practices cho HA Cluster

1. **Backup thường xuyên:** Tự động backup etcd hàng ngày
2. **Monitoring:** Setup Prometheus + Grafana để monitor control plane health
3. **Certificate rotation:** Kubernetes certificates expire sau 1 năm - set up auto-renewal
4. **Load balancer redundancy:** Consider multiple load balancers với keepalived cho true HA
5. **Odd numbers:** Luôn dùng số lẻ control plane nodes (3, 5, 7) để maintain quorum
6. **Network policies:** Implement strict network policies giữa control plane và worker nodes
7. **Updates:** Upgrade control plane nodes từng cái một, đảm bảo quorum luôn được maintain

### Script tự động cho HA Setup

#### Script cho Master Node đầu tiên

```bash
#!/bin/bash
# HA Kubernetes - First Control Plane Node

set -e

LOAD_BALANCER_ENDPOINT="k8s-api.yourdomain.com:6443"
POD_NETWORK_CIDR="192.168.0.0/16"

# [Include all system preparation steps from previous scripts]
# ...

# Initialize first control plane
echo "Initializing first control plane node..."
sudo kubeadm init \
  --control-plane-endpoint="${LOAD_BALANCER_ENDPOINT}" \
  --upload-certs \
  --pod-network-cidr="${POD_NETWORK_CIDR}"

# Configure kubectl
mkdir -p $HOME/.kube
sudo cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
sudo chown $(id -u):$(id -g) $HOME/.kube/config

echo "First control plane initialized!"
echo "SAVE THE JOIN COMMANDS FROM ABOVE!"
```

#### Script cho Control Plane Nodes tiếp theo

```bash
#!/bin/bash
# HA Kubernetes - Additional Control Plane Nodes

set -e

# [Include all system preparation steps]
# ...

echo "Ready to join as control plane node"
echo "Run the 'kubeadm join' command with --control-plane flag from the first master output"
```

## Cài đặt CNI Network Plugin

Container Network Interface (CNI) plugin là thành phần thiết yếu cho pod-to-pod communication. CoreDNS sẽ không start cho đến khi CNI được cài đặt.

### Calico - lựa chọn cho production

**Calico** cung cấp network policy support, BGP routing, và encryption capabilities. Đây là lựa chọn phù hợp cho môi trường cần security và performance cao:

```bash
kubectl create -f https://raw.githubusercontent.com/projectcalico/calico/v3.28.0/manifests/tigera-operator.yaml
kubectl create -f https://raw.githubusercontent.com/projectcalico/calico/v3.28.0/manifests/custom-resources.yaml
```

Phương pháp cài đặt trực tiếp không dùng operator:

```bash
kubectl apply -f https://raw.githubusercontent.com/projectcalico/calico/v3.31.0/manifests/calico.yaml
```

Verify Calico đang chạy:

```bash
kubectl get pods -n calico-system
```

### Flannel - đơn giản và ổn định

**Flannel** cung cấp simple overlay network, dễ setup nhưng không hỗ trợ network policies:

```bash
kubectl apply -f https://raw.githubusercontent.com/flannel-io/flannel/master/Documentation/kube-flannel.yml
```

**Quan trọng:** Khi dùng Flannel, phải khởi tạo kubeadm với `--pod-network-cidr=10.244.0.0/16`.

### Cilium - advanced eBPF-based networking

**Cilium** sử dụng eBPF technology cho high-performance networking và observability:

```bash
CILIUM_CLI_VERSION=$(curl -s https://raw.githubusercontent.com/cilium/cilium-cli/main/stable.txt)
CLI_ARCH=amd64
curl -L --fail --remote-name-all https://github.com/cilium/cilium-cli/releases/download/${CILIUM_CLI_VERSION}/cilium-linux-${CLI_ARCH}.tar.gz{,.sha256sum}
sha256sum --check cilium-linux-${CLI_ARCH}.tar.gz.sha256sum
sudo tar xzvfC cilium-linux-${CLI_ARCH}.tar.gz /usr/local/bin

cilium install --version 1.14.0
```

### Xác minh CNI đã hoạt động

```bash
kubectl get pods --all-namespaces
kubectl get nodes
kubectl get pods -n kube-system -l k8s-app=kube-dns
```

Node status phải là **Ready** và CoreDNS pods phải ở trạng thái **Running**.

## Cài đặt Metrics Server

Metrics Server là thành phần quan trọng thu thập resource metrics từ Kubelets và expose chúng trong Kubernetes API server thông qua Metrics API. Đây là yêu cầu bắt buộc để:
- Sử dụng các lệnh `kubectl top nodes` và `kubectl top pods`
- Horizontal Pod Autoscaler (HPA) hoạt động
- Vertical Pod Autoscaler (VPA) hoạt động
- Kubernetes Dashboard hiển thị metrics

### Cài đặt cho multi-node cluster (High Availability)

Đối với production cluster có nhiều nodes, sử dụng manifest HA với multiple replicas:

```bash
kubectl apply -f https://github.com/kubernetes-sigs/metrics-server/releases/latest/download/high-availability-1.21+.yaml
```

### Patch cho môi trường development/test

Trong môi trường development hoặc test không có proper TLS certificates, cần patch metrics-server để skip TLS verification:

```bash
kubectl patch deployment metrics-server -n kube-system --type='json' -p='[
  {
    "op": "add",
    "path": "/spec/template/spec/containers/0/args/-",
    "value": "--kubelet-insecure-tls"
  }
]'
```

**Lưu ý:** Flag `--kubelet-insecure-tls` chỉ nên sử dụng trong môi trường dev/test. Trong production, nên cấu hình proper TLS certificates cho kubelet.

### Điều chỉnh cho single-node hoặc cluster nhỏ

Nếu cluster chỉ có 1 worker node hoặc là single-node cluster, cần giảm số replicas của metrics-server xuống 1:

```bash
kubectl scale deployment metrics-server -n kube-system --replicas=1
```

Hoặc edit deployment trực tiếp:

```bash
kubectl edit deployment metrics-server -n kube-system
```

Sau đó thay đổi `replicas: 2` thành `replicas: 1`.

### Cài đặt phiên bản standard (single replica)

Nếu muốn cài đặt trực tiếp với 1 replica từ đầu:

```bash
kubectl apply -f https://github.com/kubernetes-sigs/metrics-server/releases/latest/download/components.yaml

# Patch để skip TLS verification (dev/test only)
kubectl patch deployment metrics-server -n kube-system --type='json' -p='[
  {
    "op": "add",
    "path": "/spec/template/spec/containers/0/args/-",
    "value": "--kubelet-insecure-tls"
  }
]'
```

### Verify metrics-server đang hoạt động

Kiểm tra pods của metrics-server:

```bash
kubectl get deployment metrics-server -n kube-system
kubectl get pods -n kube-system -l k8s-app=metrics-server
```

Đợi cho đến khi pods ở trạng thái **Running** và **Ready**, sau đó test metrics:

```bash
# Xem resource usage của nodes
kubectl top nodes

# Xem resource usage của pods
kubectl top pods --all-namespaces
```

Output mẫu:
```
NAME              CPU(cores)   CPU%   MEMORY(bytes)   MEMORY%
control-plane     156m         7%     1234Mi          32%
worker-node-1     89m          4%     876Mi           22%
```

### Troubleshooting metrics-server

Nếu metrics-server không hoạt động, kiểm tra logs:

```bash
kubectl logs -n kube-system deployment/metrics-server
```

Các lỗi thường gặp:

**"unable to fully scrape metrics"**: Đảm bảo kubelet port 10250 được mở và accessible.

**"x509: certificate signed by unknown authority"**: Cần thêm flag `--kubelet-insecure-tls` như hướng dẫn bên trên (chỉ cho dev/test).

**"no metrics available"**: Đợi vài phút để metrics-server thu thập dữ liệu lần đầu.

## Join worker nodes vào cluster (multi-node)

Để mở rộng cluster với worker nodes, thực hiện các bước sau trên mỗi worker node.

### Chuẩn bị worker nodes

Trên mỗi worker node, hoàn thành đầy đủ các bước:
- Chuẩn bị hệ thống (section 2)
- Cài đặt containerd (section 3)
- Cài đặt kubeadm, kubelet, kubectl (section 4)

### Thực thi join command

Chạy join command thu được từ output của `kubeadm init`:

```bash
sudo kubeadm join <control-plane-host>:6443 \
  --token <token> \
  --discovery-token-ca-cert-hash sha256:<hash>
```

### Nếu mất join command

Tạo token mới từ control plane node:

```bash
kubeadm token create --print-join-command
```

Hoặc tạo thủ công:

```bash
kubeadm token create
openssl x509 -pubkey -in /etc/kubernetes/pki/ca.crt | openssl rsa -pubin -outform der 2>/dev/null | \
   openssl dgst -sha256 -hex | sed 's/^.* //'
```

### Verify worker nodes đã join

Từ control plane, kiểm tra:

```bash
kubectl get nodes
```

Tất cả nodes phải có status **Ready**. Label worker nodes (optional):

```bash
kubectl label node <node-name> node-role.kubernetes.io/worker=worker
```

## Cấu hình single-node cluster

Đối với môi trường development/testing muốn chạy pods trên control plane node, cần remove control plane taint:

```bash
kubectl taint nodes --all node-role.kubernetes.io/control-plane-
```

Command này cho phép scheduler đặt pods lên control plane node, biến nó thành single-node cluster có khả năng chạy workloads.

## Lệnh kiểm tra và verify cluster

Một tập hợp lệnh quan trọng để verify cluster health và troubleshoot issues.

### Kiểm tra cluster health

```bash
# Thông tin cluster
kubectl cluster-info

# Status các nodes
kubectl get nodes -o wide

# Tất cả pods hệ thống
kubectl get pods --all-namespaces

# API server health
kubectl get --raw /healthz

# Component status
kubectl get componentstatuses
```

### Kiểm tra node-level services

```bash
# Kubelet status và logs
sudo systemctl status kubelet
sudo journalctl -u kubelet -f

# Containerd status và logs
sudo systemctl status containerd
sudo journalctl -u containerd -f
```

### Test pod và network

```bash
# Tạo test pod
kubectl run test-nginx --image=nginx
kubectl get pods

# Test DNS resolution
kubectl run test-dns --image=busybox --rm -it -- nslookup kubernetes.default

# Test inter-pod communication
kubectl run test-pod1 --image=nginx
kubectl run test-pod2 --image=busybox --rm -it -- wget -O- <pod1-ip>
```

### Troubleshooting commands

```bash
# Xem chi tiết node
kubectl describe node <node-name>

# Xem events
kubectl get events --all-namespaces --sort-by='.lastTimestamp'

# Kiểm tra CNI config
ls -la /etc/cni/net.d/

# Xem logs của pod
kubectl logs <pod-name> -n <namespace>

# Execute trong pod
kubectl exec -it <pod-name> -- /bin/bash
```

### Reset cluster khi cần

```bash
sudo kubeadm reset
sudo iptables -F && sudo iptables -t nat -F && sudo iptables -t mangle -F && sudo iptables -X
sudo rm -rf /etc/cni/net.d
sudo rm -rf /var/lib/cni/
sudo rm -rf /var/lib/kubelet
```

## Best practices và lưu ý quan trọng

### Security considerations

**KHÔNG chia sẻ** file `/etc/kubernetes/admin.conf` - nó chứa cluster-admin privileges. Giữ tokens an toàn vì bất kỳ ai có token đều có thể join nodes vào cluster. Backup etcd data thường xuyên (lưu tại `/var/lib/etcd`). Implement network policies bằng Calico hoặc Cilium để tăng cường security.

### Version management và upgrades

Không tự động upgrade Kubernetes packages (đã được pin bằng `apt-mark hold`). Luôn test upgrades trong môi trường non-production trước. Thứ tự upgrade đúng: control plane → worker nodes. Version skew policy cho phép kubelet chậm hơn API server tối đa 3 minor versions.

### Production deployment considerations

Đối với production, sử dụng **multiple control plane nodes** cho High Availability. Cần load balancer cho HA control plane setup. Với large deployments, cân nhắc external etcd cluster riêng biệt. Cài đặt monitoring stack (Prometheus, Grafana) và implement backup strategy với etcdctl hoặc Velero.

### Các vấn đề thường gặp

**Pods stuck Pending**: Kiểm tra CNI installation và network configuration. **CoreDNS CrashLoopBackOff**: Verify CNI đã cài đúng, kiểm tra SELinux issues. **Node NotReady**: Xem kubelet logs, verify container runtime, check network connectivity. **Token expired**: Tokens hết hạn sau 24 giờ, tạo mới bằng `kubeadm token create`.

## Scripts cài đặt tự động hoàn chỉnh

### Script cho Control Plane Node

```bash
#!/bin/bash
# Kubernetes v1.34 Control Plane Installation Script

set -e

# Disable swap
sudo swapoff -a
sudo sed -i '/ swap / s/^/#/' /etc/fstab

# Load kernel modules
cat <<EOF | sudo tee /etc/modules-load.d/k8s.conf
overlay
br_netfilter
EOF

sudo modprobe overlay
sudo modprobe br_netfilter

# Configure sysctl
cat <<EOF | sudo tee /etc/sysctl.d/k8s.conf
net.ipv4.ip_forward = 1
EOF

sudo sysctl --system

# Install containerd
sudo apt-get update
sudo apt-get install -y apt-transport-https ca-certificates curl gnupg lsb-release

sudo install -m 0755 -d /etc/apt/keyrings
curl -fsSL https://download.docker.com/linux/ubuntu/gpg | sudo gpg --dearmor -o /etc/apt/keyrings/docker.gpg
sudo chmod a+r /etc/apt/keyrings/docker.gpg

echo \
  "deb [arch=$(dpkg --print-architecture) signed-by=/etc/apt/keyrings/docker.gpg] https://download.docker.com/linux/ubuntu \
  $(. /etc/os-release && echo "$VERSION_CODENAME") stable" | \
  sudo tee /etc/apt/sources.list.d/docker.list > /dev/null

sudo apt-get update
sudo apt-get install -y containerd.io

# Configure containerd
sudo mkdir -p /etc/containerd
sudo containerd config default | sudo tee /etc/containerd/config.toml
sudo sed -i 's/SystemdCgroup = false/SystemdCgroup = true/' /etc/containerd/config.toml
sudo systemctl restart containerd
sudo systemctl enable containerd

# Install kubeadm, kubelet, kubectl
sudo apt-get install -y apt-transport-https ca-certificates curl gpg
sudo mkdir -p -m 755 /etc/apt/keyrings
curl -fsSL https://pkgs.k8s.io/core:/stable:/v1.34/deb/Release.key | sudo gpg --dearmor -o /etc/apt/keyrings/kubernetes-apt-keyring.gpg

echo 'deb [signed-by=/etc/apt/keyrings/kubernetes-apt-keyring.gpg] https://pkgs.k8s.io/core:/stable:/v1.34/deb/ /' | sudo tee /etc/apt/sources.list.d/kubernetes.list

sudo apt-get update
sudo apt-get install -y kubelet kubeadm kubectl
sudo apt-mark hold kubelet kubeadm kubectl
sudo systemctl enable --now kubelet

# Initialize control plane (with Calico)
sudo kubeadm init --pod-network-cidr=192.168.0.0/16

# Configure kubectl
mkdir -p $HOME/.kube
sudo cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
sudo chown $(id -u):$(id -g) $HOME/.kube/config

# Install Calico CNI
kubectl create -f https://raw.githubusercontent.com/projectcalico/calico/v3.28.0/manifests/tigera-operator.yaml
kubectl create -f https://raw.githubusercontent.com/projectcalico/calico/v3.28.0/manifests/custom-resources.yaml

# Install Metrics Server (single replica for small cluster)
kubectl apply -f https://github.com/kubernetes-sigs/metrics-server/releases/latest/download/components.yaml
sleep 10
kubectl patch deployment metrics-server -n kube-system --type='json' -p='[
  {
    "op": "add",
    "path": "/spec/template/spec/containers/0/args/-",
    "value": "--kubelet-insecure-tls"
  }
]'

echo "Control plane initialized! Use 'kubeadm token create --print-join-command' to get worker join command"
echo "Wait a few minutes for all pods to be ready, then run 'kubectl top nodes' to verify metrics-server"
```

### Script cho Worker Nodes

```bash
#!/bin/bash
# Kubernetes v1.34 Worker Node Installation Script

set -e

# Disable swap
sudo swapoff -a
sudo sed -i '/ swap / s/^/#/' /etc/fstab

# Load kernel modules
cat <<EOF | sudo tee /etc/modules-load.d/k8s.conf
overlay
br_netfilter
EOF

sudo modprobe overlay
sudo modprobe br_netfilter

# Configure sysctl
cat <<EOF | sudo tee /etc/sysctl.d/k8s.conf
net.ipv4.ip_forward = 1
EOF

sudo sysctl --system

# Install containerd
sudo apt-get update
sudo apt-get install -y apt-transport-https ca-certificates curl gnupg lsb-release

sudo install -m 0755 -d /etc/apt/keyrings
curl -fsSL https://download.docker.com/linux/ubuntu/gpg | sudo gpg --dearmor -o /etc/apt/keyrings/docker.gpg
sudo chmod a+r /etc/apt/keyrings/docker.gpg

echo \
  "deb [arch=$(dpkg --print-architecture) signed-by=/etc/apt/keyrings/docker.gpg] https://download.docker.com/linux/ubuntu \
  $(. /etc/os-release && echo "$VERSION_CODENAME") stable" | \
  sudo tee /etc/apt/sources.list.d/docker.list > /dev/null

sudo apt-get update
sudo apt-get install -y containerd.io

# Configure containerd
sudo mkdir -p /etc/containerd
sudo containerd config default | sudo tee /etc/containerd/config.toml
sudo sed -i 's/SystemdCgroup = false/SystemdCgroup = true/' /etc/containerd/config.toml
sudo systemctl restart containerd
sudo systemctl enable containerd

# Install kubeadm, kubelet, kubectl
sudo apt-get install -y apt-transport-https ca-certificates curl gpg
sudo mkdir -p -m 755 /etc/apt/keyrings
curl -fsSL https://pkgs.k8s.io/core:/stable:/v1.34/deb/Release.key | sudo gpg --dearmor -o /etc/apt/keyrings/kubernetes-apt-keyring.gpg

echo 'deb [signed-by=/etc/apt/keyrings/kubernetes-apt-keyring.gpg] https://pkgs.k8s.io/core:/stable:/v1.34/deb/ /' | sudo tee /etc/apt/sources.list.d/kubernetes.list

sudo apt-get update
sudo apt-get install -y kubelet kubeadm kubectl
sudo apt-mark hold kubelet kubeadm kubectl
sudo systemctl enable --now kubelet

echo "Worker node ready! Run the kubeadm join command from your control plane"
```

## Tài liệu tham khảo chính thức

**Kubernetes Official Documentation:**
- Installation Guide: https://kubernetes.io/docs/setup/production-environment/tools/kubeadm/install-kubeadm/
- Creating Cluster: https://kubernetes.io/docs/setup/production-environment/tools/kubeadm/create-cluster-kubeadm/
- Container Runtimes: https://kubernetes.io/docs/setup/production-environment/container-runtimes/
- kubeadm init Reference: https://kubernetes.io/docs/reference/setup-tools/kubeadm/kubeadm-init/
- Kubernetes Releases: https://kubernetes.io/releases/

**CNI Plugin Documentation:**
- Calico: https://docs.tigera.io/calico/latest/getting-started/kubernetes/
- Network Add-ons: https://kubernetes.io/docs/concepts/cluster-administration/addons/#networking-and-network-policy

**Metrics Server:**
- Metrics Server GitHub: https://github.com/kubernetes-sigs/metrics-server
- Metrics Server Documentation: https://kubernetes-sigs.github.io/metrics-server/

**High Availability:**
- HA Clusters: https://kubernetes.io/docs/setup/production-environment/tools/kubeadm/high-availability/

## Kết luận

Hướng dẫn này cung cấp đầy đủ các bước để triển khai Kubernetes cluster production-ready trên Ubuntu bằng kubeadm. Từ việc chuẩn bị hệ thống, cài đặt container runtime, bootstrap cluster với kubeadm, cấu hình network plugins, cài đặt metrics server, đến việc join worker nodes và verify cluster health - tất cả đều được trình bày chi tiết với commands cụ thể và best practices.

Với tài liệu từ kubernetes.io và các scripts tự động hóa được cung cấp, bạn có thể nhanh chóng triển khai cả single-node cluster cho development hoặc multi-node cluster cho production. Các lệnh troubleshooting và verification commands giúp đảm bảo cluster hoạt động ổn định và dễ dàng xử lý issues khi phát sinh.

Metrics Server đã được tích hợp vào hướng dẫn, cho phép bạn monitor resource usage và enable autoscaling capabilities ngay từ đầu, giúp cluster sẵn sàng cho các workloads thực tế.
