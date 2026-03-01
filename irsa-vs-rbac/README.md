IRSA vs Pod Identity vs RBAC – Amazon EKS Security Deep Dive

This lab demonstrates the difference between:

IRSA (IAM Roles for Service Accounts) → Controls access to AWS services using OIDC federation

EKS Pod Identity → Modern simplified workload identity for AWS access

Kubernetes RBAC → Controls access to Kubernetes API resources

Understanding this separation is critical for secure EKS workload design.

Conceptual Overview

Amazon EKS operates with two independent identity systems:

Layer	Identity System	Controls
AWS Layer	IAM	Access to AWS services (S3, DynamoDB, etc.)
Kubernetes Layer	RBAC	Access to Kubernetes resources (pods, secrets, deployments)

IRSA and Pod Identity operate at the AWS layer

RBAC operates at the Kubernetes layer

These systems do not replace each other.

Removing IRSA or Pod Identity → Pod loses AWS access

Removing RBAC → Pod loses Kubernetes API access

They solve different security problems.

Architecture Flow
IRSA Flow

Pod uses a Kubernetes Service Account

Service Account is linked to an IAM Role via OIDC

Pod presents OIDC token to AWS STS

STS issues temporary credentials

IAM policy determines AWS access

Pod → ServiceAccount → OIDC → STS → IAM Role
Pod Identity Flow

Pod uses a Kubernetes Service Account

Service Account is associated with IAM Role using EKS API

Pod Identity Agent delivers credentials

IAM policy determines AWS access

Pod → ServiceAccount → Pod Identity Agent → IAM Role

No manual OIDC configuration required for association.

Prerequisites

AWS CLI configured

kubectl installed

eksctl installed

IAM permissions to:

Create EKS cluster

Create IAM roles and policies

Create Pod Identity associations

Step 1 – Create EKS Cluster
eksctl create cluster \
  --name demo \
  --region us-east-1 \
  --nodegroup-name demo-nodes \
  --node-type t3.medium \
  --nodes 2

aws eks update-kubeconfig --name demo --region us-east-1
Step 2 – Enable OIDC Provider (Required for IRSA Only)
eksctl utils associate-iam-oidc-provider \
  --region us-east-1 \
  --cluster demo \
  --approve

Without this step, IRSA role assumption will fail.

Step 3 – Create S3 Bucket
aws s3 mb s3://YOUR_BUCKET_NAME

echo "EKS SECURITY DEMO" > test.txt
aws s3 cp test.txt s3://YOUR_BUCKET_NAME/
Step 4 – Create IAM Policy

Create file s3-read-policy.json

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

Create policy:

aws iam create-policy \
  --policy-name EKS-S3-Read \
  --policy-document file://s3-read-policy.json
PART 1 – IRSA Demo
Step 5 – Create IAM Service Account (IRSA)
eksctl create iamserviceaccount \
  --name s3-reader \
  --namespace default \
  --cluster demo \
  --attach-policy-arn arn:aws:iam::YOUR_ACCOUNT_ID:policy/EKS-S3-Read \
  --approve

This creates:

IAM Role

OIDC trust relationship

Kubernetes Service Account

Annotation linking IAM role

Step 6 – Test Pod WITH IRSA

Create irsa-pod.yaml

apiVersion: v1
kind: Pod
metadata:
  name: irsa-demo
spec:
  serviceAccountName: s3-reader
  containers:
    - name: aws-cli
      image: amazon/aws-cli
      command: ["sleep", "3600"]

Apply and test:

kubectl apply -f irsa-pod.yaml
kubectl exec -it irsa-demo -- sh

aws sts get-caller-identity
aws s3 ls
aws s3 cp s3://YOUR_BUCKET_NAME/test.txt .
cat test.txt

Expected:

EKS SECURITY DEMO
Step 7 – Test Pod WITHOUT IRSA

Create no-irsa.yaml

apiVersion: v1
kind: Pod
metadata:
  name: no-irsa
spec:
  containers:
    - name: aws-cli
      image: amazon/aws-cli
      command: ["sleep", "3600"]
kubectl apply -f no-irsa.yaml
kubectl exec -it no-irsa -- sh
aws s3 ls

Expected result:

AccessDenied

This confirms AWS access is controlled by IRSA.

PART 2 – EKS Pod Identity Demo
Step 8 – Create IAM Role for Pod Identity

Create trust policy pod-identity-trust.json

{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Principal": {
        "Service": "pods.eks.amazonaws.com"
      },
      "Action": "sts:AssumeRole"
    }
  ]
}

Create role:

aws iam create-role \
  --role-name EKS-PodIdentity-S3-Read \
  --assume-role-policy-document file://pod-identity-trust.json

Attach policy:

aws iam attach-role-policy \
  --role-name EKS-PodIdentity-S3-Read \
  --policy-arn arn:aws:iam::YOUR_ACCOUNT_ID:policy/EKS-S3-Read
Step 9 – Create Service Account (No Annotation Required)
kubectl create serviceaccount podid-reader
Step 10 – Create Pod Identity Association
aws eks create-pod-identity-association \
  --cluster-name demo \
  --namespace default \
  --service-account podid-reader \
  --role-arn arn:aws:iam::YOUR_ACCOUNT_ID:role/EKS-PodIdentity-S3-Read

This links IAM role directly to the service account.

Step 11 – Deploy Pod Using Pod Identity

Create podid-demo.yaml

apiVersion: v1
kind: Pod
metadata:
  name: podid-demo
spec:
  serviceAccountName: podid-reader
  containers:
    - name: aws-cli
      image: amazon/aws-cli
      command: ["sleep", "3600"]

Apply and test:

kubectl apply -f podid-demo.yaml
kubectl exec -it podid-demo -- sh

aws sts get-caller-identity
aws s3 ls
aws s3 cp s3://YOUR_BUCKET_NAME/test.txt .
cat test.txt

Expected:

EKS SECURITY DEMO

Pod Identity works without OIDC annotations.

PART 3 – Kubernetes RBAC Demo
Step 12 – Create Role

rbac-role.yaml

apiVersion: rbac.authorization.k8s.io/v1
kind: Role
metadata:
  name: pod-viewer
rules:
- apiGroups: [""]
  resources: ["pods"]
  verbs: ["get", "list"]

Apply:

kubectl apply -f rbac-role.yaml
Step 13 – Create Service Account & Binding
kubectl create serviceaccount rbac-demo

rolebinding.yaml

apiVersion: rbac.authorization.k8s.io/v1
kind: RoleBinding
metadata:
  name: pod-viewer-binding
subjects:
- kind: ServiceAccount
  name: rbac-demo
roleRef:
  kind: Role
  name: pod-viewer
  apiGroup: rbac.authorization.k8s.io

Apply:

kubectl apply -f rolebinding.yaml
Step 14 – Test RBAC
kubectl auth can-i list pods \
  --as=system:serviceaccount:default:rbac-demo

Expected:

yes
kubectl auth can-i delete pods \
  --as=system:serviceaccount:default:rbac-demo

Expected:

no

RBAC controls Kubernetes API permissions.

Security Observations

IRSA and Pod Identity control AWS authorization

RBAC controls Kubernetes authorization

Node IAM roles should NOT be used for application access

Always apply least privilege

Identity complexity increases misconfiguration risk

Simplifying identity reduces attack surface

Key Takeaways

IRSA → OIDC-based workload identity

Pod Identity → Simplified workload identity model

RBAC → Kubernetes API authorization

IAM and RBAC are separate identity systems

Both layers must be configured correctly

Misunderstanding these boundaries leads to privilege escalation

Cleanup
eksctl delete cluster --name demo --region us-east-1
Conclusion

IRSA, Pod Identity, and RBAC are complementary security mechanisms.

Secure EKS workloads require:

Proper IAM design

Proper RBAC configuration

Strict least privilege enforcement

Clear separation of identity boundaries

Simplified identity architecture where possible

Understanding this separation prevents privilege escalation and lateral movement risks in cloud-native environments.
