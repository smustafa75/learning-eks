# Module 3: Deploying Workloads


---

Deploy your first applications on EKS — from running a single pod to managing multi-replica deployments with services. Learn both imperative (CLI) and declarative (YAML) approaches.

## Concepts

| K8s Object | What It Does | Analogy |
|------------|-------------|---------|
| **Pod** | Runs 1+ containers | A single running task |
| **Deployment** | Manages pod replicas + rolling updates | Auto Scaling Group |
| **Service** | Stable network endpoint for pods | Load Balancer |
| **Namespace** | Logical isolation | Separate environments |

---

## 3.1 Run Your First Pod

```bash
# Run a single nginx pod
kubectl run nginx --image=nginx:latest

# Check it's running
kubectl get pods

# See more details
kubectl describe pod nginx

# Check logs
kubectl logs nginx

# Access the pod shell
kubectl exec -it nginx -- /bin/bash
# Inside: curl localhost → you'll see nginx welcome page
# Type: exit

# Delete the pod
kubectl delete pod nginx
```

**Key learning:** Pods are ephemeral — if they die, they're gone. That's why we use Deployments.

---

## 3.2 Create a Deployment

A Deployment ensures your desired number of pods are always running.

```bash
# Create deployment with 3 replicas
kubectl create deployment nginx-app --image=nginx:latest --replicas=3

# Watch pods come up
kubectl get pods -w
# (Ctrl+C to stop watching)

# Check deployment status
kubectl get deployment nginx-app

# Check replica set (manages the pods)
kubectl get replicaset
```

**Scale it:**
```bash
# Scale to 5
kubectl scale deployment nginx-app --replicas=5
kubectl get pods

# Scale back to 2
kubectl scale deployment nginx-app --replicas=2
kubectl get pods
```

**Update image (rolling update):**
```bash
# Update to a specific version
kubectl set image deployment/nginx-app nginx=nginx:1.27

# Watch the rollout
kubectl rollout status deployment/nginx-app

# Check rollout history
kubectl rollout history deployment/nginx-app

# Rollback if needed
kubectl rollout undo deployment/nginx-app
```

---

## 3.3 Expose with a Service

Pods get random IPs that change. A Service gives a **stable endpoint**.

### ClusterIP (internal only — default)
```bash
kubectl expose deployment nginx-app --port=80 --target-port=80 --type=ClusterIP --name=nginx-svc

# Check service
kubectl get svc nginx-svc

# Test from inside the cluster (spin up a temp pod)
kubectl run curl-test --image=curlimages/curl --rm -it -- curl nginx-svc
```

### NodePort (accessible on node IP:port)
```bash
kubectl expose deployment nginx-app --port=80 --target-port=80 --type=NodePort --name=nginx-nodeport

# Check — note the assigned port (30000-32767 range)
kubectl get svc nginx-nodeport
```

### LoadBalancer (creates AWS ELB)
```bash
kubectl expose deployment nginx-app --port=80 --target-port=80 --type=LoadBalancer --name=nginx-lb

# Check — wait for EXTERNAL-IP to appear
kubectl get svc nginx-lb -w
```

⚠️ Since nodes are private, LoadBalancer type creates an internal NLB. For public access, you'll need an Ingress Controller (Module 4).

---

## 3.4 YAML Way (Declarative — Production Approach)

Create a file `nginx-deployment.yaml`:
```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: nginx-web
  labels:
    app: nginx-web
spec:
  replicas: 3
  selector:
    matchLabels:
      app: nginx-web
  template:
    metadata:
      labels:
        app: nginx-web
    spec:
      containers:
      - name: nginx
        image: nginx:1.27
        ports:
        - containerPort: 80
        resources:
          requests:
            cpu: 100m
            memory: 128Mi
          limits:
            cpu: 250m
            memory: 256Mi
---
apiVersion: v1
kind: Service
metadata:
  name: nginx-web-svc
spec:
  selector:
    app: nginx-web
  ports:
  - port: 80
    targetPort: 80
  type: ClusterIP
```

```bash
# Apply
kubectl apply -f nginx-deployment.yaml

# Check everything
kubectl get all -l app=nginx-web

# Delete everything
kubectl delete -f nginx-deployment.yaml
```

**File:** `module-03-workloads/nginx-deployment.yaml`

---

## 3.5 Namespaces

```bash
# Create a namespace
kubectl create namespace dev

# Deploy into it
kubectl create deployment nginx-dev --image=nginx --replicas=2 -n dev

# List pods in that namespace
kubectl get pods -n dev

# List pods in ALL namespaces
kubectl get pods -A

# Delete namespace (deletes everything inside)
kubectl delete namespace dev
```

---

## 3.6 Useful Commands Cheat Sheet

| Command | What It Does |
|---------|-------------|
| `kubectl get pods` | List pods |
| `kubectl get pods -o wide` | List pods with node/IP info |
| `kubectl describe pod <name>` | Detailed pod info + events |
| `kubectl logs <pod>` | View pod logs |
| `kubectl logs <pod> -f` | Stream logs live |
| `kubectl exec -it <pod> -- /bin/bash` | Shell into pod |
| `kubectl get all` | List all resources |
| `kubectl delete pod <name>` | Delete a pod |
| `kubectl apply -f <file>` | Apply YAML config |
| `kubectl delete -f <file>` | Delete resources from YAML |

---

## Exercises

| # | Task | Done |
|---|------|------|
| 1 | Run a single pod and access its shell | [ ] |
| 2 | Create a deployment with 3 replicas | [ ] |
| 3 | Scale it up to 5, then down to 2 | [ ] |
| 4 | Expose it as ClusterIP and test with curl pod | [ ] |
| 5 | Create a deployment using YAML file | [ ] |
| 6 | Deploy into a custom namespace | [ ] |
| 7 | Clean up all resources | [ ] |

---

## Key Takeaways

1. **Pods** are ephemeral — always use Deployments to manage them
2. **Deployments** handle rolling updates, scaling, and self-healing
3. **Services** give stable networking to pods (ClusterIP for internal, NodePort/LoadBalancer for external)
4. **YAML > CLI** in production — declarative, version-controlled, repeatable
5. **Namespaces** isolate workloads logically — use them from the start

---

## Next: Module 4 — Networking (VPC CNI, Ingress, ALB Controller)
