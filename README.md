# CKA (Certified Kubernetes Administrator) Complete Preparation Guide

## 📋 Table of Contents
1. [CKA Exam Overview](#cka-exam-overview)
2. [Cluster Architecture & Installation](#cluster-architecture--installation)
3. [Workloads & Scheduling](#workloads--scheduling)
4. [Services & Networking](#services--networking)
5. [Storage](#storage)
6. [Troubleshooting](#troubleshooting)
7. [Practical Lab Exercises](#practical-lab-exercises)
8. [Exam Tips & Strategies](#exam-tips--strategies)
9. [Resources & References](#resources--references)

---

## 🎯 CKA Exam Overview

### Exam Details
- **Duration**: 2 hours
- **Format**: Performance-based, hands-on tasks
- **Passing Score**: 66%
- **Environment**: Browser-based terminal to real Kubernetes clusters
- **Kubernetes Version**: v1.29 (as of 2024)

### Exam Domains & Weights
| Domain | Weight |
|--------|--------|
| Cluster Architecture, Installation & Configuration | 25% |
| Workloads & Scheduling | 15% |
| Services & Networking | 20% |
| Storage | 10% |
| Troubleshooting | 30% |

### Key Skills Required
- kubectl command proficiency
- YAML manifest creation and editing
- Cluster troubleshooting and debugging
- Understanding Kubernetes architecture
- Networking concepts and CNI plugins
- Storage configuration and management

---

## 🏗️ Cluster Architecture & Installation

### 1.1 Kubernetes Architecture Overview

```
┌─────────────────────────────────────────────────────────────┐
│                    KUBERNETES CLUSTER                       │
├─────────────────────────────────────────────────────────────┤
│  CONTROL PLANE (Master Node)                               │
│  ┌─────────────┐ ┌─────────────┐ ┌─────────────┐          │
│  │   kube-api  │ │    etcd     │ │ kube-scheduler│         │
│  │   server    │ │             │ │               │         │
│  └─────────────┘ └─────────────┘ └─────────────┘          │
│  ┌─────────────┐ ┌─────────────┐                           │
│  │kube-controller│ │ cloud-      │                          │
│  │   manager   │ │ controller  │                          │
│  └─────────────┘ └─────────────┘                           │
├─────────────────────────────────────────────────────────────┤
│  WORKER NODES                                               │
│  ┌───────────────┐ ┌───────────────┐ ┌───────────────┐    │
│  │   Node 1      │ │   Node 2      │ │   Node 3      │    │
│  │ ┌───────────┐ │ │ ┌───────────┐ │ │ ┌───────────┐ │    │
│  │ │  kubelet  │ │ │ │  kubelet  │ │ │ │  kubelet  │ │    │
│  │ └───────────┘ │ │ └───────────┘ │ │ └───────────┘ │    │
│  │ ┌───────────┐ │ │ ┌───────────┐ │ │ ┌───────────┐ │    │
│  │ │kube-proxy │ │ │ │kube-proxy │ │ │ │kube-proxy │ │    │
│  │ └───────────┘ │ │ └───────────┘ │ │ └───────────┘ │    │
│  │ ┌───────────┐ │ │ ┌───────────┐ │ │ ┌───────────┐ │    │
│  │ │Container  │ │ │ │Container  │ │ │ │Container  │ │    │
│  │ │ Runtime   │ │ │ │ Runtime   │ │ │ │ Runtime   │ │    │
│  │ └───────────┘ │ │ └───────────┘ │ │ └───────────┘ │    │
│  └───────────────┘ └───────────────┘ └───────────────┘    │
└─────────────────────────────────────────────────────────────┘
```

### 1.2 Control Plane Components

#### API Server (kube-apiserver)
- Central management entity
- Exposes Kubernetes API
- Entry point for all administrative tasks

#### etcd
- Distributed key-value store
- Stores all cluster data
- Highly available and consistent

#### Scheduler (kube-scheduler)
- Assigns pods to nodes
- Considers resource requirements and constraints

#### Controller Manager (kube-controller-manager)
- Runs controller processes
- Node controller, Replication controller, etc.

### 1.3 Node Components

#### kubelet
- Primary node agent
- Communicates with API server
- Manages pod lifecycle

#### kube-proxy
- Network proxy
- Maintains network rules
- Enables service abstraction

#### Container Runtime
- Runs containers (Docker, containerd, CRI-O)

### 1.4 Cluster Installation Methods

#### kubeadm Installation (Most Common for CKA)

**Prerequisites:**
```bash
# Disable swap
sudo swapoff -a
sudo sed -i '/ swap / s/^\(.*\)$/#\1/g' /etc/fstab

# Configure kernel modules
cat <<EOF | sudo tee /etc/modules-load.d/k8s.conf
overlay
br_netfilter
EOF

sudo modprobe overlay
sudo modprobe br_netfilter

# Configure sysctl
cat <<EOF | sudo tee /etc/sysctl.d/k8s.conf
net.bridge.bridge-nf-call-iptables  = 1
net.bridge.bridge-nf-call-ip6tables = 1
net.ipv4.ip_forward                 = 1
EOF

sudo sysctl --system
```

**Install Container Runtime (containerd):**
```bash
# Install containerd
sudo apt-get update
sudo apt-get install -y containerd

# Configure containerd
sudo mkdir -p /etc/containerd
containerd config default | sudo tee /etc/containerd/config.toml
sudo sed -i 's/SystemdCgroup \= false/SystemdCgroup \= true/g' /etc/containerd/config.toml
sudo systemctl restart containerd
sudo systemctl enable containerd
```

**Install kubeadm, kubelet, kubectl:**
```bash
sudo apt-get update
sudo apt-get install -y apt-transport-https ca-certificates curl

curl -fsSL https://pkgs.k8s.io/core:/stable:/v1.29/deb/Release.key | sudo gpg --dearmor -o /etc/apt/keyrings/kubernetes-apt-keyring.gpg

echo 'deb [signed-by=/etc/apt/keyrings/kubernetes-apt-keyring.gpg] https://pkgs.k8s.io/core:/stable:/v1.29/deb/ /' | sudo tee /etc/apt/sources.list.d/kubernetes.list

sudo apt-get update
sudo apt-get install -y kubelet kubeadm kubectl
sudo apt-mark hold kubelet kubeadm kubectl
```

**Initialize Control Plane:**
```bash
sudo kubeadm init --pod-network-cidr=10.244.0.0/16

# Configure kubectl for regular user
mkdir -p $HOME/.kube
sudo cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
sudo chown $(id -u):$(id -g) $HOME/.kube/config

# Install CNI plugin (Flannel example)
kubectl apply -f https://github.com/flannel-io/flannel/releases/latest/download/kube-flannel.yml
```

**Join Worker Nodes:**
```bash
# Run on worker nodes (token from kubeadm init output)
sudo kubeadm join <control-plane-ip>:6443 --token <token> --discovery-token-ca-cert-hash sha256:<hash>
```

### 1.5 Cluster Management Commands

```bash
# View cluster info
kubectl cluster-info
kubectl get nodes
kubectl get nodes -o wide

# View component status
kubectl get componentstatuses

# View system pods
kubectl get pods -n kube-system

# Check cluster configuration
kubectl config view
kubectl config current-context
kubectl config get-contexts

# Manage kubeconfig
kubectl config set-context --current --namespace=<namespace>
kubectl config use-context <context-name>
```

---

## ⚙️ Workloads & Scheduling

### 2.1 Pod Fundamentals

#### Pod Lifecycle
```
┌─────────────┐    ┌─────────────┐    ┌─────────────┐    ┌─────────────┐
│   Pending   │───▶│   Running   │───▶│ Succeeded/  │───▶│  Terminated │
│             │    │             │    │   Failed    │    │             │
└─────────────┘    └─────────────┘    └─────────────┘    └─────────────┘
```

#### Basic Pod YAML
```yaml
apiVersion: v1
kind: Pod
metadata:
  name: nginx-pod
  labels:
    app: nginx
  namespace: default
spec:
  containers:
  - name: nginx
    image: nginx:1.20
    ports:
    - containerPort: 80
    resources:
      requests:
        memory: "64Mi"
        cpu: "250m"
      limits:
        memory: "128Mi"
        cpu: "500m"
    env:
    - name: ENV_VAR
      value: "production"
  restartPolicy: Always
  nodeSelector:
    kubernetes.io/os: linux
```

#### Multi-container Pod Example
```yaml
apiVersion: v1
kind: Pod
metadata:
  name: multi-container-pod
spec:
  containers:
  - name: web-server
    image: nginx
    ports:
    - containerPort: 80
    volumeMounts:
    - name: shared-data
      mountPath: /usr/share/nginx/html
  - name: sidecar
    image: busybox
    command: ['sh', '-c', 'echo Hello from sidecar > /pod-data/index.html && sleep 3600']
    volumeMounts:
    - name: shared-data
      mountPath: /pod-data
  volumes:
  - name: shared-data
    emptyDir: {}
```

### 2.2 Deployments

#### Deployment Architecture
```
Deployment
├── ReplicaSet (v1)
│   ├── Pod 1
│   ├── Pod 2
│   └── Pod 3
└── ReplicaSet (v2) [Rolling Update]
    ├── Pod 1 (new)
    ├── Pod 2 (new)
    └── Pod 3 (new)
```

#### Deployment YAML
```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: nginx-deployment
  labels:
    app: nginx
spec:
  replicas: 3
  strategy:
    type: RollingUpdate
    rollingUpdate:
      maxSurge: 1
      maxUnavailable: 1
  selector:
    matchLabels:
      app: nginx
  template:
    metadata:
      labels:
        app: nginx
    spec:
      containers:
      - name: nginx
        image: nginx:1.20
        ports:
        - containerPort: 80
        readinessProbe:
          httpGet:
            path: /
            port: 80
          initialDelaySeconds: 5
          periodSeconds: 10
        livenessProbe:
          httpGet:
            path: /
            port: 80
          initialDelaySeconds: 15
          periodSeconds: 20
```

#### Deployment Management Commands
```bash
# Create deployment
kubectl create deployment nginx --image=nginx:1.20 --replicas=3

# Update deployment
kubectl set image deployment/nginx nginx=nginx:1.21
kubectl rollout restart deployment/nginx

# Scale deployment
kubectl scale deployment nginx --replicas=5

# View rollout status
kubectl rollout status deployment/nginx
kubectl rollout history deployment/nginx

# Rollback deployment
kubectl rollout undo deployment/nginx
kubectl rollout undo deployment/nginx --to-revision=2

# Pause/Resume rollout
kubectl rollout pause deployment/nginx
kubectl rollout resume deployment/nginx
```

### 2.3 DaemonSets

```yaml
apiVersion: apps/v1
kind: DaemonSet
metadata:
  name: fluentd-daemonset
  namespace: kube-system
spec:
  selector:
    matchLabels:
      name: fluentd
  template:
    metadata:
      labels:
        name: fluentd
    spec:
      tolerations:
      - key: node-role.kubernetes.io/master
        operator: Exists
        effect: NoSchedule
      containers:
      - name: fluentd
        image: fluent/fluentd:v1.14
        volumeMounts:
        - name: varlog
          mountPath: /var/log
        - name: varlibdockercontainers
          mountPath: /var/lib/docker/containers
      volumes:
      - name: varlog
        hostPath:
          path: /var/log
      - name: varlibdockercontainers
        hostPath:
          path: /var/lib/docker/containers
```

### 2.4 StatefulSets

```yaml
apiVersion: apps/v1
kind: StatefulSet
metadata:
  name: mysql-statefulset
spec:
  serviceName: "mysql"
  replicas: 3
  selector:
    matchLabels:
      app: mysql
  template:
    metadata:
      labels:
        app: mysql
    spec:
      containers:
      - name: mysql
        image: mysql:8.0
        env:
        - name: MYSQL_ROOT_PASSWORD
          valueFrom:
            secretKeyRef:
              name: mysql-secret
              key: password
        ports:
        - containerPort: 3306
        volumeMounts:
        - name: mysql-storage
          mountPath: /var/lib/mysql
  volumeClaimTemplates:
  - metadata:
      name: mysql-storage
    spec:
      accessModes: ["ReadWriteOnce"]
      resources:
        requests:
          storage: 10Gi
```

### 2.5 Jobs and CronJobs

#### Job YAML
```yaml
apiVersion: batch/v1
kind: Job
metadata:
  name: batch-job
spec:
  completions: 3
  parallelism: 2
  backoffLimit: 4
  template:
    spec:
      containers:
      - name: batch-worker
        image: busybox
        command: ["sh", "-c", "echo Processing item; sleep 30; echo Done"]
      restartPolicy: Never
```

#### CronJob YAML
```yaml
apiVersion: batch/v1
kind: CronJob
metadata:
  name: backup-cronjob
spec:
  schedule: "0 2 * * *"  # Daily at 2 AM
  jobTemplate:
    spec:
      template:
        spec:
          containers:
          - name: backup
            image: busybox
            command: ["sh", "-c", "echo Performing backup; sleep 60"]
          restartPolicy: OnFailure
  successfulJobsHistoryLimit: 3
  failedJobsHistoryLimit: 1
```

### 2.6 Pod Scheduling

#### Node Affinity
```yaml
apiVersion: v1
kind: Pod
metadata:
  name: node-affinity-pod
spec:
  affinity:
    nodeAffinity:
      requiredDuringSchedulingIgnoredDuringExecution:
        nodeSelectorTerms:
        - matchExpressions:
          - key: kubernetes.io/arch
            operator: In
            values:
            - amd64
      preferredDuringSchedulingIgnoredDuringExecution:
      - weight: 1
        preference:
          matchExpressions:
          - key: node-type
            operator: In
            values:
            - high-memory
  containers:
  - name: nginx
    image: nginx
```

#### Pod Affinity/Anti-Affinity
```yaml
apiVersion: v1
kind: Pod
metadata:
  name: pod-affinity-example
spec:
  affinity:
    podAffinity:
      requiredDuringSchedulingIgnoredDuringExecution:
      - labelSelector:
          matchExpressions:
          - key: app
            operator: In
            values:
            - database
        topologyKey: kubernetes.io/hostname
    podAntiAffinity:
      preferredDuringSchedulingIgnoredDuringExecution:
      - weight: 100
        podAffinityTerm:
          labelSelector:
            matchExpressions:
            - key: app
              operator: In
              values:
              - web
          topologyKey: kubernetes.io/hostname
  containers:
  - name: app
    image: nginx
```

#### Taints and Tolerations
```bash
# Add taint to node
kubectl taint nodes node1 key1=value1:NoSchedule

# Remove taint
kubectl taint nodes node1 key1=value1:NoSchedule-

# View node taints
kubectl describe node node1 | grep -i taint
```

```yaml
# Pod with toleration
apiVersion: v1
kind: Pod
metadata:
  name: toleration-pod
spec:
  tolerations:
  - key: "key1"
    operator: "Equal"
    value: "value1"
    effect: "NoSchedule"
  - key: "key2"
    operator: "Exists"
    effect: "NoExecute"
    tolerationSeconds: 300
  containers:
  - name: nginx
    image: nginx
```

---

## 🌐 Services & Networking

### 3.1 Service Types

#### Service Architecture Overview
```
┌─────────────────────────────────────────────────────────────┐
│                    KUBERNETES SERVICES                     │
├─────────────────────────────────────────────────────────────┤
│  ClusterIP (Default)     │  NodePort              │  LoadBalancer    │
│  ┌─────────────────┐     │  ┌───────────────┐     │  ┌─────────────┐ │
│  │   Internal      │     │  │  External     │     │  │  Cloud LB   │ │
│  │   Access Only   │     │  │  Port on      │     │  │  External   │ │
│  │                 │     │  │  All Nodes    │     │  │  IP         │ │
│  └─────────────────┘     │  └───────────────┘     │  └─────────────┘ │
│                          │                        │                  │
│                          │  ExternalName          │                  │
│                          │  ┌───────────────┐     │                  │
│                          │  │  DNS CNAME    │     │                  │
│                          │  │  Mapping      │     │                  │
│                          │  └───────────────┘     │                  │
└─────────────────────────────────────────────────────────────────────┘
```

#### ClusterIP Service
```yaml
apiVersion: v1
kind: Service
metadata:
  name: nginx-clusterip
spec:
  type: ClusterIP
  selector:
    app: nginx
  ports:
  - port: 80
    targetPort: 80
    protocol: TCP
```

#### NodePort Service
```yaml
apiVersion: v1
kind: Service
metadata:
  name: nginx-nodeport
spec:
  type: NodePort
  selector:
    app: nginx
  ports:
  - port: 80
    targetPort: 80
    nodePort: 30080
    protocol: TCP
```

#### LoadBalancer Service
```yaml
apiVersion: v1
kind: Service
metadata:
  name: nginx-loadbalancer
spec:
  type: LoadBalancer
  selector:
    app: nginx
  ports:
  - port: 80
    targetPort: 80
    protocol: TCP
```

#### Headless Service
```yaml
apiVersion: v1
kind: Service
metadata:
  name: nginx-headless
spec:
  type: ClusterIP
  clusterIP: None
  selector:
    app: nginx
  ports:
  - port: 80
    targetPort: 80
```

### 3.2 Ingress

#### Ingress Architecture
```
Internet
    |
┌───▼────┐
│        │
│ Ingress│
│Controller│
│        │
└────────┘
    |
┌───▼────┐
│        │
│ Ingress│
│Resource│
│        │
└────────┘
    |
┌───▼────┐
│Services│
└────────┘
    |
┌───▼────┐
│  Pods  │
└────────┘
```

#### Ingress Resource
```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: example-ingress
  annotations:
    kubernetes.io/ingress.class: nginx
    nginx.ingress.kubernetes.io/rewrite-target: /
spec:
  tls:
  - hosts:
    - myapp.example.com
    secretName: tls-secret
  rules:
  - host: myapp.example.com
    http:
      paths:
      - path: /app
        pathType: Prefix
        backend:
          service:
            name: app-service
            port:
              number: 80
  - host: api.example.com
    http:
      paths:
      - path: /
        pathType: Prefix
        backend:
          service:
            name: api-service
            port:
              number: 8080
```

### 3.3 Network Policies

```yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: deny-all-ingress
  namespace: production
spec:
  podSelector: {}
  policyTypes:
  - Ingress
---
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: allow-frontend-to-backend
  namespace: production
spec:
  podSelector:
    matchLabels:
      role: backend
  policyTypes:
  - Ingress
  ingress:
  - from:
    - podSelector:
        matchLabels:
          role: frontend
    ports:
    - protocol: TCP
      port: 8080
---
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: allow-egress-to-dns
  namespace: production
spec:
  podSelector: {}
  policyTypes:
  - Egress
  egress:
  - to: []
    ports:
    - protocol: UDP
      port: 53
```

### 3.4 DNS in Kubernetes

#### DNS Resolution Pattern
```
Service: nginx-service.default.svc.cluster.local
         │        │       │   │      │
         │        │       │   │      └─ Top-level domain
         │        │       │   └─ Service type
         │        │       └─ Namespace
         │        └─ Service name
         └─ Pod can resolve as: nginx-service
```

#### Custom DNS Configuration
```yaml
apiVersion: v1
kind: Pod
metadata:
  name: dns-test
spec:
  containers:
  - name: test
    image: busybox
    command: ['sh', '-c', 'sleep 3600']
  dnsPolicy: "None"
  dnsConfig:
    nameservers:
    - 8.8.8.8
    searches:
    - ns1.svc.cluster.local
    options:
    - name: ndots
      value: "2"
```

### 3.5 CNI Plugins

#### Common CNI Plugins Comparison
| Plugin | Network Model | Features |
|--------|---------------|----------|
| Flannel | Overlay (VXLAN) | Simple, lightweight |
| Calico | BGP/IPIP | Network policies, performance |
| Weave | Overlay | Encryption, mesh networking |
| Cilium | eBPF | Advanced features, observability |

---

## 💾 Storage

### 4.1 Storage Architecture

#### Storage Stack Overview
```
┌─────────────────────────────────────────────────────────────┐
│                        POD                                  │
│  ┌─────────────────────────────────────────────────────┐   │
│  │                 CONTAINER                            │   │
│  │  ┌─────────────────────────────────────────────┐    │   │
│  │  │            VOLUME MOUNT                     │    │   │
│  │  └─────────────────────────────────────────────┘    │   │
│  └─────────────────────────────────────────────────────┘   │
└─────────────────────────────────────────────────────────────┘
                           │
                           ▼
┌─────────────────────────────────────────────────────────────┐
│                  PERSISTENT VOLUME CLAIM                   │
└─────────────────────────────────────────────────────────────┘
                           │
                           ▼
┌─────────────────────────────────────────────────────────────┐
│                  PERSISTENT VOLUME                         │
│  ┌─────────────────────────────────────────────────────┐   │
│  │              STORAGE CLASS                          │   │
│  │  ┌─────────────────────────────────────────────┐    │   │
│  │  │           CSI DRIVER                        │    │   │
│  │  │  ┌─────────────────────────────────────┐    │    │   │
│  │  │  │        PHYSICAL STORAGE             │    │    │   │
│  │  │  └─────────────────────────────────────┘    │    │   │
│  │  └─────────────────────────────────────────────┘    │   │
│  └─────────────────────────────────────────────────────┘   │
└─────────────────────────────────────────────────────────────┘
```

### 4.2 Volume Types

#### EmptyDir Volume
```yaml
apiVersion: v1
kind: Pod
metadata:
  name: emptydir-pod
spec:
  containers:
  - name: writer
    image: busybox
    command: ['sh', '-c', 'echo "Hello World" > /data/hello.txt; sleep 3600']
    volumeMounts:
    - name: shared-data
      mountPath: /data
  - name: reader
    image: busybox
    command: ['sh', '-c', 'cat /data/hello.txt; sleep 3600']
    volumeMounts:
    - name: shared-data
      mountPath: /data
  volumes:
  - name: shared-data
    emptyDir: {}
```

#### HostPath Volume
```yaml
apiVersion: v1
kind: Pod
metadata:
  name: hostpath-pod
spec:
  containers:
  - name: nginx
    image: nginx
    volumeMounts:
    - name: host-volume
      mountPath: /usr/share/nginx/html
  volumes:
  - name: host-volume
    hostPath:
      path: /data
      type: Directory
```

#### ConfigMap Volume
```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: nginx-config
data:
  nginx.conf: |
    events {}
    http {
      server {
        listen 80;
        location / {
          return 200 "Hello from ConfigMap!";
        }
      }
    }
---
apiVersion: v1
kind: Pod
metadata:
  name: configmap-pod
spec:
  containers:
  - name: nginx
    image: nginx
    volumeMounts:
    - name: config-volume
      mountPath: /etc/nginx/nginx.conf
      subPath: nginx.conf
  volumes:
  - name: config-volume
    configMap:
      name: nginx-config
```

### 4.3 Persistent Volumes and Claims

#### StorageClass
```yaml
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
  name: fast-ssd
provisioner: kubernetes.io/aws-ebs
parameters:
  type: gp3
  iops: "3000"
  throughput: "125"
reclaimPolicy: Delete
allowVolumeExpansion: true
volumeBindingMode: WaitForFirstConsumer
```

#### Persistent Volume (Static Provisioning)
```yaml
apiVersion: v1
kind: PersistentVolume
metadata:
  name: mysql-pv
spec:
  capacity:
    storage: 20Gi
  accessModes:
  - ReadWriteOnce
  persistentVolumeReclaimPolicy: Retain
  storageClassName: manual
  hostPath:
    path: /data/mysql
```

#### Persistent Volume Claim
```yaml
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: mysql-pvc
spec:
  accessModes:
  - ReadWriteOnce
  resources:
    requests:
      storage: 20Gi
  storageClassName: fast-ssd
```

#### Pod Using PVC
```yaml
apiVersion: v1
kind: Pod
metadata:
  name: mysql-pod
spec:
  containers:
  - name: mysql
    image: mysql:8.0
    env:
    - name: MYSQL_ROOT_PASSWORD
      value: "password123"
    volumeMounts:
    - name: mysql-storage
      mountPath: /var/lib/mysql
  volumes:
  - name: mysql-storage
    persistentVolumeClaim:
      claimName: mysql-pvc
```

### 4.4 Storage Commands

```bash
# View storage classes
kubectl get storageclass
kubectl describe storageclass gp2

# View persistent volumes
kubectl get pv
kubectl describe pv mysql-pv

# View persistent volume claims
kubectl get pvc
kubectl describe pvc mysql-pvc

# Check volume usage in pods
kubectl describe pod mysql-pod | grep -A 5 "Volumes"

# Create PVC dynamically
kubectl apply -f - <<EOF
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: dynamic-pvc
spec:
  accessModes:
  - ReadWriteOnce
  resources:
    requests:
      storage: 10Gi
  storageClassName: gp2
EOF
```

---

# CKA Study Guide - Sections 5-8: Advanced Topics & Exam Preparation

## Section 5.1: Troubleshooting (Continued)

### Essential Troubleshooting Commands

#### Quick Diagnostic Commands
```bash
# Get cluster info and health
kubectl cluster-info
kubectl get nodes -o wide
kubectl get componentstatuses

# Check system pods
kubectl get pods -n kube-system
kubectl get events --sort-by=.metadata.creationTimestamp

# Resource usage
kubectl top nodes
kubectl top pods --all-namespaces

# Detailed resource inspection
kubectl describe node <node-name>
kubectl describe pod <pod-name> -n <namespace>
kubectl logs <pod-name> -n <namespace> --previous
```

#### Advanced Troubleshooting Flow
```
Problem Detection
       ↓
Check Events & Logs
       ↓
Verify Resource Status
       ↓
Inspect Configuration
       ↓
Test Network Connectivity
       ↓
Check Resource Constraints
       ↓
Validate Permissions
```

### Application Troubleshooting

#### Pod Troubleshooting Scenarios

**Scenario 1: Pod Stuck in Pending State**
```bash
# Check pod status
kubectl get pods -o wide

# Examine pod details
kubectl describe pod <pod-name>

# Common causes and solutions:
# 1. Resource constraints
kubectl describe nodes
kubectl top nodes

# 2. Node selector issues
kubectl get nodes --show-labels

# 3. Taints and tolerations
kubectl describe node <node-name> | grep Taints
```

**Example YAML for testing:**
```yaml
apiVersion: v1
kind: Pod
metadata:
  name: test-pod
spec:
  containers:
  - name: nginx
    image: nginx
    resources:
      requests:
        memory: "1Gi"
        cpu: "500m"
  nodeSelector:
    disktype: ssd  # This might not exist
```

**Scenario 2: Pod CrashLoopBackOff**
```bash
# Check pod logs
kubectl logs <pod-name> --previous
kubectl logs <pod-name> -c <container-name>

# Check resource limits
kubectl describe pod <pod-name>

# Common fixes:
# 1. Fix application configuration
# 2. Adjust resource limits
# 3. Fix image or command issues
```

#### Deployment Troubleshooting

**Scenario: Deployment Not Rolling Out**
```bash
# Check deployment status
kubectl get deployments
kubectl describe deployment <deployment-name>

# Check replica set
kubectl get rs
kubectl describe rs <replicaset-name>

# Check rollout status
kubectl rollout status deployment/<deployment-name>

# View rollout history
kubectl rollout history deployment/<deployment-name>

# Rollback if needed
kubectl rollout undo deployment/<deployment-name>
```

#### Service Troubleshooting

**Service Connection Issues Flowchart:**
```
Service Not Accessible
       ↓
Check Service Endpoints
kubectl get endpoints <service-name>
       ↓
Verify Pod Labels Match Service Selector
kubectl get pods --show-labels
       ↓
Test Service from Within Cluster
kubectl run test-pod --image=busybox -it --rm
       ↓
Check Service Type and Port Configuration
kubectl describe service <service-name>
```

**Testing Service Connectivity:**
```bash
# Create test pod for internal testing
kubectl run test-pod --image=busybox -it --rm -- sh

# Inside the test pod:
nslookup <service-name>
wget -qO- <service-name>:<port>
telnet <service-name> <port>

# From outside (if NodePort/LoadBalancer):
curl <node-ip>:<node-port>
```

### Network Troubleshooting

#### Network Connectivity Testing
```bash
# Test DNS resolution
kubectl exec -it <pod-name> -- nslookup kubernetes.default

# Test inter-pod communication
kubectl exec -it <pod1> -- ping <pod2-ip>

# Check network policies
kubectl get networkpolicies --all-namespaces
kubectl describe networkpolicy <policy-name>

# Verify CNI plugin status
kubectl get pods -n kube-system | grep -E "(flannel|calico|weave)"
```

#### DNS Troubleshooting
```bash
# Check CoreDNS pods
kubectl get pods -n kube-system -l k8s-app=kube-dns

# Check CoreDNS configuration
kubectl get configmap coredns -n kube-system -o yaml

# Test DNS from pod
kubectl exec -it <pod-name> -- cat /etc/resolv.conf
kubectl exec -it <pod-name> -- nslookup kubernetes.default.svc.cluster.local
```

### Storage Troubleshooting

#### PVC Issues
```bash
# Check PVC status
kubectl get pvc
kubectl describe pvc <pvc-name>

# Check storage class
kubectl get storageclass
kubectl describe storageclass <sc-name>

# Check persistent volumes
kubectl get pv
kubectl describe pv <pv-name>
```

**Common Storage Issues:**
```yaml
# Issue: PVC stuck in Pending
# Check: StorageClass exists and is available
# Check: Sufficient storage capacity
# Check: Access modes compatibility

# Example troubleshooting:
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: test-pvc
spec:
  accessModes:
    - ReadWriteOnce
  resources:
    requests:
      storage: 1Gi
  storageClassName: standard  # Ensure this exists
```

### Node Troubleshooting

#### Node Status Issues
```bash
# Check node status
kubectl get nodes -o wide
kubectl describe node <node-name>

# Check node conditions
kubectl get nodes -o jsonpath='{range .items[*]}{.metadata.name}{"\t"}{.status.conditions[?(@.type=="Ready")].status}{"\n"}{end}'

# Node not ready troubleshooting:
# 1. Check kubelet service
sudo systemctl status kubelet
sudo journalctl -u kubelet -f

# 2. Check container runtime
sudo systemctl status docker  # or containerd
sudo docker ps  # or crictl ps

# 3. Check disk space and memory
df -h
free -h
```

#### Resource Pressure Issues
```bash
# Check node resources
kubectl describe node <node-name> | grep -A 10 "Allocated resources"

# Identify pods consuming resources
kubectl top pods --all-namespaces --sort-by=cpu
kubectl top pods --all-namespaces --sort-by=memory

# Check for resource requests/limits
kubectl describe pod <pod-name> | grep -A 10 "Requests\|Limits"
```

---

## Section 6: Practical Lab Exercises

### Lab 1: Multi-Pod Application Setup (15 minutes)

**Scenario:** Deploy a web application with database backend, configure services, and troubleshoot connectivity.

#### Step-by-Step Solution:

**Step 1: Create Namespace**
```bash
kubectl create namespace webapp
```

**Step 2: Deploy MySQL Database**
```yaml
# mysql-deployment.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: mysql
  namespace: webapp
spec:
  selector:
    matchLabels:
      app: mysql
  template:
    metadata:
      labels:
        app: mysql
    spec:
      containers:
      - name: mysql
        image: mysql:5.7
        env:
        - name: MYSQL_ROOT_PASSWORD
          value: "password"
        ports:
        - containerPort: 3306
        volumeMounts:
        - name: mysql-storage
          mountPath: /var/lib/mysql
      volumes:
      - name: mysql-storage
        emptyDir: {}
---
apiVersion: v1
kind: Service
metadata:
  name: mysql-service
  namespace: webapp
spec:
  selector:
    app: mysql
  ports:
  - port: 3306
    targetPort: 3306
  type: ClusterIP
```

**Step 3: Deploy Web Application**
```yaml
# web-deployment.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: webapp
  namespace: webapp
spec:
  replicas: 3
  selector:
    matchLabels:
      app: webapp
  template:
    metadata:
      labels:
        app: webapp
    spec:
      containers:
      - name: webapp
        image: nginx
        ports:
        - containerPort: 80
        env:
        - name: DB_HOST
          value: "mysql-service"
---
apiVersion: v1
kind: Service
metadata:
  name: webapp-service
  namespace: webapp
spec:
  selector:
    app: webapp
  ports:
  - port: 80
    targetPort: 80
    nodePort: 30080
  type: NodePort
```

**Step 4: Verify Deployment**
```bash
kubectl apply -f mysql-deployment.yaml
kubectl apply -f web-deployment.yaml

# Check status
kubectl get all -n webapp
kubectl describe service webapp-service -n webapp

# Test connectivity
curl http://<node-ip>:30080
```

### Lab 2: RBAC Configuration (10 minutes)

**Scenario:** Create a service account with limited permissions for a specific namespace.

```yaml
# rbac-lab.yaml
apiVersion: v1
kind: ServiceAccount
metadata:
  name: pod-reader
  namespace: webapp
---
apiVersion: rbac.authorization.k8s.io/v1
kind: Role
metadata:
  name: pod-reader-role
  namespace: webapp
rules:
- apiGroups: [""]
  resources: ["pods"]
  verbs: ["get", "list", "watch"]
---
apiVersion: rbac.authorization.k8s.io/v1
kind: RoleBinding
metadata:
  name: pod-reader-binding
  namespace: webapp
subjects:
- kind: ServiceAccount
  name: pod-reader
  namespace: webapp
roleRef:
  kind: Role
  name: pod-reader-role
  apiGroup: rbac.authorization.k8s.io
```

**Testing RBAC:**
```bash
kubectl apply -f rbac-lab.yaml

# Test permissions
kubectl auth can-i get pods --as=system:serviceaccount:webapp:pod-reader -n webapp
kubectl auth can-i create pods --as=system:serviceaccount:webapp:pod-reader -n webapp
```

### Lab 3: Pod Scheduling and Affinity (12 minutes)

**Scenario:** Deploy pods with specific scheduling requirements.

```yaml
# scheduling-lab.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: frontend
spec:
  replicas: 3
  selector:
    matchLabels:
      app: frontend
  template:
    metadata:
      labels:
        app: frontend
    spec:
      affinity:
        podAntiAffinity:
          requiredDuringSchedulingIgnoredDuringExecution:
          - labelSelector:
              matchLabels:
                app: frontend
            topologyKey: kubernetes.io/hostname
        nodeAffinity:
          requiredDuringSchedulingIgnoredDuringExecution:
            nodeSelectorTerms:
            - matchExpressions:
              - key: node-role.kubernetes.io/worker
                operator: Exists
      containers:
      - name: nginx
        image: nginx
        resources:
          requests:
            cpu: 100m
            memory: 128Mi
          limits:
            cpu: 200m
            memory: 256Mi
```

### Lab 4: ConfigMap and Secret Management (8 minutes)

**Scenario:** Create and use ConfigMaps and Secrets in a pod.

```bash
# Create ConfigMap
kubectl create configmap app-config \
  --from-literal=database_url=mysql-service:3306 \
  --from-literal=app_mode=production

# Create Secret
kubectl create secret generic app-secret \
  --from-literal=db_password=supersecret \
  --from-literal=api_key=abc123def456
```

```yaml
# configmap-lab.yaml
apiVersion: v1
kind: Pod
metadata:
  name: config-test
spec:
  containers:
  - name: test-container
    image: busybox
    command: ['sh', '-c', 'echo "Database URL: $DATABASE_URL" && echo "Password: $DB_PASSWORD" && sleep 3600']
    env:
    - name: DATABASE_URL
      valueFrom:
        configMapKeyRef:
          name: app-config
          key: database_url
    - name: DB_PASSWORD
      valueFrom:
        secretKeyRef:
          name: app-secret
          key: db_password
    volumeMounts:
    - name: config-volume
      mountPath: /etc/config
    - name: secret-volume
      mountPath: /etc/secrets
      readOnly: true
  volumes:
  - name: config-volume
    configMap:
      name: app-config
  - name: secret-volume
    secret:
      secretName: app-secret
```

### Lab 5: Network Policy Implementation (10 minutes)

**Scenario:** Implement network policies to control traffic between pods.

```yaml
# network-policy-lab.yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: deny-all
  namespace: webapp
spec:
  podSelector: {}
  policyTypes:
  - Ingress
  - Egress
---
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: allow-frontend-to-backend
  namespace: webapp
spec:
  podSelector:
    matchLabels:
      app: mysql
  policyTypes:
  - Ingress
  ingress:
  - from:
    - podSelector:
        matchLabels:
          app: webapp
    ports:
    - protocol: TCP
      port: 3306
---
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: allow-external-to-frontend
  namespace: webapp
spec:
  podSelector:
    matchLabels:
      app: webapp
  policyTypes:
  - Ingress
  ingress:
  - ports:
    - protocol: TCP
      port: 80
```

### Lab 6: Persistent Storage Configuration (15 minutes)

**Scenario:** Set up persistent storage for a database.

```yaml
# storage-lab.yaml
apiVersion: v1
kind: PersistentVolume
metadata:
  name: mysql-pv
spec:
  capacity:
    storage: 2Gi
  accessModes:
    - ReadWriteOnce
  persistentVolumeReclaimPolicy: Retain
  storageClassName: manual
  hostPath:
    path: /mnt/mysql-data
---
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: mysql-pvc
  namespace: webapp
spec:
  accessModes:
    - ReadWriteOnce
  resources:
    requests:
      storage: 1Gi
  storageClassName: manual
---
apiVersion: apps/v1
kind: Deployment
metadata:
  name: mysql-persistent
  namespace: webapp
spec:
  selector:
    matchLabels:
      app: mysql-persistent
  template:
    metadata:
      labels:
        app: mysql-persistent
    spec:
      containers:
      - name: mysql
        image: mysql:5.7
        env:
        - name: MYSQL_ROOT_PASSWORD
          value: "password"
        ports:
        - containerPort: 3306
        volumeMounts:
        - name: mysql-persistent-storage
          mountPath: /var/lib/mysql
      volumes:
      - name: mysql-persistent-storage
        persistentVolumeClaim:
          claimName: mysql-pvc
```

### Time-Bound Practice Exercises

#### Exercise 1: Emergency Pod Recovery (5 minutes)
```bash
# Scenario: Critical pod is failing, need to recover quickly

# Task: Find and fix a failing pod in the 'production' namespace
# 1. Identify the failing pod
kubectl get pods -n production

# 2. Get more details
kubectl describe pod <failing-pod> -n production

# 3. Check logs
kubectl logs <failing-pod> -n production --previous

# 4. Quick fix strategies:
# - Restart deployment: kubectl rollout restart deployment/<name> -n production
# - Scale down/up: kubectl scale deployment/<name> --replicas=0 -n production; kubectl scale deployment/<name> --replicas=3 -n production
# - Edit resource on the fly: kubectl edit pod <name> -n production
```

#### Exercise 2: Service Exposure (3 minutes)
```bash
# Task: Expose a deployment as NodePort service
kubectl create deployment nginx --image=nginx --replicas=3
kubectl expose deployment nginx --port=80 --target-port=80 --type=NodePort --name=nginx-service

# Verify
kubectl get service nginx-service
```

#### Exercise 3: Quick Troubleshooting (7 minutes)
```bash
# Scenario: Users report application is slow

# 1. Check resource usage (1 min)
kubectl top nodes
kubectl top pods --all-namespaces --sort-by=cpu

# 2. Check for resource constraints (2 min)
kubectl describe nodes | grep -A 5 "Allocated resources"

# 3. Check application logs (2 min)
kubectl logs -l app=<app-name> --tail=50

# 4. Check service connectivity (2 min)
kubectl get endpoints
kubectl run test-pod --image=busybox -it --rm -- wget -qO- <service-name>
```

---

## Section 7: Exam Tips & Strategies

### Time Management Techniques

#### Exam Time Allocation (180 minutes total)
```
Quick Win Questions (20-30 mins):
- Simple kubectl commands
- YAML creation/editing
- Basic troubleshooting

Medium Complexity (90-100 mins):
- Multi-step configurations
- RBAC setup
- Network policies
- Storage configuration

Complex Scenarios (50-60 mins):
- Cluster troubleshooting
- Multi-component applications
- Advanced scheduling

Final Review (10 mins):
- Double-check answers
- Complete any skipped questions
```

#### Time-Saving Strategies
1. **Read all questions first** (5 minutes) - Mark easy ones
2. **Start with high-value, quick questions**
3. **Use bookmarks** for questions to return to
4. **Don't get stuck** - move on after 10-15 minutes
5. **Use kubectl shortcuts** extensively

### kubectl Shortcuts and Aliases

#### Essential Aliases
```bash
# Add to ~/.bashrc or ~/.zshrc
alias k='kubectl'
alias kg='kubectl get'
alias kd='kubectl describe'
alias kdel='kubectl delete'
alias ka='kubectl apply'
alias kl='kubectl logs'
alias ke='kubectl edit'
alias kx='kubectl exec'

# Namespace shortcuts
alias kgp='kubectl get pods'
alias kgs='kubectl get services'
alias kgd='kubectl get deployments'
alias kgn='kubectl get nodes'

# With output formatting
alias kgpo='kubectl get pods -o wide'
alias kgno='kubectl get nodes -o wide'
```

#### Efficient kubectl Usage
```bash
# Use short names
kubectl get po          # instead of pods
kubectl get svc         # instead of services
kubectl get deploy      # instead of deployments
kubectl get ns          # instead of namespaces

# Useful flags
-o wide                 # More details
-o yaml                 # YAML output
-o json                 # JSON output
--dry-run=client -o yaml # Generate YAML
--watch                 # Watch for changes
--show-labels           # Show labels
--selector app=nginx    # Label selector
-n namespace           # Specific namespace
--all-namespaces       # All namespaces

# Powerful combinations
kubectl get events --sort-by=.metadata.creationTimestamp
kubectl get pods --field-selector=status.phase=Failed
kubectl get pods -o jsonpath='{.items[*].metadata.name}'
```

#### Resource Creation Shortcuts
```bash
# Create deployments quickly
kubectl create deployment nginx --image=nginx --replicas=3 --dry-run=client -o yaml > deployment.yaml

# Create services
kubectl expose deployment nginx --port=80 --type=NodePort --dry-run=client -o yaml > service.yaml

# Create configmaps
kubectl create configmap myconfig --from-literal=key1=value1 --dry-run=client -o yaml > configmap.yaml

# Create secrets
kubectl create secret generic mysecret --from-literal=password=secret123 --dry-run=client -o yaml > secret.yaml

# Create jobs
kubectl create job myjob --image=busybox -- echo "Hello World" --dry-run=client -o yaml > job.yaml

# Create cronjobs
kubectl create cronjob mycron --image=busybox --schedule="0 */2 * * *" -- echo "Hello" --dry-run=client -o yaml > cronjob.yaml
```

### Exam Environment Navigation

#### Browser and Terminal Tips
```bash
# Terminal shortcuts
Ctrl+A          # Beginning of line
Ctrl+E          # End of line
Ctrl+U          # Clear line
Ctrl+L          # Clear screen
Ctrl+R          # Search command history
Ctrl+C          # Cancel current command

# Copy/Paste in exam environment
Ctrl+Shift+C    # Copy
Ctrl+Shift+V    # Paste

# Multiple terminal tabs
Ctrl+Shift+T    # New tab
Ctrl+PageUp     # Previous tab
Ctrl+PageDown   # Next tab
```

#### Documentation Usage
```bash
# Bookmark these pages:
# - kubectl Cheat Sheet
# - Pod specifications
# - Service specifications
# - ConfigMap and Secret docs
# - RBAC documentation

# Search effectively:
# Use Ctrl+F to find specific sections
# Look for "Example" sections in docs
# Copy and modify existing examples
```

### Common Pitfalls and How to Avoid Them

#### YAML Formatting Issues
```yaml
# WRONG - Incorrect indentation
apiVersion: v1
kind: Pod
metadata:
name: test-pod    # Should be indented
spec:
containers:      # Missing dash for list
name: nginx      # Should be indented under containers
image: nginx

# CORRECT - Proper indentation
apiVersion: v1
kind: Pod
metadata:
  name: test-pod
spec:
  containers:
  - name: nginx
    image: nginx
```

#### Common kubectl Mistakes
```bash
# WRONG
kubectl get pod nginx -o yaml > pod.yaml    # Missing 's' in pods
kubectl apply -f pod.yml                    # Wrong file extension
kubectl delete po nginx --namespace=default # Verbose namespace flag

# CORRECT
kubectl get pods nginx -o yaml > pod.yaml
kubectl apply -f pod.yaml
kubectl delete po nginx -n default
```

#### Resource Management Errors
```bash
# Always specify namespace when required
kubectl get pods -n kube-system

# Check resource quotas before creating
kubectl describe quota -n <namespace>

# Verify resource requests/limits
kubectl describe pod <pod-name> | grep -A 10 "Requests\|Limits"
```

### Last-Minute Review Checklist

#### 15 Minutes Before Exam
- [ ] Clear browser cache and cookies
- [ ] Test copy/paste functionality
- [ ] Verify kubectl is working: `kubectl version`
- [ ] Check cluster access: `kubectl get nodes`
- [ ] Review question format and navigation

#### Core Commands to Remember
```bash
# Essential troubleshooting
kubectl get events --sort-by=.metadata.creationTimestamp
kubectl describe <resource> <name>
kubectl logs <pod> -f --previous

# Quick resource creation
kubectl run <name> --image=<image> --dry-run=client -o yaml
kubectl create deployment <name> --image=<image> --dry-run=client -o yaml
kubectl expose deployment <name> --port=<port> --type=<type>

# Editing resources
kubectl edit <resource> <name>
kubectl patch <resource> <name> -p '{"spec":{"replicas":3}}'
kubectl scale deployment <name> --replicas=<number>

# RBAC verification
kubectl auth can-i <verb> <resource> --as=<user>
```

#### Final Mental Checklist
- [ ] Always read the question twice
- [ ] Note the namespace requirement
- [ ] Check if dry-run is needed
- [ ] Verify your answer before submitting
- [ ] Use `kubectl get` to confirm resources were created
- [ ] Remember that partial credit is given

---

## Section 8: Resources & References

### Official Kubernetes Documentation

#### Core Documentation Links
- **Main Documentation**: https://kubernetes.io/docs/
- **kubectl Cheat Sheet**: https://kubernetes.io/docs/reference/kubectl/cheatsheet/
- **API Reference**: https://kubernetes.io/docs/reference/generated/kubernetes-api/v1.28/
- **kubectl Reference**: https://kubernetes.io/docs/reference/generated/kubectl/kubectl-commands

#### Specific Topic Documentation
```
Pods:
https://kubernetes.io/docs/concepts/workloads/pods/

Deployments:
https://kubernetes.io/docs/concepts/workloads/controllers/deployment/

Services:
https://kubernetes.io/docs/concepts/services-networking/service/

ConfigMaps and Secrets:
https://kubernetes.io/docs/concepts/configuration/configmap/
https://kubernetes.io/docs/concepts/configuration/secret/

Persistent Volumes:
https://kubernetes.io/docs/concepts/storage/persistent-volumes/

RBAC:
https://kubernetes.io/docs/reference/access-authn-authz/rbac/

Network Policies:
https://kubernetes.io/docs/concepts/services-networking/network-policies/

Scheduling:
https://kubernetes.io/docs/concepts/scheduling-eviction/assign-pod-node/

Troubleshooting:
https://kubernetes.io/docs/tasks/debug-application-cluster/troubleshooting/
```

### Practice Platforms and Mock Exams

#### Recommended Practice Platforms
```
1. Killer.sh (killer.sh/cka)
   - Most realistic exam simulation
   - 36-hour access with two sessions
   - Detailed explanations

2. Katacoda Kubernetes Playground
   - Free interactive scenarios
   - Browser-based clusters
   - Good for specific topic practice

3. Play with Kubernetes (labs.play-with-k8s.com)
   - Free 4-hour sessions
   - Multi-node clusters
   - Community-driven scenarios

4. KodeKloud CKA Course
   - Comprehensive video tutorials
   - Hands-on labs
   - Practice exams

5. Linux Foundation Training (training.linuxfoundation.org)
   - Official CKA preparation course
   - Includes exam voucher
   - Expert-led training
```

#### Self-Practice Setup
```bash
# Local cluster options:
# 1. minikube
minikube start --nodes=3

# 2. kind (Kubernetes in Docker)
kind create cluster --config=multi-node-config.yaml

# 3. k3s (lightweight Kubernetes)
curl -sfL https://get.k3s.io | sh -

# Practice cluster configuration:
# - Multi-node setup (minimum 3 nodes)
# - Different node roles (master, worker)
# - Network plugin (Flannel, Calico, Weave)
# - Storage classes configured
```

### Useful kubectl Cheat Sheets

#### Resource Management Cheat Sheet
```bash
# Create resources
kubectl create -f <file>
kubectl apply -f <file>
kubectl create deployment <name> --image=<image>
kubectl run <name> --image=<image>

# View resources
kubectl get <resource>
kubectl get <resource> -o wide
kubectl get <resource> -o yaml
kubectl get <resource> --selector=<selector>
kubectl get events --sort-by=.metadata.creationTimestamp

# Describe resources
kubectl describe <resource> <name>
kubectl describe <resource> --selector=<selector>

# Edit resources
kubectl edit <resource> <name>
kubectl patch <resource> <name> -p '<json-patch>'
kubectl annotate <resource> <name> <annotation>
kubectl label <resource> <name> <label>

# Delete resources
kubectl delete <resource> <name>
kubectl delete <resource> --selector=<selector>
kubectl delete -f <file>

# Scale resources
kubectl scale deployment <name> --replicas=<number>
kubectl autoscale deployment <name> --min=<min> --max=<max> --cpu-percent=<cpu>
```

#### Debugging and Troubleshooting Cheat Sheet
```bash
# Pod debugging
kubectl logs <pod-name> -f
kubectl logs <pod-name> --previous
kubectl logs <pod-name> -c <container-name>
kubectl exec -it <pod-name> -- <command>
kubectl exec -it <pod-name> -c <container-name> -- <command>

# Service debugging
kubectl get endpoints <service-name>
kubectl port-forward <pod-name> <local-port>:<remote-port>
kubectl port-forward service/<service-name> <local-port>:<remote-port>

# Node debugging
kubectl get nodes -o wide
kubectl describe node <node-name>
kubectl top nodes
kubectl top pods --all-namespaces

# Cluster debugging
kubectl cluster-info
kubectl get componentstatuses
kubectl get events --all-namespaces --sort-by=.metadata.creationTimestamp
```

### Community Resources

#### Kubernetes Community Platforms
```
1. Kubernetes Slack (slack.k8s.io)
   - #kubernetes-novice (beginner questions)
   - #cka-exam-prep (CKA specific discussions)
   - #kubectl (kubectl usage help)

2. Reddit Communities
   - r/kubernetes (general discussions)
   - r/devops (broader DevOps context)

3. Stack Overflow
   - Tag: kubernetes
   - Tag: kubectl
   - Active community support

4. GitHub Repositories
   - kubernetes/kubernetes (main repository)
   - kubernetes/website (documentation)
   - cncf/curriculum (exam curriculum)
```

#### Useful GitHub Resources
```
1. CKA Study Guides:
   - github.com/walidshaari/Kubernetes-Certified-Administrator
   - github.com/bmuschko/cka-study-guide
   - github.com/David-VTUK/CKA-StudyGuide

2. Practice Exercises:
   - github.com/dgkanatsios/CKAD-exercises
   - github.com/alijahnas/CKA-practice-exercises

3. YAML Examples:
   - github.com/kubernetes/examples
   - github.com/kubernetes/ingress-nginx/tree/main/docs/examples
```

#### Blogs and Articles
```
1. Official Kubernetes Blog (kubernetes.io/blog/)
   - Latest features and updates
   - Best practices articles
   - Community spotlights

2. CNCF Blog (www.cncf.io/blog/)
   - Cloud native ecosystem updates
   - Project announcements
   - Technical deep dives

3. Individual Tech Blogs:
   - Kelsey Hightower (@kelseyhightower)
   - Brendan Burns (@brendandburns)
   - Tim Hockin (@thockin)
```

### Final Preparation Timeline

#### 2 Weeks Before Exam
- [ ] Complete all practice labs
- [ ] Take first mock exam
- [ ] Identify weak areas
- [ ] Review documentation for weak topics
- [ ] Set up local practice environment

#### 1 Week Before Exam
- [ ] Take second mock exam
- [ ] Practice time management
- [ ] Review all kubectl shortcuts
- [ ] Study troubleshooting scenarios
- [ ] Practice YAML creation without docs

#### 3 Days Before Exam
- [ ] Final mock exam
- [ ] Review common pitfalls
- [ ] Practice browser navigation
- [ ] Verify exam system requirements
- [ ] Get good rest

#### Day of Exam
- [ ] Arrive 15 minutes early
- [ ] Clear workspace
- [ ] Have government ID ready
- [ ] Test camera and microphone
- [ ] Take deep breaths and stay calm

### Success Metrics and Goals

#### Target Scores by Topic
```
Troublesho