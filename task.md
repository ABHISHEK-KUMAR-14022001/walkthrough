# VPC/Subnet Restructuring Task

## Tasks

- [x] Create implementation plan with new subnet structure
- [x] Modify `subnet.tf` - new subnet layout (1 public, 3 EKS private, 1 DB private)
- [x] Modify `bastion.tf` - use single public subnet
- [x] Modify `database-vm.tf` - use dedicated DB subnet
- [x] Modify `nodegroup.tf` - use EKS private subnets across AZs
- [x] Modify `eks.tf` - use EKS private subnets
- [ ] Update walkthrough with new architecture, SG, NACL explanations
