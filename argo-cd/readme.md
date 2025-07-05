# Kubernetes NGINX Microservice Deployment Guide

This deployment creates a production-ready NGINX microservice on AWS EKS with comprehensive infrastructure components. Here's a detailed breakdown:

## 1. Foundation Components

### Namespace
```yaml
apiVersion: v1
kind: Namespace
metadata:
  name: nginx-app
```
- Creates an isolated environment called `nginx-app`
- All resources will be deployed within this namespace
- Provides logical separation from other applications

### Service Account
```yaml
apiVersion: v1
kind: ServiceAccount
metadata:
  name: nginx-app-sa
  annotations:
    eks.amazonaws.com/role-arn: "arn:aws:iam::ACCOUNT_ID:role/..."
```
- Creates identity for pods to interact with AWS services
- Links to IAM role for accessing AWS Secrets Manager
- Enables secure access to AWS resources without hardcoded credentials

## 2. Configuration Management

### ConfigMap
```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: nginx-config
data:
  nginx.conf: |
    # NGINX configuration
  index.html: |
    # HTML content
```
- Stores NGINX configuration and HTML content
- Separates configuration from container images
- Includes:
  - Custom NGINX server configuration
  - Gzip compression settings
  - Proxy configuration for `/api` endpoints
  - Health check endpoint at `/health`
  - Metrics endpoint at `/metrics`
  - Custom HTML page with application information

### Secrets Management
**Local Secrets:**
```yaml
apiVersion: v1
kind: Secret
metadata:
  name: app-secrets
type: Opaque
data:
  database-url: cG9zdGdyZXNxbDovL...  # Base64 encoded
```

**AWS Secrets Manager Integration:**
```yaml
apiVersion: secrets-store.csi.x-k8s.io/v1
kind: SecretProviderClass
metadata:
  name: aws-secrets
spec:
  provider: aws
  parameters:
    objects: |
      - objectName: "production-eks-cluster-app-secrets"
```
- Provides secure storage for sensitive data
- Local secrets for development/testing
- AWS Secrets Manager for production secrets
- Automatically syncs secrets from AWS into Kubernetes

## 3. Storage Solutions

### EBS Storage (Single-node access)
```yaml
apiVersion: v1
kind: PersistentVolume
metadata:
  name: nginx-app-pv
spec:
  capacity:
    storage: 10Gi
  accessModes:
    - ReadWriteOnce
  storageClassName: ebs-gp3
```
- 10GB EBS volume for application data
- ReadWriteOnce: Can only be mounted by one node
- Used for NGINX logs and cache
- Persistent across pod restarts

### EFS Storage (Multi-node access)
```yaml
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: nginx-app-efs-pvc
spec:
  accessModes:
    - ReadWriteMany
  storageClassName: efs
```
- 5GB EFS volume for shared storage
- ReadWriteMany: Can be accessed by multiple pods simultaneously
- Used for shared files, uploads, and cross-pod communication

## 4. Application Deployment

### Deployment Configuration
```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: nginx-app-deployment
spec:
  replicas: 3
  template:
    spec:
      containers:
      - name: nginx
        image: nginx:1.25-alpine
        resources:
          requests:
            memory: "128Mi"
            cpu: "100m"
          limits:
            memory: "256Mi"
            cpu: "200m"
```

**Key Features:**
- **3 replicas** for high availability
- **Resource limits** prevent resource exhaustion
- **Health checks** ensure pod readiness
- **Volume mounts** for configuration and storage
- **Environment variables** from secrets
- **Pod anti-affinity** spreads pods across nodes

### Volume Mounts
```yaml
volumeMounts:
- name: nginx-config
  mountPath: /etc/nginx/nginx.conf
  subPath: nginx.conf
- name: nginx-logs
  mountPath: /var/log/nginx
- name: shared-storage
  mountPath: /shared
- name: aws-secrets
  mountPath: /mnt/secrets
```
- Configuration files from ConfigMap
- Persistent logs storage
- Shared file system access
- Secure secrets mounting

## 5. Networking & Exposure

### Service Types

**ClusterIP Service (Internal):**
```yaml
apiVersion: v1
kind: Service
metadata:
  name: nginx-app-service
spec:
  type: ClusterIP
  ports:
  - port: 80
    targetPort: 80
```
- Internal cluster communication
- Not accessible from outside the cluster

**NodePort Service (External):**
```yaml
spec:
  type: NodePort
  ports:
  - port: 80
    nodePort: 30080
```
- Exposes service on each node's IP at port 30080
- Accessible from outside cluster via `<NodeIP>:30080`

**LoadBalancer Service (NLB):**
```yaml
spec:
  type: LoadBalancer
  annotations:
    service.beta.kubernetes.io/aws-load-balancer-type: nlb
```
- Creates AWS Network Load Balancer
- Provides external IP address
- High performance, low latency

### Ingress (Application Load Balancer)
```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  annotations:
    kubernetes.io/ingress.class: alb
    alb.ingress.kubernetes.io/scheme: internet-facing
    alb.ingress.kubernetes.io/ssl-redirect: '443'
spec:
  rules:
  - host: nginx-app.example.com
    http:
      paths:
      - path: /
        pathType: Prefix
```
- Creates AWS Application Load Balancer
- Supports SSL termination
- Host-based routing
- Path-based routing for different services

## 6. Scaling & Availability

### Horizontal Pod Autoscaler
```yaml
apiVersion: autoscaling/v2
kind: HorizontalPodAutoscaler
spec:
  minReplicas: 3
  maxReplicas: 10
  metrics:
  - type: Resource
    resource:
      name: cpu
      target:
        type: Utilization
        averageUtilization: 70
```
- Automatically scales pods based on CPU/memory usage
- Scales up when CPU > 70% or memory > 80%
- Maintains minimum 3 replicas
- Maximum 10 replicas

### Pod Disruption Budget
```yaml
apiVersion: policy/v1
kind: PodDisruptionBudget
spec:
  minAvailable: 2
  selector:
    matchLabels:
      app: nginx-app
```
- Ensures at least 2 pods remain available during updates
- Prevents service disruption during maintenance
- Maintains high availability

## 7. Security

### Network Policy
```yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
spec:
  podSelector:
    matchLabels:
      app: nginx-app
  policyTypes:
  - Ingress
  - Egress
  ingress:
  - from:
    - namespaceSelector:
        matchLabels:
          name: kube-system
    ports:
    - protocol: TCP
      port: 80
```
- Controls network traffic flow
- Allows ingress only from specific namespaces
- Restricts egress to necessary services (DNS, HTTP/HTTPS)
- Implements zero-trust networking

## 8. Monitoring & Observability

### ServiceMonitor (Prometheus)
```yaml
apiVersion: monitoring.coreos.com/v1
kind: ServiceMonitor
spec:
  selector:
    matchLabels:
      app: nginx-app
  endpoints:
  - port: http
    path: /metrics
    interval: 30s
```
- Configures Prometheus to scrape metrics
- Collects metrics every 30 seconds
- Monitors application health and performance

## 9. Maintenance Tasks

### Initialization Job
```yaml
apiVersion: batch/v1
kind: Job
metadata:
  name: nginx-app-init-job
spec:
  template:
    spec:
      containers:
      - name: init-container
        command:
        - /bin/sh
        - -c
        - |
          echo "Creating required directories..."
          mkdir -p /shared/logs /shared/cache /shared/uploads
```
- Runs once to initialize the application
- Creates necessary directories
- Sets up initial configuration

### Log Rotation CronJob
```yaml
apiVersion: batch/v1
kind: CronJob
metadata:
  name: nginx-app-logrotate
spec:
  schedule: "0 2 * * *"  # Run at 2 AM daily
  jobTemplate:
    spec:
      template:
        spec:
          containers:
          - name: logrotate
            command:
            - /bin/sh
            - -c
            - |
              find /var/log/nginx -name "*.log" -type f -mtime +7 -delete
```
- Runs daily at 2 AM
- Removes log files older than 7 days
- Prevents disk space issues

## 10. Deployment Flow

1. **Namespace Creation** → Isolated environment
2. **Service Account** → AWS IAM integration
3. **ConfigMap & Secrets** → Configuration and credentials
4. **Storage Setup** → EBS and EFS volumes
5. **Deployment** → Application pods with health checks
6. **Services** → Internal and external networking
7. **Ingress** → Load balancer and SSL termination
8. **Scaling** → Auto-scaling and availability policies
9. **Security** → Network policies and RBAC
10. **Monitoring** → Metrics collection
11. **Maintenance** → Initialization and cleanup jobs

## Key Benefits

- **High Availability**: Multiple replicas with anti-affinity
- **Scalability**: Auto-scaling based on resource usage
- **Security**: Network policies, secrets management, IAM integration
- **Persistence**: EBS for single-node, EFS for multi-node storage
- **Monitoring**: Prometheus metrics and health checks
- **Maintenance**: Automated log rotation and initialization
- **AWS Integration**: Native EKS, ALB, NLB, Secrets Manager support

This deployment provides a robust, production-ready foundation for running NGINX microservices on AWS EKS with enterprise-grade features for security, monitoring, and operations.




This Kubernetes deployment is a comprehensive example of a production-ready NGINX microservice on AWS EKS. Here are the key highlights:
Architecture Overview
The deployment creates a 3-tier architecture:

Frontend: NGINX serving static content and proxying API calls
Networking: Multiple service types (ClusterIP, NodePort, LoadBalancer, Ingress)
Storage: Both EBS (single-node) and EFS (multi-node) persistent storage

Production Features

High Availability: 3 replicas with pod anti-affinity rules
Auto-scaling: HPA scales from 3-10 pods based on CPU/memory
Security: Network policies, AWS Secrets Manager integration, IAM roles
Monitoring: Prometheus metrics collection via ServiceMonitor
Maintenance: Automated log rotation via CronJob

AWS Integration

EKS Cluster: Native Kubernetes on AWS
ALB/NLB: Application and Network Load Balancers
EBS/EFS: Block and file storage
Secrets Manager: Secure credential management
IAM: Role-based access control
