# Module 10: Production Best Practices & DR


---

## Overview

This module covers everything needed to run EKS in production — from cluster hardening and multi-AZ resilience to disaster recovery, cost optimization, and operational runbooks. It builds on all previous modules and ties them into a production-ready architecture.

---

## Concepts

| Area | What It Covers |
|------|---------------|
| **Cluster Hardening** | Private endpoints, encryption, audit logs, Pod Security Standards |
| **Multi-AZ Resilience** | Topology spread, PDB, node group strategy |
| **Disaster Recovery** | Velero backups, cross-region, RPO/RTO planning |
| **Cost Optimization** | Spot instances, Karpenter, right-sizing, Savings Plans |
| **Operational Excellence** | Runbooks, alerting, upgrade strategy, GitOps promotion |
| **Compliance & Governance** | OPA/Gatekeeper, resource quotas, tagging |

---

## Part A: Cluster Hardening

### 10.1 Private API Endpoint

Production clusters should NOT expose the API server publicly.

```bash
# Check current endpoint access
aws eks describe-cluster --name eks-learning-cluster --region us-east-1 \
  --query "cluster.resourcesVpcConfig.{Public:endpointPublicAccess,Private:endpointPrivateAccess}"

# Update to private + restricted public (CIDR allowlist)
aws eks update-cluster-config --name eks-learning-cluster --region us-east-1 \
  --resources-vpc-config endpointPublicAccess=true,endpointPrivateAccess=true,publicAccessCidrs="<YOUR_IP>/32"
```

⚠️ **Don't disable public access entirely unless you have VPN/Direct Connect to VPC.**

### 10.2 Envelope Encryption (Secrets at Rest)

```bash
# Create KMS key for EKS secrets
KMS_KEY_ARN=$(aws kms create-key --description "EKS secrets encryption" \
  --region us-east-1 --query "KeyMetadata.Arn" --output text)

# Enable on cluster
aws eks associate-encryption-config --cluster-name eks-learning-cluster \
  --region us-east-1 \
  --encryption-config '[{"resources":["secrets"],"provider":{"keyArn":"'$KMS_KEY_ARN'"}}]'
```

### 10.3 Control Plane Logging

```bash
aws eks update-cluster-config --name eks-learning-cluster --region us-east-1 \
  --logging '{"clusterLogging":[{"types":["api","audit","authenticator","controllerManager","scheduler"],"enabled":true}]}'
```

**What each log type tells you:**
- `api` — All API server requests
- `audit` — Who did what, when (critical for compliance)
- `authenticator` — Authentication successes/failures
- `controllerManager` — Controller loop issues
- `scheduler` — Pod scheduling decisions

### 10.4 Pod Security Standards (Enforce)

```yaml
# In production, enforce restricted baseline at namespace level
apiVersion: v1
kind: Namespace
metadata:
  name: production
  labels:
    pod-security.kubernetes.io/enforce: restricted
    pod-security.kubernetes.io/warn: restricted
    pod-security.kubernetes.io/audit: restricted
```

```bash
kubectl apply -f - <<'EOF'
apiVersion: v1
kind: Namespace
metadata:
  name: prod-test
  labels:
    pod-security.kubernetes.io/enforce: restricted
    pod-security.kubernetes.io/warn: restricted
    pod-security.kubernetes.io/audit: restricted
EOF
```

**File:** `module-10-production/restrict-pods.yaml`

```bash
# Test: this pod will be REJECTED (runs as root)
kubectl run bad-pod --image=nginx --namespace=prod-test
# Should fail with: "violates PodSecurity restricted"
```

# Test: this pod will succeed (non-root, read-only, with writable scratch dirs)
kubectl apply -f - <<'EOF'
apiVersion: v1
kind: Pod
metadata:
  name: good-pod
spec:
  securityContext:
    runAsNonRoot: true
    runAsUser: 1000
    seccompProfile:
      type: RuntimeDefault
  containers:
  - name: app
    image: nginxinc/nginx-unprivileged:1.27
    securityContext:
      allowPrivilegeEscalation: false
      capabilities:
        drop: ["ALL"]
      readOnlyRootFilesystem: true
    volumeMounts:
    - name: tmp
      mountPath: /tmp
    - name: cache
      mountPath: /var/cache/nginx
    - name: run
      mountPath: /var/run
  volumes:
  - name: tmp
    emptyDir: {}
  - name: cache
    emptyDir: {}
  - name: run
    emptyDir: {}
EOF
```

⚠️ **Lesson Learned — readOnlyRootFilesystem + nginx:**
- Standard `nginx:1.27` will **crash** even though it passes Pod Security admission — it needs writable dirs at runtime
- Use `nginxinc/nginx-unprivileged:1.27` (listens on 8080, runs as non-root natively)
- Add `emptyDir` volumes for `/tmp`, `/var/cache/nginx`, `/var/run` to satisfy write requirements
- Pod Security checks happen at **admission time** — runtime failures are a separate concern

**File:** `module-10-production/secure-pod-app.yaml`

---

## Part B: Multi-AZ Resilience

### 10.5 Topology Spread Constraints

Ensure pods spread evenly across AZs (not all on one node):

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: resilient-app
spec:
  replicas: 6
  selector:
    matchLabels:
      app: resilient-app
  template:
    metadata:
      labels:
        app: resilient-app
    spec:
      topologySpreadConstraints:
      - maxSkew: 1
        topologyKey: topology.kubernetes.io/zone
        whenUnsatisfiable: DoNotSchedule
        labelSelector:
          matchLabels:
            app: resilient-app
      containers:
      - name: app
        image: nginx:1.27
        resources:
          requests:
            cpu: 100m
            memory: 128Mi
          limits:
            cpu: 200m
            memory: 256Mi
```

### 10.6 Pod Disruption Budgets (PDB)

Prevent Kubernetes from taking down too many pods during node upgrades:

```yaml
apiVersion: policy/v1
kind: PodDisruptionBudget
metadata:
  name: resilient-app-pdb
spec:
  minAvailable: 4    # Always keep at least 4 of 6 pods running
  selector:
    matchLabels:
      app: resilient-app
```

```bash
kubectl apply -f resilient-pods.yaml
kubectl apply -f pdb.yaml

# Verify spread across AZs
kubectl get pods -l app=resilient-app -o wide
# Should show pods distributed across different nodes/AZs

# Verify PDB
kubectl get pdb
```

**Files:**
- `module-10-production/resilient-pods.yaml` — Deployment with topology spread constraints
- `module-10-production/pdb.yaml` — Pod Disruption Budget (create this after verifying spread)

### 10.7 Node Group Strategy

In production, don't run all workloads on a single node group. Separate nodes by purpose so that cluster add-ons (DNS, networking) are always available, stateful apps get guaranteed capacity, and stateless apps benefit from cheaper Spot pricing. Use node labels, taints, and pod tolerations/affinity to route workloads to the correct group.

| Group | Instance Type | Purpose | Scaling |
|-------|-------------|---------|---------|
| system | t3.medium | CoreDNS, kube-proxy, add-ons | Fixed (2) |
| app-ondemand | m5.large | Stateful workloads (DBs, queues) | 2–10 |
| app-spot | m5.large, m5a.large, m4.large | Stateless workloads (APIs, web) | 0–20 |

**Best practice:** Use mixed instance types for Spot to reduce interruption risk — if one pool runs out, others keep running.

---

## Part C: Disaster Recovery

### 10.8 Velero Backup Setup

```bash
# Install Velero CLI
brew install velero

# Export account ID (use in all subsequent commands)
export AWS_ACCOUNT_ID=$(aws sts get-caller-identity --query Account --output text)
export AWS_REGION="us-east-1"

# Create S3 bucket for backups
aws s3 mb s3://eks-learning-velero-backups --region $AWS_REGION

# Create IAM policy for Velero (refer to velero-policy.json in this folder)
aws iam create-policy \
  --policy-name VeleroAccessPolicy \
  --policy-document file://velero-policy.json \
  --description "Velero backup/restore access to S3 and EBS snapshots"

# Create IRSA role for Velero
eksctl create iamserviceaccount \
  --cluster=eks-learning-cluster \
  --region=$AWS_REGION \
  --name=velero \
  --namespace=velero \
  --attach-policy-arn=arn:aws:iam::$AWS_ACCOUNT_ID:policy/VeleroAccessPolicy \
  --approve
```

**File:** `module-10-production/velero-policy.json`

### 10.9 Install Velero (Helm)

```bash
# Add Helm repo
helm repo add vmware-tanzu https://vmware-tanzu.github.io/helm-charts
helm repo update

# Get the IRSA role ARN (created by eksctl in 10.8)
VELERO_ROLE_ARN=$(aws iam list-roles --query "Roles[?contains(RoleName,'velero')].Arn" --output text)
echo $VELERO_ROLE_ARN

# Install Velero (note: single quotes required in zsh to escape [0])
helm install velero vmware-tanzu/velero \
  --namespace velero --create-namespace \
  --set credentials.useSecret=false \
  --set 'configuration.backupStorageLocation[0].name=default' \
  --set 'configuration.backupStorageLocation[0].provider=aws' \
  --set 'configuration.backupStorageLocation[0].bucket=eks-learning-velero-backups' \
  --set 'configuration.backupStorageLocation[0].config.region=us-east-1' \
  --set 'configuration.volumeSnapshotLocation[0].name=default' \
  --set 'configuration.volumeSnapshotLocation[0].provider=aws' \
  --set 'configuration.volumeSnapshotLocation[0].config.region=us-east-1' \
  --set serviceAccount.server.create=true \
  --set serviceAccount.server.name=velero-server \
  --set "serviceAccount.server.annotations.eks\.amazonaws\.com/role-arn=$VELERO_ROLE_ARN" \
  --set 'initContainers[0].name=velero-plugin-for-aws' \
  --set 'initContainers[0].image=velero/velero-plugin-for-aws:v1.10.0' \
  --set 'initContainers[0].volumeMounts[0].mountPath=/target' \
  --set 'initContainers[0].volumeMounts[0].name=plugins'
```

⚠️ **Critical notes:**
- `credentials.useSecret=false` — required for IRSA to work (otherwise an empty credentials file overrides the web identity token)
- Single quotes around `[0]` values — zsh interprets brackets as glob patterns
- `serviceAccount.server.name=velero-server` — must match the trust policy subject
- If BSL shows `Unavailable`, check `troubleshooting.md` for common fixes

**After install, update trust policy** to match the SA name used by the Helm chart:

```bash
# The trust policy must reference: system:serviceaccount:velero:velero-server
# See velero-trust-policy.json in this folder
aws iam update-assume-role-policy \
  --role-name <ROLE_NAME_FROM_EKSCTL> \
  --policy-document file://velero-trust-policy.json
```

**File:** `module-10-production/velero-trust-policy.json`

**Verify BSL is Available:**
```bash
velero backup-location get
# Should show Phase: Available
```

### 10.10 Backup & Restore Test

```bash
# Create a test namespace with data
kubectl create namespace dr-test
kubectl run dr-app --image=nginx --namespace=dr-test
kubectl expose pod dr-app --port=80 --namespace=dr-test

# Backup
velero backup create dr-test-backup --include-namespaces dr-test

# Check status
velero backup describe dr-test-backup
velero backup logs dr-test-backup

# Simulate disaster
kubectl delete namespace dr-test

# Restore
velero restore create --from-backup dr-test-backup

# Verify
kubectl get all -n dr-test
```

### 10.11 Scheduled Backups

```bash
# Daily backup with 7-day retention
velero schedule create daily-backup \
  --schedule="0 2 * * *" \
  --ttl 168h0m0s \
  --include-namespaces default,production

# Verify schedule
velero schedule get
```

### 10.12 Cross-Region DR (Theory)

```
Primary (us-east-1)                    DR (eu-west-1)
┌─────────────────┐                   ┌─────────────────┐
│  EKS Cluster    │                   │  EKS Cluster    │
│  (Active)       │                   │  (Warm Standby) │
└────────┬────────┘                   └────────┬────────┘
         │                                     │
         ▼                                     ▼
┌─────────────────┐    S3 Replication  ┌─────────────────┐
│  Velero BSL     │ ──────────────────▶│  Velero BSL     │
│  (S3 bucket)    │                    │  (S3 bucket)    │
└─────────────────┘                    └─────────────────┘
         │
         ▼
┌─────────────────┐
│  Route53        │  ── Failover DNS
└─────────────────┘
```

**DR Strategies (choose based on RTO/RPO):**

| Strategy | RTO | RPO | Cost | Notes |
|----------|-----|-----|------|-------|
| Backup & Restore | 2–4h | 24h | $ | Cheapest, longest recovery |
| Pilot Light | 30–60min | 4h | $$ | Minimal infra running |
| Warm Standby | 10–15min | Minutes | $$$ | Scaled-down cluster running |
| Active-Active | ~0 | ~0 | $$$$ | Full capacity both regions |

---

## Part D: Cost Optimization

### 10.13 Spot Instances — Theory Only

> **Not creating Spot node group in this lab** — documented for production reference.

```bash
# Create a Spot node group (for stateless workloads)
aws eks create-nodegroup \
  --cluster-name eks-learning-cluster \
  --nodegroup-name spot-workers \
  --instance-types m5.large m5a.large m4.large c5.large \
  --capacity-type SPOT \
  --scaling-config minSize=0,maxSize=10,desiredSize=2 \
  --region us-east-1 \
  --subnets $PRIVATE_SUBNET_1 $PRIVATE_SUBNET_2 \
  --node-role $NODE_ROLE_ARN
```

**Spot Best Practices:**
- Use `topologySpreadConstraints` across AZs
- Mix 4+ instance types to reduce interruption
- Use PDB to protect minimum availability
- Only for stateless, fault-tolerant workloads
- Handle interruption with AWS Node Termination Handler

### 10.14 Right-Sizing with VPA Recommendations

```bash
# From Module 8 — use VPA in recommend mode (don't auto-apply in prod)
kubectl get vpa -A -o custom-columns="NAME:.metadata.name,CPU_REQ:.status.recommendation.containerRecommendations[0].target.cpu,MEM_REQ:.status.recommendation.containerRecommendations[0].target.memory"
```

### 10.15 Cost Monitoring

```bash
# Install kubecost (free tier — pin version to avoid v2.9→v3 migration issues)
helm repo add kubecost https://kubecost.github.io/cost-analyzer/
helm repo update

helm install kubecost kubecost/cost-analyzer \
  --namespace kubecost --create-namespace \
  --set global.clusterId=eks-learning-cluster \
  --set finopsAgent.clusterId=eks-learning-cluster \
  --set 'persistentVolume.storageClass=gp2' \
  --set 'prometheus.server.persistentVolume.storageClass=gp2' \
  --version 2.6.1

# Wait for pods (may take 2-3 minutes for PVs to provision)
kubectl get pods -n kubecost -w

# Access dashboard
kubectl port-forward svc/kubecost-cost-analyzer -n kubecost 9090:9090
```

⚠️ **Notes:**
- Pin to `--version 2.6.1` — Kubecost v2.9+ is a transitional release for v3 migration
- Set `persistentVolume.storageClass=gp2` — without it, PVCs stay Pending (no default StorageClass)
- Dashboard needs **15–30 min** to show data, **24h** for accurate daily costs
- See `troubleshooting.md` for PVC and version issues

---

## Part E: Operational Excellence

### 10.16 EKS Upgrade Strategy — Theory Only

> **Not performing an upgrade in this lab** — documented as a production runbook reference.

**EKS releases a new K8s version every ~4 months. Upgrade path:**

```
1. Read release notes + deprecation notices
2. Update add-ons first (VPC CNI, CoreDNS, kube-proxy, EBS CSI)
3. Update control plane (aws eks update-cluster-version)
4. Update node groups (rolling update or blue/green)
5. Test workloads
6. Update kubectl + Helm charts if needed
```

```bash
# Check current version
kubectl version --short

# Check available versions
aws eks describe-addon-versions --kubernetes-version 1.36 --query "addons[].addonName" --output table

# Upgrade control plane
aws eks update-cluster-version --name eks-learning-cluster --kubernetes-version 1.36 --region us-east-1

# Upgrade add-ons
aws eks update-addon --cluster-name eks-learning-cluster --addon-name vpc-cni --region us-east-1
aws eks update-addon --cluster-name eks-learning-cluster --addon-name coredns --region us-east-1
aws eks update-addon --cluster-name eks-learning-cluster --addon-name kube-proxy --region us-east-1
```

### 10.17 GitOps Promotion (Dev → Staging → Prod) — Theory Only

> **Not implementing multi-env in this lab** — documented as a production GitOps pattern reference.
> Requires multiple namespaces/clusters and ArgoCD ApplicationSets.

```
eks-learn/
├── environments/
│   ├── dev/
│   │   └── values.yaml       # replicas: 1, resources: small
│   ├── staging/
│   │   └── values.yaml       # replicas: 2, resources: medium
│   └── prod/
│       └── values.yaml        # replicas: 6, resources: large, PDB, topology spread
├── base/
│   ├── deployment.yaml
│   ├── service.yaml
│   └── kustomization.yaml
```

**ArgoCD ApplicationSet for multi-env:**

```yaml
apiVersion: argoproj.io/v1alpha1
kind: ApplicationSet
metadata:
  name: my-app
  namespace: argocd
spec:
  generators:
  - list:
      elements:
      - cluster: eks-learning-cluster
        env: dev
      - cluster: eks-learning-cluster
        env: staging
      - cluster: eks-prod-cluster
        env: prod
  template:
    metadata:
      name: 'my-app-{{env}}'
    spec:
      project: default
      source:
        repoURL: https://github.com/<YOUR_ORG>/eks-app.git
        path: 'environments/{{env}}'
      destination:
        server: https://kubernetes.default.svc
        namespace: '{{env}}'
      syncPolicy:
        automated:
          prune: true
```

### 10.18 Alerting Rules

Proactive alerting is critical in production — you need to know when pods are crash looping, nodes go down, or resources are exhausted **before** users report issues. PrometheusRules define alert conditions that trigger notifications via Alertmanager.

```bash
# Apply alerting rules (requires Prometheus Operator CRD)
kubectl apply -f alerting-rules.yaml

# Verify
kubectl get prometheusrule eks-alerts
```

**File:** `module-10-production/alerting-rules.yaml`

Defines 3 critical/warning alerts:
- **PodCrashLooping** — pod restarting > 0.1/sec for 5 min
- **NodeNotReady** — node not ready for 2 min
- **HighMemoryUsage** — node memory above 90% for 5 min

---

## Part F: Compliance & Governance

### 10.19 Resource Quotas & Limit Ranges

In multi-team environments, one namespace can starve others by consuming all cluster resources. **ResourceQuotas** cap total resource usage per namespace, while **LimitRanges** set default/max values per container — ensuring no single pod can request excessive CPU/memory and every pod has sensible defaults even if the developer forgets to set them.

```bash
# Create the namespace first
kubectl create namespace production

# Apply quotas and limit ranges
kubectl apply -f quota-limit.yaml

# Verify
kubectl get resourcequota -n production
kubectl get limitrange -n production
```

**File:** `module-10-production/quota-limit.yaml`

### 10.20 OPA Gatekeeper (Policy as Code) — Skipped

> **Removed from hands-on exercises.** Gatekeeper CRD registration has timing issues that make it unreliable in lab environments. Concept is documented below for reference.

**What it does:** Gatekeeper acts as a Kubernetes admission controller that rejects resources at deploy time if they violate policies (e.g., blocking images from untrusted registries, requiring resource limits, mandating labels). Policies are defined as Rego code — version-controlled and auditable.

**When to use in production:** When you need to enforce organizational policies across multiple teams/namespaces without relying on developer discipline.

**Alternative:** Kyverno — simpler YAML-based policies, no Rego required, better suited for smaller teams.

---

## Production Checklist

| # | Item | Category |
|---|------|----------|
| 1 | Private API endpoint (or CIDR-restricted) | Security |
| 2 | Secrets encryption with KMS | Security |
| 3 | Control plane logging enabled (all 5 types) | Observability |
| 4 | Pod Security Standards enforced | Security |
| 5 | Topology spread across AZs | Resilience |
| 6 | PDBs on all critical workloads | Resilience |
| 7 | Resource requests/limits on all pods | Stability |
| 8 | VPA recommendations reviewed quarterly | Cost |
| 9 | Velero scheduled backups + tested restore | DR |
| 10 | EKS upgrade plan documented | Operations |
| 11 | GitOps (ArgoCD) for all deployments | Operations |
| 12 | Alerting rules for crash loops, node failures | Observability |
| 13 | Resource quotas per namespace | Governance |
| 14 | Image registry restriction (ECR only in prod) | Security (theory) |
| 15 | Spot instances for stateless workloads | Cost |

---

## Exercises

| # | Task | Done |
|---|------|------|
| 1 | Enable control plane logging | [ ] |
| 2 | Restrict API endpoint (CIDR allowlist) | [ ] |
| 3 | Create namespace with Pod Security restricted | [ ] |
| 4 | Deploy app with topology spread + PDB | [ ] |
| 5 | Install Velero and run backup/restore test | [ ] |
| 6 | Create scheduled backup (daily, 7-day TTL) | [ ] |
| 7 | Set up resource quotas + limit ranges | [ ] |
| 8 | ~~Gatekeeper + ECR-only policy~~ | Skipped |
| 9 | Review production checklist (audit your cluster) | [ ] |
| 10 | Clean up | [ ] |

---

## Cleanup

Remove all resources created during Module 10 exercises:

```bash
# Velero + scheduled backup
velero schedule delete daily-backup --confirm
helm uninstall velero -n velero
kubectl delete namespace velero

# Kubecost
helm uninstall kubecost -n kubecost
kubectl delete namespace kubecost

# Resilient app + PDB
kubectl delete deployment resilient-app
kubectl delete pdb resilient-app-pdb

# Alerting rules
kubectl delete prometheusrule eks-alerts

# Resource quotas namespace
kubectl delete namespace production

# DR test namespace
kubectl delete namespace dr-test

# Gatekeeper (if installed)
kubectl delete namespace gatekeeper-system 2>/dev/null

# S3 bucket (optional — keep if you want backups preserved)
# aws s3 rb s3://eks-learning-velero-backups --force

# IAM cleanup (optional)
# aws iam delete-policy --policy-arn arn:aws:iam::$AWS_ACCOUNT_ID:policy/VeleroAccessPolicy
# eksctl delete iamserviceaccount --cluster=eks-learning-cluster --region=us-east-1 --name=velero --namespace=velero
```

---

## Key Takeaways

1. **Security first** — Private endpoint, KMS encryption, Pod Security, audit logs
2. **Resilience** — Topology spread + PDB = survive AZ failures and upgrades
3. **DR is not optional** — Velero + tested restores. Untested backups = no backups
4. **Cost awareness** — Spot for stateless, right-size with VPA, monitor with Kubecost
5. **GitOps everything** — ArgoCD + Helm + environment overlays = repeatable, auditable
6. **Policy as Code** — OPA Gatekeeper prevents bad configs from reaching the cluster
7. **Upgrade early, upgrade often** — Don't fall more than 2 versions behind

---

## Relationship to Your DR Work

This module's concepts directly apply to real-world DR projects:
- **10.8–10.12**: Velero patterns for cross-platform DR (e.g., on-prem → EKS)
- **10.17**: GitOps promotion with ArgoCD ApplicationSets
- **10.7**: Node group strategy for warm standby cost optimization

---

## Next Steps After Module 10

🎉 **Curriculum complete!** Apply these skills to:
1. Build Terraform modules for production EKS
2. Implement Velero DR for on-prem → EKS migration
3. Design ArgoCD multi-cluster GitOps for client deployments
