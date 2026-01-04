# EKS Production Infrastructure - Complete Walkthrough

## Architecture Overview

```mermaid
graph TB
    subgraph "VPC: 10.0.0.0/16"
        subgraph "Public Subnet (10.0.0.0/24)"
            PUB["📍 Public Subnet<br/>🖥️ Bastion + 🌐 NAT Gateway"]
        end
        
        subgraph "EKS Private Subnets (Multi-AZ)"
            EKS1["🔒 10.0.10.0/24 (AZ-1)<br/>⚙️ Worker Nodes"]
            EKS2["🔒 10.0.11.0/24 (AZ-2)<br/>⚙️ Worker Nodes"]
            EKS3["🔒 10.0.12.0/24 (AZ-3)<br/>⚙️ Worker Nodes"]
        end
        
        subgraph "Database Subnet (10.0.100.0/24)"
            DBSUB["🔒 DB Subnet<br/>🗄️ DB VM + VPC Endpoints"]
        end
        
        IGW["🚪 Internet Gateway"]
        CP["🎛️ EKS Control Plane"]
    end
    
    IGW --> PUB
    PUB -->|NAT| EKS1 & EKS2 & EKS3 & DBSUB
    EKS1 & EKS2 & EKS3 --> CP
    EKS1 & EKS2 & EKS3 -->|Pods| DBSUB
    PUB -->|kubectl| CP
    PUB -->|SSM| DBSUB
```

---

## Subnet Layout

| Subnet | CIDR | AZ | Purpose | Resources |
|--------|------|-----|---------|-----------|
| public | 10.0.0.0/24 | ap-south-1a | Bastion, NAT Gateway | Bastion Host, NAT Gateway |
| eks-private-0 | 10.0.10.0/24 | ap-south-1a | EKS Workers | Node Groups |
| eks-private-1 | 10.0.11.0/24 | ap-south-1b | EKS Workers | Node Groups |
| eks-private-2 | 10.0.12.0/24 | ap-south-1c | EKS Workers | Node Groups |
| db-private | 10.0.100.0/24 | ap-south-1a | Database | DB VM, VPC Endpoints |

---

## Security Groups

### How Access is Controlled

```mermaid
graph LR
    subgraph "Security Groups"
        BSG["🛡️ bastion_sg"]
        CSG["🛡️ eks_cluster_sg"]
        NSG["🛡️ eks_node_sg"]
        DSG["🛡️ db_vm_sg"]
    end
    
    BSG -->|443| CSG
    NSG -->|443| CSG
    NSG -->|5432, 6379, 9200| DSG
    CSG -->|All| NSG
```

### Security Group Rules

| Security Group | Direction | Port | Source/Destination | Purpose |
|----------------|-----------|------|-------------------|---------|
| **bastion_sg** | Egress | All | 0.0.0.0/0 | Outbound for SSM, kubectl |
| **eks_cluster_sg** | Ingress | 443 | bastion_sg | kubectl from Bastion |
| **eks_cluster_sg** | Ingress | 443 | eks_node_sg | Nodes → Control Plane |
| **eks_cluster_sg** | Ingress | 1025-65535 | eks_node_sg | Webhooks, kubelet |
| **eks_cluster_sg** | Egress | All | 0.0.0.0/0 | Control plane outbound |
| **eks_node_sg** | Ingress | All | eks_node_sg (self) | Node ↔ Node (CNI) |
| **eks_node_sg** | Ingress | All | eks_cluster_sg | Control Plane → Nodes |
| **eks_node_sg** | Egress | All | 0.0.0.0/0 | Outbound for registry, APIs |
| **db_vm_sg** | Ingress | 5432 | eks_node_sg | PostgreSQL from pods |
| **db_vm_sg** | Ingress | 6379 | eks_node_sg | Redis from pods |
| **db_vm_sg** | Ingress | 9200 | eks_node_sg | Elasticsearch from pods |
| **db_vm_sg** | Ingress | 443 | VPC CIDR | SSM VPC Endpoints |
| **db_vm_sg** | Egress | All | 0.0.0.0/0 | Outbound for apt, docker |

---

## NACLs (Network ACLs)

> [!NOTE]
> **We use DEFAULT NACLs** which allow all inbound and outbound traffic. Security is enforced at the **Security Group level** instead.

### Why Security Groups are Sufficient

| Feature | Security Groups | NACLs |
|---------|----------------|-------|
| State | **Stateful** (return traffic auto-allowed) | Stateless |
| Level | Instance/ENI level | Subnet level |
| Rules | Allow only | Allow + Deny |
| Default | Deny all, add allows | Allow all |

For this setup, Security Groups provide sufficient isolation:
- ✅ DB only accepts connections from EKS nodes
- ✅ EKS nodes only accept connections from control plane
- ✅ Bastion has no inbound rules (SSM only)

---

## How Bastion Connects to EKS

```
Bastion Host (public subnet)
    │
    ├── [1] aws eks update-kubeconfig
    │       Uses IAM role to get cluster credentials
    │
    ├── [2] kubectl get nodes
    │       Request goes to private EKS endpoint (443)
    │       Security Group: bastion_sg → eks_cluster_sg ✅
    │
    └── [3] Response returns via same path (stateful)
```

### EKS Access Entries

| Principal | Type | Policy |
|-----------|------|--------|
| Bastion IAM Role | STANDARD | AmazonEKSClusterAdminPolicy |
| devops IAM User | STANDARD | AmazonEKSClusterAdminPolicy |

---

## How Pods Connect to Database

```
Pod (in EKS node)
    │
    ├── [1] Connect to DB private IP (e.g., 10.0.100.x:5432)
    │       Security Group: eks_node_sg → db_vm_sg ✅
    │       Route: Same VPC, direct routing via VPC router
    │
    └── [2] Response returns (stateful SG allows return traffic)
```

### Database Connection Strings

```bash
# Get DB VM private IP
terraform output db_vm_private_ip

# PostgreSQL
postgresql://<user>:<password>@<db_vm_private_ip>:5432/<database>

# Redis
redis://<db_vm_private_ip>:6379

# Elasticsearch
http://<db_vm_private_ip>:9200
```

---

## How SSM Works for DB VM

```
User's Machine
    │
    ├── [1] aws ssm start-session --target <db-instance-id>
    │       Goes to AWS SSM service (internet)
    │
AWS SSM Service
    │
    ├── [2] Routes to VPC Endpoint (in db-private subnet)
    │       VPC Endpoint: com.amazonaws.ap-south-1.ssm
    │
    ├── [3] DB VM receives session via endpoint
    │       Security Group: db_vm_sg allows 443 from VPC CIDR ✅
    │
    └── [4] Secure shell session established
```

---

## Quick Start Commands

```bash
# Connect to Bastion via SSM
aws ssm start-session --target <bastion-instance-id> --region ap-south-1

# Configure kubectl on Bastion
aws eks update-kubeconfig --region ap-south-1 --name prod-private-eks
kubectl get nodes

# Connect to DB VM via SSM
aws ssm start-session --target <db-instance-id> --region ap-south-1
```

---

## Terraform Commands

```bash
terraform init
terraform plan
terraform apply
terraform output
```
