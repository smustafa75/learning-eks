# Module 4: Networking


---

Expose your applications to the outside world. Set up the AWS Load Balancer Controller, create ALB Ingress for HTTP traffic, and NLB Services for TCP — the networking layer that connects users to your pods.

## Concepts

| Component | What It Does |
|-----------|-------------|
| **VPC CNI** | Gives each pod a real VPC IP address |
| **kube-proxy** | Routes service traffic (ClusterIP/NodePort) |
| **CoreDNS** | DNS resolution inside the cluster |
| **AWS Load Balancer Controller** | Creates ALB/NLB for Ingress & LoadBalancer services |
| **Ingress** | HTTP/HTTPS routing rules → routes traffic to services |

---

## 4.1 Understanding VPC CNI

Every pod gets a **real VPC IP** from your subnet CIDR. This is unique to AWS EKS.

```bash
# See pod IPs — they're from your VPC subnet range (192.168.x.x)
kubectl get pods -o wide

# Check max pods per node (reflects IP capacity from VPC CNI)
kubectl get nodes -o custom-columns=NAME:.metadata.name,PODS:.status.capacity.pods
```

**Why this matters:**
- Pods can communicate directly with other AWS services (RDS, ElastiCache) without NAT
- Security groups can be applied at pod level
- No overlay network overhead = better performance

---

## 4.2 Install AWS Load Balancer Controller

This controller watches for Ingress/Service resources and creates ALBs/NLBs automatically.

### Pre-requisites

**A. Create IAM OIDC Provider (one-time per cluster):**
```bash
eksctl utils associate-iam-oidc-provider --cluster eks-learning-cluster --region us-east-1 --approve
```

**B. Create IAM Policy for the controller:**
```bash
# Download the LATEST policy (always use 'main' branch, not versioned tags)
curl -o iam-policy.json https://raw.githubusercontent.com/kubernetes-sigs/aws-load-balancer-controller/main/docs/install/iam_policy.json

# Create the policy
aws iam create-policy \
  --policy-name AWSLoadBalancerControllerIAMPolicy \
  --policy-document file://iam-policy.json \
  --region us-east-1
```

⚠️ **If the downloaded policy is incomplete** (causes AccessDenied errors), attach these managed policies to the IRSA role as fallback:
- `ElasticLoadBalancingFullAccess`
- `AmazonEC2FullAccess`

**C. Create Service Account with IAM role (IRSA):**
```bash
eksctl create iamserviceaccount \
  --cluster=eks-learning-cluster \
  --namespace=kube-system \
  --name=aws-load-balancer-controller \
  --attach-policy-arn=arn:aws:iam::$AWS_ACCOUNT_ID:policy/AWSLoadBalancerControllerIAMPolicy \
  --override-existing-serviceaccounts \
  --region us-east-1 \
  --approve
```

### Install via Helm

```bash
# Add Helm repo
helm repo add eks https://aws.github.io/eks-charts
helm repo update

# Install the controller
helm install aws-load-balancer-controller eks/aws-load-balancer-controller \
  -n kube-system \
  --set clusterName=eks-learning-cluster \
  --set serviceAccount.create=false \
  --set serviceAccount.name=aws-load-balancer-controller \
  --set region=us-east-1 \
  --set vpcId=$VPC_ID
```

> ⚠️ Set `VPC_ID` first: `export VPC_ID=$(aws eks describe-cluster --name eks-learning-cluster --region us-east-1 --query "cluster.resourcesVpcConfig.vpcId" --output text)`

### Validate
```bash
# Check controller is running
kubectl get deployment -n kube-system aws-load-balancer-controller

# Check logs
kubectl logs -n kube-system deployment/aws-load-balancer-controller
```

---

## 4.3 Ingress — Expose App via ALB

### Tag Your Subnets (Required for ALB auto-discovery)

**Public subnets** (for internet-facing ALB):
```bash
# Set your subnet IDs (from VPC console or: aws ec2 describe-subnets --filters "Name=vpc-id,Values=$VPC_ID" --query "Subnets[].{Id:SubnetId,Public:MapPublicIpOnLaunch,AZ:AvailabilityZone}")
export PUBLIC_SUBNET_1="<your-public-subnet-id>"
export PRIVATE_SUBNET_1="<your-private-subnet-1>"
export PRIVATE_SUBNET_2="<your-private-subnet-2>"

aws ec2 create-tags --resources $PUBLIC_SUBNET_1 \
  --tags Key=kubernetes.io/role/elb,Value=1 --region us-east-1
```

**Private subnets** (for internal ALB):
```bash
aws ec2 create-tags --resources $PRIVATE_SUBNET_1 $PRIVATE_SUBNET_2 \
  --tags Key=kubernetes.io/role/internal-elb,Value=1 --region us-east-1
```

### Deploy a Sample App + Ingress

```bash
cat > nginx-ingress.yaml << 'EOF'
apiVersion: apps/v1
kind: Deployment
metadata:
  name: web-app
  labels:
    app: web-app
spec:
  replicas: 2
  selector:
    matchLabels:
      app: web-app
  template:
    metadata:
      labels:
        app: web-app
    spec:
      containers:
      - name: nginx
        image: nginx:1.27
        ports:
        - containerPort: 80
---
apiVersion: v1
kind: Service
metadata:
  name: web-app-svc
spec:
  selector:
    app: web-app
  ports:
  - port: 80
    targetPort: 80
  type: ClusterIP
---
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: web-app-ingress
  annotations:
    alb.ingress.kubernetes.io/scheme: internet-facing
    alb.ingress.kubernetes.io/target-type: ip
    alb.ingress.kubernetes.io/subnets: <YOUR_PUBLIC_SUBNET_1>,<YOUR_PUBLIC_SUBNET_2>
spec:
  ingressClassName: alb
  rules:
  - http:
      paths:
      - path: /
        pathType: Prefix
        backend:
          service:
            name: web-app-svc
            port:
              number: 80
EOF

kubectl apply -f nginx-ingress.yaml
```

**File:** `module-04-networking/nginx-ingress.yaml`

### Validate
```bash
# Check ingress — wait for ADDRESS to appear
kubectl get ingress web-app-ingress -w

# Once ADDRESS shows (ALB DNS name), test it:
curl http://<ALB-DNS-NAME>
```

⚠️ ALB creation takes 2-3 minutes. The ADDRESS field will show the ALB DNS name.

⚠️ **Important notes:**
- ALB requires **2+ subnets in different AZs** — single subnet will fail
- Subnet IDs must be **comma-separated, no spaces**: `subnet-aaa,subnet-bbb`
- After IAM policy changes, restart controller: `kubectl rollout restart deployment/aws-load-balancer-controller -n kube-system`
- ALB DNS may take 1-2 min to propagate — test with `curl` first

---

## 4.4 Internal Ingress (Private ALB)

For apps that should only be accessible within VPC:

```yaml
annotations:
  alb.ingress.kubernetes.io/scheme: internal          # ← change to internal
  alb.ingress.kubernetes.io/target-type: ip
  alb.ingress.kubernetes.io/subnets: <YOUR_PRIVATE_SUBNET_1>,<YOUR_PRIVATE_SUBNET_2>
```

---

## 4.5 NLB via Service (Layer 4)

For TCP/UDP traffic (non-HTTP):

```bash
cat > nlb-service.yaml << 'EOF'
apiVersion: v1
kind: Service
metadata:
  name: web-app-nlb
  annotations:
    service.beta.kubernetes.io/aws-load-balancer-type: external
    service.beta.kubernetes.io/aws-load-balancer-nlb-target-type: ip
    service.beta.kubernetes.io/aws-load-balancer-scheme: internet-facing
spec:
  selector:
    app: web-app
  ports:
  - port: 80
    targetPort: 80
  type: LoadBalancer
EOF

kubectl apply -f nlb-service.yaml
kubectl get svc web-app-nlb -w
```

---

## 4.6 Cleanup

```bash
kubectl delete -f nginx-ingress.yaml
kubectl delete -f nlb-service.yaml
# ALB/NLB will be automatically deleted
```

---

## Key Takeaways

1. **VPC CNI** = pods get real VPC IPs (no overlay)
2. **AWS LB Controller** = required for ALB/NLB integration
3. **Ingress** = Layer 7 (HTTP) routing via ALB
4. **Service type LoadBalancer** = Layer 4 (TCP) via NLB
5. **Subnet tags** = how the controller finds subnets for ALBs
6. **IRSA** = secure way to give pods AWS permissions (no node-level access)

---

## Lessons Learned (From Troubleshooting)

| Issue | Cause | Fix |
|-------|-------|-----|
| `subnets count less than minimal required count: 1 < 2` | ALB needs 2+ subnets in different AZs | Create 2nd public subnet in different AZ |
| Subnet IDs treated as one string | Space-separated instead of comma-separated | Use commas: `subnet-aaa,subnet-bbb` |
| `AccessDenied: ec2:DescribeSecurityGroups` | IAM policy from GitHub was outdated/incomplete | Attach `ElasticLoadBalancingFullAccess` + `AmazonEC2FullAccess` to IRSA role |
| `DescribeListenerAttributes` denied | Newer ELB API action not in downloaded policy | Re-download latest policy from `main` branch, not versioned tag |
| ALB DNS not resolving immediately | DNS propagation delay | Wait 1-2 minutes, test with curl first |

---

## Troubleshooting Commands

```bash
# Check controller is running
kubectl get deployment -n kube-system aws-load-balancer-controller

# Check controller logs (first place to look)
kubectl logs -n kube-system deployment/aws-load-balancer-controller --tail=10

# Restart controller (after IAM policy changes)
kubectl rollout restart deployment/aws-load-balancer-controller -n kube-system

# Check ingress status
kubectl get ingress

# Check ALB in AWS
aws elbv2 describe-load-balancers --region us-east-1 --query "LoadBalancers[].{Name:LoadBalancerName,DNS:DNSName,State:State.Code}"
```

---

## Exercises

| # | Task | Done |
|---|------|------|
| 1 | Check pod IPs match VPC CIDR | [ ] |
| 2 | Check max pods per node | [ ] |
| 3 | Associate OIDC provider | [ ] |
| 4 | Create IAM policy for LB controller | [ ] |
| 5 | Create IRSA service account | [ ] |
| 6 | Install Helm | [ ] |
| 7 | Install LB Controller via Helm | [ ] |
| 8 | Verify controller running | [ ] |
| 9 | Tag public subnets | [ ] |
| 10 | Tag private subnets | [ ] |
| 11 | Deploy app + ingress (nginx-ingress.yaml) | [ ] |
| 12 | Wait for ALB DNS | [ ] |
| 13 | Test ALB with curl | [ ] |
| 14 | Clean up | [ ] |

---

## Next: Module 5 — Storage (EBS CSI, EFS CSI, PV/PVC)
