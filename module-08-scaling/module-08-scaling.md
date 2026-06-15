# Module 8: Scaling


---

---

Make your cluster respond to demand automatically. Configure HPA to scale pods based on CPU/memory, VPA to right-size resource requests, and Cluster Autoscaler to add/remove nodes when capacity runs out.

## Concepts

| Component | What It Does |
|-----------|-------------|
| **HPA** | Horizontal Pod Autoscaler — scales pod replicas based on CPU/memory/custom metrics |
| **VPA** | Vertical Pod Autoscaler — adjusts pod resource requests/limits automatically |
| **Cluster Autoscaler** | Scales node group size based on pending pods |
| **Karpenter** | AWS-native node provisioner — faster, more flexible than Cluster Autoscaler |

---

## Part A: Horizontal Pod Autoscaler (HPA)

### 8.1 Verify Metrics Server is Running

```bash
kubectl get deployment metrics-server -n kube-system
kubectl top nodes
kubectl top pods -A
```

⚠️ HPA requires Metrics Server. If not installed, refer to **Module 7 (step 7.8–7.9)** for full installation and validation instructions:
```bash
kubectl apply -f https://github.com/kubernetes-sigs/metrics-server/releases/latest/download/components.yaml
```

### 8.2 Deploy a Test Application with Resource Requests

```bash
cat > hpa-test-app.yaml << 'EOF'
apiVersion: apps/v1
kind: Deployment
metadata:
  name: hpa-demo
  namespace: default
spec:
  replicas: 1
  selector:
    matchLabels:
      app: hpa-demo
  template:
    metadata:
      labels:
        app: hpa-demo
    spec:
      containers:
      - name: hpa-demo
        image: registry.k8s.io/hpa-example
        ports:
        - containerPort: 80
        resources:
          requests:
            cpu: 200m
          limits:
            cpu: 500m
---
apiVersion: v1
kind: Service
metadata:
  name: hpa-demo
  namespace: default
spec:
  selector:
    app: hpa-demo
  ports:
  - port: 80
    targetPort: 80
EOF

kubectl apply -f hpa-test-app.yaml
kubectl get pods -l app=hpa-demo
```

**File:** `module-08-scaling/hpa-test-app.yaml`

### 8.3 Create HPA

```bash
kubectl autoscale deployment hpa-demo \
  --cpu-percent=50 \
  --min=1 \
  --max=10

# Verify HPA
kubectl get hpa
```

Expected output:
```
NAME       REFERENCE             TARGETS   MINPODS   MAXPODS   REPLICAS
hpa-demo   Deployment/hpa-demo   0%/50%    1         10        1
```

### 8.4 Generate Load to Trigger Scale-Up

```bash
# In a separate terminal, run a load generator
kubectl run load-generator --image=busybox:1.28 --restart=Never -- /bin/sh -c "while sleep 0.01; do wget -q -O- http://hpa-demo; done"

# Watch HPA in action (wait 1-2 minutes)
kubectl get hpa --watch
```

Expected: Replicas increase from 1 → multiple as CPU exceeds 50%.

### 8.5 Stop Load and Watch Scale-Down

```bash
kubectl delete pod load-generator

# Watch scale-down (takes ~5 minutes by default)
kubectl get hpa --watch
kubectl get pods -l app=hpa-demo
```

### 8.6 HPA with YAML Definition

```bash
cat > hpa-manifest.yaml << 'EOF'
apiVersion: autoscaling/v2
kind: HorizontalPodAutoscaler
metadata:
  name: hpa-demo-v2
  namespace: default
spec:
  scaleTargetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: hpa-demo
  minReplicas: 2
  maxReplicas: 15
  metrics:
  - type: Resource
    resource:
      name: cpu
      target:
        type: Utilization
        averageUtilization: 60
  - type: Resource
    resource:
      name: memory
      target:
        type: Utilization
        averageUtilization: 70
  behavior:
    scaleDown:
      stabilizationWindowSeconds: 120
      policies:
      - type: Percent
        value: 50
        periodSeconds: 60
    scaleUp:
      stabilizationWindowSeconds: 0
      policies:
      - type: Percent
        value: 100
        periodSeconds: 15
EOF

kubectl apply -f hpa-manifest.yaml
kubectl get hpa hpa-demo-v2
```

**File:** `module-08-scaling/hpa-manifest.yaml`

**Key fields:**
- `behavior.scaleDown.stabilizationWindowSeconds` — cooldown before scaling down
- `behavior.scaleUp` — how aggressively to scale up

---

## Part B: Vertical Pod Autoscaler (VPA)

### 8.7 Install VPA

```bash
# Clone the VPA repo
git clone https://github.com/kubernetes/autoscaler.git
cd autoscaler/vertical-pod-autoscaler

# Install VPA components
./hack/vpa-up.sh

# Verify
kubectl get pods -n kube-system | grep vpa
```

Expected pods:
- `vpa-admission-controller`
- `vpa-recommender`
- `vpa-updater`

### 8.8 Deploy VPA for a Workload

```bash
cat > vpa-demo.yaml << 'EOF'
apiVersion: apps/v1
kind: Deployment
metadata:
  name: vpa-demo
  namespace: default
spec:
  replicas: 2
  selector:
    matchLabels:
      app: vpa-demo
  template:
    metadata:
      labels:
        app: vpa-demo
    spec:
      containers:
      - name: vpa-demo
        image: registry.k8s.io/hpa-example
        resources:
          requests:
            cpu: 100m
            memory: 50Mi
          limits:
            cpu: 500m
            memory: 500Mi
---
apiVersion: autoscaling.k8s.io/v1
kind: VerticalPodAutoscaler
metadata:
  name: vpa-demo
  namespace: default
spec:
  targetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: vpa-demo
  updatePolicy:
    updateMode: "Off"  # Start with "Off" to just get recommendations
EOF

kubectl apply -f vpa-demo.yaml
```

**File:** `module-08-scaling/vpa-demo.yaml`

### 8.9 Check VPA Recommendations

```bash
# Wait 2-3 minutes for recommendations to appear
kubectl get vpa vpa-demo -o yaml | grep -A 20 "recommendation"
```

**Update modes:**
| Mode | Behavior |
|------|----------|
| `Off` | Only provides recommendations, no changes |
| `Initial` | Sets resources only at pod creation |
| `Auto` | Evicts and recreates pods with new resource values |

⚠️ **VPA and HPA should NOT target the same metric (CPU/memory) on the same deployment.** Use VPA for right-sizing, HPA for scaling replicas.

---

## Part C: Cluster Autoscaler

### Troubleshooting: Set Environment Variables

If `$AWS_ACCOUNT_ID` is not set, IRSA commands will fail with policy ARN errors. Always run this at the start of your session:

```bash
export AWS_ACCOUNT_ID=$(aws sts get-caller-identity --query Account --output text)
echo $AWS_ACCOUNT_ID
```

### 8.10 Create IAM Policy for Cluster Autoscaler

```bash
cat > cluster-autoscaler-policy.json << 'EOF'
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Action": [
        "autoscaling:DescribeAutoScalingGroups",
        "autoscaling:DescribeAutoScalingInstances",
        "autoscaling:DescribeLaunchConfigurations",
        "autoscaling:DescribeScalingActivities",
        "autoscaling:DescribeTags",
        "autoscaling:SetDesiredCapacity",
        "autoscaling:TerminateInstanceInAutoScalingGroup",
        "ec2:DescribeLaunchTemplateVersions",
        "ec2:DescribeInstanceTypes",
        "ec2:DescribeImages",
        "ec2:GetInstanceTypesFromInstanceRequirements",
        "eks:DescribeNodegroup"
      ],
      "Resource": "*"
    }
  ]
}
EOF

aws iam create-policy \
  --policy-name EKSClusterAutoscalerPolicy \
  --policy-document file://cluster-autoscaler-policy.json \
  --region us-east-1
```

**File:** `module-08-scaling/cluster-autoscaler-policy.json`

### 8.11 Create IRSA for Cluster Autoscaler

```bash
eksctl create iamserviceaccount \
  --cluster=eks-learning-cluster \
  --namespace=kube-system \
  --name=cluster-autoscaler \
  --attach-policy-arn=arn:aws:iam::${AWS_ACCOUNT_ID}:policy/EKSClusterAutoscalerPolicy \
  --region us-east-1 \
  --approve --override-existing-serviceaccounts
```

### 8.12 Create Kubernetes RBAC for Cluster Autoscaler

⚠️ IRSA gives AWS permissions, but the pod also needs Kubernetes RBAC to access the K8s API (list nodes, pods, etc.)

```bash
cat > cluster-autoscaler-rbac.yaml << 'EOF'
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRole
metadata:
  name: cluster-autoscaler
rules:
- apiGroups: [""]
  resources: ["nodes", "pods", "replicationcontrollers", "services", "persistentvolumeclaims", "persistentvolumes", "namespaces"]
  verbs: ["get", "list", "watch"]
- apiGroups: ["apps"]
  resources: ["daemonsets", "replicasets", "statefulsets"]
  verbs: ["get", "list", "watch"]
- apiGroups: ["batch"]
  resources: ["jobs"]
  verbs: ["get", "list", "watch"]
- apiGroups: ["policy"]
  resources: ["poddisruptionbudgets"]
  verbs: ["get", "list", "watch"]
- apiGroups: ["storage.k8s.io"]
  resources: ["storageclasses", "csinodes", "csidrivers", "csistoragecapacities"]
  verbs: ["get", "list", "watch"]
- apiGroups: ["coordination.k8s.io"]
  resources: ["leases"]
  verbs: ["get", "list", "watch", "create", "update"]
- apiGroups: [""]
  resources: ["events"]
  verbs: ["create", "patch"]
- apiGroups: [""]
  resources: ["configmaps"]
  verbs: ["create", "delete", "get", "update"]
---
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRoleBinding
metadata:
  name: cluster-autoscaler
roleRef:
  apiGroup: rbac.authorization.k8s.io
  kind: ClusterRole
  name: cluster-autoscaler
subjects:
- kind: ServiceAccount
  name: cluster-autoscaler
  namespace: kube-system
EOF

kubectl apply -f cluster-autoscaler-rbac.yaml
```

**File:** `module-08-scaling/cluster-autoscaler-rbac.yaml`

### 8.13 Deploy Cluster Autoscaler

```bash
cat > cluster-autoscaler.yaml << 'EOF'
apiVersion: apps/v1
kind: Deployment
metadata:
  name: cluster-autoscaler
  namespace: kube-system
  labels:
    app: cluster-autoscaler
spec:
  replicas: 1
  selector:
    matchLabels:
      app: cluster-autoscaler
  template:
    metadata:
      labels:
        app: cluster-autoscaler
    spec:
      serviceAccountName: cluster-autoscaler
      containers:
      - name: cluster-autoscaler
        image: registry.k8s.io/autoscaling/cluster-autoscaler:v1.29.0
        command:
        - ./cluster-autoscaler
        - --v=4
        - --stderrthreshold=info
        - --cloud-provider=aws
        - --skip-nodes-with-local-storage=false
        - --expander=least-waste
        - --node-group-auto-discovery=asg:tag=k8s.io/cluster-autoscaler/enabled,k8s.io/cluster-autoscaler/eks-learning-cluster
        - --balance-similar-node-groups
        - --skip-nodes-with-system-pods=false
        resources:
          requests:
            cpu: 100m
            memory: 300Mi
          limits:
            cpu: 100m
            memory: 600Mi
EOF

kubectl apply -f cluster-autoscaler.yaml
kubectl get pods -n kube-system -l app=cluster-autoscaler
```

**File:** `module-08-scaling/cluster-autoscaler.yaml`

### 8.14 Test Cluster Autoscaler — Scale Up Nodes

```bash
# Deploy many pods that exceed current node capacity
cat > inflate.yaml << 'EOF'
apiVersion: apps/v1
kind: Deployment
metadata:
  name: inflate
  namespace: default
spec:
  replicas: 20
  selector:
    matchLabels:
      app: inflate
  template:
    metadata:
      labels:
        app: inflate
    spec:
      containers:
      - name: inflate
        image: public.ecr.aws/eks-distro/kubernetes/pause:3.7
        resources:
          requests:
            cpu: 250m
            memory: 256Mi
EOF

kubectl apply -f inflate.yaml

# Watch pods — some will be Pending (not enough nodes)
kubectl get pods -l app=inflate --watch

# Check Cluster Autoscaler logs
kubectl logs -l app=cluster-autoscaler -n kube-system --tail=20
```

**File:** `module-08-scaling/inflate.yaml`

### 8.15 Verify Node Scale-Up

```bash
# Watch nodes increase
kubectl get nodes --watch

# Check ASG in AWS Console or CLI
aws autoscaling describe-auto-scaling-groups \
  --query "AutoScalingGroups[?contains(Tags[?Key=='eks:cluster-name'].Value, 'eks-learning-cluster')].{Name:AutoScalingGroupName,Desired:DesiredCapacity,Min:MinSize,Max:MaxSize}" \
  --output table \
  --region us-east-1
```

### 8.16 Test Scale-Down

```bash
kubectl delete deployment inflate

# Wait 10+ minutes — Cluster Autoscaler removes underutilized nodes
kubectl get nodes --watch
```

---

## Part D: Karpenter (Modern Alternative) — ⚠️ THEORY ONLY

### 8.17 Karpenter vs Cluster Autoscaler

| Feature | Cluster Autoscaler | Karpenter |
|---------|-------------------|-----------|
| Speed | 2-5 min to add node | 30-60 sec |
| Flexibility | Tied to ASG/node groups | Provisions any instance type |
| Right-sizing | No | Yes — picks optimal instance |
| Consolidation | No | Yes — bin-packs and removes waste |
| AWS-native | No (K8s SIG project) | Yes (AWS open-source) |

### 8.18 Karpenter Prerequisites

⚠️ **Karpenter replaces Cluster Autoscaler — do NOT run both.**

```bash
# Remove Cluster Autoscaler first
kubectl delete deployment cluster-autoscaler -n kube-system
```

**Step 1: Set environment variables**

```bash
export AWS_ACCOUNT_ID=$(aws sts get-caller-identity --query Account --output text)
export AWS_REGION="us-east-1"
export CLUSTER_NAME="eks-learning-cluster"
export KARPENTER_NAMESPACE="kube-system"
export KARPENTER_VERSION="1.1.0"

echo "Account: $AWS_ACCOUNT_ID | Region: $AWS_REGION | Cluster: $CLUSTER_NAME"
```

**Step 2: Create KarpenterNode IAM Role**

```bash
cat > karpenter-node-trust-policy.json << EOF
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Principal": {
        "Service": "ec2.amazonaws.com"
      },
      "Action": "sts:AssumeRole"
    }
  ]
}
EOF

aws iam create-role \
  --role-name "KarpenterNodeRole-${CLUSTER_NAME}" \
  --assume-role-policy-document file://karpenter-node-trust-policy.json

# Attach required policies for nodes
aws iam attach-role-policy --role-name "KarpenterNodeRole-${CLUSTER_NAME}" \
  --policy-arn arn:aws:iam::aws:policy/AmazonEKSWorkerNodePolicy
aws iam attach-role-policy --role-name "KarpenterNodeRole-${CLUSTER_NAME}" \
  --policy-arn arn:aws:iam::aws:policy/AmazonEKS_CNI_Policy
aws iam attach-role-policy --role-name "KarpenterNodeRole-${CLUSTER_NAME}" \
  --policy-arn arn:aws:iam::aws:policy/AmazonEC2ContainerRegistryReadOnly
aws iam attach-role-policy --role-name "KarpenterNodeRole-${CLUSTER_NAME}" \
  --policy-arn arn:aws:iam::aws:policy/AmazonSSMManagedInstanceCore
```

**Step 3: Create access entry for Karpenter nodes**

```bash
aws eks create-access-entry \
  --cluster-name ${CLUSTER_NAME} \
  --principal-arn arn:aws:iam::${AWS_ACCOUNT_ID}:role/KarpenterNodeRole-${CLUSTER_NAME} \
  --type EC2_LINUX \
  --region ${AWS_REGION}
```

**Step 4: Create Karpenter Controller IAM Policy**

```bash
cat > karpenter-controller-policy.json << 'EOF'
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Sid": "Karpenter",
      "Effect": "Allow",
      "Action": [
        "ec2:CreateFleet",
        "ec2:CreateLaunchTemplate",
        "ec2:CreateTags",
        "ec2:DeleteLaunchTemplate",
        "ec2:DescribeAvailabilityZones",
        "ec2:DescribeImages",
        "ec2:DescribeInstances",
        "ec2:DescribeInstanceTypeOfferings",
        "ec2:DescribeInstanceTypes",
        "ec2:DescribeLaunchTemplates",
        "ec2:DescribeSecurityGroups",
        "ec2:DescribeSpotPriceHistory",
        "ec2:DescribeSubnets",
        "ec2:RunInstances",
        "ec2:TerminateInstances",
        "iam:AddRoleToInstanceProfile",
        "iam:CreateInstanceProfile",
        "iam:DeleteInstanceProfile",
        "iam:GetInstanceProfile",
        "iam:PassRole",
        "iam:RemoveRoleFromInstanceProfile",
        "iam:TagInstanceProfile",
        "pricing:GetProducts",
        "ssm:GetParameter",
        "eks:DescribeCluster"
      ],
      "Resource": "*"
    },
    {
      "Sid": "KarpenterSQS",
      "Effect": "Allow",
      "Action": [
        "sqs:DeleteMessage",
        "sqs:GetQueueAttributes",
        "sqs:GetQueueUrl",
        "sqs:ReceiveMessage"
      ],
      "Resource": "*"
    }
  ]
}
EOF

aws iam create-policy \
  --policy-name "KarpenterControllerPolicy-${CLUSTER_NAME}" \
  --policy-document file://karpenter-controller-policy.json
```

**Step 5: Create IRSA for Karpenter Controller**

```bash
eksctl create iamserviceaccount \
  --cluster=${CLUSTER_NAME} \
  --namespace=${KARPENTER_NAMESPACE} \
  --name=karpenter \
  --attach-policy-arn=arn:aws:iam::${AWS_ACCOUNT_ID}:policy/KarpenterControllerPolicy-${CLUSTER_NAME} \
  --region ${AWS_REGION} \
  --approve --override-existing-serviceaccounts
```

**Step 6: Create SQS Queue for Spot Interruption Handling**

```bash
aws sqs create-queue \
  --queue-name ${CLUSTER_NAME} \
  --region ${AWS_REGION}

# Create EventBridge rules to forward interruption events to SQS
aws events put-rule \
  --name "KarpenterInterruptionRule-${CLUSTER_NAME}" \
  --event-pattern '{"source":["aws.ec2"],"detail-type":["EC2 Spot Instance Interruption Warning","EC2 Instance Rebalance Recommendation","EC2 Instance State-change Notification"]}' \
  --region ${AWS_REGION}

# Get SQS ARN
SQS_ARN=$(aws sqs get-queue-attributes --queue-url $(aws sqs get-queue-url --queue-name ${CLUSTER_NAME} --region ${AWS_REGION} --query QueueUrl --output text) --attribute-names QueueArn --query Attributes.QueueArn --output text --region ${AWS_REGION})

aws events put-targets \
  --rule "KarpenterInterruptionRule-${CLUSTER_NAME}" \
  --targets "Id=KarpenterTarget,Arn=${SQS_ARN}" \
  --region ${AWS_REGION}
```

**Step 7: Tag subnets and security groups for Karpenter discovery**

```bash
# Tag private subnets (Karpenter launches nodes here)
aws ec2 create-tags --resources ${PRIVATE_SUBNET_1} ${PRIVATE_SUBNET_2} \
  --tags Key=karpenter.sh/discovery,Value=${CLUSTER_NAME} \
  --region ${AWS_REGION}

# Tag the cluster security group
CLUSTER_SG=$(aws eks describe-cluster --name ${CLUSTER_NAME} --region ${AWS_REGION} --query "cluster.resourcesVpcConfig.clusterSecurityGroupId" --output text)
aws ec2 create-tags --resources ${CLUSTER_SG} \
  --tags Key=karpenter.sh/discovery,Value=${CLUSTER_NAME} \
  --region ${AWS_REGION}
```

### 8.19 Install Karpenter

**Troubleshooting: CRD Conflict with `eks-auto-mode`**

If Helm install fails with `conflicts with "eks-auto-mode"` on CRDs, delete leftover CRDs first:

```bash
kubectl delete crd nodeclaims.karpenter.sh nodepools.karpenter.sh ec2nodeclasses.karpenter.k8s.aws 2>/dev/null

# If delete hangs due to finalizers:
kubectl patch crd nodeclaims.karpenter.sh -p '{"metadata":{"finalizers":[]}}' --type=merge
kubectl patch crd nodepools.karpenter.sh -p '{"metadata":{"finalizers":[]}}' --type=merge
kubectl patch crd ec2nodeclasses.karpenter.k8s.aws -p '{"metadata":{"finalizers":[]}}' --type=merge
```

**Install Karpenter via Helm:**

```bash
helm upgrade --install karpenter oci://public.ecr.aws/karpenter/karpenter \
  --version "${KARPENTER_VERSION}" \
  --namespace "${KARPENTER_NAMESPACE}" \
  --set "settings.clusterName=${CLUSTER_NAME}" \
  --set "settings.interruptionQueue=${CLUSTER_NAME}" \
  --set controller.env[0].name=AWS_REGION \
  --set controller.env[0].value=${AWS_REGION} \
  --set controller.resources.requests.cpu=1 \
  --set controller.resources.requests.memory=1Gi \
  --set controller.resources.limits.cpu=1 \
  --set controller.resources.limits.memory=1Gi \
  --wait
```

⚠️ **Troubleshooting: IMDS/Region error** — If pods crash with `ec2imds: GetRegion, context deadline exceeded`, ensure `controller.env[0]` sets `AWS_REGION` explicitly as shown above.

Verify:
```bash
kubectl get pods -n kube-system -l app.kubernetes.io/name=karpenter
```

### 8.20 Create NodePool + EC2NodeClass

```bash
cat > karpenter-nodepool.yaml << 'EOF'
apiVersion: karpenter.sh/v1
kind: NodePool
metadata:
  name: default
spec:
  template:
    spec:
      requirements:
      - key: kubernetes.io/arch
        operator: In
        values: ["amd64"]
      - key: karpenter.sh/capacity-type
        operator: In
        values: ["on-demand", "spot"]
      - key: karpenter.k8s.aws/instance-category
        operator: In
        values: ["c", "m", "r", "t"]
      - key: karpenter.k8s.aws/instance-size
        operator: In
        values: ["medium", "large", "xlarge"]
      nodeClassRef:
        group: karpenter.k8s.aws
        kind: EC2NodeClass
        name: default
  limits:
    cpu: "100"
    memory: 200Gi
  disruption:
    consolidationPolicy: WhenEmptyOrUnderutilized
    consolidateAfter: 1m
---
apiVersion: karpenter.k8s.aws/v1
kind: EC2NodeClass
metadata:
  name: default
spec:
  amiSelectorTerms:
  - alias: al2023@latest
  role: "KarpenterNodeRole-${CLUSTER_NAME}"
  subnetSelectorTerms:
  - tags:
      karpenter.sh/discovery: "${CLUSTER_NAME}"
  securityGroupSelectorTerms:
  - tags:
      karpenter.sh/discovery: "${CLUSTER_NAME}"
EOF

kubectl apply -f karpenter-nodepool.yaml
```

### 8.21 Test Karpenter Scaling

```bash
# Same inflate test
kubectl apply -f inflate.yaml

# Watch — Karpenter provisions nodes much faster
kubectl get nodes --watch
kubectl logs -l app.kubernetes.io/name=karpenter -n kube-system --tail=20

# Cleanup
kubectl delete deployment inflate
# Karpenter consolidates and removes empty nodes automatically
```

---

## Cleanup

```bash
# Remove HPA test
kubectl delete -f hpa-test-app.yaml
kubectl delete hpa hpa-demo
kubectl delete -f hpa-manifest.yaml 2>/dev/null

# Remove VPA
kubectl delete -f vpa-demo.yaml
cd autoscaler/vertical-pod-autoscaler && ./hack/vpa-down.sh && cd -

# Remove Cluster Autoscaler (if still running)
kubectl delete -f cluster-autoscaler.yaml 2>/dev/null
aws iam delete-policy --policy-arn arn:aws:iam::${AWS_ACCOUNT_ID}:policy/EKSClusterAutoscalerPolicy

# Remove Karpenter (if installed)
helm uninstall karpenter -n kube-system
kubectl delete -f karpenter-nodepool.yaml 2>/dev/null

# Remove inflate
kubectl delete deployment inflate 2>/dev/null
```

---

## Key Takeaways

1. **HPA** = scale pods horizontally based on metrics (CPU, memory, custom)
2. **VPA** = right-size pod resource requests (don't combine with HPA on same metric)
3. **Cluster Autoscaler** = adds/removes nodes via ASG (slower, proven)
4. **Karpenter** = modern, fast, right-sizes nodes, consolidates waste (recommended for new clusters)
5. **Metrics Server** is required for HPA and `kubectl top`
6. **Scale-up is fast, scale-down is intentionally slow** (stabilization windows prevent flapping)

---

## Exercises

| # | Task | Done |
|---|------|------|
| 1 | Deploy test app with resource requests | [ ] |
| 2 | Create HPA and verify with `kubectl get hpa` | [ ] |
| 3 | Generate load and watch pods scale up | [ ] |
| 4 | Stop load and watch scale-down | [ ] |
| 5 | Create HPA v2 manifest with behavior policies | [ ] |
| 6 | Install VPA and get recommendations | [ ] |
| 7 | Deploy Cluster Autoscaler with IRSA | [ ] |
| 8 | Inflate pods to trigger node scale-up | [ ] |
| 9 | (Optional) Karpenter — theory only, review concepts | [ ] |
| 10 | Clean up all resources | [ ] |

---

## Next: Module 9 — CI/CD with EKS (CodePipeline, ArgoCD, Flux)
