# Module 6: Security


---

Lock down your cluster. Implement least-privilege access with IRSA and RBAC, protect sensitive data with Secrets, enforce Pod Security Standards, and isolate network traffic between namespaces.

## Concepts

| Component | What It Does |
|-----------|-------------|
| **RBAC** | Controls who can do what inside the cluster |
| **IRSA** | Gives pods AWS permissions without node-level access |
| **Service Accounts** | Identity for pods (like IAM users for containers) |
| **Secrets** | Store sensitive data (passwords, tokens, keys) |
| **Pod Security Standards** | Restrict what pods can do (no root, no host network, etc.) |
| **Network Policies** | Firewall rules between pods |

---

## Part A: RBAC (Role-Based Access Control)

### 6.1 Understand RBAC Objects

| Object | Scope | Purpose |
|--------|-------|---------|
| `Role` | Namespace | Permissions within a namespace |
| `ClusterRole` | Cluster-wide | Permissions across all namespaces |
| `RoleBinding` | Namespace | Assigns Role to a user/group/SA |
| `ClusterRoleBinding` | Cluster-wide | Assigns ClusterRole to a user/group/SA |

### 6.2 Create a Namespace + Service Account

```bash
# Create namespace
kubectl create namespace dev-team

# Create service account
kubectl create serviceaccount dev-user -n dev-team
```

### 6.3 Create a Role (namespace-scoped permissions)

```bash
cat > dev-role.yaml << 'EOF'
apiVersion: rbac.authorization.k8s.io/v1
kind: Role
metadata:
  name: dev-role
  namespace: dev-team
rules:
- apiGroups: [""]
  resources: ["pods", "services", "configmaps"]
  verbs: ["get", "list", "watch", "create", "delete"]
- apiGroups: ["apps"]
  resources: ["deployments"]
  verbs: ["get", "list", "watch", "create", "update", "delete"]
EOF

kubectl apply -f dev-role.yaml
```

**File:** `module-06-security/dev-role.yaml`

### 6.4 Bind Role to Service Account

```bash
cat > dev-rolebinding.yaml << 'EOF'
apiVersion: rbac.authorization.k8s.io/v1
kind: RoleBinding
metadata:
  name: dev-rolebinding
  namespace: dev-team
subjects:
- kind: ServiceAccount
  name: dev-user
  namespace: dev-team
roleRef:
  kind: Role
  name: dev-role
  apiGroup: rbac.authorization.k8s.io
EOF

kubectl apply -f dev-rolebinding.yaml
```

**File:** `module-06-security/dev-rolebinding.yaml`

### 6.5 Test RBAC Permissions

```bash
# Can dev-user list pods in dev-team? (should be YES)
kubectl auth can-i list pods -n dev-team --as=system:serviceaccount:dev-team:dev-user

# Can dev-user list pods in default? (should be NO)
kubectl auth can-i list pods -n default --as=system:serviceaccount:dev-team:dev-user

# Can dev-user delete nodes? (should be NO)
kubectl auth can-i delete nodes --as=system:serviceaccount:dev-team:dev-user
```

**Validate:**
```bash
kubectl get role,rolebinding -n dev-team
```

---

## Part B: IRSA (IAM Roles for Service Accounts)

### 6.6 Concept

IRSA = Pod gets its own IAM role (not the node's role). Principle of least privilege.

```
Pod → Service Account → IAM Role → AWS Permissions
```

### 6.7 Create an IRSA Role (example: S3 read access)

```bash
# Create service account with IAM role
eksctl create iamserviceaccount \
  --cluster=eks-learning-cluster \
  --namespace=default \
  --name=s3-reader \
  --attach-policy-arn=arn:aws:iam::aws:policy/AmazonS3ReadOnlyAccess \
  --region us-east-1 \
  --approve
```

### 6.8 Test IRSA — Pod with S3 Access

```bash
cat > s3-test-pod.yaml << 'EOF'
apiVersion: v1
kind: Pod
metadata:
  name: s3-test
spec:
  serviceAccountName: s3-reader
  containers:
  - name: aws-cli
    image: amazon/aws-cli:latest
    command: ["sleep", "3600"]
EOF

kubectl apply -f s3-test-pod.yaml

# Wait for pod to be running
kubectl get pod s3-test -w

# Exec into pod and test S3 access
kubectl exec -it s3-test -- aws s3 ls
# Should list buckets (read-only)

kubectl exec -it s3-test -- aws ec2 describe-instances --region us-east-1
# Should FAIL (no EC2 permissions)
```

**File:** `module-06-security/s3-test-pod.yaml`

**Validate:**
```bash
kubectl describe sa s3-reader
# Look for annotation: eks.amazonaws.com/role-arn
```

---

## Part C: Kubernetes Secrets

### 6.9 Create Secrets

```bash
# From literal values
kubectl create secret generic db-creds \
  --from-literal=username=admin \
  --from-literal=password=SuperSecret123

# View secret (base64 encoded)
kubectl get secret db-creds -o yaml

# Decode
kubectl get secret db-creds -o jsonpath='{.data.password}' | base64 -d
```

### 6.10 Use Secrets in a Pod

```bash
cat > secret-pod.yaml << 'EOF'
apiVersion: v1
kind: Pod
metadata:
  name: secret-test
spec:
  containers:
  - name: app
    image: nginx
    env:
    - name: DB_USER
      valueFrom:
        secretKeyRef:
          name: db-creds
          key: username
    - name: DB_PASS
      valueFrom:
        secretKeyRef:
          name: db-creds
          key: password
    volumeMounts:
    - name: secret-volume
      mountPath: /etc/secrets
      readOnly: true
  volumes:
  - name: secret-volume
    secret:
      secretName: db-creds
EOF

kubectl apply -f secret-pod.yaml

# Verify env vars
kubectl exec secret-test -- env | grep DB_

# Verify mounted files
kubectl exec secret-test -- cat /etc/secrets/username
kubectl exec secret-test -- cat /etc/secrets/password
```

**File:** `module-06-security/secret-pod.yaml`

---

## Part D: Pod Security

### 6.11 Pod Security Standards (PSS)

Three levels:
| Level | What It Allows |
|-------|---------------|
| `privileged` | Everything (no restrictions) |
| `baseline` | Blocks known privilege escalations |
| `restricted` | Hardened (no root, no host access, read-only root FS) |

### 6.12 Enforce Pod Security on a Namespace

```bash
# Create namespace with restricted security
kubectl create namespace secure-ns

# Label it to enforce restricted policy
kubectl label namespace secure-ns \
  pod-security.kubernetes.io/enforce=restricted \
  pod-security.kubernetes.io/warn=restricted

# Try deploying a privileged pod (should FAIL)
kubectl run priv-test --image=nginx --privileged -n secure-ns
# Error: violates PodSecurity "restricted"

# Deploy a compliant pod (should SUCCEED)
cat > secure-pod.yaml << 'EOF'
apiVersion: v1
kind: Pod
metadata:
  name: secure-app
  namespace: secure-ns
spec:
  securityContext:
    runAsNonRoot: true
    runAsUser: 1000
    seccompProfile:
      type: RuntimeDefault
  containers:
  - name: app
    image: nginx
    securityContext:
      allowPrivilegeEscalation: false
      capabilities:
        drop: ["ALL"]
      readOnlyRootFilesystem: true
EOF

kubectl apply -f secure-pod.yaml
```

**File:** `module-06-security/secure-pod.yaml`

---

## Part E: Network Policies

### 6.13 Restrict Pod-to-Pod Traffic

```bash
# Deploy two apps
kubectl create namespace netpol-test
kubectl create deployment web --image=nginx --replicas=1 -n netpol-test
kubectl create deployment api --image=nginx --replicas=1 -n netpol-test
kubectl expose deployment web --port=80 -n netpol-test
kubectl expose deployment api --port=80 -n netpol-test

# By default, all pods can talk to each other
# Now restrict: only web can talk to api

cat > network-policy.yaml << 'EOF'
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: api-allow-web-only
  namespace: netpol-test
spec:
  podSelector:
    matchLabels:
      app: api
  policyTypes:
  - Ingress
  ingress:
  - from:
    - podSelector:
        matchLabels:
          app: web
    ports:
    - port: 80
EOF

kubectl apply -f network-policy.yaml
```

**File:** `module-06-security/network-policy.yaml`

**Test:**
```bash
# From web pod → api (should WORK)
kubectl exec -n netpol-test deploy/web -- curl -s --max-time 3 api

# From a random pod → api (should TIMEOUT)
kubectl run test-curl --image=curlimages/curl --rm -it -n netpol-test -- curl -s --max-time 3 api
```

⚠️ **Note:** Network Policies require a CNI that supports them. VPC CNI supports network policies on EKS 1.25+.

---

## Cleanup

```bash
kubectl delete namespace dev-team secure-ns netpol-test
kubectl delete pod s3-test secret-test
kubectl delete secret db-creds
kubectl delete -f dev-role.yaml -f dev-rolebinding.yaml 2>/dev/null
eksctl delete iamserviceaccount --cluster=eks-learning-cluster --namespace=default --name=s3-reader --region us-east-1
```

---

## Key Takeaways

1. **RBAC** = least privilege inside the cluster (who can do what)
2. **IRSA** = least privilege for AWS access (pod-level, not node-level)
3. **Secrets** = store sensitive data (env vars or mounted files)
4. **Pod Security Standards** = enforce security at namespace level
5. **Network Policies** = firewall between pods (east-west traffic)

---

## Exercises

| # | Task | Done |
|---|------|------|
| 1 | Create namespace + service account | [ ] |
| 2 | Create Role with limited permissions | [ ] |
| 3 | Bind Role and test with `auth can-i` | [ ] |
| 4 | Create IRSA role for S3 access | [ ] |
| 5 | Test pod with IRSA (S3 works, EC2 fails) | [ ] |
| 6 | Create and use Secrets (env + volume) | [ ] |
| 7 | Enforce restricted Pod Security on namespace | [ ] |
| 8 | Create Network Policy and test isolation | [ ] |
| 9 | Clean up all resources | [ ] |

---

## Next: Module 7 — Observability (CloudWatch, Prometheus, Grafana)
