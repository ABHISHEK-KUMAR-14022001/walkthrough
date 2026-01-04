# VPC/Subnet Restructuring Plan

## New Architecture

```mermaid
graph TB
    subgraph "VPC: 10.0.0.0/16"
        subgraph "Public Subnet (1)"
            PUB["📍 Public Subnet<br/>10.0.0.0/24<br/>🖥️ Bastion + 🌐 NAT"]
        end
        
        subgraph "EKS Private Subnets (3 AZs)"
            EKS1["🔒 EKS Subnet AZ-1<br/>10.0.10.0/24<br/>⚙️ Worker Nodes"]
            EKS2["🔒 EKS Subnet AZ-2<br/>10.0.11.0/24<br/>⚙️ Worker Nodes"]
            EKS3["🔒 EKS Subnet AZ-3<br/>10.0.12.0/24<br/>⚙️ Worker Nodes"]
        end
        
        subgraph "Database Private Subnet (1)"
            DBSUB["🔒 Database Subnet<br/>10.0.100.0/24<br/>🗄️ DB VM + VPC Endpoints"]
        end
        
        IGW["🚪 Internet Gateway"]
        NAT["🌐 NAT Gateway"]
        CP["🎛️ EKS Control Plane"]
    end
    
    IGW --> PUB
    PUB --> NAT
    NAT --> EKS1 & EKS2 & EKS3 & DBSUB
    EKS1 & EKS2 & EKS3 --> CP
    EKS1 & EKS2 & EKS3 -->|5432, 6379, 9200| DBSUB
    PUB -->|SSM| DBSUB
```

---

## Subnet Layout

| Subnet | CIDR | Purpose | AZ |
|--------|------|---------|-----|
| public | 10.0.0.0/24 | Bastion, NAT Gateway | ap-south-1a |
| eks-private-0 | 10.0.10.0/24 | EKS Worker Nodes | ap-south-1a |
| eks-private-1 | 10.0.11.0/24 | EKS Worker Nodes | ap-south-1b |
| eks-private-2 | 10.0.12.0/24 | EKS Worker Nodes | ap-south-1c |
| db-private | 10.0.100.0/24 | Database VM, VPC Endpoints | ap-south-1a |

**Total: 5 subnets** (1 public + 3 EKS private + 1 DB private)

---

## Proposed Changes

### 1. [MODIFY] subnet.tf

- Change from 2 public + 2 private to new structure
- Create 1 public subnet
- Create 3 EKS private subnets (across all AZs)
- Create 1 dedicated DB private subnet
- Update route table associations

### 2. [MODIFY] bastion.tf

- Update to use single public subnet: `aws_subnet.public.id`

### 3. [MODIFY] database-vm.tf

- Update DB VM to use dedicated DB subnet: `aws_subnet.db_private.id`
- Update VPC endpoints to use DB subnet

### 4. [MODIFY] nodegroup.tf

- Update node groups to use EKS private subnets: `aws_subnet.eks_private[*].id`

### 5. [MODIFY] eks.tf

- Update EKS cluster to use EKS private subnets: `aws_subnet.eks_private[*].id`

### 6. [NEW] nacl.tf (Optional)

NACLs are not strictly required since security groups handle traffic filtering. Default NACLs allow all traffic. We'll skip NACLs unless you specifically want them.

---

## Security Group Connectivity (No Changes Needed)

| Source | Destination | Ports | Status |
|--------|-------------|-------|--------|
| EKS Nodes (eks_node_sg) | DB VM (db_vm_sg) | 5432, 6379, 9200 | ✅ Already configured |
| Bastion (bastion_sg) | EKS Cluster (eks_cluster_sg) | 443 | ✅ Already configured |

The existing security group rules reference security groups (not CIDRs), so they will continue to work across different subnets.

---

## User Review Required

> [!IMPORTANT]
> This change will require **destroying and recreating** the EKS cluster, node groups, and database VM since they're moving to new subnets.
> 
> Run: `terraform destroy` then `terraform apply`

Do you want me to proceed with these changes?
