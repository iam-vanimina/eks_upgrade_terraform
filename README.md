# eks_upgrade_terraform

# 🚀 EKS Kubernetes Cluster Upgrade Using Terraform — Zero Downtime

This project demonstrates how to perform an **Amazon EKS Kubernetes version upgrade using Terraform**, while maintaining application availability through:

* Terraform-managed EKS versions
* Managed Node Groups
* Rolling node updates
* PodDisruptionBudgets
* Multiple application replicas
* Readiness probes
* Spare node capacity
* EKS add-on management
* Prometheus/Grafana monitoring
* Argo CD/GitOps validation

> **Example:** Kubernetes `1.35 → 1.36`

Always verify that the target Kubernetes version is currently supported by EKS before applying the change.

---

# 🏗️ Architecture

```text
                         AWS
                          │
                          ▼
                    ┌──────────┐
                    │   ALB    │
                    └────┬─────┘
                         │
                         ▼
                ┌─────────────────┐
                │   Amazon EKS    │
                │                 │
                │ Control Plane   │
                │   1.35 → 1.36  │
                └────────┬────────┘
                         │
              ┌──────────┴──────────┐
              │                     │
              ▼                     ▼
       Managed Node Group    Managed Node Group
            1.36                  1.36
              │                     │
        ┌─────┴─────┐         ┌─────┴─────┐
        ▼     ▼     ▼         ▼     ▼     ▼
       Pod   Pod   Pod        Pod   Pod   Pod
        │                     │
        └──────────┬──────────┘
                   ▼
            Application
                   │
        ┌──────────┴──────────┐
        ▼                     ▼
   Prometheus              Grafana
```

---

# 📁 Repository Structure

```text
eks-zero-downtime-upgrade/
│
├── README.md
│
├── terraform/
│   ├── versions.tf
│   ├── providers.tf
│   ├── variables.tf
│   ├── terraform.tfvars
│   ├── main.tf
│   ├── eks.tf
│   ├── node-groups.tf
│   ├── addons.tf
│   ├── outputs.tf
│   └── backend.tf
│
├── kubernetes/
│   ├── namespace.yaml
│   ├── deployment.yaml
│   ├── service.yaml
│   ├── pdb.yaml
│   └── ingress.yaml
│
├── scripts/
│   ├── pre-upgrade-check.sh
│   ├── post-upgrade-check.sh
│   └── health-check.sh
│
└── docs/
    ├── architecture.md
    ├── upgrade-runbook.md
    ├── rollback.md
    └── troubleshooting.md
```

---

# 🔄 Terraform Upgrade Flow

The GitOps/Terraform workflow is:

```text
GitHub
   │
   ▼
Terraform Code
   │
   ▼
terraform plan
   │
   ▼
Review Changes
   │
   ▼
terraform apply
   │
   ├───────────────┐
   ▼               ▼
EKS Control      Node Groups
Plane             1.35 → 1.36
   │               │
   └───────┬───────┘
           ▼
      EKS Add-ons
           │
           ▼
    Application Health
           │
           ▼
      Prometheus
           │
           ▼
        Grafana
```

---

# 1. Terraform Provider

`terraform/providers.tf`

```hcl
terraform {
  required_version = ">= 1.6.0"

  required_providers {
    aws = {
      source  = "hashicorp/aws"
      version = "~> 6.0"
    }
  }
}

provider "aws" {
  region = var.aws_region
}
```

The AWS provider exposes `aws_eks_cluster` and `aws_eks_node_group` resources for Terraform-managed EKS infrastructure.

---

# 2. Terraform Variables

`terraform/variables.tf`

```hcl
variable "aws_region" {
  description = "AWS region"
  type        = string
  default     = "ap-south-1"
}

variable "cluster_name" {
  description = "EKS cluster name"
  type        = string
  default     = "eks-zero-downtime"
}

variable "kubernetes_version" {
  description = "EKS Kubernetes version"
  type        = string
  default     = "1.36"
}

variable "node_group_version" {
  description = "Managed node group Kubernetes version"
  type        = string
  default     = "1.36"
}

variable "node_instance_types" {
  description = "EKS worker node instance types"
  type        = list(string)

  default = [
    "t3.medium"
  ]
}
```

---

# 3. Terraform EKS Cluster

`terraform/eks.tf`

```hcl
resource "aws_eks_cluster" "this" {

  name     = var.cluster_name
  role_arn = aws_iam_role.eks_cluster.arn

  version = var.kubernetes_version

  vpc_config {
    subnet_ids = var.private_subnet_ids
  }

  access_config {
    authentication_mode = "API"
  }

  depends_on = [
    aws_iam_role_policy_attachment.eks_cluster_policy
  ]
}
```

The important upgrade-controlled attribute is:

```hcl
version = var.kubernetes_version
```

Therefore:

```text
1.35
```

can become:

```text
1.36
```

through a Terraform variable change.

---

# 4. Managed Node Group

`terraform/node-groups.tf`

```hcl
resource "aws_eks_node_group" "main" {

  cluster_name = aws_eks_cluster.this.name

  node_group_name = "${var.cluster_name}-ng"

  node_role_arn = aws_iam_role.eks_nodes.arn

  subnet_ids = var.private_subnet_ids

  version = var.node_group_version

  instance_types = var.node_instance_types

  capacity_type = "ON_DEMAND"

  scaling_config {

    desired_size = 3
    min_size     = 3
    max_size     = 6

  }

  update_config {

    max_unavailable = 1

  }

  depends_on = [

    aws_iam_role_policy_attachment.eks_worker_node_policy,

    aws_iam_role_policy_attachment.eks_cni_policy,

    aws_iam_role_policy_attachment.eks_container_registry_policy

  ]

  tags = {

    Name = "${var.cluster_name}-node"

    Environment = "production"

  }
}
```

The `update_config` block controls the node-group rolling update behavior. Terraform's AWS provider exposes `version`, scaling configuration and update configuration on `aws_eks_node_group`.

---

# 5. Zero-Downtime Node Configuration

For production:

```hcl
scaling_config {

  desired_size = 3

  min_size = 3

  max_size = 6
}

update_config {

  max_unavailable = 1
}
```

This provides:

```text
Minimum nodes = 3
Maximum nodes = 6

During upgrade:

Node 1 → replaced
Node 2 → running
Node 3 → running
New node → created
```

EKS drains pods from nodes before terminating them during managed node-group updates.

---

# 6. EKS Add-ons

`terraform/addons.tf`

```hcl
resource "aws_eks_addon" "vpc_cni" {

  cluster_name = aws_eks_cluster.this.name

  addon_name = "vpc-cni"

  resolve_conflicts_on_update = "OVERWRITE"

}

resource "aws_eks_addon" "coredns" {

  cluster_name = aws_eks_cluster.this.name

  addon_name = "coredns"

  resolve_conflicts_on_update = "OVERWRITE"

}

resource "aws_eks_addon" "kube_proxy" {

  cluster_name = aws_eks_cluster.this.name

  addon_name = "kube-proxy"

  resolve_conflicts_on_update = "OVERWRITE"

}

resource "aws_eks_addon" "ebs_csi" {

  cluster_name = aws_eks_cluster.this.name

  addon_name = "aws-ebs-csi-driver"

  resolve_conflicts_on_update = "OVERWRITE"

}
```

After a Kubernetes version upgrade, AWS recommends updating cluster components/add-ons and bringing nodes to the same Kubernetes minor version as the control plane.

---

# 7. Application Deployment

`kubernetes/deployment.yaml`

```yaml
apiVersion: apps/v1
kind: Deployment

metadata:
  name: frontend
  namespace: production

spec:

  replicas: 3

  strategy:

    type: RollingUpdate

    rollingUpdate:

      maxUnavailable: 0

      maxSurge: 1

  selector:

    matchLabels:
      app: frontend

  template:

    metadata:
      labels:
        app: frontend

    spec:

      containers:

      - name: frontend

        image: nginx:1.27

        ports:

        - containerPort: 8080

        readinessProbe:

          httpGet:

            path: /health

            port: 8080

          initialDelaySeconds: 10

          periodSeconds: 5

        livenessProbe:

          httpGet:

            path: /health

            port: 8080

          initialDelaySeconds: 30

          periodSeconds: 10
```

---

# 8. PodDisruptionBudget

`kubernetes/pdb.yaml`

```yaml
apiVersion: policy/v1

kind: PodDisruptionBudget

metadata:
  name: frontend-pdb
  namespace: production

spec:

  minAvailable: 2

  selector:

    matchLabels:

      app: frontend
```

Check:

```bash
kubectl get pdb -A
```

Expected:

```text
NAME           MIN AVAILABLE   ALLOWED DISRUPTIONS
frontend-pdb   2               1
```

This is important because EKS's rolling managed-node-group update respects PodDisruptionBudgets; AWS documents a separate force-update mode that does not respect PDBs.

---

# 9. Terraform Backend

For production, store Terraform state remotely.

`terraform/backend.tf`

```hcl
terraform {
  backend "s3" {

    bucket = "my-eks-terraform-state"

    key = "eks/production/terraform.tfstate"

    region = "ap-south-1"

    encrypt = true

  }
}
```

Recommended:

```text
S3
 +
Versioning
 +
Encryption
 +
State locking
```

Do not store:

```text
terraform.tfstate
```

in GitHub.

---

# 10. Initialize Terraform

```bash
cd terraform
```

Initialize:

```bash
terraform init
```

Validate:

```bash
terraform validate
```

Format:

```bash
terraform fmt -recursive
```

---

# 11. Create Terraform Plan

```bash
terraform plan
```

For a production upgrade:

```bash
terraform plan \
  -out=eks-upgrade.tfplan
```

Review carefully.

You want to see changes related to:

```text
aws_eks_cluster
aws_eks_node_group
aws_eks_addon
```

You **do not** want Terraform to propose destroying the entire EKS cluster.

If you see:

```text
-/+ aws_eks_cluster
```

STOP.

Investigate the plan before applying.

---

# 12. Initial Cluster

Example:

```hcl
kubernetes_version = "1.35"

node_group_version = "1.35"
```

Apply:

```bash
terraform apply
```

Verify:

```bash
kubectl get nodes -o wide
```

Example:

```text
NAME       STATUS   VERSION
node-01    Ready    v1.35.x
node-02    Ready    v1.35.x
node-03    Ready    v1.35.x
```

---

# 13. Start the Upgrade

Change:

```hcl
kubernetes_version = "1.35"
```

to:

```hcl
kubernetes_version = "1.36"
```

And:

```hcl
node_group_version = "1.35"
```

to:

```hcl
node_group_version = "1.36"
```

Therefore:

```text
BEFORE

Control Plane = 1.35
Nodes          = 1.35


AFTER

Control Plane = 1.36
Nodes          = 1.36
```

EKS requires the node-group Kubernetes version not to be greater than the control-plane version.

---

# 14. Terraform Plan

```bash
terraform plan \
  -out=eks-upgrade.tfplan
```

Review:

```bash
terraform show \
  eks-upgrade.tfplan
```

Look for:

```text
version: 1.35 → 1.36
```

---

# 15. Apply Upgrade

```bash
terraform apply \
  eks-upgrade.tfplan
```

Terraform will submit the EKS changes.

The EKS control-plane update is asynchronous; AWS reports the cluster as `UPDATING` during the operation and `ACTIVE` when it completes.

---

# 16. Monitor Control Plane

```bash
aws eks describe-cluster \
  --name eks-zero-downtime \
  --region ap-south-1 \
  --query 'cluster.status'
```

Expected:

```text
UPDATING
```

then:

```text
ACTIVE
```

Check version:

```bash
aws eks describe-cluster \
  --name eks-zero-downtime \
  --region ap-south-1 \
  --query 'cluster.version'
```

Expected:

```text
"1.36"
```

---

# 17. Monitor Node Upgrade

```bash
watch kubectl get nodes -o wide
```

During the transition you may see:

```text
NAME       STATUS   VERSION
node-01    Ready    v1.35
node-02    Ready    v1.35
node-03    Ready    v1.36
```

Eventually:

```text
NAME       STATUS   VERSION
node-01    Ready    v1.36
node-02    Ready    v1.36
node-03    Ready    v1.36
```

---

# 18. Monitor Pods

```bash
watch kubectl get pods -A -o wide
```

Check Pending:

```bash
kubectl get pods -A | grep Pending
```

Check failures:

```bash
kubectl get pods -A |
grep -E "CrashLoopBackOff|ImagePullBackOff|Error"
```

Check events:

```bash
kubectl get events \
  -A \
  --sort-by=.lastTimestamp
```

---

# 19. Validate Application

```bash
kubectl get deployments -A
```

Expected:

```text
NAME       READY   UP-TO-DATE   AVAILABLE
frontend   3/3     3            3
backend    3/3     3            3
```

Check rollout:

```bash
kubectl rollout status \
  deployment/frontend \
  -n production
```

Test:

```bash
curl -f https://example.com/health
```

---

# 20. Terraform State Validation

After the upgrade:

```bash
terraform state list
```

Check:

```bash
terraform plan
```

The desired result is:

```text
No changes.
```

This is important.

It means:

```text
AWS Infrastructure
       =
Terraform Configuration
       =
Terraform State
```

---

# 21. Complete Upgrade Workflow

```text
                 GitHub
                    │
                    ▼
             Terraform Code
                    │
                    ▼
            Change Version
             1.35 → 1.36
                    │
                    ▼
           terraform validate
                    │
                    ▼
             terraform plan
                    │
                    ▼
               Review
                    │
                    ▼
             terraform apply
                    │
           ┌────────┴────────┐
           ▼                 ▼
     EKS Control        Managed Node
        Plane               Groups
     1.35 → 1.36        1.35 → 1.36
           │                 │
           └────────┬────────┘
                    ▼
               EKS Add-ons
                    │
                    ▼
             Application Test
                    │
                    ▼
               Monitoring
                    │
                    ▼
              Final Validation
```

---

# 🛡️ Zero-Downtime Design

The Terraform upgrade should be combined with Kubernetes workload protection:

```text
                   Production
                       │
           ┌───────────┼───────────┐
           ▼           ▼           ▼
       Replicas       PDB       Readiness
          >= 3                    Probe
           │           │           │
           └───────────┼───────────┘
                       ▼
                Spare Capacity
                       │
                       ▼
              Managed Node Group
                Rolling Update
                       │
                       ▼
                Zero-Downtime
                 Application
```

---

# 🚨 Important Terraform Rule

Do **not** use:

```hcl
lifecycle {
  ignore_changes = [
    version
  ]
}
```

if Terraform is intended to manage the Kubernetes version.

Otherwise Terraform will stop detecting version changes.

Instead:

```hcl
resource "aws_eks_cluster" "this" {

  name = var.cluster_name

  version = var.kubernetes_version

}
```

---

# 🚨 Do Not Force Node Updates

Avoid manually forcing node updates simply because a PDB blocks eviction.

For example, do not automatically resort to:

```text
force update
```

A forced EKS managed-node-group update can bypass PDB protection. AWS documents rolling updates as respecting PDBs and force updates as not respecting them.

Investigate:

```bash
kubectl get pdb -A
```

and:

```bash
kubectl describe pdb <pdb-name> \
  -n <namespace>
```

before proceeding.

---

# 📊 Prometheus / Grafana Monitoring

Monitor these metrics during the upgrade:

```text
Node CPU
Node Memory
Pod Restarts
Pod Availability
Pending Pods
HTTP 5xx
HTTP Latency
Request Rate
API Server Errors
Application Errors
```

Recommended dashboard:

```text
┌─────────────────────────────────────┐
│          EKS UPGRADE                 │
├─────────────────────────────────────┤
│ Nodes Ready             3 / 3       │
│ Pods Available          18 / 18     │
│ Pending Pods            0           │
│ HTTP 5xx                0           │
│ P99 Latency             120 ms      │
│ CPU                     42%         │
│ Memory                  58%         │
└─────────────────────────────────────┘
```

---

# 🔁 Terraform + Argo CD

Use different responsibilities:

```text
Terraform
   │
   ├── VPC
   ├── EKS
   ├── Node Groups
   ├── IAM
   └── EKS Add-ons
          │
          ▼
        EKS
          │
          ▼
       Argo CD
          │
          ├── Deployments
          ├── Services
          ├── Ingress
          ├── PDB
          ├── ConfigMaps
          └── Applications
```

Recommended separation:

```text
Terraform
= Infrastructure

Argo CD
= Kubernetes Applications
```

This avoids having two systems fight over application resources.

---

# 🔐 Production Upgrade Checklist

### Terraform

* [ ] Terraform state backed up
* [ ] Remote backend configured
* [ ] `terraform fmt`
* [ ] `terraform validate`
* [ ] `terraform plan`
* [ ] Review destroy/recreate operations
* [ ] Save upgrade plan

### EKS

* [ ] Current version verified
* [ ] Target version supported
* [ ] Deprecated APIs checked
* [ ] Control plane upgrade
* [ ] Node group upgrade
* [ ] Add-ons upgraded

### Workloads

* [ ] 3+ replicas for critical services
* [ ] Readiness probes
* [ ] Liveness probes
* [ ] PDB
* [ ] Pod topology/anti-affinity where appropriate
* [ ] Spare capacity

### Monitoring

* [ ] Prometheus
* [ ] Grafana
* [ ] Application health check
* [ ] HTTP error monitoring
* [ ] Node monitoring
* [ ] Pod monitoring
* [ ] Logs

### GitOps

* [ ] Argo CD synced
* [ ] Applications healthy
* [ ] No unexpected drift

### Final

* [ ] All nodes Ready
* [ ] All nodes target Kubernetes version
* [ ] No Pending pods
* [ ] No CrashLoopBackOff
* [ ] PDB healthy
* [ ] Services healthy
* [ ] Load Balancer healthy
* [ ] Application smoke test successful
* [ ] `terraform plan` shows no unexpected changes

---

# 🧪 Useful Commands

```bash
terraform init
```

```bash
terraform fmt -recursive
```

```bash
terraform validate
```

```bash
terraform plan
```

```bash
terraform apply
```

```bash
terraform show
```

```bash
terraform state list
```

Kubernetes:

```bash
kubectl get nodes -o wide
```

```bash
kubectl get pods -A
```

```bash
kubectl get pdb -A
```

```bash
kubectl get events -A \
  --sort-by=.lastTimestamp
```

AWS:

```bash
aws eks describe-cluster \
  --name $CLUSTER_NAME \
  --region $AWS_REGION
```

```bash
aws eks list-nodegroups \
  --cluster-name $CLUSTER_NAME \
  --region $AWS_REGION
```

```bash
aws eks list-addons \
  --cluster-name $CLUSTER_NAME \
  --region $AWS_REGION
```

---

# 🎯 Key Concept

The Terraform change is intentionally simple:

```diff
- kubernetes_version = "1.35"
+ kubernetes_version = "1.36"

- node_group_version = "1.35"
+ node_group_version = "1.36"
```

Then:

```bash
terraform plan
terraform apply
```

The important part is **not the version change itself**. The zero-downtime behavior comes from the complete design:

```text
Terraform
    +
EKS Managed Node Groups
    +
Multiple Replicas
    +
RollingUpdate
    +
PodDisruptionBudget
    +
Readiness Probes
    +
Spare Capacity
    +
Monitoring
    =
Production Upgrade Strategy
```

AWS recommends keeping the control plane and nodes on the same Kubernetes minor version after the upgrade.

---

# 👨‍💻 Author

## Venkata Ram Vanimina

**DevOps Engineer | AWS | Kubernetes | Terraform | Jenkins | Argo CD | Observability**

```text
Learn → Build → Automate → Monitor → Upgrade Safely
```

---

# ⭐ GitHub Topics

```text
aws
amazon-eks
eks
kubernetes
terraform
devops
aws-devops
kubernetes-upgrade
eks-upgrade
zero-downtime
terraform-aws
gitops
argocd
prometheus
grafana
cloud-engineering
```

---

# 📚 Official References

* [Amazon EKS — Update an existing cluster](https://docs.aws.amazon.com/eks/latest/userguide/update-cluster.html?utm_source=chatgpt.com)
* [Amazon EKS — Update managed node groups](https://docs.aws.amazon.com/eks/latest/userguide/update-managed-node-group.html?utm_source=chatgpt.com)
* [Terraform AWS Provider — EKS Node Group](https://registry.terraform.io/providers/hashicorp/aws/latest/docs/resources/eks_node_group.html?utm_source=chatgpt.com)
* [Terraform AWS Provider — EKS Cluster](https://registry.terraform.io/providers/hashicorp/aws/6.24.0/docs/resources/eks_cluster?utm_source=chatgpt.com)
* [EKS Kubernetes version lifecycle](https://docs.aws.amazon.com/eks/latest/userguide/kubernetes-versions.html?utm_source=chatgpt.com)
