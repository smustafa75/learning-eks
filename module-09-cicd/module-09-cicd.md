# Module 9: CI/CD with EKS


---

Automate your deployment pipeline. Build container images with ECR, deploy via ArgoCD GitOps (Git = source of truth), and package applications with Helm charts for repeatable, version-controlled releases.

## Concepts

| Component | What It Does |
|-----------|-------------|
| **ECR** | AWS container registry — stores Docker images |
| **CodePipeline** | AWS-native CI/CD orchestrator |
| **CodeBuild** | Builds Docker images + runs tests |
| **ArgoCD** | GitOps — syncs K8s manifests from Git to cluster |
| **Flux** | Alternative GitOps tool (lighter than ArgoCD) |
| **Helm** | Package manager for K8s (charts = reusable templates) |

### ArgoCD vs CodePipeline — Why This Module Focuses on ArgoCD

| Aspect | CodePipeline | ArgoCD |
|--------|-------------|--------|
| Approach | Imperative (`kubectl set image`) | Declarative (Git = source of truth) |
| Drift detection | None | Continuous — auto-corrects |
| Rollback | Re-run pipeline | `git revert` or UI click |
| Multi-cluster | Complex | Native |
| Audit trail | CloudTrail | Git history |
| Observability | CloudWatch | ArgoCD UI + Git diff |

> **This module focuses on ArgoCD (GitOps)** as the recommended production approach.
> CodePipeline/CodeBuild is documented as theory only (Part C) for awareness.
> Helm is covered as the packaging format that ArgoCD deploys.

---

## Part A: ECR (Container Registry)

### 9.1 Create ECR Repository

```bash
aws ecr create-repository \
  --repository-name eks-learning/web-app \
  --region us-east-1

# Get the registry URI
ECR_URI=$(aws ecr describe-repositories --repository-name eks-learning/web-app \
  --region us-east-1 --query "repositories[0].repositoryUri" --output text)
echo $ECR_URI
```

### 9.2 Build and Push a Docker Image

```bash
# Create a simple app (within the module folder)
mkdir -p web-app && cd web-app

cat > Dockerfile << 'EOF'
FROM nginx:1.27-alpine
COPY index.html /usr/share/nginx/html/
EOF

cat > index.html << 'EOF'
<h1>EKS Learning App - v1.0</h1>
<p>Deployed via CI/CD pipeline</p>
EOF

# Login to ECR
aws ecr get-login-password --region us-east-1 | docker login --username AWS --password-stdin $ECR_URI

# Build and push (use --platform for Apple Silicon Macs → EKS x86 nodes)
docker build --platform linux/amd64 -t $ECR_URI:v1 .
docker push $ECR_URI:v1
```

### 9.3 Deploy from ECR

```bash
kubectl create deployment web-app --image=$ECR_URI:v1
kubectl expose deployment web-app --port=80 --type=ClusterIP
kubectl get pods
```

**Validate:**
```bash
kubectl run curl-test --image=curlimages/curl --rm -it --restart=Never -- curl web-app
# Should show "EKS Learning App - v1.0"
```

---

## Part B: ArgoCD (GitOps)

### 9.4 Install ArgoCD

```bash
kubectl create namespace argocd

kubectl apply -n argocd -f https://raw.githubusercontent.com/argoproj/argo-cd/stable/manifests/install.yaml

# Wait for pods
kubectl get pods -n argocd -w
```

⚠️ **Troubleshooting: CRD annotation too long error**

If you see `metadata.annotations: Too long: may not be more than 262144 bytes`, re-run with `--server-side`:

```bash
kubectl apply -n argocd -f https://raw.githubusercontent.com/argoproj/argo-cd/stable/manifests/install.yaml --server-side
```

### 9.5 Access ArgoCD UI

```bash
# Get initial admin password
kubectl -n argocd get secret argocd-initial-admin-secret -o jsonpath="{.data.password}" | base64 -d
echo

# Port-forward
kubectl port-forward svc/argocd-server -n argocd 8080:443
```

Open: https://localhost:8080
- Username: `admin`
- Password: (from above command)

### 9.6 Install ArgoCD CLI

```bash
# Linux (CloudShell)
curl -sSL -o argocd https://github.com/argoproj/argo-cd/releases/latest/download/argocd-linux-amd64
chmod +x argocd && sudo mv argocd /usr/local/bin/

# macOS
brew install argocd
```

### ⚠️ Troubleshooting: Private GitHub Repository Access

If `argocd app create` fails with:
```
error testing repository connectivity: unable to ls-remote HEAD on repository:
failed to list refs: authentication required: Repository not found.
```

**Root Cause:** ArgoCD cannot access a private repo without credentials.

**Fix:** Register the repo with a GitHub PAT before creating the app:

```bash
argocd repo add https://github.com/smustafa75/eks-learn.git \
  --username smustafa75 \
  --password <GITHUB_PAT>
```

**PAT Requirements (Fine-grained token):**
1. GitHub → Settings → Developer settings → Personal access tokens → Fine-grained tokens
2. Repository access: Only select repositories → `smustafa75/eks-learn`
3. Permissions → Repository permissions:
   - **Contents:** Read-only ✅
   - **Metadata:** Read-only ✅ (auto-selected)

**Common PAT Error:** `"Write access to repository not granted"` — means the token has wrong permissions or wrong repo scope. Regenerate with correct settings above.

---

### 9.7 Create a GitOps App

**Step 1: Create K8s manifests in your repo** (under `module-09-cicd/manifests/`)

```bash
# Manifests live at: module-09-cicd/manifests/deployment.yaml
# Contains: Deployment (nginx:1.27, 2 replicas) + ClusterIP Service
```

**File:** `module-09-cicd/manifests/deployment.yaml`

**Step 2: Register app in ArgoCD**

```bash
argocd login localhost:8080 --insecure --username admin --password <PASSWORD>

argocd app create gitops-app \
  --repo https://github.com/smustafa75/eks-learn.git \
  --path module-09-cicd/manifests \
  --dest-server https://kubernetes.default.svc \
  --dest-namespace default \
  --sync-policy automated
```

**Step 3: Verify**
```bash
argocd app get gitops-app
kubectl get pods -l app=gitops-app
```

### 9.8 GitOps Workflow — Update via Git

```bash
# Change replicas in Git → ArgoCD auto-syncs
# Edit deployment.yaml: replicas: 2 → 4
# Push to Git
# ArgoCD detects change and applies it automatically
```

**Validate:**
```bash
argocd app get gitops-app
kubectl get pods -l app=gitops-app
# Should show 4 pods after sync
```

---

## Part C: CodePipeline + CodeBuild (AWS-Native CI/CD) — ⚠️ OPTIONAL / Theory Only

> **Recommendation:** Use **ArgoCD (Part B)** as your primary GitOps CI/CD approach.
> CodePipeline/CodeBuild is documented here for awareness only — it uses imperative `kubectl set image`
> which bypasses GitOps principles (Git as single source of truth). ArgoCD is declarative, auditable,
> and the industry standard for Kubernetes deployments.

### 9.9 Concept

```
Git Push → CodePipeline → CodeBuild → ECR → kubectl set image → EKS
                                              ↑ imperative — NOT GitOps
```

**Why ArgoCD wins:**
| Aspect | CodePipeline | ArgoCD |
|--------|-------------|--------|
| Source of truth | Pipeline definition | Git repo (manifests) |
| Drift detection | None | Continuous |
| Rollback | Re-run pipeline | `git revert` or ArgoCD UI |
| Multi-cluster | Complex | Native |
| Audit trail | CloudTrail | Git history |

### 9.10 buildspec.yml (Reference Only — Do Not Create)

```yaml
version: 0.2
phases:
  pre_build:
    commands:
      - echo Logging in to ECR...
      - aws ecr get-login-password --region $AWS_DEFAULT_REGION | docker login --username AWS --password-stdin $ECR_URI
      - COMMIT_HASH=$(echo $CODEBUILD_RESOLVED_SOURCE_VERSION | cut -c 1-7)
      - IMAGE_TAG=${COMMIT_HASH:=latest}
  build:
    commands:
      - echo Building Docker image...
      - docker build -t $ECR_URI:$IMAGE_TAG .
      - docker tag $ECR_URI:$IMAGE_TAG $ECR_URI:latest
  post_build:
    commands:
      - echo Pushing to ECR...
      - docker push $ECR_URI:$IMAGE_TAG
      - docker push $ECR_URI:latest
      - echo Updating EKS deployment...
      - aws eks update-kubeconfig --name eks-learning-cluster --region $AWS_DEFAULT_REGION
      - kubectl set image deployment/web-app web-app=$ECR_URI:$IMAGE_TAG
```

### 9.11 Create Pipeline (Console) — Skip

> Not creating this pipeline. If needed in future, the steps are:
> 1. CodePipeline → Create pipeline
> 2. Source: GitHub (connect your repo)
> 3. Build: CodeBuild (create new project, use `buildspec.yml`)
> 4. Deploy: Skip (handled in buildspec via kubectl)
>
> ⚠️ CodeBuild role needs: ECR access + EKS access (add to aws-auth or access entry)

---

## Part D: Helm Charts

### Why Helm After ArgoCD?

They're **complementary, not competing**:

| | ArgoCD | Helm |
|--|--------|------|
| **Role** | Deployment engine (HOW to deploy) | Packaging format (WHAT to deploy) |
| **Analogy** | The delivery truck | The boxed product |

**In production, you use BOTH together:**

```
Helm Chart (templates + values) → Pushed to Git → ArgoCD syncs to cluster
```

**Why Helm matters even with ArgoCD:**
- **Templating** — One chart, multiple environments (dev/staging/prod) via `values.yaml` overrides
- **Reusability** — Deploy nginx, postgres, redis from community charts without writing YAML
- **Versioning** — Chart version `1.2.3` → rollback to `1.2.2` cleanly
- **Dependencies** — Bundle app + database + cache as one release
- **ArgoCD natively supports Helm** — point ArgoCD at a Helm chart in Git, it renders + deploys automatically

**Without Helm:** You write raw YAML per environment, duplicating 90% of content.
**Without ArgoCD:** You run `helm install` manually — no drift detection, no auto-sync.

**Bottom line:** Helm = packaging format. ArgoCD = delivery mechanism. They're a pair in production EKS.

---

### 9.12 Create a Helm Chart

```bash
helm create my-app
# Creates: my-app/Chart.yaml, values.yaml, templates/

# Customize values.yaml
cat > my-app/values.yaml << 'EOF'
replicaCount: 2
image:
  repository: nginx
  tag: "1.27"
service:
  type: ClusterIP
  port: 80
EOF

# Install
helm install my-release my-app/

# Verify
kubectl get all -l app.kubernetes.io/instance=my-release

# Upgrade (change replicas)
helm upgrade my-release my-app/ --set replicaCount=4

# Rollback
helm rollback my-release 1

# Uninstall
helm uninstall my-release
```

**File:** `module-09-cicd/my-app/` (full Helm chart with templates, values, Chart.yaml)

---

## Part E: AWS Resilience Hub (Optional)

### 9.13 Add EKS Cluster to Resilience Hub

Resilience Hub assesses your EKS workloads for resiliency (RTO/RPO compliance).

**Pre-requisite: Grant Resilience Hub RBAC access to the cluster:**

```bash
kubectl apply -f - <<'EOF'
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRole
metadata:
  name: resilience-hub-read
rules:
- apiGroups: [""]
  resources: ["namespaces", "pods", "services", "replicationcontrollers", "persistentvolumeclaims", "persistentvolumes", "configmaps", "nodes"]
  verbs: ["get", "list"]
- apiGroups: ["apps"]
  resources: ["deployments", "replicasets", "statefulsets", "daemonsets"]
  verbs: ["get", "list"]
- apiGroups: ["batch"]
  resources: ["jobs", "cronjobs"]
  verbs: ["get", "list"]
- apiGroups: ["networking.k8s.io"]
  resources: ["ingresses", "networkpolicies"]
  verbs: ["get", "list"]
- apiGroups: ["autoscaling"]
  resources: ["horizontalpodautoscalers"]
  verbs: ["get", "list"]
---
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRoleBinding
metadata:
  name: resilience-hub-read-binding
subjects:
- kind: Group
  name: resilience-hub
  apiGroup: rbac.authorization.k8s.io
roleRef:
  kind: ClusterRole
  name: resilience-hub-read
  apiGroup: rbac.authorization.k8s.io
EOF
```

**File:** `module-09-cicd/resiliencehub-role.yaml`

**Then add to Resilience Hub:**
1. Resilience Hub Console → Create application
2. Add resource → EKS cluster → `eks-learning-cluster`
3. Select namespace: `default`
4. Run assessment

**Add Resilience Hub role as EKS access entry:**
1. EKS Console → `eks-learning-cluster` → **Access** tab → **Create access entry**
2. IAM principal ARN: `arn:aws:iam::$AWS_ACCOUNT_ID:role/ResilienceHubAssessmentRole`
3. Type: **Standard**
4. Kubernetes groups: `resilience-hub`
5. Create

⚠️ Without this access entry, Resilience Hub gets `Access Denied` when listing pods/deployments.

**For Terraform-managed resources** — allow Resilience Hub to read the TF state bucket:

```json
{
    "Version": "2012-10-17",
    "Statement": [
        {
            "Sid": "AllowResilienceHubReadState",
            "Effect": "Allow",
            "Principal": {
                "AWS": "arn:aws:iam::${AWS_ACCOUNT_ID}:role/ResilienceHubAssessmentRole"
            },
            "Action": [
                "s3:GetObject",
                "s3:ListBucket"
            ],
            "Resource": [
                "arn:aws:s3:::<TF_STATE_BUCKET>",
                "arn:aws:s3:::<TF_STATE_BUCKET>/*"
            ]
        }
    ]
}
```

Apply via: S3 Console → Bucket → Permissions → Bucket policy → Edit → Paste above (replace variables).

---

## Cleanup

```bash
# ArgoCD
argocd app delete gitops-app
kubectl delete namespace argocd

# ECR
aws ecr delete-repository --repository-name eks-learning/web-app --force --region us-east-1

# Deployments
kubectl delete deployment web-app gitops-app
kubectl delete svc web-app gitops-app-svc

# Helm
helm uninstall my-release 2>/dev/null
```

---

## Key Takeaways

1. **ECR** = store images close to EKS (fast pulls, IAM auth)
2. **ArgoCD** = GitOps — Git is the source of truth, auto-syncs to cluster ⭐ **PRIMARY CHOICE**
3. **CodePipeline/CodeBuild** = AWS-native, imperative — use only when GitOps isn't feasible
4. **Helm** = package K8s apps, version them, rollback easily
5. **GitOps > kubectl apply** in production — auditable, repeatable, declarative

---

## Exercises

| # | Task | Done |
|---|------|------|
| 1 | Create ECR repo | [ ] |
| 2 | Build and push Docker image | [ ] |
| 3 | Deploy from ECR | [ ] |
| 4 | Install ArgoCD | [ ] |
| 5 | Access ArgoCD UI | [ ] |
| 6 | Install ArgoCD CLI | [ ] |
| 7 | Create GitOps app (auto-sync from Git) | [ ] |
| 8 | Test GitOps — update Git, watch auto-deploy | [ ] |
| 9 | Create and deploy a Helm chart | [ ] |
| 10 | Helm upgrade + rollback | [ ] |
| 11 | AWS Resilience Hub (EKS integration) | [ ] |
| 12 | Clean up | [ ] |

---

## Next: Module 10 — Production Best Practices & DR
