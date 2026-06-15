# Module 10 — Troubleshooting Guide

---

## Velero + IRSA on EKS

### Problem 1: BSL Unavailable — "no EC2 IMDS role found"

**Symptom:**
```
failed to refresh cached credentials, no EC2 IMDS role found, operation error ec2imds: GetMetadata, canceled, context deadline exceeded
```

**Root Cause:** Velero Helm chart creates a `velero-server` service account, but `eksctl create iamserviceaccount` created a separate `velero` SA with the IRSA annotation. The pod uses `velero-server` which has no IRSA.

**Fix:** Annotate the correct SA:
```bash
kubectl annotate sa velero-server -n velero \
  eks.amazonaws.com/role-arn=<ROLE_ARN> --overwrite
kubectl rollout restart deployment velero -n velero
```

---

### Problem 2: BSL Unavailable — "no EC2 IMDS role found" (even after annotation)

**Symptom:** Same error persists after annotating the correct SA.

**Root Cause:** The IAM role trust policy references `system:serviceaccount:velero:velero` but the pod uses `velero-server`.

**Fix:** Update the trust policy to match the actual SA name:
```bash
# Get current trust policy, change "velero" to "velero-server" in the sub condition:
# "system:serviceaccount:velero:velero" → "system:serviceaccount:velero:velero-server"

aws iam update-assume-role-policy \
  --role-name <ROLE_NAME> \
  --policy-document file://velero-trust-policy.json
```

---

### Problem 3: BSL Unavailable — still failing after trust policy fix

**Symptom:** Same IMDS error even though annotation + trust policy are correct.

**Root Cause:** Velero Helm chart sets `AWS_SHARED_CREDENTIALS_FILE=/credentials/cloud` which points to an empty/missing file. AWS SDK checks this **before** the IRSA web identity token and fails.

**Fix:** Upgrade Helm release with `credentials.useSecret=false`:
```bash
helm upgrade velero vmware-tanzu/velero \
  --namespace velero \
  --set credentials.useSecret=false \
  --set serviceAccount.server.name=velero-server \
  --set "serviceAccount.server.annotations.eks\.amazonaws\.com/role-arn=<ROLE_ARN>" \
  ... (rest of config)
```

**Key flag:** `credentials.useSecret=false` — stops the chart from mounting the empty credentials file.

---

### Problem 4: BSL Unavailable — "AssumeRoleWithWebIdentity... sts.us-east-1.amazonaws.com: i/o timeout"

**Symptom:**
```
operation error STS: AssumeRoleWithWebIdentity, Post "https://sts.us-east-1.amazonaws.com/": dial tcp 192.168.0.38:443: i/o timeout
```

**Root Cause:** Pod on private subnet cannot reach the STS endpoint. Either NAT Gateway route is missing or VPC endpoint isn't configured for the node subnets.

**Fix (VPC Endpoint approach):**

1. Check if STS VPC endpoint exists:
```bash
aws ec2 describe-vpc-endpoints --region us-east-1 \
  --filters "Name=vpc-id,Values=$VPC_ID" "Name=service-name,Values=com.amazonaws.us-east-1.sts" \
  --query "VpcEndpoints[].{Id:VpcEndpointId,State:State,SubnetIds:SubnetIds,Groups:Groups[].GroupId}"
```

2. Ensure endpoint is in the same AZ/subnet as your nodes (or add subnets):
```bash
aws ec2 modify-vpc-endpoint \
  --vpc-endpoint-id <VPCE_ID> \
  --add-subnet-ids <NODE_SUBNET> \
  --region us-east-1
```

3. **Fix Security Group** — the endpoint SG must allow inbound 443 from nodes:
```bash
NODE_SG=$(aws eks describe-cluster --name eks-learning-cluster --region us-east-1 \
  --query "cluster.resourcesVpcConfig.clusterSecurityGroupId" --output text)

aws ec2 authorize-security-group-ingress \
  --group-id <ENDPOINT_SG> \
  --protocol tcp \
  --port 443 \
  --source-group $NODE_SG \
  --region us-east-1
```

---

### Problem 5: "DuplicateSubnetsInSameZone" when adding subnets to endpoint

**Symptom:**
```
Found another VPC endpoint subnet in the availability zone of subnet-xxx
```

**Root Cause:** VPC endpoint already has a subnet in that AZ. Only one subnet per AZ is allowed per endpoint.

**Fix:** Only add subnets for AZs not already covered. Check existing subnets first and skip the overlapping one.

---

## Helm + Zsh

### Problem: "zsh: no matches found: configuration.backupStorageLocation[0].name=default"

**Root Cause:** Zsh interprets `[0]` as a glob pattern.

**Fix:** Wrap `--set` values in single quotes:
```bash
--set 'configuration.backupStorageLocation[0].name=default'
```

---

## eksctl

### Problem: "No cluster found for name: eks-learning-cluster"

**Root Cause:** eksctl defaults to a region that isn't where your cluster lives.

**Fix:** Always pass `--region`:
```bash
eksctl create iamserviceaccount \
  --cluster=eks-learning-cluster \
  --region=us-east-1 \
  ...
```

---

## General IRSA Debugging Checklist

| Step | Command | What to Check |
|------|---------|---------------|
| 1 | `kubectl get sa <name> -n <ns> -o yaml` | IRSA annotation present? |
| 2 | `kubectl get pod -o jsonpath='{.items[0].spec.serviceAccountName}'` | Pod using the annotated SA? |
| 3 | `kubectl get pod -o jsonpath='{.items[0].spec.volumes}'` | `aws-iam-token` volume projected? |
| 4 | `kubectl get pod -o jsonpath='{.items[0].spec.containers[0].env[*].name}'` | `AWS_ROLE_ARN` + `AWS_WEB_IDENTITY_TOKEN_FILE` set? |
| 5 | `aws iam get-role --role-name <role> --query "Role.AssumeRolePolicyDocument"` | Trust policy sub matches SA name? |
| 6 | Check for `AWS_SHARED_CREDENTIALS_FILE` env var | If set, may override IRSA — use `credentials.useSecret=false` |
| 7 | Pod logs — check if error is STS timeout vs auth failure | Timeout = networking. Auth = trust policy/SA mismatch |

---

## Kubecost Helm Install

### Problem 1: "clusterId is required. Please set .Values.global.clusterId"

**Fix:** Add `--set global.clusterId=<cluster-name>` to the Helm install.

---

### Problem 2: "Kubecost 2.9.x is only used for preparing agents to upgrade to 3.0"

**Symptom:**
```
Kubecost 2.9.x is only used for preparing agents to upgrade to 3.0.
In kubecost 2.9, cluster_id is set in two places.
```

**Root Cause:** The latest Kubecost chart (v2.9+) is a transitional release for v3 migration. It requires additional config that isn't needed for free-tier learning.

**Fix:** Pin to a stable older version:
```bash
helm install kubecost kubecost/cost-analyzer \
  --namespace kubecost --create-namespace \
  --set global.clusterId=eks-learning-cluster \
  --set finopsAgent.clusterId=eks-learning-cluster \
  --version 2.6.1
```

---

### Problem 3: Kubecost pods Pending — "unbound immediate PersistentVolumeClaims"

**Symptom:**
```
0/3 nodes are available: pod has unbound immediate PersistentVolumeClaims. not found
Failed to schedule pod, unbound pvc must define a storage class
```

**Root Cause:** Kubecost Helm chart creates PVCs without a StorageClass. EKS doesn't always have a default StorageClass set.

**Fix Option A (at install time — recommended):**
```bash
helm install kubecost kubecost/cost-analyzer \
  --namespace kubecost --create-namespace \
  --set global.clusterId=eks-learning-cluster \
  --set finopsAgent.clusterId=eks-learning-cluster \
  --set 'persistentVolume.storageClass=gp2' \
  --set 'prometheus.server.persistentVolume.storageClass=gp2' \
  --version 2.6.1
```

**Fix Option B (patch existing PVCs):**
```bash
kubectl patch pvc kubecost-cost-analyzer -n kubecost -p '{"spec":{"storageClassName":"gp2"}}'
kubectl patch pvc kubecost-prometheus-server -n kubecost -p '{"spec":{"storageClassName":"gp2"}}'
```
