# 🚀 AWS EKS IAM Access + IRSA

> 🔐 **AWS EKS Security | IAM | OIDC | IRSA | RBAC | AWS Services**

A practical hands-on guide to implementing **IAM access for EKS users** and **IAM Roles for Service Accounts (IRSA)** for Kubernetes workloads.

---

## 🌈 What You'll Learn

| 🔧 Topic           | 📚 What You'll Learn          |
| ------------------ | ----------------------------- |
| 👤 IAM             | IAM users, roles and policies |
| ☸️ EKS             | EKS authentication and access |
| 🔑 Access Entry    | Give users access to EKS      |
| 🛡️ RBAC           | Kubernetes authorization      |
| 🪪 OIDC            | EKS identity provider         |
| 🔐 IRSA            | IAM permissions for pods      |
| ☁️ S3              | Pod access to S3              |
| 🔒 Secrets Manager | Secure application secrets    |
| 🗄️ DynamoDB       | Workload-level AWS access     |
| 🧪 Testing         | Verify IAM from inside pods   |

---

# 🏗️ Architecture

```text
                         ☁️ AWS ACCOUNT
                              │
                ┌─────────────┴─────────────┐
                │                           │
                ▼                           ▼
          👤 HUMAN USER                🤖 POD / APP
                │                           │
                │                           │
        EKS ACCESS ENTRY              SERVICE ACCOUNT
                │                           │
                ▼                           ▼
        ☸️ EKS CLUSTER                    IRSA
                │                           │
                ▼                           ▼
        🔐 Kubernetes RBAC              IAM ROLE
                │                           │
                │                  ┌────────┼────────┐
                │                  │        │        │
                ▼                  ▼        ▼        ▼
          📦 roboshop             S3    Secrets   DynamoDB
           namespace
```

---

# ⭐ IAM Access vs IRSA

## 👤 Human Access

```text
👤 IAM User / IAM Role
          │
          ▼
🔑 EKS Access Entry
          │
          ▼
🛡️ Kubernetes RBAC
          │
          ▼
☸️ kubectl
```

## 🤖 Application Access

```text
📦 Kubernetes Pod
        │
        ▼
🪪 ServiceAccount
        │
        ▼
🔐 IRSA
        │
        ▼
☁️ IAM Role
        │
        ▼
AWS Services
```

> 💡 **Important:** IAM access for humans and IRSA for workloads solve different problems.

---

# 🧰 Prerequisites

Install:

* ☁️ AWS CLI
* ☸️ kubectl
* 🛠️ eksctl
* 🏗️ Terraform
* 📦 Helm

Check:

```bash
aws --version
kubectl version --client
eksctl version
terraform version
```

Configure AWS:

```bash
aws configure
```

Verify:

```bash
aws sts get-caller-identity
```

---

# ⚙️ 1. Configure Environment

```bash
export AWS_REGION=ap-south-1
export CLUSTER_NAME=roboshop-eks
export AWS_ACCOUNT_ID=$(aws sts get-caller-identity --query Account --output text)
```

Check:

```bash
echo $AWS_REGION
echo $CLUSTER_NAME
echo $AWS_ACCOUNT_ID
```

Example:

```text
🌍 Region      : ap-south-1
☸️ Cluster     : roboshop-eks
🏦 Account ID  : 123456789012
```

---

# 🔗 2. Get EKS OIDC Provider

IRSA requires an **OIDC identity provider**.

```bash
aws eks describe-cluster \
  --name $CLUSTER_NAME \
  --region $AWS_REGION \
  --query "cluster.identity.oidc.issuer" \
  --output text
```

Example:

```text
https://oidc.eks.ap-south-1.amazonaws.com/id/XXXXXXXX
```

Remove:

```text
https://
```

Result:

```text
oidc.eks.ap-south-1.amazonaws.com/id/XXXXXXXX
```

---

# 🔐 3. Enable OIDC for IRSA

```bash
eksctl utils associate-iam-oidc-provider \
  --cluster $CLUSTER_NAME \
  --region $AWS_REGION \
  --approve
```

Verify:

```bash
aws iam list-open-id-connect-providers
```

Expected:

```text
✅ OIDC Provider
     ↓
🔐 IRSA
     ↓
☁️ IAM Role
```

---

# 👤 4. Create IAM User

> ⚠️ For production, prefer **IAM Identity Center / federated roles** over long-lived IAM users.

Create:

```bash
aws iam create-user \
  --user-name eks-developer
```

Get ARN:

```bash
aws iam get-user \
  --user-name eks-developer \
  --query "User.Arn" \
  --output text
```

Example:

```text
arn:aws:iam::123456789012:user/eks-developer
```

---

# 🔑 5. Create EKS Access Entry

```bash
aws eks create-access-entry \
  --cluster-name $CLUSTER_NAME \
  --principal-arn arn:aws:iam::$AWS_ACCOUNT_ID:user/eks-developer \
  --type STANDARD \
  --region $AWS_REGION
```

Verify:

```bash
aws eks list-access-entries \
  --cluster-name $CLUSTER_NAME \
  --region $AWS_REGION
```

Expected:

```text
👤 eks-developer
       │
       ▼
🔑 EKS Access Entry
       │
       ▼
☸️ EKS Cluster
```

---

# 👀 6. Give Read-Only EKS Access

```bash
aws eks associate-access-policy \
  --cluster-name $CLUSTER_NAME \
  --principal-arn arn:aws:iam::$AWS_ACCOUNT_ID:user/eks-developer \
  --policy-arn arn:aws:eks::aws:cluster-access-policy/AmazonEKSViewPolicy \
  --access-scope type=cluster \
  --region $AWS_REGION
```

### Result

```text
👤 Developer
     │
     ▼
🔑 Access Entry
     │
     ▼
👀 AmazonEKSViewPolicy
     │
     ▼
☸️ EKS
```

---

# 📦 7. Namespace-Level Access

Restrict the developer to:

```text
roboshop
```

Run:

```bash
aws eks associate-access-policy \
  --cluster-name $CLUSTER_NAME \
  --principal-arn arn:aws:iam::$AWS_ACCOUNT_ID:user/eks-developer \
  --policy-arn arn:aws:eks::aws:cluster-access-policy/AmazonEKSViewPolicy \
  --access-scope type=namespace,namespaces=roboshop \
  --region $AWS_REGION
```

### 🔒 Result

```text
👤 Developer
     │
     ▼
🔑 EKS Access Entry
     │
     ▼
📦 roboshop
     │
     └── 👀 Read Access
```

---

# ☸️ 8. Configure kubectl

```bash
aws eks update-kubeconfig \
  --region $AWS_REGION \
  --name $CLUSTER_NAME
```

Check:

```bash
kubectl config current-context
```

Test:

```bash
kubectl get pods -n roboshop
```

---

# 🪪 9. Create IAM Policy for S3

Create:

```text
s3-read-policy.json
```

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Sid": "S3ReadAccess",
      "Effect": "Allow",
      "Action": [
        "s3:GetObject"
      ],
      "Resource": "arn:aws:s3:::roboshop-example-bucket/*"
    },
    {
      "Sid": "S3ListAccess",
      "Effect": "Allow",
      "Action": [
        "s3:ListBucket"
      ],
      "Resource": "arn:aws:s3:::roboshop-example-bucket"
    }
  ]
}
```

Create IAM policy:

```bash
aws iam create-policy \
  --policy-name RoboshopS3ReadPolicy \
  --policy-document file://s3-read-policy.json
```

---

# 🔐 10. Create IRSA ServiceAccount

Using `eksctl`:

```bash
eksctl create iamserviceaccount \
  --cluster $CLUSTER_NAME \
  --region $AWS_REGION \
  --namespace roboshop \
  --name cart-sa \
  --attach-policy-arn arn:aws:iam::$AWS_ACCOUNT_ID:policy/RoboshopS3ReadPolicy \
  --approve \
  --override-existing-serviceaccounts
```

Architecture:

```text
📦 cart Pod
     │
     ▼
🪪 cart-sa
     │
     ▼
🔐 IRSA
     │
     ▼
☁️ IAM Role
     │
     ▼
🪣 S3
```

---

# 🔎 11. Verify ServiceAccount

```bash
kubectl get serviceaccount cart-sa \
  -n roboshop
```

Detailed:

```bash
kubectl describe serviceaccount cart-sa \
  -n roboshop
```

Look for:

```text
eks.amazonaws.com/role-arn
```

---

# 📝 12. Kubernetes ServiceAccount

```yaml
apiVersion: v1
kind: ServiceAccount

metadata:
  name: cart-sa
  namespace: roboshop

  annotations:
    eks.amazonaws.com/role-arn: arn:aws:iam::123456789012:role/roboshop-cart-irsa
```

Apply:

```bash
kubectl apply -f serviceaccount.yaml
```

---

# 🚀 13. Use ServiceAccount in Deployment

```yaml
apiVersion: apps/v1
kind: Deployment

metadata:
  name: cart
  namespace: roboshop

spec:
  replicas: 2

  selector:
    matchLabels:
      app: cart

  template:
    metadata:
      labels:
        app: cart

    spec:
      serviceAccountName: cart-sa

      containers:
        - name: cart
          image: your-account/cart:1.0.0
          ports:
            - containerPort: 8080
```

Apply:

```bash
kubectl apply -f deployment.yaml
```

---

# 🧪 14. Verify Pod ServiceAccount

```bash
kubectl get pods -n roboshop
```

Then:

```bash
kubectl describe pod <POD_NAME> \
  -n roboshop
```

Look for:

```text
Service Account: cart-sa
```

---

# 🧪 15. Test IAM From Inside Pod

Create test pod:

```bash
kubectl run aws-test \
  -n roboshop \
  --image=amazon/aws-cli \
  --serviceaccount=cart-sa \
  --command -- sleep 3600
```

Run:

```bash
kubectl exec -it aws-test \
  -n roboshop -- \
  aws sts get-caller-identity
```

Expected:

```json
{
    "UserId": "AROAXXXXXXXXXXXX",
    "Account": "123456789012",
    "Arn": "arn:aws:sts::123456789012:assumed-role/roboshop-cart-irsa/..."
}
```

### 🎯 What does this prove?

```text
✅ Pod received AWS credentials
✅ Credentials are temporary
✅ Pod assumed IAM Role
✅ IRSA is working
```

---

# 🪣 16. Test S3 Access

```bash
kubectl exec -it aws-test \
  -n roboshop -- \
  aws s3 ls s3://roboshop-example-bucket
```

Download:

```bash
kubectl exec -it aws-test \
  -n roboshop -- \
  aws s3 cp \
  s3://roboshop-example-bucket/test.txt \
  /tmp/test.txt
```

---

# 🚫 17. Test Access Denied

Try an unauthorized operation:

```bash
kubectl exec -it aws-test \
  -n roboshop -- \
  aws s3 rm s3://roboshop-example-bucket/test.txt
```

Expected:

```text
❌ AccessDenied
```

This demonstrates:

> 🔐 **Least Privilege**

---

# 🔄 18. IRSA Authentication Flow

```text
             ☸️ EKS
               │
               ▼
       🪪 ServiceAccount
               │
               ▼
        🔐 Projected Token
               │
               ▼
            ☁️ AWS STS
               │
               │ AssumeRoleWithWebIdentity
               ▼
            🔑 IAM Role
               │
               ▼
        📜 IAM Permissions
               │
       ┌───────┼────────┐
       ▼       ▼        ▼
      🪣 S3   🔐 SM   🗄️ DynamoDB
```

---

# 🔏 19. IRSA Trust Policy

Example:

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Principal": {
        "Federated": "arn:aws:iam::123456789012:oidc-provider/oidc.eks.ap-south-1.amazonaws.com/id/XXXXXXXX"
      },
      "Action": "sts:AssumeRoleWithWebIdentity",
      "Condition": {
        "StringEquals": {
          "oidc.eks.ap-south-1.amazonaws.com/id/XXXXXXXX:aud": "sts.amazonaws.com",
          "oidc.eks.ap-south-1.amazonaws.com/id/XXXXXXXX:sub": "system:serviceaccount:roboshop:cart-sa"
        }
      }
    }
  ]
}
```

### 🔥 Critical value

```text
system:serviceaccount:roboshop:cart-sa
```

This means:

```text
Namespace  → roboshop
ServiceAccount → cart-sa
```

---

# 🔒 20. Secrets Manager With IRSA

Example policy:

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Action": [
        "secretsmanager:GetSecretValue"
      ],
      "Resource": "arn:aws:secretsmanager:ap-south-1:123456789012:secret:roboshop/*"
    }
  ]
}
```

Application can use:

```bash
aws secretsmanager get-secret-value \
  --secret-id roboshop/cart
```

---

# 🗄️ 21. DynamoDB With IRSA

Example:

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Action": [
        "dynamodb:GetItem",
        "dynamodb:PutItem",
        "dynamodb:Query"
      ],
      "Resource": "arn:aws:dynamodb:ap-south-1:123456789012:table/roboshop-cart"
    }
  ]
}
```

---

# 📦 22. ECR Access

For image pulling, EKS nodes commonly receive ECR permissions through the **node IAM role**.

Recommended model:

```text
              ☸️ EKS
                │
       ┌────────┴────────┐
       │                 │
       ▼                 ▼
🖥️ Node IAM Role     🤖 Pod IRSA
       │                 │
       ▼                 ▼
   📦 ECR Pull      AWS Application APIs
                         │
                  ┌──────┼──────┐
                  ▼      ▼      ▼
                 S3   Secrets DynamoDB
```

---

# 🛠️ 23. Troubleshooting

## 🔍 Check OIDC

```bash
aws eks describe-cluster \
  --name $CLUSTER_NAME \
  --query "cluster.identity.oidc.issuer" \
  --output text
```

---

## 🔍 Check ServiceAccount

```bash
kubectl get sa cart-sa \
  -n roboshop \
  -o yaml
```

Look for:

```text
eks.amazonaws.com/role-arn
```

---

## 🔍 Check Pod

```bash
kubectl describe pod <POD_NAME> \
  -n roboshop
```

Verify:

```text
Service Account: cart-sa
```

---

## 🔍 Check IAM Role

```bash
aws iam get-role \
  --role-name roboshop-cart-irsa
```

---

## 🔍 Check IAM Policies

```bash
aws iam list-attached-role-policies \
  --role-name roboshop-cart-irsa
```

---

## 🔍 Check EKS Access Entries

```bash
aws eks list-access-entries \
  --cluster-name $CLUSTER_NAME
```

---

# ❌ Common Problems

### 1️⃣ Pod gets wrong IAM permissions

Check:

```bash
kubectl get sa cart-sa \
  -n roboshop \
  -o yaml
```

Make sure:

```yaml
annotations:
  eks.amazonaws.com/role-arn: arn:aws:iam::ACCOUNT_ID:role/ROLE_NAME
```

---

### 2️⃣ Namespace mismatch

IAM trust policy:

```text
system:serviceaccount:roboshop:cart-sa
```

Kubernetes:

```text
Namespace: roboshop
ServiceAccount: cart-sa
```

They must match exactly.

---

### 3️⃣ Wrong OIDC provider

Compare:

```bash
aws eks describe-cluster \
  --name $CLUSTER_NAME \
  --query "cluster.identity.oidc.issuer"
```

with the IAM trust relationship.

---

### 4️⃣ AccessDenied

Check:

```bash
aws iam list-attached-role-policies \
  --role-name roboshop-cart-irsa
```

Then verify:

```text
IAM Action
     +
Resource ARN
     +
Trust Policy
```

---

# 🛡️ 24. Security Best Practices

### ✅ Least Privilege

Avoid:

```text
❌ AdministratorAccess
```

Prefer:

```text
✅ s3:GetObject
✅ s3:ListBucket
✅ secretsmanager:GetSecretValue
✅ dynamodb:GetItem
```

---

### ✅ Separate IAM Roles

```text
cart-sa
   ↓
cart-irsa-role
   ↓
DynamoDB

catalogue-sa
   ↓
catalogue-irsa-role
   ↓
S3

user-sa
   ↓
user-irsa-role
   ↓
Secrets Manager
```

---

### ✅ Don't Store AWS Keys in Kubernetes

Avoid:

```yaml
AWS_ACCESS_KEY_ID: ...
AWS_SECRET_ACCESS_KEY: ...
```

Use:

```text
🔐 IRSA
```

instead.

---

### ✅ Prefer Federated Human Access

Production:

```text
👤 Developer
     │
     ▼
AWS IAM Identity Center
     │
     ▼
🔑 IAM Role
     │
     ▼
EKS Access Entry
     │
     ▼
☸️ Kubernetes
```

---

# 🏭 25. Production Architecture

```text
                         ☁️ AWS
                           │
                    🔐 IAM Identity Center
                           │
                           ▼
                     👤 Developer
                           │
                           ▼
                    🔑 IAM Role
                           │
                           ▼
                  EKS Access Entry
                           │
                           ▼
                    🛡️ Kubernetes RBAC
                           │
                           ▼
                      ☸️ EKS
                           │
             ┌─────────────┴─────────────┐
             │                           │
             ▼                           ▼
        👤 Human Access             🤖 Applications
                                         │
                                         ▼
                                  🪪 ServiceAccount
                                         │
                                         ▼
                                      🔐 IRSA
                                         │
                                         ▼
                                    ☁️ IAM Role
                                         │
                          ┌──────────────┼──────────────┐
                          ▼              ▼              ▼
                         🪣 S3      🔐 Secrets      🗄️ DynamoDB
```

---

# 📋 26. Quick Command Reference

### ☁️ AWS Identity

```bash
aws sts get-caller-identity
```

### ☸️ EKS

```bash
aws eks describe-cluster \
  --name roboshop-eks
```

### 🔗 OIDC

```bash
aws eks describe-cluster \
  --name roboshop-eks \
  --query "cluster.identity.oidc.issuer"
```

### 🔑 Access Entries

```bash
aws eks list-access-entries \
  --cluster-name roboshop-eks
```

### 🪪 ServiceAccounts

```bash
kubectl get sa -n roboshop
```

### 🔐 IRSA Role

```bash
kubectl get sa cart-sa \
  -n roboshop \
  -o jsonpath='{.metadata.annotations.eks\.amazonaws\.com/role-arn}'
```

### 🧪 Pod IAM Identity

```bash
kubectl exec -it <POD_NAME> \
  -n roboshop -- \
  aws sts get-caller-identity
```

---

# 🎯 27. Interview Questions

### ❓ What is IRSA?

**IRSA = IAM Roles for Service Accounts.**

It allows EKS workloads to obtain temporary AWS credentials through an IAM role associated with a Kubernetes ServiceAccount.

---

### ❓ Why use IRSA?

It provides:

```text
🔐 Fine-grained permissions
🚫 No static AWS credentials
⏱️ Temporary credentials
🛡️ Least privilege
```

---

### ❓ IAM User vs IRSA?

| 👤 IAM User/Role       | 🤖 IRSA                 |
| ---------------------- | ----------------------- |
| Human access           | Pod access              |
| EKS authentication     | AWS API authorization   |
| EKS Access Entry       | ServiceAccount          |
| kubectl                | AWS SDK/CLI             |
| Kubernetes permissions | AWS service permissions |

---

### ❓ What is OIDC?

**OIDC — OpenID Connect** provides the identity federation mechanism used by EKS to establish trust between Kubernetes ServiceAccount tokens and AWS IAM.

---

### ❓ What does STS do?

AWS STS provides **temporary security credentials** when a workload assumes an IAM role.

---

### ❓ Can applications have different IAM permissions?

Yes.

```text
cart-sa
   ↓
cart-role
   ↓
DynamoDB

catalogue-sa
   ↓
catalogue-role
   ↓
S3
```

---

# 🧠 Key Concept

## ⭐ Remember This

```text
👤 HUMAN
IAM Identity
     ↓
EKS Access Entry
     ↓
Kubernetes RBAC
     ↓
☸️ EKS


🤖 POD
ServiceAccount
     ↓
IRSA
     ↓
IAM Role
     ↓
AWS Services
```

---

# 🏆 Final Architecture

```text
                    🔐 AWS IAM
                        │
          ┌─────────────┴─────────────┐
          │                           │
          ▼                           ▼
       👤 HUMAN                    🤖 POD
          │                           │
          ▼                           ▼
   EKS Access Entry             ServiceAccount
          │                           │
          ▼                           ▼
 Kubernetes RBAC                    IRSA
          │                           │
          ▼                           ▼
       ☸️ EKS                     IAM Role
                                      │
                         ┌────────────┼────────────┐
                         │            │            │
                         ▼            ▼            ▼
                        🪣 S3      🔐 Secrets   🗄️ DynamoDB
```

---

# 🚀 Hands-On Roboshop Lab

Build a production-style setup:

```text
☸️ EKS
│
└── 📦 roboshop
    │
    ├── 🛒 cart
    │   ├── cart-sa
    │   └── cart-irsa-role
    │       └── 🗄️ DynamoDB
    │
    ├── 📚 catalogue
    │   ├── catalogue-sa
    │   └── catalogue-irsa-role
    │       └── 🪣 S3
    │
    └── 👤 user
        ├── user-sa
        └── user-irsa-role
            └── 🔐 Secrets Manager
```

### 👨‍💻 Developer Access

```text
Developer
   ↓
IAM Identity Center / IAM Role
   ↓
EKS Access Entry
   ↓
roboshop namespace
   ↓
kubectl
```

### 🤖 Application Access

```text
Application
   ↓
ServiceAccount
   ↓
IRSA
   ↓
IAM Role
   ↓
AWS Services
```

---

# 🌟 Technologies

```text
☁️ AWS
☸️ Amazon EKS
🔐 AWS IAM
🪪 OIDC
🔑 EKS Access Entries
🛡️ Kubernetes RBAC
🔐 IRSA
🪣 Amazon S3
🔒 AWS Secrets Manager
🗄️ Amazon DynamoDB
📦 Amazon ECR
🛠️ eksctl
⌨️ AWS CLI
```

---

# 📌 Conclusion

The core security model is:

> 👤 **Human → EKS Access Entry → Kubernetes permissions**

and

> 🤖 **Pod → ServiceAccount → IRSA → IAM Role → AWS permissions**

Using these separately provides a clean foundation for **secure EKS authentication, Kubernetes authorization, workload-level AWS permissions, and least-privilege access**.

---

## ⭐ If this helped you

```text
⭐ Star the repository
🍴 Fork the project
📢 Share with the DevOps community
💬 Open an issue for questions
```

**#AWS #EKS #IAM #IRSA #Kubernetes #DevOps #Cloud #AWSDevOps #CloudSecurity #SRE #Terraform #Docker #CICD**
