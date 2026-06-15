# Module 1: EKS Fundamentals & Concepts


---

---

Understand what EKS is, how it fits into the Kubernetes ecosystem, and explore the AWS console to see the service first-hand. No cluster creation yet — just building the mental model.

## 1.1 What is Kubernetes? (5 min)

Kubernetes (K8s) is a container orchestration platform that automates:
- **Deployment** — Roll out containers across machines
- **Scaling** — Add/remove containers based on demand
- **Self-healing** — Restart failed containers automatically
- **Load balancing** — Distribute traffic across containers

**Analogy:** Think of K8s as a "fleet manager" for containers, like how an airline manages planes across airports.

---

## 1.2 What is Amazon EKS? (5 min)

EKS = **Elastic Kubernetes Service** — AWS-managed Kubernetes control plane.

**What AWS manages for you:**
- Control plane (API server, etcd, scheduler, controller manager)
- High availability (multi-AZ control plane)
- Patching and upgrades
- Integration with AWS services (IAM, VPC, ELB, EBS, etc.)

**What YOU manage:**
- Worker nodes (EC2 or Fargate)
- Your applications (pods, deployments)
- Networking configuration
- Security policies

---

## 1.3 EKS Architecture Diagram

```
┌─────────────────────────────────────────────────────────┐
│                    AWS CLOUD                              │
│                                                          │
│  ┌──────────────────────────────────────────────────┐   │
│  │           EKS CONTROL PLANE (AWS Managed)         │   │
│  │                                                    │   │
│  │  ┌──────────┐  ┌──────────┐  ┌───────────────┐  │   │
│  │  │API Server│  │Scheduler │  │Controller Mgr │  │   │
│  │  └──────────┘  └──────────┘  └───────────────┘  │   │
│  │  ┌──────────────────────────────────────────┐    │   │
│  │  │         etcd (cluster state)              │    │   │
│  │  └──────────────────────────────────────────┘    │   │
│  └──────────────────────────────────────────────────┘   │
│                         │                                │
│                    kubectl / API                          │
│                         │                                │
│  ┌──────────────────────────────────────────────────┐   │
│  │           DATA PLANE (You Manage)                 │   │
│  │                                                    │   │
│  │  ┌─────────────┐  ┌─────────────┐  ┌──────────┐ │   │
│  │  │  Node (EC2) │  │  Node (EC2) │  │  Fargate │ │   │
│  │  │ ┌─────────┐ │  │ ┌─────────┐ │  │ ┌──────┐ │ │   │
│  │  │ │  Pod A  │ │  │ │  Pod C  │ │  │ │Pod E │ │ │   │
│  │  │ │  Pod B  │ │  │ │  Pod D  │ │  │ └──────┘ │ │   │
│  │  │ └─────────┘ │  │ └─────────┘ │  └──────────┘ │   │
│  │  └─────────────┘  └─────────────┘                │   │
│  └──────────────────────────────────────────────────┘   │
└─────────────────────────────────────────────────────────┘
```

---

## 1.4 Key Concepts (10 min)

| Concept | What It Is | AWS Equivalent |
|---------|-----------|----------------|
| **Pod** | Smallest deployable unit (1+ containers) | Like a single task in ECS |
| **Node** | A machine running pods (EC2 instance) | EC2 instance in ECS cluster |
| **Cluster** | Control plane + nodes | ECS Cluster |
| **Deployment** | Manages pod replicas & updates | ECS Service |
| **Service** | Stable network endpoint for pods | ECS Service + Load Balancer |
| **Namespace** | Logical isolation within a cluster | Like separate ECS clusters |
| **ConfigMap** | Configuration data (non-secret) | SSM Parameter Store |
| **Secret** | Sensitive data (passwords, keys) | Secrets Manager |
| **Ingress** | HTTP routing rules | ALB Listener Rules |
| **PersistentVolume** | Storage attached to pods | EBS/EFS volumes |

---

## 1.5 EKS Node Types — Which to Choose?

| Type | Best For | Cost Model |
|------|----------|-----------|
| **Managed Node Groups** | Most workloads (recommended for beginners) | EC2 pricing |
| **Self-Managed Nodes** | Custom AMIs, GPU, special requirements | EC2 pricing |
| **Fargate** | Serverless, no node management | Per pod (vCPU + memory) |

**Start with:** Managed Node Groups (simplest to learn)

---

## 1.6 EKS Pricing

| Component | Cost |
|-----------|------|
| EKS Control Plane | $0.10/hour (~$73/month) |
| Worker Nodes | Standard EC2/Fargate pricing |
| Data Transfer | Standard AWS data transfer rates |

💡 **Tip:** For learning, use a small cluster (2x t3.medium nodes) — costs ~$3-4/day total.

---

## 1.7 ✅ Hands-On: Explore EKS in Console (Do This Now)

### Step 1: Navigate to EKS
- [ ] Open AWS Console → Search "EKS" → Click **Amazon Elastic Kubernetes Service**

### Step 2: Observe the Dashboard
- [ ] Note the left menu: Clusters, Add-ons, Pod Identity
- [ ] Click **Clusters** — you'll see any existing clusters (or empty)

### Step 3: Explore Cluster Creation (DON'T CREATE YET — just look)
- [ ] Click **Create cluster** (or **Add cluster** → Create)
- [ ] Observe the required fields:
  - Cluster name
  - Kubernetes version (latest stable)
  - Cluster IAM role
  - VPC & Subnets
  - Security groups
  - Cluster endpoint access (Public/Private/Both)
- [ ] **Cancel** — we'll create properly in Module 2

### Step 4: Check Prerequisites
- [ ] Open CloudShell (terminal icon top-right in console)
- [ ] Run: `aws --version` (should show AWS CLI v2)
- [ ] Run: `kubectl version --client` (may need to install)
- [ ] Run: `eksctl version` (may need to install)

---

## 1.8 Key Takeaways

1. EKS = AWS-managed Kubernetes control plane
2. You manage the data plane (nodes + workloads)
3. Pods are the smallest unit, Deployments manage them
4. Start with Managed Node Groups for simplicity
5. Control plane costs $0.10/hr — nodes are separate EC2 costs

---

## Next Step

When you've completed the console exploration above, tell me and we'll move to **Module 2: Cluster Setup** where we'll actually create your first EKS cluster using:
- Option A: AWS Console (click-through)
- Option B: eksctl (CLI — recommended)
- Option C: Terraform (IaC — your favourite!)

---

## Exercises

| # | Task | Done |
|---|------|------|
| 1 | Read: What is Kubernetes | [ ] |
| 2 | Read: What is Amazon EKS | [ ] |
| 3 | Read: Architecture Diagram | [ ] |
| 4 | Read: Key Concepts | [ ] |
| 5 | Read: Node Types | [ ] |
| 6 | Read: Pricing | [ ] |
| 7 | Hands-On Console Exploration | [ ] |
| 8 | Review Key Takeaways | [ ] |

## Notes from Session
- Console shows Quick Config and Custom Config options
- kubectl installed in CloudShell
- eksctl NOT installed — will install in Module 2
- Will use Custom Config approach for learning
