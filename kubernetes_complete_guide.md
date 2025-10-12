# Complete Kubernetes Guide: Beginner to Advanced

## Table of Contents

1. [Introduction to Kubernetes](#introduction-to-kubernetes)
2. [Beginner Level](#beginner-level)
   - [Installation and Setup](#installation-and-setup)
   - [Basic Concepts](#basic-concepts)
   - [Your First Pod](#your-first-pod)
   - [Services and Networking](#services-and-networking)
   - [Basic kubectl Commands](#basic-kubectl-commands)
3. [Intermediate Level](#intermediate-level)
   - [Deployments and ReplicaSets](#deployments-and-replicasets)
   - [ConfigMaps and Secrets](#configmaps-and-secrets)
   - [Volumes and Storage](#volumes-and-storage)
   - [Namespaces](#namespaces)
   - [Ingress Controllers](#ingress-controllers)
4. [Advanced Level](#advanced-level)
   - [StatefulSets and DaemonSets](#statefulsets-and-daemonsets)
   - [Jobs and CronJobs](#jobs-and-cronjobs)
   - [Auto-scaling (HPA/VPA)](#auto-scaling)
   - [RBAC and Security](#rbac-and-security)
   - [Custom Resources and Operators](#custom-resources-and-operators)
5. [Production Deployment](#production-deployment)
6. [Monitoring and Observability](#monitoring-and-observability)
7. [Best Practices](#best-practices)
8. [Troubleshooting](#troubleshooting)

---

## Introduction to Kubernetes

Kubernetes (K8s) is an open-source container orchestration platform that automates deployment, scaling, and management of containerized applications. It provides a robust platform for running distributed systems resiliently.

### Key Benefits:
- **Container Orchestration**: Manages container lifecycle
- **Service Discovery**: Automatic service discovery and load balancing
- **Auto-scaling**: Horizontal and vertical scaling
- **Self-healing**: Automatic restart, replacement, and rescheduling
- **Rolling Updates**: Zero-downtime deployments
- **Storage Orchestration**: Automatic mounting of storage systems

### Architecture Overview:
- **Master Node**: API Server, etcd, Controller Manager, Scheduler
- **Worker Nodes**: kubelet, kube-proxy, Container Runtime
- **Pods**: Smallest deployable units
- **Services**: Network abstraction for Pods
- **Ingress**: External access to services

---

## Beginner Level

### Installation and Setup

#### Local Development Options

##### 1. Minikube (Recommended for beginners)
```bash
# Install Minikube (Linux)
curl -Lo minikube https://storage.googleapis.com/minikube/releases/latest/minikube-linux-amd64
chmod +x minikube
sudo mv minikube /usr/local/bin/

# Start Minikube
minikube start

# Enable addons
minikube addons enable dashboard
minikube addons enable ingress

# Access dashboard
minikube dashboard
```

##### 2. Kind (Kubernetes in Docker)
```bash
# Install Kind
go install sigs.k8s.io/kind@v0.20.0

# Create cluster
kind create cluster --name my-cluster

# Create cluster with config
cat <<EOF | kind create cluster --config=-
kind: Cluster
apiVersion: kind.x-k8s.io/v1alpha4
nodes:
- role: control-plane
- role: worker
- role: worker
EOF
```

##### 3. Docker Desktop
```bash
# Enable Kubernetes in Docker Desktop settings
# Docker Desktop > Settings > Kubernetes > Enable Kubernetes
```

#### kubectl Installation
```bash
# Linux
curl -LO "https://dl.k8s.io/release/$(curl -L -s https://dl.k8s.io/release/stable.txt)/bin/linux/amd64/kubectl"
chmod +x kubectl
sudo mv kubectl /usr/local/bin/

# macOS
brew install kubectl

# Windows (PowerShell)
Invoke-WebRequest -Uri "https://dl.k8s.io/release/v1.28.0/bin/windows/amd64/kubectl.exe" -OutFile "kubectl.exe"

# Verify installation
kubectl version --client
```

#### Configure kubectl
```bash
# View current context
kubectl config current-context

# List all contexts
kubectl config get-contexts

# Switch context
kubectl config use-context minikube

# Set default namespace
kubectl config set-context --current --namespace=my-namespace
```

### Basic Concepts

#### Pods
- Smallest deployable unit in Kubernetes
- Can contain one or more containers
- Containers in a pod share network and storage

#### Services
- Stable network endpoint for pods
- Load balances traffic across pods
- Types: ClusterIP, NodePort, LoadBalancer, ExternalName

#### Deployments
- Manages pod replicas
- Provides declarative updates
- Supports rolling updates and rollbacks

#### Namespaces
- Virtual clusters within a physical cluster
- Provides resource isolation
- Default namespaces: default, kube-system, kube-public

### Your First Pod

#### Simple Pod Definition
```yaml
# pod.yaml
apiVersion: v1
kind: Pod
metadata:
  name: my-pod
  labels:
    app: my-app
spec:
  containers:
  - name: nginx
    image: nginx:1.20
    ports:
    - containerPort: 80
```

#### Create and Manage Pod
```bash
# Create pod
kubectl apply -f pod.yaml

# List pods
kubectl get pods

# Describe pod
kubectl describe pod my-pod

# Get pod logs
kubectl logs my-pod

# Execute command in pod
kubectl exec -it my-pod -- /bin/bash

# Delete pod
kubectl delete pod my-pod
```

#### Multi-Container Pod
```yaml
# multi-container-pod.yaml
apiVersion: v1
kind: Pod
metadata:
  name: multi-container-pod
spec:
  containers:
  - name: web
    image: nginx:1.20
    ports:
    - containerPort: 80
    volumeMounts:
    - name: shared-data
      mountPath: /usr/share/nginx/html
  
  - name: sidecar
    image: busybox
    command: ["/bin/sh"]
    args: ["-c", "while true; do echo $(date) >> /data/index.html; sleep 60; done"]
    volumeMounts:
    - name: shared-data
      mountPath: /data
  
  volumes:
  - name: shared-data
    emptyDir: {}
```

### Services and Networking

#### ClusterIP Service (Default)
```yaml
# service-clusterip.yaml
apiVersion: v1
kind: Service
metadata:
  name: my-service
spec:
  selector:
    app: my-app
  ports:
  - protocol: TCP
    port: 80
    targetPort: 8080
  type: ClusterIP
```

#### NodePort Service
```yaml
# service-nodeport.yaml
apiVersion: v1
kind: Service
metadata:
  name: my-nodeport-service
spec:
  selector:
    app: my-app
  ports:
  - protocol: TCP
    port: 80
    targetPort: 8080
    nodePort: 30080
  type: NodePort
```

#### LoadBalancer Service
```yaml
# service-loadbalancer.yaml
apiVersion: v1
kind: Service
metadata:
  name: my-loadbalancer-service
spec:
  selector:
    app: my-app
  ports:
  - protocol: TCP
    port: 80
    targetPort: 8080
  type: LoadBalancer
```

#### Service Management Commands
```bash
# List services
kubectl get services

# Describe service
kubectl describe service my-service

# Get service endpoints
kubectl get endpoints my-service

# Port forwarding
kubectl port-forward service/my-service 8080:80

# Test service connectivity
kubectl run test-pod --image=busybox --rm -it --restart=Never -- wget -qO- my-service
```

### Basic kubectl Commands

#### Resource Management
```bash
# Apply configuration
kubectl apply -f file.yaml
kubectl apply -f directory/

# Create resource
kubectl create -f file.yaml

# Delete resource
kubectl delete -f file.yaml
kubectl delete pod my-pod
kubectl delete service my-service

# Replace resource
kubectl replace -f file.yaml
```

#### Information Commands
```bash
# Get resources
kubectl get pods
kubectl get services
kubectl get deployments
kubectl get all

# Describe resources
kubectl describe pod my-pod
kubectl describe node node-name

# Show labels
kubectl get pods --show-labels

# Filter by labels
kubectl get pods -l app=my-app

# Get resource YAML/JSON
kubectl get pod my-pod -o yaml
kubectl get pod my-pod -o json
```

#### Debugging Commands
```bash
# View logs
kubectl logs pod-name
kubectl logs -f pod-name  # follow logs
kubectl logs pod-name -c container-name  # specific container

# Execute commands
kubectl exec -it pod-name -- /bin/bash
kubectl exec pod-name -- ls /

# Copy files
kubectl cp pod-name:/path/file.txt ./file.txt
kubectl cp ./file.txt pod-name:/path/

# Resource usage
kubectl top nodes
kubectl top pods
```

---

## Intermediate Level

### Deployments and ReplicaSets

#### Basic Deployment
```yaml
# deployment.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: nginx-deployment
  labels:
    app: nginx
spec:
  replicas: 3
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
        resources:
          requests:
            memory: "64Mi"
            cpu: "250m"
          limits:
            memory: "128Mi"
            cpu: "500m"
```

#### Advanced Deployment Features
```yaml
# advanced-deployment.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: web-app
  labels:
    app: web-app
spec:
  replicas: 5
  strategy:
    type: RollingUpdate
    rollingUpdate:
      maxSurge: 1
      maxUnavailable: 1
  selector:
    matchLabels:
      app: web-app
  template:
    metadata:
      labels:
        app: web-app
    spec:
      containers:
      - name: web
        image: nginx:1.20
        ports:
        - containerPort: 80
        livenessProbe:
          httpGet:
            path: /
            port: 80
          initialDelaySeconds: 30
          periodSeconds: 30
        readinessProbe:
          httpGet:
            path: /
            port: 80
          initialDelaySeconds: 5
          periodSeconds: 5
        resources:
          requests:
            memory: "128Mi"
            cpu: "100m"
          limits:
            memory: "256Mi"
            cpu: "200m"
```

#### Deployment Management
```bash
# Create deployment
kubectl apply -f deployment.yaml

# Scale deployment
kubectl scale deployment nginx-deployment --replicas=5

# Update deployment image
kubectl set image deployment/nginx-deployment nginx=nginx:1.21

# Rolling update status
kubectl rollout status deployment/nginx-deployment

# Rollback deployment
kubectl rollout undo deployment/nginx-deployment

# View rollout history
kubectl rollout history deployment/nginx-deployment

# Pause/Resume rollout
kubectl rollout pause deployment/nginx-deployment
kubectl rollout resume deployment/nginx-deployment
```

### ConfigMaps and Secrets

#### ConfigMap Examples
```yaml
# configmap.yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: app-config
data:
  app.properties: |
    debug=true
    database.host=localhost
    database.port=5432
  log-level: "INFO"
  max-connections: "100"
```

```yaml
# configmap-from-file.yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: nginx-config
data:
  nginx.conf: |
    server {
        listen 80;
        location / {
            proxy_pass http://backend;
        }
    }
```

#### Using ConfigMap in Pods
```yaml
# pod-with-configmap.yaml
apiVersion: v1
kind: Pod
metadata:
  name: app-pod
spec:
  containers:
  - name: app
    image: busybox
    command: ["/bin/sh", "-c", "env"]
    env:
    - name: LOG_LEVEL
      valueFrom:
        configMapKeyRef:
          name: app-config
          key: log-level
    envFrom:
    - configMapRef:
        name: app-config
    volumeMounts:
    - name: config-volume
      mountPath: /etc/config
  volumes:
  - name: config-volume
    configMap:
      name: app-config
```

#### Secrets
```yaml
# secret.yaml
apiVersion: v1
kind: Secret
metadata:
  name: app-secret
type: Opaque
data:
  username: YWRtaW4=  # admin (base64 encoded)
  password: cGFzc3dvcmQ=  # password (base64 encoded)
```

```yaml
# pod-with-secret.yaml
apiVersion: v1
kind: Pod
metadata:
  name: secret-pod
spec:
  containers:
  - name: app
    image: nginx
    env:
    - name: USERNAME
      valueFrom:
        secretKeyRef:
          name: app-secret
          key: username
    volumeMounts:
    - name: secret-volume
      mountPath: /etc/secrets
      readOnly: true
  volumes:
  - name: secret-volume
    secret:
      secretName: app-secret
```

#### ConfigMap and Secret Management
```bash
# Create ConfigMap from literals
kubectl create configmap app-config --from-literal=key1=value1 --from-literal=key2=value2

# Create ConfigMap from file
kubectl create configmap nginx-config --from-file=nginx.conf

# Create Secret from literals
kubectl create secret generic app-secret --from-literal=username=admin --from-literal=password=secret

# Create Secret from file
kubectl create secret generic ssl-secret --from-file=tls.crt --from-file=tls.key

# Encode/Decode base64
echo -n 'admin' | base64
echo 'YWRtaW4=' | base64 --decode
```

### Volumes and Storage

#### EmptyDir Volume
```yaml
# emptydir-volume.yaml
apiVersion: v1
kind: Pod
metadata:
  name: emptydir-pod
spec:
  containers:
  - name: writer
    image: busybox
    command: ["/bin/sh", "-c", "echo 'Hello World' > /data/hello.txt && sleep 3600"]
    volumeMounts:
    - name: shared-data
      mountPath: /data
  - name: reader
    image: busybox
    command: ["/bin/sh", "-c", "cat /data/hello.txt && sleep 3600"]
    volumeMounts:
    - name: shared-data
      mountPath: /data
  volumes:
  - name: shared-data
    emptyDir: {}
```

#### HostPath Volume
```yaml
# hostpath-volume.yaml
apiVersion: v1
kind: Pod
metadata:
  name: hostpath-pod
spec:
  containers:
  - name: app
    image: nginx
    volumeMounts:
    - name: host-volume
      mountPath: /usr/share/nginx/html
  volumes:
  - name: host-volume
    hostPath:
      path: /tmp/nginx-html
      type: DirectoryOrCreate
```

#### PersistentVolume and PersistentVolumeClaim
```yaml
# persistent-volume.yaml
apiVersion: v1
kind: PersistentVolume
metadata:
  name: pv-storage
spec:
  capacity:
    storage: 10Gi
  accessModes:
    - ReadWriteOnce
  hostPath:
    path: /tmp/pv-data

---
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: pvc-storage
spec:
  accessModes:
    - ReadWriteOnce
  resources:
    requests:
      storage: 5Gi
```

```yaml
# pod-with-pvc.yaml
apiVersion: v1
kind: Pod
metadata:
  name: pvc-pod
spec:
  containers:
  - name: app
    image: nginx
    volumeMounts:
    - name: storage
      mountPath: /usr/share/nginx/html
  volumes:
  - name: storage
    persistentVolumeClaim:
      claimName: pvc-storage
```

#### StorageClass
```yaml
# storage-class.yaml
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
  name: fast-storage
provisioner: kubernetes.io/aws-ebs
parameters:
  type: gp2
  zones: us-west-2a
reclaimPolicy: Delete
allowVolumeExpansion: true
```

### Namespaces

#### Create and Manage Namespaces
```yaml
# namespace.yaml
apiVersion: v1
kind: Namespace
metadata:
  name: development
  labels:
    environment: dev
```

```bash
# Create namespace
kubectl create namespace production

# List namespaces
kubectl get namespaces

# Set default namespace
kubectl config set-context --current --namespace=development

# Delete namespace
kubectl delete namespace development

# Create resources in specific namespace
kubectl apply -f deployment.yaml -n production
```

#### Resource Quotas
```yaml
# resource-quota.yaml
apiVersion: v1
kind: ResourceQuota
metadata:
  name: compute-quota
  namespace: development
spec:
  hard:
    requests.cpu: "1"
    requests.memory: 1Gi
    limits.cpu: "2"
    limits.memory: 2Gi
    persistentvolumeclaims: "4"
    pods: "10"
```

#### Limit Ranges
```yaml
# limit-range.yaml
apiVersion: v1
kind: LimitRange
metadata:
  name: mem-limit-range
  namespace: development
spec:
  limits:
  - default:
      memory: "512Mi"
      cpu: "500m"
    defaultRequest:
      memory: "256Mi"
      cpu: "100m"
    type: Container
```

### Ingress Controllers

#### Basic Ingress
```yaml
# ingress.yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: web-ingress
  annotations:
    kubernetes.io/ingress.class: nginx
spec:
  rules:
  - host: myapp.example.com
    http:
      paths:
      - path: /
        pathType: Prefix
        backend:
          service:
            name: web-service
            port:
              number: 80
```

#### Advanced Ingress with TLS
```yaml
# ingress-tls.yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: secure-ingress
  annotations:
    kubernetes.io/ingress.class: nginx
    cert-manager.io/cluster-issuer: letsencrypt-prod
    nginx.ingress.kubernetes.io/ssl-redirect: "true"
spec:
  tls:
  - hosts:
    - myapp.example.com
    - api.example.com
    secretName: tls-secret
  rules:
  - host: myapp.example.com
    http:
      paths:
      - path: /
        pathType: Prefix
        backend:
          service:
            name: frontend-service
            port:
              number: 80
  - host: api.example.com
    http:
      paths:
      - path: /api
        pathType: Prefix
        backend:
          service:
            name: api-service
            port:
              number: 8080
```

#### Install NGINX Ingress Controller
```bash
# Using Helm
helm repo add ingress-nginx https://kubernetes.github.io/ingress-nginx
helm repo update
helm install ingress-nginx ingress-nginx/ingress-nginx

# Using kubectl
kubectl apply -f https://raw.githubusercontent.com/kubernetes/ingress-nginx/controller-v1.8.1/deploy/static/provider/cloud/deploy.yaml

# Check installation
kubectl get pods -n ingress-nginx
kubectl get services -n ingress-nginx
```

---

## Advanced Level

### StatefulSets and DaemonSets

#### StatefulSet Example
```yaml
# statefulset.yaml
apiVersion: apps/v1
kind: StatefulSet
metadata:
  name: mongodb
spec:
  serviceName: mongodb-service
  replicas: 3
  selector:
    matchLabels:
      app: mongodb
  template:
    metadata:
      labels:
        app: mongodb
    spec:
      containers:
      - name: mongodb
        image: mongo:4.4
        ports:
        - containerPort: 27017
        volumeMounts:
        - name: mongodb-storage
          mountPath: /data/db
        env:
        - name: MONGO_INITDB_ROOT_USERNAME
          value: admin
        - name: MONGO_INITDB_ROOT_PASSWORD
          valueFrom:
            secretKeyRef:
              name: mongodb-secret
              key: password
  volumeClaimTemplates:
  - metadata:
      name: mongodb-storage
    spec:
      accessModes: ["ReadWriteOnce"]
      resources:
        requests:
          storage: 10Gi
```

#### Headless Service for StatefulSet
```yaml
# headless-service.yaml
apiVersion: v1
kind: Service
metadata:
  name: mongodb-service
spec:
  clusterIP: None
  selector:
    app: mongodb
  ports:
  - port: 27017
    targetPort: 27017
```

#### DaemonSet Example
```yaml
# daemonset.yaml
apiVersion: apps/v1
kind: DaemonSet
metadata:
  name: log-collector
  labels:
    app: log-collector
spec:
  selector:
    matchLabels:
      app: log-collector
  template:
    metadata:
      labels:
        app: log-collector
    spec:
      tolerations:
      - key: node-role.kubernetes.io/master
        effect: NoSchedule
      containers:
      - name: fluentd
        image: fluent/fluentd:v1.14
        resources:
          limits:
            memory: 200Mi
          requests:
            cpu: 100m
            memory: 200Mi
        volumeMounts:
        - name: varlog
          mountPath: /var/log
        - name: varlibdockercontainers
          mountPath: /var/lib/docker/containers
          readOnly: true
      volumes:
      - name: varlog
        hostPath:
          path: /var/log
      - name: varlibdockercontainers
        hostPath:
          path: /var/lib/docker/containers
```

### Jobs and CronJobs

#### Job Example
```yaml
# job.yaml
apiVersion: batch/v1
kind: Job
metadata:
  name: data-migration
spec:
  completions: 1
  parallelism: 1
  backoffLimit: 3
  template:
    spec:
      restartPolicy: Never
      containers:
      - name: migration
        image: postgres:13
        command: ["/bin/sh"]
        args:
        - -c
        - |
          psql -h $DB_HOST -U $DB_USER -d $DB_NAME -c "
          CREATE TABLE IF NOT EXISTS migrations (
            id SERIAL PRIMARY KEY,
            name VARCHAR(255),
            executed_at TIMESTAMP DEFAULT NOW()
          );"
        env:
        - name: DB_HOST
          value: postgres-service
        - name: DB_USER
          valueFrom:
            secretKeyRef:
              name: db-secret
              key: username
        - name: DB_NAME
          value: myapp
        - name: PGPASSWORD
          valueFrom:
            secretKeyRef:
              name: db-secret
              key: password
```

#### CronJob Example
```yaml
# cronjob.yaml
apiVersion: batch/v1
kind: CronJob
metadata:
  name: backup-job
spec:
  schedule: "0 2 * * *"  # Daily at 2 AM
  jobTemplate:
    spec:
      template:
        spec:
          restartPolicy: OnFailure
          containers:
          - name: backup
            image: postgres:13
            command: ["/bin/sh"]
            args:
            - -c
            - |
              pg_dump -h $DB_HOST -U $DB_USER $DB_NAME > /backup/backup-$(date +%Y%m%d-%H%M%S).sql
              find /backup -name "backup-*.sql" -mtime +7 -delete
            env:
            - name: DB_HOST
              value: postgres-service
            - name: DB_USER
              valueFrom:
                secretKeyRef:
                  name: db-secret
                  key: username
            - name: DB_NAME
              value: myapp
            - name: PGPASSWORD
              valueFrom:
                secretKeyRef:
                  name: db-secret
                  key: password
            volumeMounts:
            - name: backup-storage
              mountPath: /backup
          volumes:
          - name: backup-storage
            persistentVolumeClaim:
              claimName: backup-pvc
  successfulJobsHistoryLimit: 3
  failedJobsHistoryLimit: 1
```

### Auto-scaling

#### Horizontal Pod Autoscaler (HPA)
```yaml
# hpa.yaml
apiVersion: autoscaling/v2
kind: HorizontalPodAutoscaler
metadata:
  name: web-hpa
spec:
  scaleTargetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: web-deployment
  minReplicas: 2
  maxReplicas: 10
  metrics:
  - type: Resource
    resource:
      name: cpu
      target:
        type: Utilization
        averageUtilization: 70
  - type: Resource
    resource:
      name: memory
      target:
        type: Utilization
        averageUtilization: 80
  behavior:
    scaleDown:
      stabilizationWindowSeconds: 300
      policies:
      - type: Percent
        value: 10
        periodSeconds: 60
    scaleUp:
      stabilizationWindowSeconds: 0
      policies:
      - type: Percent
        value: 100
        periodSeconds: 15
      - type: Pods
        value: 4
        periodSeconds: 15
      selectPolicy: Max
```

#### Vertical Pod Autoscaler (VPA)
```yaml
# vpa.yaml
apiVersion: autoscaling.k8s.io/v1
kind: VerticalPodAutoscaler
metadata:
  name: web-vpa
spec:
  targetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: web-deployment
  updatePolicy:
    updateMode: "Auto"
  resourcePolicy:
    containerPolicies:
    - containerName: web
      maxAllowed:
        cpu: 1
        memory: 500Mi
      minAllowed:
        cpu: 100m
        memory: 50Mi
```

#### Cluster Autoscaler
```yaml
# cluster-autoscaler.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: cluster-autoscaler
  namespace: kube-system
spec:
  replicas: 1
  selector:
    matchLabels:
      app: cluster-autoscaler
  template:
    metadata:
      labels:
        app: cluster-autoscaler
    spec:
      serviceAccountName: cluster-autoscaler
      containers:
      - image: k8s.gcr.io/autoscaling/cluster-autoscaler:v1.21.0
        name: cluster-autoscaler
        resources:
          limits:
            cpu: 100m
            memory: 300Mi
          requests:
            cpu: 100m
            memory: 300Mi
        command:
        - ./cluster-autoscaler
        - --v=4
        - --stderrthreshold=info
        - --cloud-provider=aws
        - --skip-nodes-with-local-storage=false
        - --expander=least-waste
        - --node-group-auto-discovery=asg:tag=k8s.io/cluster-autoscaler/enabled,k8s.io/cluster-autoscaler/my-cluster
```

### RBAC and Security

#### Service Account
```yaml
# service-account.yaml
apiVersion: v1
kind: ServiceAccount
metadata:
  name: my-service-account
  namespace: default
```

#### Role and RoleBinding
```yaml
# role.yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: Role
metadata:
  namespace: default
  name: pod-reader
rules:
- apiGroups: [""]
  resources: ["pods"]
  verbs: ["get", "watch", "list"]

---
apiVersion: rbac.authorization.k8s.io/v1
kind: RoleBinding
metadata:
  name: read-pods
  namespace: default
subjects:
- kind: ServiceAccount
  name: my-service-account
  namespace: default
roleRef:
  kind: Role
  name: pod-reader
  apiGroup: rbac.authorization.k8s.io
```

#### ClusterRole and ClusterRoleBinding
```yaml
# cluster-role.yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRole
metadata:
  name: node-reader
rules:
- apiGroups: [""]
  resources: ["nodes"]
  verbs: ["get", "watch", "list"]

---
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRoleBinding
metadata:
  name: read-nodes
subjects:
- kind: ServiceAccount
  name: my-service-account
  namespace: default
roleRef:
  kind: ClusterRole
  name: node-reader
  apiGroup: rbac.authorization.k8s.io
```

#### Pod Security Standards
```yaml
# pod-security-context.yaml
apiVersion: v1
kind: Pod
metadata:
  name: secure-pod
spec:
  securityContext:
    runAsNonRoot: true
    runAsUser: 1000
    runAsGroup: 3000
    fsGroup: 2000
    seccompProfile:
      type: RuntimeDefault
  containers:
  - name: app
    image: nginx
    securityContext:
      allowPrivilegeEscalation: false
      readOnlyRootFilesystem: true
      capabilities:
        drop:
        - ALL
    volumeMounts:
    - name: tmp
      mountPath: /tmp
    - name: cache
      mountPath: /var/cache/nginx
    - name: run
      mountPath: /var/run
  volumes:
  - name: tmp
    emptyDir: {}
  - name: cache
    emptyDir: {}
  - name: run
    emptyDir: {}
```

#### Network Policies
```yaml
# network-policy.yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: deny-all
  namespace: production
spec:
  podSelector: {}
  policyTypes:
  - Ingress
  - Egress

---
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: allow-web-traffic
  namespace: production
spec:
  podSelector:
    matchLabels:
      app: web
  policyTypes:
  - Ingress
  - Egress
  ingress:
  - from:
    - podSelector:
        matchLabels:
          app: frontend
    ports:
    - protocol: TCP
      port: 8080
  egress:
  - to:
    - podSelector:
        matchLabels:
          app: database
    ports:
    - protocol: TCP
      port: 5432
```

### Custom Resources and Operators

#### Custom Resource Definition (CRD)
```yaml
# crd.yaml
apiVersion: apiextensions.k8s.io/v1
kind: CustomResourceDefinition
metadata:
  name: webapps.example.com
spec:
  group: example.com
  versions:
  - name: v1
    served: true
    storage: true
    schema:
      openAPIV3Schema:
        type: object
        properties:
          spec:
            type: object
            properties:
              replicas:
                type: integer
                minimum: 1
                maximum: 10
              image:
                type: string
              port:
                type: integer
          status:
            type: object
            properties:
              availableReplicas:
                type: integer
  scope: Namespaced
  names:
    plural: webapps
    singular: webapp
    kind: WebApp
```

#### Custom Resource Instance
```yaml
# webapp-instance.yaml
apiVersion: example.com/v1
kind: WebApp
metadata:
  name: my-webapp
spec:
  replicas: 3
  image: nginx:1.20
  port: 80
```

#### Simple Operator Example (Go)
```go
// Simple operator controller logic
func (r *WebAppReconciler) Reconcile(ctx context.Context, req ctrl.Request) (ctrl.Result, error) {
    log := r.Log.WithValues("webapp", req.NamespacedName)

    // Fetch the WebApp instance
    var webapp examplev1.WebApp
    if err := r.Get(ctx, req.NamespacedName, &webapp); err != nil {
        return ctrl.Result{}, client.IgnoreNotFound(err)
    }

    // Create or update Deployment
    deployment := &appsv1.Deployment{
        ObjectMeta: metav1.ObjectMeta{
            Name:      webapp.Name,
            Namespace: webapp.Namespace,
        },
    }

    _, err := ctrl.CreateOrUpdate(ctx, r.Client, deployment, func() error {
        deployment.Spec.Replicas = &webapp.Spec.Replicas
        deployment.Spec.Selector = &metav1.LabelSelector{
            MatchLabels: map[string]string{"app": webapp.Name},
        }
        deployment.Spec.Template = corev1.PodTemplateSpec{
            ObjectMeta: metav1.ObjectMeta{
                Labels: map[string]string{"app": webapp.Name},
            },
            Spec: corev1.PodSpec{
                Containers: []corev1.Container{
                    {
                        Name:  "web",
                        Image: webapp.Spec.Image,
                        Ports: []corev1.ContainerPort{
                            {
                                ContainerPort: webapp.Spec.Port,
                            },
                        },
                    },
                },
            },
        }
        return ctrl.SetControllerReference(&webapp, deployment, r.Scheme)
    })

    return ctrl.Result{}, err
}
```

---

## Production Deployment

### Complete Application Stack
```yaml
# production-stack.yaml
apiVersion: v1
kind: Namespace
metadata:
  name: production

---
apiVersion: v1
kind: ConfigMap
metadata:
  name: app-config
  namespace: production
data:
  database.url: "postgresql://postgres:5432/myapp"
  redis.url: "redis://redis:6379"
  log.level: "INFO"

---
apiVersion: v1
kind: Secret
metadata:
  name: app-secrets
  namespace: production
type: Opaque
data:
  database.password: cGFzc3dvcmQ=
  jwt.secret: c2VjcmV0a2V5

---
apiVersion: apps/v1
kind: Deployment
metadata:
  name: web-app
  namespace: production
spec:
  replicas: 3
  strategy:
    type: RollingUpdate
    rollingUpdate:
      maxSurge: 1
      maxUnavailable: 0
  selector:
    matchLabels:
      app: web-app
  template:
    metadata:
      labels:
        app: web-app
    spec:
      serviceAccountName: web-app-sa
      securityContext:
        runAsNonRoot: true
        runAsUser: 1000
        fsGroup: 2000
      containers:
      - name: web
        image: myapp:v1.2.3
        imagePullPolicy: Always
        ports:
        - containerPort: 8080
        env:
        - name: DATABASE_URL
          valueFrom:
            configMapKeyRef:
              name: app-config
              key: database.url
        - name: DATABASE_PASSWORD
          valueFrom:
            secretKeyRef:
              name: app-secrets
              key: database.password
        resources:
          requests:
            memory: "256Mi"
            cpu: "100m"
          limits:
            memory: "512Mi"
            cpu: "500m"
        livenessProbe:
          httpGet:
            path: /health
            port: 8080
          initialDelaySeconds: 30
          periodSeconds: 30
          timeoutSeconds: 5
          failureThreshold: 3
        readinessProbe:
          httpGet:
            path: /ready
            port: 8080
          initialDelaySeconds: 5
          periodSeconds: 5
          timeoutSeconds: 3
          failureThreshold: 3
        securityContext:
          allowPrivilegeEscalation: false
          readOnlyRootFilesystem: true
          capabilities:
            drop:
            - ALL
        volumeMounts:
        - name: tmp
          mountPath: /tmp
        - name: logs
          mountPath: /app/logs
      volumes:
      - name: tmp
        emptyDir: {}
      - name: logs
        emptyDir: {}

---
apiVersion: v1
kind: Service
metadata:
  name: web-app-service
  namespace: production
spec:
  selector:
    app: web-app
  ports:
  - port: 80
    targetPort: 8080
  type: ClusterIP

---
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: web-app-ingress
  namespace: production
  annotations:
    kubernetes.io/ingress.class: nginx
    cert-manager.io/cluster-issuer: letsencrypt-prod
    nginx.ingress.kubernetes.io/ssl-redirect: "true"
    nginx.ingress.kubernetes.io/rate-limit: "100"
spec:
  tls:
  - hosts:
    - myapp.example.com
    secretName: web-app-tls
  rules:
  - host: myapp.example.com
    http:
      paths:
      - path: /
        pathType: Prefix
        backend:
          service:
            name: web-app-service
            port:
              number: 80

---
apiVersion: autoscaling/v2
kind: HorizontalPodAutoscaler
metadata:
  name: web-app-hpa
  namespace: production
spec:
  scaleTargetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: web-app
  minReplicas: 3
  maxReplicas: 20
  metrics:
  - type: Resource
    resource:
      name: cpu
      target:
        type: Utilization
        averageUtilization: 70
  - type: Resource
    resource:
      name: memory
      target:
        type: Utilization
        averageUtilization: 80

---
apiVersion: policy/v1
kind: PodDisruptionBudget
metadata:
  name: web-app-pdb
  namespace: production
spec:
  minAvailable: 2
  selector:
    matchLabels:
      app: web-app
```

### Database Deployment with StatefulSet
```yaml
# postgres-statefulset.yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: postgres-config
  namespace: production
data:
  postgresql.conf: |
    max_connections = 200
    shared_buffers = 256MB
    effective_cache_size = 1GB
    maintenance_work_mem = 64MB
    checkpoint_completion_target = 0.9
    wal_buffers = 16MB
    default_statistics_target = 100

---
apiVersion: apps/v1
kind: StatefulSet
metadata:
  name: postgres
  namespace: production
spec:
  serviceName: postgres-service
  replicas: 1
  selector:
    matchLabels:
      app: postgres
  template:
    metadata:
      labels:
        app: postgres
    spec:
      containers:
      - name: postgres
        image: postgres:13-alpine
        ports:
        - containerPort: 5432
        env:
        - name: POSTGRES_DB
          value: myapp
        - name: POSTGRES_USER
          value: postgres
        - name: POSTGRES_PASSWORD
          valueFrom:
            secretKeyRef:
              name: postgres-secret
              key: password
        - name: PGDATA
          value: /var/lib/postgresql/data/pgdata
        volumeMounts:
        - name: postgres-storage
          mountPath: /var/lib/postgresql/data
        - name: postgres-config
          mountPath: /etc/postgresql/postgresql.conf
          subPath: postgresql.conf
        resources:
          requests:
            memory: "256Mi"
            cpu: "250m"
          limits:
            memory: "1Gi"
            cpu: "1000m"
        livenessProbe:
          exec:
            command:
            - /bin/sh
            - -c
            - exec pg_isready -U "postgres" -h 127.0.0.1 -p 5432
          initialDelaySeconds: 30
          periodSeconds: 10
          timeoutSeconds: 5
          successThreshold: 1
          failureThreshold: 6
        readinessProbe:
          exec:
            command:
            - /bin/sh
            - -c
            - exec pg_isready -U "postgres" -h 127.0.0.1 -p 5432
          initialDelaySeconds: 5
          periodSeconds: 10
          timeoutSeconds: 5
          successThreshold: 1
          failureThreshold: 3
      volumes:
      - name: postgres-config
        configMap:
          name: postgres-config
  volumeClaimTemplates:
  - metadata:
      name: postgres-storage
    spec:
      accessModes: ["ReadWriteOnce"]
      resources:
        requests:
          storage: 20Gi

---
apiVersion: v1
kind: Service
metadata:
  name: postgres-service
  namespace: production
spec:
  selector:
    app: postgres
  ports:
  - port: 5432
    targetPort: 5432
  type: ClusterIP
```

---

## Monitoring and Observability

### Prometheus and Grafana Setup
```yaml
# monitoring-namespace.yaml
apiVersion: v1
kind: Namespace
metadata:
  name: monitoring

---
# prometheus-config.yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: prometheus-config
  namespace: monitoring
data:
  prometheus.yml: |
    global:
      scrape_interval: 15s
      evaluation_interval: 15s
    
    rule_files:
      - "*.rules"
    
    scrape_configs:
      - job_name: 'kubernetes-apiservers'
        kubernetes_sd_configs:
        - role: endpoints
        scheme: https
        tls_config:
          ca_file: /var/run/secrets/kubernetes.io/serviceaccount/ca.crt
        bearer_token_file: /var/run/secrets/kubernetes.io/serviceaccount/token
        relabel_configs:
        - source_labels: [__meta_kubernetes_namespace, __meta_kubernetes_service_name, __meta_kubernetes_endpoint_port_name]
          action: keep
          regex: default;kubernetes;https
      
      - job_name: 'kubernetes-nodes'
        kubernetes_sd_configs:
        - role: node
        scheme: https
        tls_config:
          ca_file: /var/run/secrets/kubernetes.io/serviceaccount/ca.crt
        bearer_token_file: /var/run/secrets/kubernetes.io/serviceaccount/token
        relabel_configs:
        - action: labelmap
          regex: __meta_kubernetes_node_label_(.+)
      
      - job_name: 'kubernetes-pods'
        kubernetes_sd_configs:
        - role: pod
        relabel_configs:
        - source_labels: [__meta_kubernetes_pod_annotation_prometheus_io_scrape]
          action: keep
          regex: true
        - source_labels: [__meta_kubernetes_pod_annotation_prometheus_io_path]
          action: replace
          target_label: __metrics_path__
          regex: (.+)

---
apiVersion: apps/v1
kind: Deployment
metadata:
  name: prometheus
  namespace: monitoring
spec:
  replicas: 1
  selector:
    matchLabels:
      app: prometheus
  template:
    metadata:
      labels:
        app: prometheus
    spec:
      serviceAccountName: prometheus
      containers:
      - name: prometheus
        image: prom/prometheus:v2.35.0
        args:
          - '--config.file=/etc/prometheus/prometheus.yml'
          - '--storage.tsdb.path=/prometheus/'
          - '--web.console.libraries=/etc/prometheus/console_libraries'
          - '--web.console.templates=/etc/prometheus/consoles'
          - '--storage.tsdb.retention.time=200h'
          - '--web.enable-lifecycle'
        ports:
        - containerPort: 9090
        volumeMounts:
        - name: prometheus-config
          mountPath: /etc/prometheus/
        - name: prometheus-storage
          mountPath: /prometheus/
        resources:
          requests:
            cpu: 200m
            memory: 1000Mi
          limits:
            cpu: 1000m
            memory: 2000Mi
      volumes:
      - name: prometheus-config
        configMap:
          name: prometheus-config
      - name: prometheus-storage
        emptyDir: {}

---
apiVersion: v1
kind: Service
metadata:
  name: prometheus-service
  namespace: monitoring
spec:
  selector:
    app: prometheus
  ports:
  - port: 9090
    targetPort: 9090
  type: ClusterIP
```

### Application Metrics
```yaml
# app-with-metrics.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: web-app-metrics
  namespace: production
spec:
  replicas: 3
  selector:
    matchLabels:
      app: web-app-metrics
  template:
    metadata:
      labels:
        app: web-app-metrics
      annotations:
        prometheus.io/scrape: "true"
        prometheus.io/port: "8080"
        prometheus.io/path: "/metrics"
    spec:
      containers:
      - name: web
        image: myapp:v1.2.3
        ports:
        - containerPort: 8080
        - containerPort: 9090
          name: metrics
        env:
        - name: METRICS_ENABLED
          value: "true"
        - name: METRICS_PORT
          value: "9090"
```

### Logging with Fluentd
```yaml
# fluentd-daemonset.yaml
apiVersion: apps/v1
kind: DaemonSet
metadata:
  name: fluentd
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
        effect: NoSchedule
      containers:
      - name: fluentd
        image: fluent/fluentd-kubernetes-daemonset:v1.14-debian-elasticsearch7-1
        env:
        - name: FLUENT_ELASTICSEARCH_HOST
          value: "elasticsearch.logging.svc.cluster.local"
        - name: FLUENT_ELASTICSEARCH_PORT
          value: "9200"
        - name: FLUENT_ELASTICSEARCH_SCHEME
          value: "http"
        resources:
          limits:
            memory: 200Mi
          requests:
            cpu: 100m
            memory: 200Mi
        volumeMounts:
        - name: varlog
          mountPath: /var/log
        - name: varlibdockercontainers
          mountPath: /var/lib/docker/containers
          readOnly: true
      volumes:
      - name: varlog
        hostPath:
          path: /var/log
      - name: varlibdockercontainers
        hostPath:
          path: /var/lib/docker/containers
```

---

## Best Practices

### Resource Management
```yaml
# Best practices deployment
apiVersion: apps/v1
kind: Deployment
metadata:
  name: best-practices-app
spec:
  replicas: 3
  strategy:
    type: RollingUpdate
    rollingUpdate:
      maxSurge: 1
      maxUnavailable: 0
  selector:
    matchLabels:
      app: best-practices-app
  template:
    metadata:
      labels:
        app: best-practices-app
      annotations:
        prometheus.io/scrape: "true"
        prometheus.io/port: "9090"
    spec:
      # Use specific service account
      serviceAccountName: app-service-account
      
      # Security context
      securityContext:
        runAsNonRoot: true
        runAsUser: 1000
        fsGroup: 2000
      
      containers:
      - name: app
        image: myapp:v1.2.3
        imagePullPolicy: Always
        
        # Resource requests and limits
        resources:
          requests:
            memory: "128Mi"
            cpu: "100m"
          limits:
            memory: "256Mi"
            cpu: "200m"
        
        # Health checks
        livenessProbe:
          httpGet:
            path: /health
            port: 8080
          initialDelaySeconds: 30
          periodSeconds: 30
          timeoutSeconds: 5
          failureThreshold: 3
        
        readinessProbe:
          httpGet:
            path: /ready
            port: 8080
          initialDelaySeconds: 5
          periodSeconds: 5
          timeoutSeconds: 3
          failureThreshold: 3
        
        # Security context for container
        securityContext:
          allowPrivilegeEscalation: false
          readOnlyRootFilesystem: true
          capabilities:
            drop:
            - ALL
        
        # Environment variables from ConfigMap and Secret
        env:
        - name: DATABASE_URL
          valueFrom:
            configMapKeyRef:
              name: app-config
              key: database-url
        - name: DATABASE_PASSWORD
          valueFrom:
            secretKeyRef:
              name: app-secret
              key: database-password
        
        # Volume mounts
        volumeMounts:
        - name: tmp
          mountPath: /tmp
        - name: cache
          mountPath: /app/cache
      
      # Volumes
      volumes:
      - name: tmp
        emptyDir: {}
      - name: cache
        emptyDir: {}
      
      # Node selection
      nodeSelector:
        kubernetes.io/os: linux
      
      # Pod anti-affinity for high availability
      affinity:
        podAntiAffinity:
          preferredDuringSchedulingIgnoredDuringExecution:
          - weight: 100
            podAffinityTerm:
              labelSelector:
                matchExpressions:
                - key: app
                  operator: In
                  values:
                  - best-practices-app
              topologyKey: kubernetes.io/hostname
```

### Security Best Practices

1. **Use specific service accounts**
2. **Run as non-root user**
3. **Use read-only root filesystem**
4. **Drop all capabilities**
5. **Use network policies**
6. **Scan images for vulnerabilities**
7. **Use RBAC for access control**
8. **Enable Pod Security Standards**

### Performance Best Practices

1. **Set resource requests and limits**
2. **Use horizontal pod autoscaling**
3. **Implement proper health checks**
4. **Use pod anti-affinity for HA**
5. **Configure pod disruption budgets**
6. **Use rolling updates**
7. **Monitor resource usage**

---

## Troubleshooting

### Common Issues and Solutions

#### Pod Issues
```bash
# Pod stuck in Pending state
kubectl describe pod pod-name
kubectl get events --sort-by=.metadata.creationTimestamp

# Pod CrashLoopBackOff
kubectl logs pod-name --previous
kubectl describe pod pod-name

# Pod OOMKilled
kubectl describe pod pod-name
kubectl top pod pod-name

# Check resource usage
kubectl top pods
kubectl top nodes
```

#### Service Issues
```bash
# Service not accessible
kubectl get endpoints service-name
kubectl describe service service-name

# Test service connectivity
kubectl run test-pod --image=busybox --rm -it --restart=Never -- nslookup service-name
kubectl run test-pod --image=busybox --rm -it --restart=Never -- wget -qO- service-name:port
```

#### Networking Issues
```bash
# Check network policies
kubectl get networkpolicies

# Test pod-to-pod communication
kubectl exec -it pod1 -- ping pod2-ip

# Check DNS resolution
kubectl exec -it pod-name -- nslookup kubernetes.default
```

#### Storage Issues
```bash
# Check PVC status
kubectl get pvc
kubectl describe pvc pvc-name

# Check PV status
kubectl get pv
kubectl describe pv pv-name

# Check storage class
kubectl get storageclass
```

### Debugging Commands
```bash
# Get cluster information
kubectl cluster-info
kubectl get nodes -o wide

# Check component status
kubectl get componentstatuses

# View events
kubectl get events --sort-by=.metadata.creationTimestamp
kubectl get events --field-selector type=Warning

# Check resource usage
kubectl top nodes
kubectl top pods --all-namespaces

# Debug networking
kubectl run debug-pod --image=nicolaka/netshoot --rm -it --restart=Never

# Check RBAC permissions
kubectl auth can-i create pods --as=user
kubectl auth can-i '*' '*' --as=system:serviceaccount:default:my-sa
```

### Useful Scripts
```bash
#!/bin/bash
# cluster-health-check.sh

echo "=== Cluster Health Check ==="

echo "Checking nodes..."
kubectl get nodes

echo "Checking system pods..."
kubectl get pods -n kube-system

echo "Checking resource usage..."
kubectl top nodes

echo "Checking events..."
kubectl get events --sort-by=.metadata.creationTimestamp | tail -10

echo "Checking persistent volumes..."
kubectl get pv

echo "Checking storage classes..."
kubectl get storageclass
```

This comprehensive Kubernetes guide covers everything from basic concepts to advanced production deployments. Each section builds upon previous knowledge and provides practical examples for real-world scenarios.