# EKS: From Fundamentals to Production — Level 200-300

A hands-on, self-paced Amazon EKS curriculum that takes you from zero to production-ready in 10 modules. Every command is tested, every mistake documented, and every lesson earned the hard way.

---

## 🎯 Who Is This For?

- Cloud engineers moving from EC2/ECS to Kubernetes
- Solution architects preparing for EKS engagements
- DevOps engineers building production EKS platforms
- Anyone who learns best by doing (not just reading)

**Prerequisites:** AWS account, basic CLI skills, familiarity with containers.

---

## 📚 Modules

| # | Module | Level | Key Skills |
|---|--------|-------|------------|
| 01 | [Fundamentals](module-01-fundamentals/) | 100 | EKS architecture, concepts, console |
| 02 | [Cluster Setup](module-02-cluster-setup/) | 200 | VPC, IAM, eksctl, node groups |
| 03 | [Workloads](module-03-workloads/) | 200 | Pods, Deployments, Services, YAML |
| 04 | [Networking](module-04-networking/) | 200-300 | VPC CNI, ALB Controller, Ingress |
| 05 | [Storage](module-05-storage/) | 200-300 | EBS CSI, EFS CSI, PV/PVC |
| 06 | [Security](module-06-security/) | 300 | IRSA, RBAC, Pod Security, Network Policy |
| 07 | [Observability](module-07-observability/) | 200-300 | Container Insights, Metrics Server |
| 08 | [Scaling](module-08-scaling/) | 300 | HPA, VPA, Cluster Autoscaler, Karpenter |
| 09 | [CI/CD](module-09-cicd/) | 300 | ArgoCD GitOps, Helm, ECR |
| 10 | [Production & DR](module-10-production/) | 300-400 | Velero, PDB, Topology Spread, Cost |

---

## 🏗️ What You'll Build

```
Module 1-3:  Cluster + first workloads
Module 4-6:  Networking, storage, security hardening
Module 7-8:  Observability + auto-scaling
Module 9:    GitOps pipeline (ArgoCD + Helm)
Module 10:   Production hardening + disaster recovery
```

---

## 🛠️ Tools Used

| Tool | Purpose |
|------|---------|
| `kubectl` | Cluster interaction |
| `aws cli` | AWS resource management |
| `helm` | Package management |
| `argocd` | GitOps deployments |
| `velero` | Backup & disaster recovery |
| `eksctl` | Cluster bootstrapping |

---

## 🚀 Getting Started

```bash
# Clone
git clone https://github.com/smustafa75/eks-learn.git
cd eks-learn

# Set environment variables
export AWS_ACCOUNT_ID=$(aws sts get-caller-identity --query Account --output text)
export AWS_REGION="us-east-1"
export CLUSTER_NAME="eks-learning-cluster"

# Start with Module 1
open module-01-fundamentals/module-01-eks-fundamentals.md
```

Each module has:
- **Concepts** — theory with diagrams/tables
- **Hands-on steps** — tested commands with expected output
- **YAML files** — ready to apply (`kubectl apply -f <file>`)
- **Exercises** — structured tasks with checkboxes
- **Key Takeaways** — what to remember
- **Cleanup** — how to tear down resources

---

## 💡 What Makes This Different

- **Battle-tested** — Every error encountered is documented in [troubleshooting.md](module-10-production/troubleshooting.md)
- **Real-world patterns** — IRSA, GitOps, Velero DR — not toy examples
- **Lessons from failure** — Node group failures, IRSA mismatches, VPC endpoint issues — all captured
- **Cost-conscious** — Uses t3.medium nodes, cleans up resources, covers Spot instances

---

## 📁 Structure

```
eks-learn/
├── module-01-fundamentals/     # EKS concepts + console exploration
├── module-02-cluster-setup/    # Create your first cluster
├── module-03-workloads/        # Deploy apps (pods, deployments, services)
├── module-04-networking/       # ALB, Ingress, VPC CNI
├── module-05-storage/          # EBS + EFS with CSI drivers
├── module-06-security/         # IRSA, RBAC, Pod Security
├── module-07-observability/    # CloudWatch Container Insights
├── module-08-scaling/          # HPA, VPA, Cluster Autoscaler
├── module-09-cicd/             # ArgoCD, Helm, ECR
├── module-10-production/       # DR, hardening, cost, compliance
│   └── troubleshooting.md      # All issues & fixes documented
└── README.md
```

---

## ⏱️ Time Investment

| Pace | Duration |
|------|----------|
| Full-time (4-6 hrs/day) | ~2 weeks |
| Part-time (1-2 hrs/day) | ~4-5 weeks |
| Weekends only | ~6-8 weeks |

---

## 📝 License

MIT — use it, fork it, learn from it.

---

## 🤝 Contributing

Found an error or have a better approach? Open an issue or PR. This curriculum improves with community input.
