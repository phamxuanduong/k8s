# Hướng dẫn cài đặt Kubernetes v1.34 trên Ubuntu

Tài liệu hướng dẫn chi tiết cài đặt và triển khai Kubernetes cluster trên Ubuntu với kubeadm. Tất cả hướng dẫn dựa trên tài liệu chính thức từ kubernetes.io.

## Tổng quan

Hướng dẫn này bao gồm:
- ✅ Cài đặt Kubernetes v1.34 (phiên bản mới nhất tháng 10/2025)
- ✅ Triển khai cụm All-in-One (1 node) cho dev/test/learning
- ✅ Triển khai cụm HA (3 masters + workers) cho production
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

### File 02: Cụm All-in-One
**`02-tao-cum-don.md`** (15 KB)

Triển khai cluster với **1 node duy nhất**, vừa làm control plane vừa làm worker.

Bao gồm:
- Khởi tạo control plane với kubeadm init
- Cấu hình kubectl
- Cài đặt CNI plugin (Calico/Flannel/Cilium)
- Remove taint để chạy workloads trên control plane
- Tối ưu resource limits
- Verify cluster

**Phù hợp cho**: Development, Testing, Learning, Lab, Demo
**KHÔNG phù hợp**: Production

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

### Scenario 1: All-in-One Cluster (1 node duy nhất)

```
01-cai-dat-co-ban.md (trên 1 node)
    ↓
02-tao-cum-don.md (all-in-one)
    ↓
04-cai-dat-metrics-server.md (optional)
    ↓
05-best-practices.md (backup, monitoring)
```

**Phù hợp cho**: Dev, Test, Learning, Lab

### Scenario 2: HA Cluster (3 masters + workers)

```
01-cai-dat-co-ban.md (trên tất cả nodes + LB)
    ↓
03-tao-cum-ha.md (setup 3 masters + workers)
    ↓
04-cai-dat-metrics-server.md (khuyến nghị)
    ↓
05-best-practices.md (backup, monitoring)
```

**Phù hợp cho**: Production

---

## Yêu cầu hệ thống

### Hardware tối thiểu

**All-in-One Cluster (1 node):**
- CPU: 4+ cores
- RAM: 8+ GB
- Disk: 50+ GB

**HA Cluster:**
- **3 Control Plane Nodes**: Mỗi node 2+ CPUs, 4+ GB RAM, 20+ GB disk
- **3+ Worker Nodes**: Mỗi node 1+ CPUs, 2+ GB RAM, 20+ GB disk
- **1 Load Balancer**: 1 CPU, 1 GB RAM

### Software

- Ubuntu 18.04, 20.04, 22.04, hoặc 24.04 (amd64)
- Linux kernel >= 3.10
- containerd (sẽ được cài tự động)

---

## Quick Start

### Cài đặt nhanh All-in-One Cluster

```bash
# 1. Làm theo File 01: Cài đặt cơ bản
# Chuẩn bị hệ thống, cài containerd, kubeadm, kubelet, kubectl

# 2. Làm theo File 02: Cụm All-in-One
# Khởi tạo cluster, cài CNI, remove taint

# 3. Verify
kubectl get nodes
kubectl create deployment nginx --image=nginx
```

### Cài đặt nhanh HA Cluster

```bash
# 1. Chuẩn bị load balancer (HAProxy)

# 2. Trên TẤT CẢ 3 master nodes và worker nodes:
#    Làm theo File 01: Cài đặt cơ bản

# 3. Làm theo File 03: Tạo cụm HA
#    - Khởi tạo master node đầu tiên
#    - Join 2 master nodes còn lại
#    - Cài CNI
#    - Join worker nodes

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
