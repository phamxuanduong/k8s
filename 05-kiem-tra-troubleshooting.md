# 11 - Kiểm tra và Troubleshooting

Một tập hợp lệnh quan trọng để verify cluster health và troubleshoot issues.

## Kiểm tra Cluster Health

### Thông tin cluster

```bash
kubectl cluster-info
```

Output mẫu:
```
Kubernetes control plane is running at https://192.168.1.10:6443
CoreDNS is running at https://192.168.1.10:6443/api/v1/namespaces/kube-system/services/kube-dns:dns/proxy
```

### Status các nodes

```bash
kubectl get nodes
```

Output mẫu (tất cả phải Ready):
```
NAME           STATUS   ROLES           AGE   VERSION
k8s-master     Ready    control-plane   30m   v1.34.0
k8s-worker-1   Ready    worker          20m   v1.34.0
```

### Chi tiết nodes

```bash
kubectl get nodes -o wide
```

Hiển thị thêm: INTERNAL-IP, EXTERNAL-IP, OS-IMAGE, KERNEL-VERSION, CONTAINER-RUNTIME

### Tất cả pods hệ thống

```bash
kubectl get pods --all-namespaces
```

Hoặc:
```bash
kubectl get pods -A
```

### API server health

```bash
kubectl get --raw /healthz
```

Output mong đợi: `ok`

### Component status

```bash
kubectl get componentstatuses
```

**Lưu ý**: Từ Kubernetes 1.19+, componentstatuses có thể deprecated. Sử dụng lệnh khác để kiểm tra:

```bash
kubectl get pods -n kube-system
```

## Kiểm tra Node-Level Services

### Kubelet status

```bash
sudo systemctl status kubelet
```

Status phải là **active (running)**.

### Kubelet logs

```bash
# Xem logs realtime
sudo journalctl -u kubelet -f

# Xem 100 dòng cuối
sudo journalctl -u kubelet -n 100

# Xem logs từ 1 giờ trước
sudo journalctl -u kubelet --since "1 hour ago"
```

### Containerd status

```bash
sudo systemctl status containerd
```

### Containerd logs

```bash
sudo journalctl -u containerd -f
```

### Kiểm tra containers đang chạy

```bash
sudo crictl ps
```

### Kiểm tra images

```bash
sudo crictl images
```

## Test Pod và Network

### Tạo test pod

```bash
kubectl run test-nginx --image=nginx
kubectl get pods
```

### Kiểm tra pod details

```bash
kubectl describe pod test-nginx
```

### Xem logs của pod

```bash
kubectl logs test-nginx
```

### Execute command trong pod

```bash
kubectl exec -it test-nginx -- /bin/bash
```

### Test DNS resolution

```bash
kubectl run test-dns --image=busybox --rm -it -- nslookup kubernetes.default
```

Output mẫu:
```
Server:    10.96.0.10
Address 1: 10.96.0.10 kube-dns.kube-system.svc.cluster.local

Name:      kubernetes.default
Address 1: 10.96.0.1 kubernetes.default.svc.cluster.local
```

### Test inter-pod communication

```bash
# Deploy 2 pods
kubectl run test-pod1 --image=nginx
kubectl run test-pod2 --image=busybox --rm -it -- sh

# Trong test-pod2, lấy IP của test-pod1
POD1_IP=$(kubectl get pod test-pod1 -o jsonpath='{.status.podIP}')

# Test connectivity
wget -O- http://$POD1_IP
```

## Troubleshooting Commands

### Xem chi tiết node

```bash
kubectl describe node <node-name>
```

Các section quan trọng:
- **Conditions**: Node health status
- **Capacity/Allocatable**: Resources available
- **Non-terminated Pods**: Pods đang chạy
- **Allocated resources**: Resource usage

### Xem events

```bash
# Tất cả events, sorted by time
kubectl get events --all-namespaces --sort-by='.lastTimestamp'

# Events của một namespace cụ thể
kubectl get events -n kube-system --sort-by='.lastTimestamp'

# Watch events realtime
kubectl get events --all-namespaces --watch
```

### Kiểm tra CNI config

```bash
ls -la /etc/cni/net.d/
```

Phải có ít nhất một `.conf` hoặc `.conflist` file.

### Xem CNI plugin config

```bash
cat /etc/cni/net.d/*.conf
```

### Xem logs của một pod cụ thể

```bash
kubectl logs <pod-name> -n <namespace>

# Nếu pod có nhiều containers
kubectl logs <pod-name> -c <container-name> -n <namespace>

# Xem logs trước khi restart (nếu pod đã crashed)
kubectl logs <pod-name> --previous
```

### Execute trong pod

```bash
kubectl exec -it <pod-name> -- /bin/bash

# Hoặc sh nếu không có bash
kubectl exec -it <pod-name> -- /bin/sh

# Execute command cụ thể
kubectl exec <pod-name> -- ls -la /
```

### Port forwarding

```bash
kubectl port-forward <pod-name> 8080:80
```

Sau đó truy cập `http://localhost:8080` trên máy local.

## Các vấn đề thường gặp

### 1. Pods stuck ở Pending

**Nguyên nhân**:
- CNI chưa cài
- Không đủ resources
- Taints/tolerations issues
- PersistentVolume không available

**Kiểm tra**:
```bash
kubectl describe pod <pod-name>
```

Xem section **Events** để biết lý do.

**Giải pháp**:
```bash
# Nếu CNI chưa cài
# -> Cài CNI (File 07)

# Nếu không đủ resources
kubectl top nodes
kubectl describe nodes

# Nếu taint issue (cho single-node)
kubectl taint nodes --all node-role.kubernetes.io/control-plane-
```

### 2. CoreDNS CrashLoopBackOff

**Nguyên nhân**:
- CNI chưa cài đúng
- SELinux issues
- Loop plugin config issues

**Kiểm tra**:
```bash
kubectl logs -n kube-system <coredns-pod>
```

**Giải pháp**:
```bash
# Verify CNI installed
kubectl get pods -A | grep -E 'calico|flannel|cilium'

# Nếu CNI chưa có, cài lại (File 07)

# Disable SELinux nếu cần (CentOS/RHEL)
sudo setenforce 0
```

### 3. Node NotReady

**Nguyên nhân**:
- Kubelet issue
- Container runtime issue
- Network issue
- Disk/Memory pressure

**Kiểm tra**:
```bash
kubectl describe node <node-name>
```

Xem section **Conditions** để biết chi tiết.

**Giải pháp**:
```bash
# Trên node đó, kiểm tra kubelet
sudo systemctl status kubelet
sudo journalctl -u kubelet -n 100

# Restart kubelet
sudo systemctl restart kubelet

# Kiểm tra containerd
sudo systemctl status containerd
sudo systemctl restart containerd

# Kiểm tra network
ping <other-node-ip>

# Kiểm tra disk space
df -h
```

### 4. Token expired

**Nguyên nhân**: Tokens hết hạn sau 24 giờ.

**Giải pháp**:
```bash
# Từ control plane, tạo token mới
kubeadm token create --print-join-command
```

### 5. Pods không có IP address

**Nguyên nhân**: CNI plugin issue.

**Kiểm tra**:
```bash
kubectl get pods --all-namespaces | grep -E 'calico|flannel|cilium'
```

**Giải pháp**:
```bash
# Restart CNI pods
kubectl delete pod -n kube-system <cni-pod-name>

# Hoặc reinstall CNI (File 07)
```

### 6. Cannot connect to API server

**Nguyên nhân**:
- API server down
- Network issue
- Firewall blocking port 6443

**Kiểm tra**:
```bash
# Trên control plane
sudo systemctl status kubelet
kubectl get pods -n kube-system | grep apiserver

# Từ worker node, test connectivity
telnet <control-plane-ip> 6443
curl -k https://<control-plane-ip>:6443/version
```

**Giải pháp**:
```bash
# Trên control plane, restart kubelet
sudo systemctl restart kubelet

# Mở port 6443
sudo ufw allow 6443/tcp
```

### 7. High CPU/Memory usage

**Kiểm tra**:
```bash
kubectl top nodes
kubectl top pods --all-namespaces --sort-by=memory
kubectl top pods --all-namespaces --sort-by=cpu
```

**Giải pháp**:
```bash
# Set resource limits
kubectl set resources deployment <deployment-name> \
  --limits=cpu=200m,memory=256Mi \
  --requests=cpu=100m,memory=128Mi

# Scale down
kubectl scale deployment <deployment-name> --replicas=1
```

### 8. Pods evicted

**Nguyên nhân**: Node pressure (memory/disk).

**Kiểm tra**:
```bash
kubectl get pods --all-namespaces | grep Evicted
kubectl describe node | grep -A 10 Conditions
```

**Giải pháp**:
```bash
# Clean up evicted pods
kubectl get pods --all-namespaces | grep Evicted | \
  awk '{print $2 " -n " $1}' | xargs kubectl delete pod

# Free up disk space
sudo crictl rmi --prune
sudo journalctl --vacuum-time=3d
docker system prune -a  # Nếu có docker
```

## Kiểm tra Certificates

### Check expiration

```bash
sudo kubeadm certs check-expiration
```

Output hiển thị thời gian hết hạn của tất cả certificates.

### Renew certificates

```bash
sudo kubeadm certs renew all
```

Sau đó restart kubelet:
```bash
sudo systemctl restart kubelet
```

## Debug Commands nâng cao

### Xem tất cả resources

```bash
kubectl api-resources
```

### Xem config

```bash
kubectl config view
```

### Kiểm tra RBAC

```bash
# Xem permissions của user hiện tại
kubectl auth can-i --list

# Check specific permission
kubectl auth can-i create deployments
kubectl auth can-i delete pods --namespace=kube-system
```

### Xem service endpoints

```bash
kubectl get endpoints
```

### Xem persistent volumes

```bash
kubectl get pv
kubectl get pvc
```

### Network troubleshooting pod

Deploy một pod với network tools:
```bash
kubectl run netshoot --rm -it --image=nicolaka/netshoot -- /bin/bash

# Trong pod, có các tools: ping, curl, nslookup, traceroute, tcpdump, etc.
```

## Reset Cluster khi cần

```bash
# Reset cluster
sudo kubeadm reset -f

# Clean up
sudo iptables -F && sudo iptables -t nat -F && sudo iptables -t mangle -F && sudo iptables -X
sudo rm -rf /etc/cni/net.d
sudo rm -rf /var/lib/cni/
sudo rm -rf /var/lib/kubelet
sudo rm -rf /etc/kubernetes

# Restart containerd
sudo systemctl restart containerd
```

## Script kiểm tra cluster health tổng hợp

```bash
#!/bin/bash
# Cluster Health Check Script

echo "=== Kubernetes Cluster Health Check ==="
echo

echo "[1] Cluster Info:"
kubectl cluster-info
echo

echo "[2] Node Status:"
kubectl get nodes -o wide
echo

echo "[3] System Pods:"
kubectl get pods -n kube-system
echo

echo "[4] All Namespaces Pods:"
kubectl get pods -A
echo

echo "[5] Top Nodes:"
kubectl top nodes || echo "Metrics Server not available"
echo

echo "[6] Recent Events:"
kubectl get events --all-namespaces --sort-by='.lastTimestamp' | tail -20
echo

echo "[7] Component Status:"
kubectl get pods -n kube-system -o wide
echo

echo "[8] CNI Status:"
kubectl get pods -A | grep -E 'calico|flannel|cilium'
echo

echo "=== Health Check Complete ==="
```

Lưu thành `cluster-health-check.sh`, sau đó chạy:
```bash
chmod +x cluster-health-check.sh
./cluster-health-check.sh
```

## Logs and Diagnostics locations

### Kubernetes logs
- Kubelet logs: `sudo journalctl -u kubelet`
- Containerd logs: `sudo journalctl -u containerd`
- Pod logs: `/var/log/pods/`
- Container logs: `/var/log/containers/`

### Kubernetes configs
- Main config: `/etc/kubernetes/`
- Kubelet config: `/var/lib/kubelet/config.yaml`
- Certificates: `/etc/kubernetes/pki/`
- Manifests: `/etc/kubernetes/manifests/`

### CNI configs
- CNI config: `/etc/cni/net.d/`
- CNI binaries: `/opt/cni/bin/`

## Checklist troubleshooting

- [ ] Nodes status là Ready
- [ ] Tất cả system pods Running
- [ ] CNI pods Running
- [ ] CoreDNS pods Running
- [ ] Metrics Server pods Running
- [ ] Không có Pending/CrashLoopBackOff pods
- [ ] kubectl top nodes hoạt động
- [ ] DNS resolution hoạt động
- [ ] Pods có thể communicate với nhau
- [ ] Services accessible

## Tiếp theo

Sau khi verify cluster health:
- **File 12**: Best Practices
- **File 13**: Scripts tự động
