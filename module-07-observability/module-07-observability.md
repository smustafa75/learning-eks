# Module 7: Observability


---

See what's happening inside your cluster. Set up CloudWatch Container Insights for logs and metrics, install Prometheus + Grafana for dashboards, and enable the Metrics Server that powers auto-scaling in Module 8.

## Concepts

| Component | What It Does |
|-----------|-------------|
| **CloudWatch Container Insights** | AWS-native metrics + logs for EKS |
| **Prometheus** | Open-source metrics collection + alerting |
| **Grafana** | Visualization dashboards for Prometheus data |
| **Fluent Bit** | Log forwarding (pods → CloudWatch/S3/etc.) |
| **ADOT** | AWS Distro for OpenTelemetry (traces + metrics) |

---

## Part A: CloudWatch Container Insights

### 7.1 Enable Container Insights (CloudWatch Agent + Fluent Bit)

**Step 1: Create IRSA for CloudWatch Agent:**
```bash
eksctl create iamserviceaccount \
  --cluster=eks-learning-cluster \
  --namespace=amazon-cloudwatch \
  --name=cloudwatch-agent \
  --attach-policy-arn=arn:aws:iam::aws:policy/CloudWatchAgentServerPolicy \
  --attach-policy-arn=arn:aws:iam::aws:policy/AWSXrayWriteOnlyAccess \
  --region us-east-1 \
  --approve --override-existing-serviceaccounts

# Get the role ARN
ROLE_ARN=$(kubectl get sa cloudwatch-agent -n amazon-cloudwatch -o jsonpath='{.metadata.annotations.eks\.amazonaws\.com/role-arn}')
echo $ROLE_ARN
```

**Step 2: Install add-on with IRSA role:**
```bash
aws eks create-addon \
  --cluster-name eks-learning-cluster \
  --addon-name amazon-cloudwatch-observability \
  --service-account-role-arn $ROLE_ARN \
  --region us-east-1
```

Or via Console: EKS → Cluster → Add-ons → Add → **Amazon CloudWatch Observability** → Select the IRSA role

⚠️ **Without IRSA**, the agent falls back to the node role which lacks CloudWatch permissions → `AccessDenied` errors in logs.

### 7.2 Validate Container Insights

```bash
# Check pods running in amazon-cloudwatch namespace
kubectl get pods -n amazon-cloudwatch

# Check DaemonSets (should see cloudwatch-agent + fluent-bit)
kubectl get daemonset -n amazon-cloudwatch
```

**View in Console:**
- CloudWatch → Container Insights → Select cluster `eks-learning-cluster`
- See: CPU, Memory, Network, Pod count per node

### 7.3 View Pod Logs in CloudWatch

```bash
# Deploy a test app that generates logs
kubectl run log-test --image=busybox -- sh -c "while true; do echo 'Hello from EKS - $(date)'; sleep 5; done"

# Verify it's running
kubectl logs log-test --tail=5
```

**View in Console:**
- CloudWatch → Log groups → `/aws/containerinsights/eks-learning-cluster/application`
- Find your pod's log stream

**Cleanup:**
```bash
kubectl delete pod log-test
```

---

## Part B: Prometheus + Grafana

### 7.4 Install Prometheus (via Helm)

```bash
# Add Helm repos
helm repo add prometheus-community https://prometheus-community.github.io/helm-charts
helm repo update

# Create namespace
kubectl create namespace monitoring

# Install kube-prometheus-stack (Prometheus + Grafana + AlertManager)
helm install prometheus prometheus-community/kube-prometheus-stack \
  --namespace monitoring \
  --set grafana.adminPassword=EksLearning123
```

### 7.5 Validate Prometheus

```bash
# Check all pods in monitoring namespace
kubectl get pods -n monitoring

# Check services
kubectl get svc -n monitoring
```

Expected pods:
- `prometheus-kube-prometheus-operator`
- `prometheus-prometheus-kube-prometheus-prometheus`
- `prometheus-grafana`
- `alertmanager`
- `node-exporter` (DaemonSet)

### 7.6 Access Grafana Dashboard

```bash
# Port-forward Grafana to localhost
kubectl port-forward svc/prometheus-grafana 3000:80 -n monitoring
```

Then open: http://localhost:3000
- Username: `admin`
- Password: `EksLearning123`

**Pre-built dashboards:**
- Go to Dashboards → Browse
- Look for: "Kubernetes / Compute Resources / Cluster"
- Also: "Kubernetes / Compute Resources / Namespace (Pods)"

⚠️ In CloudShell, port-forward won't work (no browser). Use from your local machine instead, or expose via LoadBalancer (not recommended for production).

### 7.7 Access Prometheus UI

```bash
kubectl port-forward svc/prometheus-kube-prometheus-prometheus 9090:9090 -n monitoring
```

Open: http://localhost:9090
- Try query: `container_cpu_usage_seconds_total`
- Try query: `kube_pod_status_phase{phase="Running"}`

### 7.8 Create a Custom Alert Rule

PrometheusRules define alert conditions that fire when metrics exceed thresholds. This example alerts when any pod's CPU exceeds 80% for 2 minutes.

```bash
kubectl apply -f high-cpu-alert.yaml

# Verify
kubectl get prometheusrule -n monitoring
```

**File:** `module-07-observability/high-cpu-alert.yaml`

---

## Part C: Metrics Server (for HPA — needed in Module 8)

### 7.9 Install Metrics Server

```bash
# Check if already installed
kubectl get deployment metrics-server -n kube-system

# If not installed:
kubectl apply -f https://github.com/kubernetes-sigs/metrics-server/releases/latest/download/components.yaml
```

### 7.10 Validate Metrics Server

```bash
# Check it's running
kubectl get pods -n kube-system | grep metrics-server

# Test — should show CPU/memory per node
kubectl top nodes

# Test — should show CPU/memory per pod
kubectl top pods -A
```

---

## Part D: Custom Alerts (Optional)

---

## Cleanup

```bash
# Remove Prometheus stack
helm uninstall prometheus -n monitoring
kubectl delete namespace monitoring

# Remove Container Insights add-on
aws eks delete-addon --cluster-name eks-learning-cluster --addon-name amazon-cloudwatch-observability --region us-east-1

# Remove metrics server
kubectl delete -f https://github.com/kubernetes-sigs/metrics-server/releases/latest/download/components.yaml
```

---

## Key Takeaways

1. **Container Insights** = quickest AWS-native monitoring (add-on, no config)
2. **Prometheus** = industry standard for K8s metrics (pull-based, PromQL)
3. **Grafana** = visualization layer on top of Prometheus
4. **Metrics Server** = lightweight, required for `kubectl top` and HPA
5. **Fluent Bit** = log forwarding to CloudWatch/S3
6. **Port-forward** = access internal services without exposing them

---

## Exercises

| # | Task | Done |
|---|------|------|
| 1 | Enable Container Insights add-on | [ ] |
| 2 | View pod metrics in CloudWatch Console | [ ] |
| 3 | View pod logs in CloudWatch Logs | [ ] |
| 4 | Install Prometheus + Grafana via Helm | [ ] |
| 5 | Access Grafana dashboard (port-forward) | [ ] |
| 6 | Run a PromQL query in Prometheus UI | [ ] |
| 7 | Install Metrics Server | [ ] |
| 8 | Run `kubectl top nodes` and `kubectl top pods` | [ ] |
| 9 | Clean up | [ ] |

---

## Next: Module 8 — Scaling (HPA, VPA, Karpenter, Cluster Autoscaler)
