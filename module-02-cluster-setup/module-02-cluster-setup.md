# Module 2: EKS Cluster Setup


---

Create your first EKS cluster from scratch — VPC, IAM roles, control plane, and worker nodes. By the end, `kubectl get nodes` returns Ready.

## Pre-Requisites

### A. VPC Setup

**Create VPC:**
- VPC → Create VPC → **VPC and more** (creates subnets, route tables, IGW, NAT GW together)
- CIDR: `192.168.0.0/20` (or your choice)
- AZs: 2
- Public subnets: 1 | Private subnets: 2
- NAT Gateway: 1 (in public subnet)
- DNS hostnames: ✅ Enable
- DNS resolution: ✅ Enable

**Or configure existing VPC — ensure:**

| Requirement | Why |
|-------------|-----|
| DNS Hostnames = enabled | Nodes resolve EKS endpoint |
| DNS Resolution = enabled | Internal DNS works |
| NAT GW in public subnet | Private nodes reach internet |
| Private subnets → route 0.0.0.0/0 → NAT GW | Outbound connectivity |
| 2+ private subnets in different AZs | HA for node group |

**Validate:**
```bash
aws ec2 describe-vpc-attribute --vpc-id <VPC_ID> --attribute enableDnsHostnames
aws ec2 describe-vpc-attribute --vpc-id <VPC_ID> --attribute enableDnsSupport
aws ec2 describe-nat-gateways --filter "Name=vpc-id,Values=<VPC_ID>" --query "NatGateways[].{State:State,PublicIp:NatGatewayAddresses[0].PublicIp}"
```

---

### B. IAM Roles

**Cluster Role (`eks-cluster-role`):**
- Trust: `eks.amazonaws.com`
- Policy: `AmazonEKSClusterPolicy`

**Node Role (`eks-node-role`):**
- Trust: `ec2.amazonaws.com`
- Policies: `AmazonEKSWorkerNodePolicy` + `AmazonEKS_CNI_Policy` + `AmazonEC2ContainerRegistryReadOnly`

**Validate:**
```bash
aws iam list-attached-role-policies --role-name eks-cluster-role
aws iam list-attached-role-policies --role-name eks-node-role
```

---

## Step 1: Create Cluster

EKS → Create cluster → Custom config

| Field | Value |
|-------|-------|
| Name | `eks-learning-cluster` |
| Version | Latest |
| Role | `eks-cluster-role` |
| Subnets | Private subnets only |
| Endpoint | **Public and Private** ⭐ |
| Auth mode | **EKS API and ConfigMap** ⭐ |
| Add-ons | CoreDNS + kube-proxy + VPC CNI (defaults) |

→ Wait ~15 min

**Validate:**
```bash
aws eks describe-cluster --name eks-learning-cluster --region us-east-1 \
  --query "cluster.{Status:status,Endpoint:endpoint}"
aws eks update-kubeconfig --name eks-learning-cluster --region us-east-1
kubectl get svc
```

---

## Step 2: Node Access (BEFORE node group!)

EKS → Cluster → Access tab → Create access entry:
- ARN: `arn:aws:iam::<ACCOUNT_ID>:role/eks-node-role`
- **Type: `EC2 Linux`** ⭐ (NOT `EC2`)

**Validate:**
```bash
kubectl get configmap aws-auth -n kube-system -o yaml
```

---

## Step 3: Create Node Group

EKS → Cluster → Compute → Add Node Group

| Field | Value |
|-------|-------|
| Name | `eks-learning-nodes` |
| Role | `eks-node-role` |
| Subnets | Same private subnets |
| Type | `t3.medium` |
| Desired/Min/Max | 2 / 1 / 3 |

→ Wait ~5-10 min

**Validate:**
```bash
kubectl get nodes -o wide
kubectl get pods -A
kubectl cluster-info
eksctl get nodegroup --cluster eks-learning-cluster --region us-east-1
```

---

## Exercises

| # | Task | Done |
|---|------|------|
| 1 | Create VPC with public + private subnets | [ ] |
| 2 | Create cluster IAM role | [ ] |
| 3 | Create EKS cluster (API + ConfigMap auth) | [ ] |
| 4 | Create node IAM role + access entry (EC2 Linux) | [ ] |
| 5 | Create managed node group (private subnets + NAT GW) | [ ] |
| 6 | Verify: kubectl get nodes shows Ready | [ ] |

---

## Key Takeaways

1. **Order:** VPC → IAM → Cluster → Access Entry → Node Group
2. **Access Entry type:** `EC2 Linux` (not `EC2`)
3. **Endpoint:** Public + Private
4. **Auth mode:** EKS API and ConfigMap
5. **Nodes:** Private subnets + NAT GW for outbound

---

## Cleanup

```bash
aws eks delete-nodegroup --cluster-name eks-learning-cluster \
  --nodegroup-name eks-learning-nodes --region us-east-1
# Wait ~5-10 min, then:
aws eks delete-cluster --name eks-learning-cluster --region us-east-1
```
