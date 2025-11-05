# Hướng dẫn cài đặt Kubernetes v1.34 trên Ubuntu

Tài liệu hướng dẫn chi tiết cài đặt và triển khai Kubernetes cluster trên Ubuntu với kubeadm. Tất cả hướng dẫn dựa trên tài liệu chính thức từ kubernetes.io.

## Tổng quan

Hướng dẫn này bao gồm:
- ✅ Cài đặt Kubernetes v1.34 (phiên bản mới nhất tháng 10/2025)
- ✅ Triển khai cụm đơn (Single Master) cho dev/test
- ✅ Triển khai cụm đơn node (All-in-one) cho learning
- ✅ Triển khai cụm HA (High Availability) cho production
- ✅ Cài đặt CNI plugins (Calico, Flannel, Cilium)
- ✅ Cài đặt Metrics Server
- ✅ Best practices cho production

## Cấu trúc tài liệu (5 files)

### File 01: Cài đặt cơ bản
**`01-cai-dat-co-ban.md`** (9.0 KB)

Tất cả bước chuẩn bị cần thiết. Bao gồm:
- Yêu cầu hệ thống (phần cứng, OS, network)
- Chuẩn bị hệ thống (tắt swap, load kernel modules)
- Cài đặt containerd
- Cài đặt kubeadm, kubelet, kubectl

**Thực hiện trên**: TẤT CẢ CÁC NODES

### File 02: Tạo cụm đơn
**`02-tao-cum-don.md`** (15 KB)

Triển khai cluster với 1 control plane node. **Bao gồm cả multi-node và single-node.**

Bao gồm:
- Khởi tạo control plane với kubeadm init
- Cấu hình kubectl
- Cài đặt CNI plugin (Calico/Flannel/Cilium)
- **Option A**: Join worker nodes (multi-node)
- **Option B**: Cấu hình single-node (all-in-one)
- Verify cluster

**Phù hợp cho**: Development, Testing, Learning, Small Production

### File 03: Tạo cụm HA
**`03-tao-cum-ha.md`** (14 KB)

Triển khai High Availability cluster với 3 control plane nodes + load balancer.

Bao gồm:
- Cấu hình load balancer (HAProxy)
- Khởi tạo master node đầu tiên
- Join thêm 2 master nodes
- Cài đặt CNI plugin
- Join worker nodes
- Test High Availability
- Backup và restore

**Phù hợp cho**: Production environments

### File 04: Cài đặt Metrics Server
**`04-cai-dat-metrics-server.md`** (7.5 KB)

Cài đặt Metrics Server để monitor resource usage.

Bao gồm:
- Cài đặt single replica (dev/test)
- Cài đặt HA (production)
- Patch cho môi trường dev/test
- Test kubectl top nodes/pods
- Test HPA (Horizontal Pod Autoscaler)
- Troubleshooting

**Cần thiết cho**: kubectl top, HPA, VPA

### File 05: Best Practices
**`05-best-practices.md`** (12 KB)

Best practices cho production.

Bao gồm:
- Security considerations
- Backup và disaster recovery
- Version management và upgrades
- Resource management
- Monitoring và logging
- Certificate management

---

## Quy trình triển khai

### Scenario 1: Multi-Node Cluster (1 master + workers)

```
01-cai-dat-co-ban.md (trên tất cả nodes)
    ↓
02-tao-cum-don.md → Option A (join workers)
    ↓
04-cai-dat-metrics-server.md (khuyến nghị)
    ↓
05-best-practices.md (backup, monitoring)
```

### Scenario 2: Single-Node Cluster (all-in-one)

```
01-cai-dat-co-ban.md (trên 1 node)
    ↓
02-tao-cum-don.md → Option B (single-node)
    ↓
04-cai-dat-metrics-server.md (optional)
    ↓
05-best-practices.md (backup, monitoring)
```

### Scenario 3: HA Cluster (3 masters + workers)

```
01-cai-dat-co-ban.md (trên tất cả nodes + LB)
    ↓
03-tao-cum-ha.md (setup 3 masters)
    ↓
04-cai-dat-metrics-server.md (khuyến nghị)
    ↓
05-best-practices.md (backup, monitoring)
```

---

## Yêu cầu hệ thống

### Hardware tối thiểu

**Control Plane Node:**
- CPU: 2+ cores (khuyến nghị 4+ cores)
- RAM: 2 GB (khuyến nghị 4 GB+)
- Disk: 20 GB+

**Worker Nodes:**
- CPU: 1+ cores
- RAM: 2 GB
- Disk: 20 GB+

**Single-Node Cluster:**
- CPU: 4+ cores
- RAM: 8+ GB
- Disk: 50+ GB

**HA Setup thêm:**
- 3 control plane nodes
- 1 load balancer (1 CPU, 1 GB RAM)

### Software

- Ubuntu 18.04, 20.04, 22.04, hoặc 24.04 (amd64)
- Linux kernel >= 3.10
- containerd (sẽ được cài tự động)

---

## Quick Start

### Cài đặt nhanh Single-Node Cluster

```bash
# 1. Làm theo File 01: Cài đặt cơ bản
# Chuẩn bị hệ thống, cài containerd, kubeadm, kubelet, kubectl

# 2. Làm theo File 02: Tạo cụm đơn → Option B (single-node)
# Khởi tạo cluster và remove taint

# 3. Verify
kubectl get nodes
kubectl create deployment nginx --image=nginx
```

### Cài đặt nhanh Multi-Node Cluster

```bash
# 1. Trên TẤT CẢ nodes, làm theo File 01: Cài đặt cơ bản
# Chuẩn bị hệ thống, cài containerd, kubeadm, kubelet, kubectl

# 2. Trên CONTROL PLANE, làm theo File 02: Tạo cụm đơn → Option A
# Khởi tạo cluster, cài CNI

# 3. Trên WORKER NODES, join vào cluster
# Lấy join command từ control plane:
kubeadm token create --print-join-command
# Chạy command đó trên worker nodes

# 4. Verify
kubectl get nodes
kubectl get pods --all-namespaces
```

---

## Lệnh thường dùng

```bash
# Kiểm tra nodes
kubectl get nodes

# Kiểm tra pods
kubectl get pods --all-namespaces

# Kiểm tra cluster health
kubectl cluster-info

# Xem resource usage (cần Metrics Server)
kubectl top nodes
kubectl top pods --all-namespaces

# Tạo join command mới
kubeadm token create --print-join-command
```

---

## Troubleshooting

Các vấn đề thường gặp:

### Nodes NotReady
- Kiểm tra CNI plugin đã cài chưa
- Xem kubelet logs: `sudo journalctl -u kubelet -f`

### Pods Pending
- Kiểm tra CNI: `kubectl get pods -A | grep -E 'calico|flannel'`
- Verify nodes Ready: `kubectl get nodes`

### Token expired
```bash
kubeadm token create --print-join-command
```

### Single-node: Pods không schedule
```bash
# Verify taint đã remove
kubectl describe node | grep Taint

# Remove taint nếu cần
kubectl taint nodes --all node-role.kubernetes.io/control-plane-
```

---

## Tài liệu tham khảo

- [Kubernetes Official Documentation](https://kubernetes.io/docs/)
- [kubeadm Documentation](https://kubernetes.io/docs/setup/production-environment/tools/kubeadm/)
- [Calico Documentation](https://docs.tigera.io/calico/latest/)
- [Metrics Server GitHub](https://github.com/kubernetes-sigs/metrics-server)

---

## Thông tin phiên bản

- **Kubernetes**: v1.34.0
- **Calico**: v3.28.0 / v3.31.0
- **containerd**: Latest from Docker repository
- **Ngày cập nhật**: Tháng 10/2025

---

## Hỗ trợ

Nếu gặp vấn đề:
1. Kiểm tra logs: `sudo journalctl -u kubelet -f`
2. Verify cấu hình: `kubectl get pods -A`
3. Xem phần Troubleshooting phía trên

---

## License

Tài liệu này được tổng hợp từ kubernetes.io và các nguồn chính thức, phục vụ mục đích học tập và triển khai.
