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

## 🔧 Troubleshooting

### 5.1 Cluster Component Troubleshooting

#### System Pods Health Check
```bash
# Check all system pods
kubectl get pods -n kube-system
kubectl get pods -n kube-system -o wide

# Check specific components
kubectl logs