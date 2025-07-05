# EKS Controllers and Storage Setup - Step by Step Explanation

## Overview
This Terraform configuration sets up essential controllers and storage drivers for an EKS cluster, enabling it to manage AWS resources like load balancers, storage volumes, and secrets.

## 1. Initial Setup and Data Sources

### Required Providers
```hcl
terraform {
  required_providers {
    aws = "~> 5.0"           # AWS provider for infrastructure
    helm = "~> 2.0"          # Helm provider for Kubernetes packages
    kubernetes = "~> 2.0"    # Kubernetes provider for K8s resources
  }
}
```

### Data Sources
- `aws_caller_identity.current` - Gets current AWS account ID
- `aws_region.current` - Gets current AWS region
- These are used to construct ARNs and resource identifiers

## 2. Provider Configuration

### Helm and Kubernetes Providers
Both providers are configured to connect to the EKS cluster using:
- **Authentication**: AWS CLI with `eks get-token` command
- **Connection**: Cluster endpoint and CA certificate
- **API Version**: `client.authentication.k8s.io/v1beta1`

## 3. AWS Load Balancer Controller

### Purpose
Manages Application Load Balancers (ALBs) and Network Load Balancers (NLBs) for Kubernetes services and ingresses.

### IAM Setup
**Role**: `aws_iam_role.aws_load_balancer_controller`
- Uses OIDC (OpenID Connect) for service account authentication
- Allows the controller pod to assume this role

**Policy**: Comprehensive permissions including:
- **EC2 permissions**: Describe VPCs, subnets, security groups, instances
- **ELB permissions**: Create/manage load balancers, target groups, listeners
- **Security**: Create and manage security groups with specific conditions
- **Certificates**: Access ACM certificates and IAM server certificates
- **WAF**: Associate/disassociate Web Application Firewalls

### Deployment
- **Helm Chart**: `aws-load-balancer-controller` from AWS EKS charts
- **Configuration**: Cluster name, region, VPC ID
- **Service Account**: Pre-created with IAM role annotation

## 4. EBS CSI Driver

### Purpose
Enables dynamic provisioning of Amazon EBS volumes for persistent storage.

### IAM Setup
- **Role**: Uses AWS managed policy `Amazon_EBS_CSI_Driver`
- **Service Account**: `ebs-csi-controller-sa` in kube-system namespace

### Storage Class
```hcl
kubernetes_storage_class_v1.ebs_gp3:
- Type: gp3 (latest generation SSD)
- Encryption: Enabled
- Binding: WaitForFirstConsumer (zone-aware)
- Expansion: Allowed
- Default: Set as default storage class
```

## 5. EFS CSI Driver

### Purpose
Provides shared file system storage across multiple pods/nodes.

### Components
**EFS File System**:
- Performance mode: General Purpose
- Throughput: Provisioned (100 MiB/s)
- Encryption: Enabled at rest

**Mount Targets**:
- Created in each private subnet
- Uses dedicated security group for NFS traffic (port 2049)

**Security Group**:
- Allows NFS traffic (port 2049) from VPC CIDR
- Enables communication between EFS and EKS nodes

**Storage Class**:
- Provisioning mode: EFS Access Points
- Reclaim policy: Retain (data preserved after PVC deletion)
- Directory permissions: 700

## 6. Secrets Manager CSI Driver

### Purpose
Mounts AWS Secrets Manager secrets as files in Kubernetes pods.

### Components
**Two Helm Charts**:
1. **Base driver**: `secrets-store-csi-driver`
   - Enables secret rotation
   - Allows syncing secrets to Kubernetes secrets

2. **AWS provider**: `secrets-store-csi-driver-provider-aws`
   - AWS-specific implementation
   - Integrates with AWS Secrets Manager

**IAM Permissions**:
- `secretsmanager:GetSecretValue` - Read secret values
- `secretsmanager:DescribeSecret` - Get secret metadata
- Scoped to secrets in the current account/region

**Sample Secret**:
Creates a demo secret with database URL, API key, and JWT secret for testing.

## 7. Service Accounts

Each controller gets a dedicated Kubernetes service account with:
- **IRSA annotation**: `eks.amazonaws.com/role-arn`
- **Namespace**: `kube-system` (system namespace)
- **Purpose**: Enables pods to assume AWS IAM roles

## 8. Security Considerations

### IRSA (IAM Roles for Service Accounts)
- **Principle**: Least privilege access
- **Mechanism**: OIDC-based authentication
- **Benefits**: No need to store AWS credentials in pods

### Encryption
- **EBS**: Encryption at rest enabled
- **EFS**: Encryption at rest enabled
- **Secrets**: Stored securely in AWS Secrets Manager

### Network Security
- **EFS**: Dedicated security group restricts NFS access
- **Load Balancer**: Controlled security group management

## 9. Key Outputs

The configuration provides these outputs for use in other modules:
- IAM role ARNs for all controllers
- EFS file system ID
- Secrets Manager secret ARN

## 10. Dependencies and Order

The resources are deployed in this order:
1. **IAM roles and policies** (foundational)
2. **Service accounts** (links IAM to Kubernetes)
3. **Helm releases** (actual controller deployment)
4. **Storage classes** (depends on CSI drivers)
5. **EFS infrastructure** (file system and mount targets)

## Usage Examples

### Using EBS Storage
```yaml
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: my-ebs-pvc
spec:
  accessModes:
    - ReadWriteOnce
  storageClassName: ebs-gp3
  resources:
    requests:
      storage: 10Gi
```

### Using EFS Storage
```yaml
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: my-efs-pvc
spec:
  accessModes:
    - ReadWriteMany
  storageClassName: efs
  resources:
    requests:
      storage: 5Gi
```

### Using Application Load Balancer
```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: my-ingress
  annotations:
    kubernetes.io/ingress.class: alb
    alb.ingress.kubernetes.io/scheme: internet-facing
spec:
  rules:
    - host: myapp.example.com
      http:
        paths:
          - path: /
            pathType: Prefix
            backend:
              service:
                name: my-service
                port:
                  number: 80
```

This configuration provides a complete foundation for running production workloads on EKS with proper storage, load balancing, and secrets management capabilities.



This Terraform configuration is a comprehensive setup for EKS cluster controllers and storage drivers. Here's what it accomplishes:
Main Components:

AWS Load Balancer Controller - Automatically manages ALBs/NLBs for your Kubernetes services
EBS CSI Driver - Handles dynamic provisioning of EBS volumes for pod storage
EFS CSI Driver - Provides shared file system storage across multiple pods
Secrets Manager CSI Driver - Securely mounts AWS secrets as files in pods

Key Benefits:

IRSA Integration: Uses IAM Roles for Service Accounts for secure, credential-free authentication
Production-Ready: Includes encryption, proper security groups, and managed policies
Storage Options: Provides both block storage (EBS) and shared storage (EFS)
Secrets Management: Secure access to AWS Secrets Manager from pods

Variables Required:
You'll need to pass these variables from your main configuration:

cluster_name, cluster_endpoint, cluster_certificate_authority_data
cluster_oidc_issuer_url, vpc_id, private_subnet_ids

This setup is essential for any production EKS cluster as it enables core Kubernetes functionality like persistent volumes, load balancing, and secure secrets management.
