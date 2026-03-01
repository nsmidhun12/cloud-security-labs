# 🔐 IRSA vs RBAC – Amazon EKS Security Deep Dive

This lab demonstrates the difference between:

- **IRSA (IAM Roles for Service Accounts)** → Controls access to AWS services
- **Kubernetes RBAC** → Controls access to Kubernetes API resources

Understanding this separation is critical for secure EKS workload design.

---

# Conceptual Overview

Amazon EKS operates with **two independent identity systems**:

| Layer | Identity System | Controls |
|-------|-----------------|----------|
| AWS Layer | IAM | Access to AWS services (S3, DynamoDB, etc.) |
| Kubernetes Layer | RBAC | Access to Kubernetes resources (pods, secrets, deployments) |

These systems do not replace each other.

- Removing IRSA → Pod loses AWS access  
- Removing RBAC → Pod loses Kubernetes API access  

They solve different security problems.

---

# Architecture Flow

1. Pod uses a Kubernetes Service Account  
2. Service Account is linked to an IAM Role via OIDC  
3. Pod receives temporary credentials from AWS STS  
4. IAM policy determines AWS access  
5. RBAC rules determine Kubernetes API access  

---

# Prerequisites

- AWS CLI configured
- kubectl installed
- eksctl installed
- IAM permissions to:
  - Create EKS cluster
  - Create IAM roles and policies

---

# Step 1 – Create EKS Cluster

```bash
eksctl create cluster \
  --name demo \
  --region us-east-1 \
  --nodegroup-name demo-nodes \
  --node-type t3.medium \
  --nodes 2

aws eks update-kubeconfig --name demo --region us-east-1
```

---

# Step 2 – Enable OIDC Provider

IRSA requires OIDC federation between Kubernetes and AWS IAM.

```bash
eksctl utils associate-iam-oidc-provider \
  --region us-east-1 \
  --cluster demo \
  --approve
```

Without this step, IAM role assumption will fail.

---

# Step 3 – Create S3 Bucket

```bash
aws s3 mb s3://YOUR_BUCKET_NAME

echo "IRSA SUCCESS DEMO" > test.txt
aws s3 cp test.txt s3://YOUR_BUCKET_NAME/
```

---

# Step 4 – Create IAM Policy

`s3-read-policy.json`

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Action": [
        "s3:GetObject",
        "s3:ListBucket"
      ],
      "Resource": [
        "arn:aws:s3:::YOUR_BUCKET_NAME",
        "arn:aws:s3:::YOUR_BUCKET_NAME/*"
      ]
    }
  ]
}
```

Create policy:

```bash
aws iam create-policy \
  --policy-name EKS-IRSA-S3-Read \
  --policy-document file://s3-read-policy.json
```

---

# Step 5 – Create IAM Service Account (IRSA)

```bash
eksctl create iamserviceaccount \
  --name s3-reader \
  --namespace default \
  --cluster demo \
  --attach-policy-arn arn:aws:iam::YOUR_ACCOUNT_ID:policy/EKS-IRSA-S3-Read \
  --approve
```

This creates:

- IAM Role
- Trust policy with OIDC
- Kubernetes Service Account
- Annotation linking IAM role to Service Account

---

# Step 6 – Test Pod WITH IRSA

Apply:

```bash
kubectl apply -f irsa-pod.yaml
kubectl exec -it irsa-demo -- sh
```

Verify identity:

```bash
aws sts get-caller-identity
```

You should see the IAM role ARN.

Test S3 access:

```bash
aws s3 ls
aws s3 cp s3://YOUR_BUCKET_NAME/test.txt .
cat test.txt
```

Expected:

```
IRSA SUCCESS DEMO
```

This confirms AWS access via IRSA.

---

# Step 7 – Test Pod WITHOUT IRSA

Apply:

```bash
kubectl apply -f no-irsa.yaml
kubectl exec -it no-irsa -- sh
```

Run:

```bash
aws s3 ls
```

Expected result:

```
AccessDenied
```

This proves AWS access is controlled by IRSA — not RBAC.

---

# Step 8 – Test Kubernetes RBAC

Apply role:

```bash
kubectl apply -f rbac-role.yaml
kubectl create serviceaccount rbac-demo
kubectl apply -f rolebinding.yaml
```

Test permissions:

```bash
kubectl auth can-i list pods \
  --as=system:serviceaccount:default:rbac-demo
```

Expected:

```
yes
```

```bash
kubectl auth can-i delete pods \
  --as=system:serviceaccount:default:rbac-demo
```

Expected:

```
no
```

This confirms RBAC controls Kubernetes API permissions.

---

# Security Observations

- IRSA follows least privilege at AWS layer
- RBAC follows least privilege at Kubernetes layer
- Node IAM roles should NOT be used for application access
- Avoid broad permissions like `s3:*`
- Always verify trust relationships in IAM

---

# Key Takeaways

- IRSA → AWS authorization
- RBAC → Kubernetes authorization
- Both layers must be configured correctly
- Security failures often occur when these boundaries are misunderstood

---

# Cleanup

Always delete the cluster to avoid costs:

```bash
eksctl delete cluster --name demo --region us-east-1
```

---

# Conclusion

IRSA and RBAC are complementary security mechanisms.

Secure EKS workloads require:

- Proper IAM design
- Proper RBAC configuration
- Strict least privilege enforcement
- Clear separation of identity boundaries

Understanding this separation prevents privilege escalation and lateral movement risks in cloud-native environments.
