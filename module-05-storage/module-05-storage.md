# Module 5: Storage (EBS CSI, EFS CSI, PV/PVC)


---

Give your applications persistent storage that survives pod restarts. Set up EBS (block storage) and EFS (shared filesystem) using CSI drivers and IRSA — the foundation for stateful workloads like databases.

## Concepts

| Component | What It Does |
|-----------|-------------|
| **PersistentVolume (PV)** | A piece of storage provisioned in the cluster |
| **PersistentVolumeClaim (PVC)** | A request for storage by a pod |
| **StorageClass** | Defines how storage is dynamically provisioned |
| **EBS CSI Driver** | Provisions EBS volumes as PVs (block storage, single-AZ, single-pod) |
| **EFS CSI Driver** | Provisions EFS file systems as PVs (shared storage, multi-AZ, multi-pod) |

### EBS vs EFS — When to Use

| Feature | EBS | EFS |
|---------|-----|-----|
| Access mode | ReadWriteOnce (single pod) | ReadWriteMany (multiple pods) |
| AZ scope | Single AZ | Multi-AZ |
| Use case | Databases, single-instance apps | Shared config, CMS, ML data |
| Performance | High IOPS, low latency | Scalable throughput |
| Cost | Lower (per GB) | Higher (per GB) |

---

## Part A: EBS CSI Driver

### 5.1 Create IAM Role for EBS CSI Driver

The OIDC provider should already be associated from Module 4. If not:
```bash
eksctl utils associate-iam-oidc-provider --cluster eks-learning-cluster --region us-east-1 --approve
```

Create the IRSA role with AWS managed policy:
```bash
eksctl create iamserviceaccount \
  --name ebs-csi-controller-sa \
  --namespace kube-system \
  --cluster eks-learning-cluster \
  --role-name AmazonEKS_EBS_CSI_DriverRole \
  --role-only \
  --attach-policy-arn arn:aws:iam::aws:policy/service-role/AmazonEBSCSIDriverPolicy \
  --region us-east-1 \
  --approve
```

> ℹ️ AWS also offers `AmazonEBSCSIDriverPolicyV2` (ARN: `arn:aws:iam::aws:policy/AmazonEBSCSIDriverPolicyV2`) with scoped-down permissions. Either works; the `service-role/` version is what eksctl uses by default.

**Validate:**
```bash
aws iam get-role --role-name AmazonEKS_EBS_CSI_DriverRole --query "Role.Arn" --output text
```

---

### 5.2 Install EBS CSI Driver (EKS Add-on)

```bash
aws eks create-addon \
  --cluster-name eks-learning-cluster \
  --addon-name aws-ebs-csi-driver \
  --service-account-role-arn arn:aws:iam::$AWS_ACCOUNT_ID:role/AmazonEKS_EBS_CSI_DriverRole \
  --region us-east-1
```

**Validate:**
```bash
# Check add-on status (wait for ACTIVE)
aws eks describe-addon --cluster-name eks-learning-cluster \
  --addon-name aws-ebs-csi-driver --region us-east-1 \
  --query "addon.{Status:status,Version:addonVersion}"

# Check pods running
kubectl get pods -n kube-system -l app.kubernetes.io/name=aws-ebs-csi-driver
```

---

### 5.3 Create a StorageClass

```bash
cat > ebs-storageclass.yaml << 'EOF'
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
  name: ebs-gp3
provisioner: ebs.csi.aws.com
volumeBindingMode: WaitForFirstConsumer
parameters:
  type: gp3
  fsType: ext4
reclaimPolicy: Delete
allowVolumeExpansion: true
EOF

kubectl apply -f ebs-storageclass.yaml
```

**File:** `module-05-storage/ebs-storageclass.yaml`

**Key fields:**
- `WaitForFirstConsumer` — volume created in same AZ as the pod (avoids AZ mismatch)
- `gp3` — latest generation, better price/performance than gp2
- `allowVolumeExpansion` — can resize PVC later without recreating

**Validate:**
```bash
kubectl get storageclass
```

---

### 5.4 Create PVC + Pod Using EBS

```bash
cat > ebs-test-app.yaml << 'EOF'
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: ebs-claim
spec:
  accessModes:
    - ReadWriteOnce
  storageClassName: ebs-gp3
  resources:
    requests:
      storage: 5Gi
---
apiVersion: v1
kind: Pod
metadata:
  name: ebs-test-pod
spec:
  containers:
  - name: app
    image: nginx:1.27
    volumeMounts:
    - mountPath: /data
      name: ebs-volume
    command: ["/bin/sh", "-c"]
    args:
    - |
      echo "Hello from EBS volume! Written at $(date)" > /data/test.txt
      nginx -g 'daemon off;'
  volumes:
  - name: ebs-volume
    persistentVolumeClaim:
      claimName: ebs-claim
EOF

kubectl apply -f ebs-test-app.yaml
```

**File:** `module-05-storage/ebs-test-app.yaml`

**Validate:**
```bash
# Check PVC is Bound
kubectl get pvc ebs-claim

# Check pod is Running
kubectl get pod ebs-test-pod

# Verify data was written
kubectl exec ebs-test-pod -- cat /data/test.txt

# See the actual EBS volume created
kubectl get pv
```

---

### 5.5 Test Data Persistence

```bash
# Write more data
kubectl exec ebs-test-pod -- sh -c 'echo "Persistent data survives pod restart" >> /data/test.txt'

# Delete the pod (PVC and PV remain)
kubectl delete pod ebs-test-pod

# Recreate the pod (same PVC)
cat > ebs-test-pod-recreate.yaml << 'EOF'
apiVersion: v1
kind: Pod
metadata:
  name: ebs-test-pod
spec:
  containers:
  - name: app
    image: nginx:1.27
    volumeMounts:
    - mountPath: /data
      name: ebs-volume
    command: ["/bin/sh", "-c"]
    args: ["cat /data/test.txt && nginx -g 'daemon off;'"]
  volumes:
  - name: ebs-volume
    persistentVolumeClaim:
      claimName: ebs-claim
EOF

kubectl apply -f ebs-test-pod-recreate.yaml

# Verify data survived
kubectl exec ebs-test-pod -- cat /data/test.txt
```

---

### 5.6 Cleanup EBS Resources

```bash
kubectl delete pod ebs-test-pod
kubectl delete pvc ebs-claim
# PV and EBS volume auto-deleted (reclaimPolicy: Delete)
kubectl get pv  # should be empty
```

---

## Part B: EFS CSI Driver

### 5.7 Create EFS File System

```bash
# Get VPC ID
VPC_ID=$(aws eks describe-cluster --name eks-learning-cluster --region us-east-1 \
  --query "cluster.resourcesVpcConfig.vpcId" --output text)
echo "VPC: $VPC_ID"

# Get VPC CIDR for security group
VPC_CIDR=$(aws ec2 describe-vpcs --vpc-ids $VPC_ID --region us-east-1 \
  --query "Vpcs[0].CidrBlock" --output text)
echo "CIDR: $VPC_CIDR"

# Create security group for EFS
SG_ID=$(aws ec2 create-security-group \
  --group-name efs-eks-sg \
  --description "EFS access from EKS" \
  --vpc-id $VPC_ID \
  --region us-east-1 \
  --query "GroupId" --output text)
echo "SG: $SG_ID"

# Allow NFS (port 2049) from VPC CIDR
aws ec2 authorize-security-group-ingress \
  --group-id $SG_ID \
  --protocol tcp \
  --port 2049 \
  --cidr $VPC_CIDR \
  --region us-east-1

# ⭐ VALIDATE: Confirm port 2049 is open (must show the rule)
aws ec2 describe-security-groups --group-ids $SG_ID --region us-east-1 \
  --query "SecurityGroups[].IpPermissions[].{Port:FromPort,CIDR:IpRanges[0].CidrIp}"
# Expected: [{"Port": 2049, "CIDR": "192.168.0.0/20"}]

# Create EFS file system
EFS_ID=$(aws efs create-file-system \
  --performance-mode generalPurpose \
  --throughput-mode bursting \
  --encrypted \
  --tags Key=Name,Value=eks-learning-efs \
  --region us-east-1 \
  --query "FileSystemId" --output text)
echo "EFS: $EFS_ID"
```

**Create mount targets in each private subnet:**
```bash
# Get private subnet IDs (same ones used by node group)
SUBNETS=$(aws eks describe-cluster --name eks-learning-cluster --region us-east-1 \
  --query "cluster.resourcesVpcConfig.subnetIds" --output text)

# Create mount target in each subnet
for SUBNET in $SUBNETS; do
  echo "Creating mount target in $SUBNET..."
  aws efs create-mount-target \
    --file-system-id $EFS_ID \
    --subnet-id $SUBNET \
    --security-groups $SG_ID \
    --region us-east-1 2>/dev/null || echo "  (skipped — may already exist or wrong AZ)"
done
```

**Validate:**
```bash
aws efs describe-mount-targets --file-system-id $EFS_ID --region us-east-1 \
  --query "MountTargets[].{SubnetId:SubnetId,State:LifeCycleState}"
```
Wait until all mount targets show `available`.

---

### 5.8 Create IAM Role for EFS CSI Driver

```bash
eksctl create iamserviceaccount \
  --name efs-csi-controller-sa \
  --namespace kube-system \
  --cluster eks-learning-cluster \
  --role-name AmazonEKS_EFS_CSI_DriverRole \
  --role-only \
  --attach-policy-arn arn:aws:iam::aws:policy/service-role/AmazonEFSCSIDriverPolicy \
  --region us-east-1 \
  --approve
```

---

### 5.9 Install EFS CSI Driver (EKS Add-on)

```bash
aws eks create-addon \
  --cluster-name eks-learning-cluster \
  --addon-name aws-efs-csi-driver \
  --service-account-role-arn arn:aws:iam::$AWS_ACCOUNT_ID:role/AmazonEKS_EFS_CSI_DriverRole \
  --region us-east-1
```

**Validate:**
```bash
aws eks describe-addon --cluster-name eks-learning-cluster \
  --addon-name aws-efs-csi-driver --region us-east-1 \
  --query "addon.{Status:status,Version:addonVersion}"

kubectl get pods -n kube-system -l app.kubernetes.io/name=aws-efs-csi-driver
```

---

### 5.10 Create StorageClass + PV + PVC for EFS

```bash
cat > efs-storage.yaml << 'EOF'
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
  name: efs-sc
provisioner: efs.csi.aws.com
---
apiVersion: v1
kind: PersistentVolume
metadata:
  name: efs-pv
spec:
  capacity:
    storage: 5Gi
  volumeMode: Filesystem
  accessModes:
    - ReadWriteMany
  persistentVolumeReclaimPolicy: Retain
  storageClassName: efs-sc
  csi:
    driver: efs.csi.aws.com
    volumeHandle: EFS_ID_PLACEHOLDER
---
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: efs-claim
spec:
  accessModes:
    - ReadWriteMany
  storageClassName: efs-sc
  resources:
    requests:
      storage: 5Gi
EOF

# Replace placeholder with actual EFS ID
sed -i "s/EFS_ID_PLACEHOLDER/$EFS_ID/" efs-storage.yaml
# On macOS use: sed -i '' "s/EFS_ID_PLACEHOLDER/$EFS_ID/" efs-storage.yaml

kubectl apply -f efs-storage.yaml
```

**File:** `module-05-storage/efs-storage.yaml`

> ⚠️ Make sure `$EFS_ID` is still set from step 5.7. If not, retrieve it:
> ```bash
> EFS_ID=$(aws efs describe-file-systems --region us-east-1 \
>   --query "FileSystems[?Name=='eks-learning-efs'].FileSystemId" --output text)
> ```

**Validate:**
```bash
kubectl get pv efs-pv
kubectl get pvc efs-claim
```

---

### 5.11 Test EFS — Multiple Pods Sharing Storage

```bash
cat > efs-test-pods.yaml << 'EOF'
apiVersion: v1
kind: Pod
metadata:
  name: efs-writer
spec:
  containers:
  - name: writer
    image: busybox:1.36
    command: ["/bin/sh", "-c"]
    args:
    - |
      while true; do
        echo "Written by efs-writer at $(date)" >> /shared/log.txt
        sleep 5
      done
    volumeMounts:
    - mountPath: /shared
      name: efs-volume
  volumes:
  - name: efs-volume
    persistentVolumeClaim:
      claimName: efs-claim
---
apiVersion: v1
kind: Pod
metadata:
  name: efs-reader
spec:
  containers:
  - name: reader
    image: busybox:1.36
    command: ["/bin/sh", "-c"]
    args: ["tail -f /shared/log.txt"]
    volumeMounts:
    - mountPath: /shared
      name: efs-volume
  volumes:
  - name: efs-volume
    persistentVolumeClaim:
      claimName: efs-claim
EOF

kubectl apply -f efs-test-pods.yaml
```

**File:** `module-05-storage/efs-test-pods.yaml`

**Validate:**
```bash
# Wait for pods to be running
kubectl get pods efs-writer efs-reader

# Check reader sees writer's data (ReadWriteMany in action!)
kubectl logs efs-reader

# Or exec into reader
kubectl exec efs-reader -- cat /shared/log.txt
```

---

### 5.12 Cleanup EFS Resources

```bash
# Delete K8s resources
kubectl delete -f efs-test-pods.yaml
kubectl delete -f efs-storage.yaml

# Delete EFS mount targets
for MT in $(aws efs describe-mount-targets --file-system-id $EFS_ID --region us-east-1 \
  --query "MountTargets[].MountTargetId" --output text); do
  aws efs delete-mount-target --mount-target-id $MT --region us-east-1
done

# Wait ~1 min for mount targets to delete, then:
aws efs delete-file-system --file-system-id $EFS_ID --region us-east-1

# Delete security group
aws ec2 delete-security-group --group-id $SG_ID --region us-east-1
```

---

## Key Takeaways

1. **EBS CSI** = block storage, single pod, single AZ — ideal for databases
2. **EFS CSI** = shared file system, multi-pod, multi-AZ — ideal for shared data
3. **StorageClass** = defines provisioning rules (type, reclaim policy, expansion)
4. **WaitForFirstConsumer** = critical for EBS to avoid AZ mismatch
5. **IRSA** = each CSI driver needs its own IAM role
6. **EKS Add-ons** = preferred way to install CSI drivers (managed updates)
7. **PVC survives pod deletion** — data persists until PVC is deleted

---

## Troubleshooting

| Issue | Cause | Fix |
|-------|-------|-----|
| PVC stuck in `Pending` | CSI driver not installed or no IRSA | Check add-on status + IAM role |
| `UnauthorizedOperation` on volume create | Missing IAM permissions | Verify `AmazonEBSCSIDriverPolicy` attached |
| EBS pod stuck in `Pending` (different AZ) | Volume in wrong AZ | Use `WaitForFirstConsumer` binding mode |
| EFS mount timeout | Security group missing NFS rule | Allow TCP 2049 from VPC CIDR |
| EFS mount target not available | Still creating | Wait 1-2 min, check `LifeCycleState` |
| EFS `Failed to resolve` DNS error | SG created but port 2049 not added | Verify: `aws ec2 describe-security-groups --group-ids <SG> --query "SecurityGroups[].IpPermissions[]"` |
| EFS `Not authorized to perform sts:AssumeRoleWithWebIdentity` | Secondary error when DNS fails | Fix the DNS/SG issue first — this resolves itself |

---

## Exercises Summary

| # | Task | Type |
|---|------|------|
| 1 | Create IRSA role for EBS CSI | IAM |
| 2 | Install EBS CSI driver add-on | EKS Add-on |
| 3 | Create gp3 StorageClass | K8s |
| 4 | Create PVC + Pod with EBS volume | K8s |
| 5 | Test data persistence (delete pod, recreate) | K8s |
| 6 | Cleanup EBS resources | K8s |
| 7 | Create EFS file system + security group + mount targets | AWS |
| 8 | Create IRSA role for EFS CSI | IAM |
| 9 | Install EFS CSI driver add-on | EKS Add-on |
| 10 | Create StorageClass + PV + PVC for EFS | K8s |
| 11 | Test ReadWriteMany (writer + reader pods) | K8s |
| 12 | Cleanup EFS resources | AWS + K8s |

---

## Next: Module 6 — Security (IRSA, RBAC, Pod Security, Secrets)
