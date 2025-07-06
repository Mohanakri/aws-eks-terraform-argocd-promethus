# EKS Terraform Configuration - Step by Step Explanation

## 1. Terraform Setup and Provider Configuration

```hcl
terraform {
  required_version = ">= 1.0"
  required_providers {
    aws = {
      source  = "hashicorp/aws"
      version = "~> 5.0"
    }
  }
}

provider "aws" {
  region = var.aws_region
}
```

**What this does:**
- Specifies minimum Terraform version (1.0+)
- Configures AWS provider version 5.x
- Sets up AWS provider with configurable region

## 2. Data Sources

```hcl
data "aws_availability_zones" "available" {
  state = "available"
}

data "aws_caller_identity" "current" {}
```

**What this does:**
- Fetches available AWS availability zones in the region
- Gets current AWS account information (account ID, user ARN, etc.)

## 3. Variables Definition

The configuration defines several variables with defaults:

- `aws_region`: AWS region (default: us-west-2)
- `cluster_name`: EKS cluster name (default: production-eks-cluster)
- `cluster_version`: Kubernetes version (default: 1.28)
- `vpc_cidr`: VPC network range (default: 10.0.0.0/16)
- `node_instance_type`: EC2 instance type for worker nodes (default: t3.medium)
- `node_desired_size`: Desired worker nodes (default: 3)
- `node_max_size`: Maximum worker nodes (default: 5)
- `node_min_size`: Minimum worker nodes (default: 1)

## 4. VPC (Virtual Private Cloud) Setup

```hcl
resource "aws_vpc" "eks_vpc" {
  cidr_block           = var.vpc_cidr
  enable_dns_hostnames = true
  enable_dns_support   = true
  # ... tags
}
```

**What this does:**
- Creates a VPC with the specified CIDR block (10.0.0.0/16)
- Enables DNS hostnames and support (required for EKS)
- Tags the VPC for Kubernetes cluster discovery

## 5. Internet Gateway

```hcl
resource "aws_internet_gateway" "eks_igw" {
  vpc_id = aws_vpc.eks_vpc.id
}
```

**What this does:**
- Creates an Internet Gateway attached to the VPC
- Enables internet access for public subnets

## 6. Subnet Configuration

### Public Subnets
```hcl
resource "aws_subnet" "eks_public_subnet" {
  count = 2
  # Configuration for public subnets
  # CIDR: 10.0.1.0/24, 10.0.2.0/24
  map_public_ip_on_launch = true
}
```

### Private Subnets
```hcl
resource "aws_subnet" "eks_private_subnet" {
  count = 2
  # Configuration for private subnets
  # CIDR: 10.0.10.0/24, 10.0.11.0/24
}
```

**What this does:**
- Creates 2 public subnets in different AZs (for high availability)
- Creates 2 private subnets in different AZs
- Public subnets get public IPs automatically
- Private subnets are for worker nodes (more secure)
- Proper Kubernetes tags for load balancer placement

## 7. NAT Gateway Setup

```hcl
resource "aws_eip" "eks_nat_eip" {
  count = 2
  domain = "vpc"
}

resource "aws_nat_gateway" "eks_nat_gateway" {
  count = 2
  allocation_id = aws_eip.eks_nat_eip[count.index].id
  subnet_id     = aws_subnet.eks_public_subnet[count.index].id
}
```

**What this does:**
- Creates 2 Elastic IPs for NAT gateways
- Creates 2 NAT gateways in public subnets
- Enables private subnet instances to access the internet for updates/pulls

## 8. Route Tables and Associations

### Public Route Table
```hcl
resource "aws_route_table" "eks_public_rt" {
  vpc_id = aws_vpc.eks_vpc.id
  route {
    cidr_block = "0.0.0.0/0"
    gateway_id = aws_internet_gateway.eks_igw.id
  }
}
```

### Private Route Tables
```hcl
resource "aws_route_table" "eks_private_rt" {
  count = 2
  vpc_id = aws_vpc.eks_vpc.id
  route {
    cidr_block     = "0.0.0.0/0"
    nat_gateway_id = aws_nat_gateway.eks_nat_gateway[count.index].id
  }
}
```

**What this does:**
- Public route table routes internet traffic through Internet Gateway
- Private route tables route internet traffic through NAT gateways
- Associates subnets with appropriate route tables

## 9. Security Groups

### EKS Cluster Security Group
```hcl
resource "aws_security_group" "eks_cluster_sg" {
  ingress {
    from_port   = 443
    to_port     = 443
    protocol    = "tcp"
    cidr_blocks = ["0.0.0.0/0"]
  }
  egress {
    from_port   = 0
    to_port     = 0
    protocol    = "-1"
    cidr_blocks = ["0.0.0.0/0"]
  }
}
```

### EKS Node Security Group
```hcl
resource "aws_security_group" "eks_node_sg" {
  ingress {
    from_port = 0
    to_port   = 65535
    protocol  = "tcp"
    self      = true
  }
  # Additional ingress rules for cluster communication
}
```

**What this does:**
- Cluster SG: Allows HTTPS (443) inbound, all outbound
- Node SG: Allows internal communication between nodes and cluster

## 10. IAM Roles and Policies

### EKS Cluster Role
```hcl
resource "aws_iam_role" "eks_cluster_role" {
  assume_role_policy = jsonencode({
    # Allows EKS service to assume this role
  })
}

resource "aws_iam_role_policy_attachment" "eks_cluster_policy" {
  policy_arn = "arn:aws:iam::aws:policy/AmazonEKSClusterPolicy"
  role       = aws_iam_role.eks_cluster_role.name
}
```

### EKS Node Group Role
```hcl
resource "aws_iam_role" "eks_node_role" {
  assume_role_policy = jsonencode({
    # Allows EC2 service to assume this role
  })
}

# Attaches three AWS managed policies:
# - AmazonEKSWorkerNodePolicy
# - AmazonEKS_CNI_Policy  
# - AmazonEC2ContainerRegistryReadOnly
```

**What this does:**
- Creates IAM roles for EKS cluster and worker nodes
- Attaches necessary AWS managed policies
- Enables EKS service to manage resources on your behalf

## 11. EKS Cluster Creation

```hcl
resource "aws_eks_cluster" "eks_cluster" {
  name     = var.cluster_name
  role_arn = aws_iam_role.eks_cluster_role.arn
  version  = var.cluster_version

  vpc_config {
    subnet_ids              = concat(aws_subnet.eks_private_subnet[*].id, aws_subnet.eks_public_subnet[*].id)
    security_group_ids      = [aws_security_group.eks_cluster_sg.id]
    endpoint_private_access = true
    endpoint_public_access  = true
  }

  enabled_cluster_log_types = ["api", "audit", "authenticator", "controllerManager", "scheduler"]
}
```

**What this does:**
- Creates the EKS cluster with specified version
- Configures VPC settings with both private and public subnets
- Enables both private and public API endpoints
- Enables comprehensive logging for troubleshooting

## 12. EKS Node Group

```hcl
resource "aws_eks_node_group" "eks_node_group" {
  cluster_name    = aws_eks_cluster.eks_cluster.name
  node_group_name = "${var.cluster_name}-node-group"
  node_role_arn   = aws_iam_role.eks_node_role.arn
  subnet_ids      = aws_subnet.eks_private_subnet[*].id

  capacity_type  = "ON_DEMAND"
  instance_types = [var.node_instance_type]

  scaling_config {
    desired_size = var.node_desired_size
    max_size     = var.node_max_size
    min_size     = var.node_min_size
  }

  update_config {
    max_unavailable = 1
  }
}
```

**What this does:**
- Creates managed node group in private subnets
- Uses ON_DEMAND instances (can be changed to SPOT for cost savings)
- Configures auto-scaling with min/max/desired capacity
- Sets update strategy to minimize downtime

## 13. OIDC Provider Setup

```hcl
data "tls_certificate" "eks_oidc" {
  url = aws_eks_cluster.eks_cluster.identity[0].oidc[0].issuer
}

resource "aws_iam_openid_connect_provider" "eks_oidc" {
  client_id_list  = ["sts.amazonaws.com"]
  thumbprint_list = [data.tls_certificate.eks_oidc.certificates[0].sha1_fingerprint]
  url             = aws_eks_cluster.eks_cluster.identity[0].oidc[0].issuer
}
```

**What this does:**
- Sets up OIDC identity provider for the cluster
- Enables IAM roles for service accounts (IRSA)
- Allows Kubernetes pods to assume AWS IAM roles securely

## 14. Outputs

The configuration outputs important information:
- `cluster_name`: EKS cluster name
- `cluster_endpoint`: API server endpoint
- `cluster_security_group_id`: Security group ID
- `cluster_oidc_issuer_url`: OIDC issuer URL
- `cluster_certificate_authority_data`: CA certificate
- `vpc_id`: VPC ID
- `private_subnet_ids`: Private subnet IDs
- `public_subnet_ids`: Public subnet IDs

## Architecture Summary

This Terraform configuration creates a production-ready EKS cluster with:

1. **High Availability**: Multi-AZ deployment with redundant NAT gateways
2. **Security**: Worker nodes in private subnets, proper security groups
3. **Scalability**: Auto-scaling node groups with configurable capacity
4. **Monitoring**: Comprehensive logging enabled
5. **Integration**: OIDC provider for IAM roles for service accounts
6. **Networking**: Proper VPC setup with public/private subnets

The infrastructure follows AWS best practices for EKS deployment and provides a solid foundation for running containerized applications in production.


This Terraform configuration creates a comprehensive, production-ready EKS cluster infrastructure. The setup follows AWS best practices and includes all the necessary components for a secure, scalable, and highly available Kubernetes environment.
Key highlights of this configuration:

Network Architecture: Creates a VPC with both public and private subnets across multiple availability zones for high availability
Security: Places worker nodes in private subnets and configures appropriate security groups
Scalability: Implements auto-scaling for worker nodes with configurable capacity
Logging: Enables comprehensive cluster logging for monitoring and troubleshooting
IAM Integration: Sets up OIDC provider for secure pod-to-AWS service authentication
