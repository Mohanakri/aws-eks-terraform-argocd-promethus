Yes, exactly! In real companies, DevOps teams typically follow this **hybrid approach**:

**Terraform handles:**
- Infrastructure provisioning (EKS cluster, VPCs, IAM roles)
- Driver installations (EBS CSI, Load Balancer Controller, Secrets Store)
- Service account creation with IRSA
- Base cluster configuration

**ArgoCD handles:**
- Application deployments
- Microservices rollouts
- Environment-specific configurations
- Continuous deployment from Git repositories

**Real-world workflow looks like:**

```
Infrastructure Team → Terraform → Cluster + Drivers Ready
    ↓
Application Teams → ArgoCD → Deploy Apps to Ready Cluster
```

**Why this separation works:**

**Infrastructure (Terraform)** - Changes infrequently, needs precise state management, often requires elevated permissions. Perfect for "set it and forget it" components like drivers.

**Applications (ArgoCD)** - Changes frequently, needs GitOps workflows, developers can self-serve deployments. Great for rapid iteration and multiple environments.

**Typical company structure:**
- **Platform/DevOps team** - Owns Terraform configs, manages cluster infrastructure
- **Development teams** - Own ArgoCD applications, deploy their services
- **Clear boundary** - Infrastructure team provides "paved road," dev teams deploy on it

This gives you infrastructure consistency (Terraform) with application agility (ArgoCD). Most companies find this hybrid approach scales better than trying to do everything with one tool.

Some companies also use Terraform for initial ArgoCD installation itself, then ArgoCD manages everything else - creating a clean handoff point.
