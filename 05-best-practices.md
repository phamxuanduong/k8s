# 05 - Best Practices và Lưu ý quan trọng

Các best practices để vận hành Kubernetes cluster hiệu quả và an toàn.

## Security Considerations

### 1. Bảo vệ admin.conf

**KHÔNG chia sẻ** file `/etc/kubernetes/admin.conf` - nó chứa cluster-admin privileges.

```bash
# Backup admin.conf an toàn
sudo cp /etc/kubernetes/admin.conf ~/admin.conf.backup
chmod 600 ~/admin.conf.backup

# Không commit vào git
echo "admin.conf*" >> .gitignore
```

### 2. Bảo vệ tokens

Join tokens cho phép bất kỳ ai có thể join nodes vào cluster.

```bash
# List tokens hiện tại
kubeadm token list

# Delete tokens không dùng
kubeadm token delete <token>

# Tạo token với TTL ngắn (1 giờ)
kubeadm token create --ttl 1h
```

### 3. RBAC - Role-Based Access Control

Sử dụng RBAC để giới hạn permissions:

```yaml
# Tạo user với limited permissions
apiVersion: v1
kind: ServiceAccount
metadata:
  name: limited-user
  namespace: default
---
apiVersion: rbac.authorization.k8s.io/v1
kind: Role
metadata:
  name: pod-reader
  namespace: default
rules:
- apiGroups: [""]
  resources: ["pods"]
  verbs: ["get", "list"]
---
apiVersion: rbac.authorization.k8s.io/v1
kind: RoleBinding
metadata:
  name: read-pods
  namespace: default
subjects:
- kind: ServiceAccount
  name: limited-user
  namespace: default
roleRef:
  kind: Role
  name: pod-reader
  apiGroup: rbac.authorization.k8s.io
```

### 4. Network Policies

Implement network policies để restrict pod-to-pod communication:

```yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: default-deny-ingress
  namespace: default
spec:
  podSelector: {}
  policyTypes:
  - Ingress
```

**Lưu ý**: Cần CNI hỗ trợ network policies (Calico, Cilium).

### 5. Pod Security Standards

Enable Pod Security Standards:

```bash
# Label namespace với security level
kubectl label namespace default pod-security.kubernetes.io/enforce=baseline
kubectl label namespace default pod-security.kubernetes.io/warn=restricted
```

### 6. Secrets Management

Không hardcode secrets trong manifests:

```bash
# Tạo secret từ file
kubectl create secret generic db-password --from-file=password.txt

# Tạo secret từ literal
kubectl create secret generic db-credentials \
  --from-literal=username=admin \
  --from-literal=password=secretpass

# Sử dụng trong pod
apiVersion: v1
kind: Pod
metadata:
  name: app
spec:
  containers:
  - name: app
    image: myapp
    env:
    - name: DB_PASSWORD
      valueFrom:
        secretKeyRef:
          name: db-password
          key: password
```

**Production**: Sử dụng external secrets management (HashiCorp Vault, AWS Secrets Manager, etc.)

## Backup và Disaster Recovery

### 1. Backup etcd regularly

```bash
#!/bin/bash
# Backup script - chạy daily

BACKUP_DIR="/backups/etcd"
DATE=$(date +%Y%m%d_%H%M%S)

sudo ETCDCTL_API=3 etcdctl snapshot save ${BACKUP_DIR}/etcd-backup-${DATE}.db \
  --endpoints=https://127.0.0.1:2379 \
  --cacert=/etc/kubernetes/pki/etcd/ca.crt \
  --cert=/etc/kubernetes/pki/etcd/server.crt \
  --key=/etc/kubernetes/pki/etcd/server.key

# Verify backup
sudo ETCDCTL_API=3 etcdctl snapshot status ${BACKUP_DIR}/etcd-backup-${DATE}.db

# Rotate old backups (keep last 7 days)
find ${BACKUP_DIR} -name "etcd-backup-*.db" -mtime +7 -delete

# Copy to remote backup server
scp ${BACKUP_DIR}/etcd-backup-${DATE}.db user@backup-server:/backups/
```

### 2. Backup certificates và configs

```bash
sudo tar -czf k8s-backup-$(date +%Y%m%d).tar.gz \
  /etc/kubernetes/pki \
  /etc/kubernetes/*.conf \
  /etc/kubernetes/manifests
```

### 3. Backup persistent data

```bash
# Backup PersistentVolumes
kubectl get pv -o yaml > pv-backup.yaml
kubectl get pvc -A -o yaml > pvc-backup.yaml
```

### 4. Disaster Recovery Plan

Document và test disaster recovery procedures:
1. Restore etcd from backup
2. Restore certificates
3. Rejoin nodes
4. Restore applications
5. Verify services

## Version Management và Upgrades

### 1. Pin versions

Đã được thực hiện trong File 04, nhưng verify:

```bash
apt-mark showhold
```

Output phải có: `kubeadm`, `kubectl`, `kubelet`

### 2. Test upgrades trước

Luôn test upgrades trong môi trường non-production trước khi apply lên production.

### 3. Thứ tự upgrade đúng

**Thứ tự upgrade Kubernetes**:
1. Control plane node đầu tiên
2. Các control plane nodes khác
3. Worker nodes (từng node một)

**Quy trình upgrade control plane**:
```bash
# 1. Unhold packages
sudo apt-mark unhold kubeadm

# 2. Upgrade kubeadm
sudo apt-get update
sudo apt-get install -y kubeadm=1.34.1-1.1

# 3. Verify upgrade plan
sudo kubeadm upgrade plan

# 4. Apply upgrade
sudo kubeadm upgrade apply v1.34.1

# 5. Upgrade kubelet và kubectl
sudo apt-mark unhold kubelet kubectl
sudo apt-get install -y kubelet=1.34.1-1.1 kubectl=1.34.1-1.1
sudo systemctl daemon-reload
sudo systemctl restart kubelet

# 6. Hold lại
sudo apt-mark hold kubeadm kubelet kubectl
```

### 4. Version skew policy

Kubernetes hỗ trợ version skew:
- **kube-apiserver**: N (latest)
- **kubelet**: N-2 (có thể chậm hơn 2 minor versions)
- **kubectl**: N+1, N, N-1

Ví dụ: API server v1.34, kubelet có thể v1.32

## Production Deployment Considerations

### 1. High Availability setup

Đối với production, **bắt buộc** sử dụng multiple control plane nodes (File 06):
- Tối thiểu **3 control plane nodes**
- Load balancer cho API server
- etcd backup automated

### 2. Resource Planning

#### Control Plane Nodes
- **CPU**: 4+ cores
- **RAM**: 8+ GB
- **Disk**: 50+ GB (SSD khuyến nghị)

#### Worker Nodes
- Tùy workload, nhưng tối thiểu:
- **CPU**: 2+ cores
- **RAM**: 4+ GB
- **Disk**: 50+ GB

### 3. Monitoring và Logging

#### Deploy Prometheus + Grafana

```bash
# Sử dụng kube-prometheus-stack
helm repo add prometheus-community https://prometheus-community.github.io/helm-charts
helm repo update
helm install prometheus prometheus-community/kube-prometheus-stack
```

#### Deploy EFK Stack (Elasticsearch, Fluentd, Kibana)

Hoặc sử dụng managed logging solutions (CloudWatch, Stackdriver, etc.)

### 4. Resource Limits

**Luôn** set resource requests và limits:

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: myapp
spec:
  containers:
  - name: app
    image: myapp:latest
    resources:
      requests:
        memory: "128Mi"
        cpu: "100m"
      limits:
        memory: "256Mi"
        cpu: "200m"
```

### 5. Health Checks

Implement liveness và readiness probes:

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: myapp
spec:
  containers:
  - name: app
    image: myapp:latest
    livenessProbe:
      httpGet:
        path: /healthz
        port: 8080
      initialDelaySeconds: 30
      periodSeconds: 10
    readinessProbe:
      httpGet:
        path: /ready
        port: 8080
      initialDelaySeconds: 10
      periodSeconds: 5
```

### 6. Pod Disruption Budgets

Đảm bảo availability trong maintenance:

```yaml
apiVersion: policy/v1
kind: PodDisruptionBudget
metadata:
  name: myapp-pdb
spec:
  minAvailable: 2
  selector:
    matchLabels:
      app: myapp
```

## Certificate Management

### 1. Check expiration thường xuyên

```bash
sudo kubeadm certs check-expiration
```

### 2. Auto-renewal

Tạo cronjob để auto-renew:

```bash
# /etc/cron.monthly/renew-k8s-certs
#!/bin/bash
sudo kubeadm certs renew all
sudo systemctl restart kubelet
```

### 3. Certificate rotation cho kubelet

Enable automatic certificate rotation:

```yaml
# /var/lib/kubelet/config.yaml
rotateCertificates: true
serverTLSBootstrap: true
```

## Node Maintenance

### 1. Drain node trước maintenance

```bash
kubectl drain <node-name> --ignore-daemonsets --delete-emptydir-data
```

### 2. Perform maintenance

Update OS, reboot, etc.

### 3. Uncordon node

```bash
kubectl uncordon <node-name>
```

### 4. Verify pods rescheduled

```bash
kubectl get pods -o wide
```

## Namespace Organization

### 1. Sử dụng namespaces

Tách biệt environments và teams:

```bash
kubectl create namespace production
kubectl create namespace staging
kubectl create namespace development
```

### 2. Set default namespace

```bash
kubectl config set-context --current --namespace=production
```

### 3. ResourceQuotas

Giới hạn resources per namespace:

```yaml
apiVersion: v1
kind: ResourceQuota
metadata:
  name: compute-quota
  namespace: production
spec:
  hard:
    requests.cpu: "10"
    requests.memory: 20Gi
    limits.cpu: "20"
    limits.memory: 40Gi
    persistentvolumeclaims: "10"
```

## Scaling Best Practices

### 1. Horizontal Pod Autoscaler

```bash
kubectl autoscale deployment myapp --cpu-percent=70 --min=2 --max=10
```

### 2. Vertical Pod Autoscaler

Deploy VPA:
```bash
git clone https://github.com/kubernetes/autoscaler.git
cd autoscaler/vertical-pod-autoscaler
./hack/vpa-up.sh
```

### 3. Cluster Autoscaler

Cho cloud environments (AWS, GCP, Azure).

## Image Management

### 1. Sử dụng image tags cụ thể

❌ **Tránh**:
```yaml
image: nginx:latest
```

✅ **Nên**:
```yaml
image: nginx:1.21.6
```

### 2. Private registry

```bash
kubectl create secret docker-registry regcred \
  --docker-server=<your-registry-server> \
  --docker-username=<your-name> \
  --docker-password=<your-password> \
  --docker-email=<your-email>
```

Sử dụng trong pod:
```yaml
spec:
  imagePullSecrets:
  - name: regcred
```

### 3. Image scanning

Scan images cho vulnerabilities trước khi deploy (Trivy, Clair, Anchore).

## GitOps và CI/CD

### 1. Infrastructure as Code

Store tất cả Kubernetes manifests trong Git.

### 2. GitOps tools

Sử dụng ArgoCD hoặc Flux để automated deployments.

### 3. CI/CD pipeline

- Build images trong CI
- Test trong staging namespace
- Deploy sang production với approval

## Các vấn đề thường gặp và cách tránh

### 1. ❌ Chạy pods as root

✅ Sử dụng non-root user:
```yaml
securityContext:
  runAsNonRoot: true
  runAsUser: 1000
```

### 2. ❌ Không set resource limits

✅ Luôn set requests/limits.

### 3. ❌ Expose unnecessary ports

✅ Chỉ expose ports cần thiết, sử dụng NetworkPolicies.

### 4. ❌ Store secrets trong Git

✅ Sử dụng Kubernetes Secrets hoặc external secrets manager.

### 5. ❌ Single point of failure

✅ Deploy multiple replicas, sử dụng HA setup.

## Monitoring Checklist

- [ ] Node metrics (CPU, Memory, Disk)
- [ ] Pod metrics
- [ ] API server metrics
- [ ] etcd metrics (cho HA clusters)
- [ ] Application logs
- [ ] Security audit logs
- [ ] Certificate expiration alerts
- [ ] Backup success/failure alerts

## Documentation

### 1. Document cluster architecture

- Topology diagram
- IP ranges
- DNS configuration
- Load balancer setup

### 2. Document runbooks

- Deployment procedures
- Rollback procedures
- Disaster recovery steps
- Upgrade procedures

### 3. Document access

- Who has access
- Service accounts
- RBAC policies

## Compliance và Auditing

### 1. Enable audit logging

```yaml
# /etc/kubernetes/manifests/kube-apiserver.yaml
- --audit-log-path=/var/log/kubernetes/audit.log
- --audit-log-maxage=30
- --audit-log-maxbackup=10
- --audit-log-maxsize=100
```

### 2. Regular security audits

Sử dụng tools như:
- kube-bench (CIS Kubernetes Benchmark)
- kube-hunter (penetration testing)
- Falco (runtime security)

## Checklist Production Readiness

- [ ] High Availability setup (3+ control planes)
- [ ] Load balancer configured
- [ ] Automated etcd backups
- [ ] Certificate auto-renewal
- [ ] Monitoring và alerting
- [ ] Logging centralized
- [ ] RBAC configured
- [ ] Network policies implemented
- [ ] Resource quotas set
- [ ] Pod security standards enforced
- [ ] Disaster recovery plan documented và tested
- [ ] Upgrade procedures documented
- [ ] Security audit completed

---

**Nguồn tài liệu**: kubernetes.io/docs/tasks và kubernetes.io/docs/concepts
