# Module 2: EKS Node Group Prerequisites — Complete Checklist

> **Note:** Subnet IDs and VPC IDs below are from the author's environment. Substitute your own values.

> **⚠️ Context:** This document records the troubleshooting journey during initial cluster setup.
> The first attempts used public subnets and failed. The **final working setup uses private subnets + NAT Gateway**
> (as documented in `module-02-cluster-setup.md`). This file is kept as a reference for common failure patterns.

**Updated:** 14 May 2026 (after 3 hours of troubleshooting)  
**Lesson:** ALL prerequisites must be in place BEFORE creating a node group. Fixing them after launch doesn't help — nodes have a limited bootstrap retry window.

---

## ⚠️ CRITICAL: The Order Matters

```
1. VPC configured correctly          ← BEFORE cluster creation
2. Cluster created                   ← BEFORE aws-auth / access entries
3. aws-auth ConfigMap created        ← BEFORE node group creation
4. Node group created                ← LAST step
```

If you create the node group before the aws-auth exists, nodes will fail to register and the node group will timeout (~15 min) and fail.

---

## Complete Prerequisites Checklist

### VPC Requirements (Do BEFORE creating cluster)

| # | Requirement | How to Check | How to Fix |
|---|-------------|-------------|-----------|
| 1 | ✅ DNS Resolution enabled | VPC → Details → DNS resolution | VPC → Actions → Edit VPC settings |
| 2 | ✅ DNS Hostnames enabled | VPC → Details → DNS hostnames | VPC → Actions → Edit VPC settings |
| 3 | ✅ Internet Gateway attached | VPC → Internet Gateways | Create IGW → Attach to VPC |
| 4 | ✅ Route table has 0.0.0.0/0 → IGW | Route Tables → Routes | Edit routes → Add 0.0.0.0/0 → IGW |
| 5 | ✅ Subnets in 2+ AZs | Subnets → check AZ column | Create subnets in different AZs |
| 6 | ✅ Auto-assign public IP on ALL node subnets | Subnets → Auto-assign public IPv4 | Subnet → Actions → Edit settings |
| 7 | ✅ NACLs allow all (or at minimum 443 out) | Network ACLs → Inbound/Outbound | Edit rules |
| 8 | ✅ Sufficient IPs (min 6 per subnet, recommend 16+) | Subnets → Available IPs | Use larger CIDR |

### IAM Requirements (Do BEFORE creating cluster/node group)

| # | Requirement | Details |
|---|-------------|---------|
| 1 | ✅ Cluster IAM Role | Trust: `eks.amazonaws.com` / Policy: `AmazonEKSClusterPolicy` |
| 2 | ✅ Node IAM Role | Trust: `ec2.amazonaws.com` / Policies: `AmazonEKSWorkerNodePolicy` + `AmazonEKS_CNI_Policy` + `AmazonEC2ContainerRegistryReadOnly` |

### Authentication (Do AFTER cluster is Active, BEFORE node group)

| # | Requirement | Details |
|---|-------------|---------|
| 1 | ✅ aws-auth ConfigMap | Must map the node role ARN to `system:bootstrappers` + `system:nodes` groups |
| 2 | OR Access Entry | Type: `EC2_LINUX` for the node role (if cluster supports API mode) |

---

## What We Learned (Failure Log)

| Attempt | Failure Reason | Root Cause |
|---------|---------------|-----------|
| 1 | Nodes stuck "Booting Amazon Linux" | No IGW → no internet → can't reach EKS API |
| 2 | `Ec2SubnetInvalidConfiguration` | <PRIVATE_SUBNET_1> had auto-assign public IP = false |
| 3 | "Instances failed to join the kubernetes cluster" | DNS Hostnames disabled on VPC → nodes couldn't resolve EKS endpoint |
| 4 | "Instances failed to join the kubernetes cluster" | aws-auth ConfigMap didn't exist → nodes authenticated but weren't authorized |

---

## Current Cluster State (Final — Verified ✅)

| Component | Status |
|-----------|--------|
| Cluster `eks-learning-cluster` | Active ✅ |
| VPC DNS Resolution | Enabled ✅ |
| VPC DNS Hostnames | Enabled ✅ |
| IGW + Route 0.0.0.0/0 (public subnet) | Attached ✅ |
| NAT GW (in public subnet) | Available ✅ |
| Private subnet 1 (us-east-1a) | Route → NAT GW ✅ |
| Private subnet 2 (us-east-1b) | Route → NAT GW ✅ |
| NACLs | Allow all ✅ |
| Security Group (cluster) | Self-ref ingress + open egress ✅ |
| Access Entry (eks-node-role, EC2 Linux) | Configured ✅ |
| Node IAM Role | 3 policies attached ✅ |

---

## Final Working Setup: Create Node Group

### In Console:
1. EKS → Cluster → Compute → **Add Node Group**
2. Name: `eks-learning-nodes`
3. Node IAM role: `eks-node-role`
4. **Subnets: SELECT PRIVATE SUBNETS ONLY:**
   - Private subnet (us-east-1a) ✅
   - Private subnet (us-east-1b) ✅
   - ⚠️ **DO NOT** select public subnets for nodes
5. Instance type: `t3.medium`
6. Desired: 2 / Min: 1 / Max: 3
7. Create

### Why Private Subnets?
- Nodes don't need public IPs — they reach internet via NAT GW
- Reduces attack surface (no direct inbound from internet)
- Production best practice — matches module 10 recommendations

### Expected Timeline:
- 0-2 min: Instances launch in private subnets (no public IP)
- 2-5 min: Kubelet starts, reaches EKS API via NAT GW, authenticates
- 5-10 min: Nodes register, health checks pass
- 10-15 min: Node group status → **Active**

### Verify Success:
```bash
aws eks update-kubeconfig --region us-east-1 --name eks-learning-cluster
kubectl get nodes
# Should show 2 nodes in Ready state
```

---

## Why Previous Attempts Failed (Root Cause Summary)

The fundamental issue: **We created the cluster with a custom VPC that wasn't properly configured for EKS, and we didn't set up node authentication before launching nodes.**

EKS node bootstrap has a **limited retry window** (~10-15 minutes). If ANY of these are missing when nodes launch, they will fail:

1. **Internet connectivity** (IGW + route + public IP) — nodes must reach EKS API endpoint
2. **DNS resolution** (DNS hostnames + DNS support) — nodes must resolve the EKS endpoint hostname
3. **Authorization** (aws-auth ConfigMap or access entry) — nodes must be allowed to register

All three must be in place BEFORE the node group is created. You cannot fix them after nodes launch and expect the current node group to recover.
