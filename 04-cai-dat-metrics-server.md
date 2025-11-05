# 04 - Cài đặt Metrics Server

Metrics Server là thành phần quan trọng thu thập resource metrics từ Kubelets và expose chúng trong Kubernetes API server thông qua Metrics API.

**Điều kiện tiên quyết**:
- ✅ Đã có cluster đang chạy (File 02 hoặc File 03)

**LƯU Ý**: Metrics Server chỉ cần cài **một lần** từ control plane node.

---

## Tại sao cần Metrics Server?

Metrics Server là yêu cầu bắt buộc để:

- ✅ Sử dụng lệnh `kubectl top nodes` và `kubectl top pods`
- ✅ **Horizontal Pod Autoscaler (HPA)** hoạt động
- ✅ **Vertical Pod Autoscaler (VPA)** hoạt động
- ✅ Kubernetes Dashboard hiển thị metrics
- ✅ Monitor resource usage của cluster

---

## Cài đặt Standard (Khuyến nghị cho cluster nhỏ)

### Bước 1: Deploy Metrics Server

```bash
kubectl apply -f https://github.com/kubernetes-sigs/metrics-server/releases/latest/download/components.yaml
```

### Bước 2: Patch cho Development/Test environments

Trong môi trường development hoặc test không có proper TLS certificates, cần patch để skip TLS verification:

```bash
kubectl patch deployment metrics-server -n kube-system --type='json' -p='[
  {
    "op": "add",
    "path": "/spec/template/spec/containers/0/args/-",
    "value": "--kubelet-insecure-tls"
  }
]'
```

**⚠️ LƯU Ý**: Flag `--kubelet-insecure-tls` chỉ nên sử dụng trong môi trường dev/test. Trong production, nên cấu hình proper TLS certificates cho kubelet.

---

## Cài đặt High Availability (Cho production cluster lớn)

### Bước 1: Deploy HA Metrics Server

```bash
kubectl apply -f https://github.com/kubernetes-sigs/metrics-server/releases/latest/download/high-availability-1.21+.yaml
```

Manifest này tự động deploy metrics-server với 2 replicas.

### Bước 2: Patch cho Development/Test (nếu cần)

```bash
kubectl patch deployment metrics-server -n kube-system --type='json' -p='[
  {
    "op": "add",
    "path": "/spec/template/spec/containers/0/args/-",
    "value": "--kubelet-insecure-tls"
  }
]'
```

---

## Điều chỉnh số replicas

### Cho single-node cluster

```bash
kubectl scale deployment metrics-server -n kube-system --replicas=1
```

### Cho production cluster lớn

```bash
kubectl scale deployment metrics-server -n kube-system --replicas=3
```

---

## Verify Metrics Server

### Kiểm tra deployment

```bash
kubectl get deployment metrics-server -n kube-system
```

Output mong đợi:
```
NAME             READY   UP-TO-DATE   AVAILABLE   AGE
metrics-server   1/1     1            1           2m
```

### Kiểm tra pods

```bash
kubectl get pods -n kube-system -l k8s-app=metrics-server
```

Output mong đợi:
```
NAME                              READY   STATUS    RESTARTS   AGE
metrics-server-xxxxx              1/1     Running   0          2m
```

**Đợi cho đến khi pods ở trạng thái Running và Ready.**

### Test metrics

Sau khi pods Running, đợi 1-2 phút để metrics-server thu thập dữ liệu, sau đó test:

```bash
# Xem resource usage của nodes
kubectl top nodes
```

Output mẫu:
```
NAME              CPU(cores)   CPU%   MEMORY(bytes)   MEMORY%
k8s-master        156m         7%     1234Mi          32%
k8s-worker-1      89m          4%     876Mi           22%
```

```bash
# Xem resource usage của pods
kubectl top pods --all-namespaces
```

Output mẫu:
```
NAMESPACE     NAME                              CPU(cores)   MEMORY(bytes)
kube-system   coredns-xxxxx                     3m           15Mi
kube-system   etcd-k8s-master                   18m          52Mi
kube-system   kube-apiserver-k8s-master         45m          256Mi
```

---

## Troubleshooting

### Metrics-server không start

Kiểm tra logs:
```bash
kubectl logs -n kube-system deployment/metrics-server
```

### Lỗi: "unable to fully scrape metrics"

**Nguyên nhân**: Kubelet port 10250 không accessible.

**Giải pháp**:
```bash
# Đảm bảo port 10250 mở trên tất cả nodes
sudo ufw allow 10250/tcp

# Hoặc disable firewall tạm thời để test
sudo ufw disable
```

### Lỗi: "x509: certificate signed by unknown authority"

**Nguyên nhân**: TLS certificate issues.

**Giải pháp**: Thêm flag `--kubelet-insecure-tls` như hướng dẫn trên.

### Lỗi: "no metrics available"

**Nguyên nhân**: Metrics-server chưa thu thập đủ dữ liệu.

**Giải pháp**: Đợi 2-3 phút sau khi pods Running, sau đó thử lại.

### kubectl top nodes/pods trả về lỗi

```bash
# Kiểm tra metrics-server API service
kubectl get apiservice v1beta1.metrics.k8s.io -o yaml

# Phải thấy status Available: True
```

Nếu status không Available:
```bash
# Restart metrics-server
kubectl rollout restart deployment metrics-server -n kube-system

# Đợi pods mới Running
kubectl get pods -n kube-system -l k8s-app=metrics-server -w
```

---

## Test Horizontal Pod Autoscaler (HPA)

Sau khi cài Metrics Server, test HPA:

### Deploy test application

```bash
kubectl create deployment php-apache --image=registry.k8s.io/hpa-example
kubectl set resources deployment php-apache --requests=cpu=200m
kubectl expose deployment php-apache --port=80
```

### Tạo HPA

```bash
kubectl autoscale deployment php-apache --cpu-percent=50 --min=1 --max=10
```

### Verify HPA

```bash
kubectl get hpa
```

Output:
```
NAME         REFERENCE               TARGETS   MINPODS   MAXPODS   REPLICAS   AGE
php-apache   Deployment/php-apache   0%/50%    1         10        1          1m
```

### Generate load để test

```bash
kubectl run -it --rm load-generator --image=busybox --restart=Never -- /bin/sh -c "while sleep 0.01; do wget -q -O- http://php-apache; done"
```

Trong terminal khác, watch HPA:
```bash
kubectl get hpa php-apache --watch
```

Sau vài phút, bạn sẽ thấy REPLICAS tăng lên khi CPU usage tăng.

### Clean up

```bash
kubectl delete hpa php-apache
kubectl delete deployment php-apache
kubectl delete service php-apache
```

---

## Script tự động cài đặt

```bash
#!/bin/bash
# install-metrics-server.sh

set -e

echo "=== Cài đặt Metrics Server ==="

# [1] Deploy Metrics Server
echo "[1/3] Deploy Metrics Server..."
kubectl apply -f https://github.com/kubernetes-sigs/metrics-server/releases/latest/download/components.yaml

# Đợi deployment tạo
sleep 5

# [2] Patch cho dev/test environment
echo "[2/3] Patch Metrics Server (skip TLS verification)..."
kubectl patch deployment metrics-server -n kube-system --type='json' -p='[
  {
    "op": "add",
    "path": "/spec/template/spec/containers/0/args/-",
    "value": "--kubelet-insecure-tls"
  }
]'

# [3] Wait for pods ready
echo "[3/3] Đợi Metrics Server ready..."
kubectl wait --for=condition=available --timeout=300s deployment/metrics-server -n kube-system

echo -e "\n=== Metrics Server đã cài đặt! ==="
echo "Đợi 1-2 phút để metrics-server thu thập dữ liệu..."
echo "Sau đó test bằng: kubectl top nodes"
```

Lưu script thành file `install-metrics-server.sh`, sau đó chạy:
```bash
chmod +x install-metrics-server.sh
./install-metrics-server.sh
```

---

## Checklist cài đặt

- [ ] Metrics Server deployment đã được tạo
- [ ] Metrics Server pods đang Running
- [ ] `kubectl top nodes` hoạt động
- [ ] `kubectl top pods` hoạt động
- [ ] Không có errors trong metrics-server logs
- [ ] HPA test hoạt động (optional)

---

## Tiếp theo

Sau khi cài đặt Metrics Server:
- **File 05**: Best Practices và vận hành cluster

**Nguồn tài liệu**: https://github.com/kubernetes-sigs/metrics-server
