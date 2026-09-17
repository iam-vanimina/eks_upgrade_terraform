# 🚀 Amazon EKS Kubernetes Cluster Upgrade — Zero Downtime

A production-style guide for upgrading an **Amazon EKS Kubernetes cluster** with minimal disruption and a zero-downtime application strategy.

This project demonstrates how to safely upgrade:

```text
EKS Control Plane
       ↓
Managed Node Groups
       ↓
EKS Add-ons
       ↓
Controllers
       ↓
Application Validation
```

The guide focuses on **rolling upgrades, PodDisruptionBudgets, readiness probes, workload replicas, capacity planning, monitoring, and GitOps validation**.

---

## 📌 Objectives

By completing this project, you will learn how to:

* Upgrade an Amazon EKS Kubernetes cluster
* Upgrade the EKS control plane
* Upgrade managed node groups
* Perform rolling node replacement
* Configure PodDisruptionBudgets
* Configure Kubernetes rolling deployments
* Use readiness and liveness probes
* Maintain application availability during node upgrades
* Validate EKS add-on compatibility
* Monitor workloads during an upgrade
* Detect deprecated Kubernetes APIs
* Validate applications after an upgrade
* Integrate EKS upgrades with Argo CD and GitOps

---

# 🏗️ Architecture

```text
                         Internet
                            │
                            ▼
                     ┌─────────────┐
                     │ AWS ALB/NLB │
                     └──────┬──────┘
                            │
                            ▼
                  ┌───────────────────┐
                  │   Amazon EKS      │
                  │                   │
                  │ Control Plane     │
                  │ 1.35 → 1.36       │
                  └─────────┬─────────┘
                            │
              ┌─────────────┴─────────────┐
              │                           │
              ▼                           ▼
       Managed Node 1              Managed Node 2
        1.35 → 1.36                1.35 → 1.36
              │                           │
              ▼                           ▼
         ┌─────────┐                 ┌─────────┐
         │ Pod A   │                 │ Pod B   │
         └─────────┘                 └─────────┘
              │                           │
              └─────────────┬─────────────┘
                            ▼
                     ┌─────────────┐
                     │ Monitoring  │
                     │ Prometheus  │
                     │ Grafana     │
                     └─────────────┘
```

---

# 🔄 Upgrade Strategy

The recommended sequence is:

```text
                ┌──────────────────┐
                │ Current EKS      │
                │ Kubernetes 1.35  │
                └────────┬─────────┘
                         │
                         ▼
              ┌────────────────────┐
              │ Pre-Upgrade Checks │
              └────────┬───────────┘
                       │
                       ▼
              ┌────────────────────┐
              │ Deprecated APIs    │
              │ Workload Health    │
              │ PDB / Probes       │
              │ Capacity           │
              └────────┬───────────┘
                       │
                       ▼
              ┌────────────────────┐
              │ Upgrade Control    │
              │ Plane 1.35 → 1.36  │
              └────────┬───────────┘
                       │
                       ▼
              ┌────────────────────┐
              │ Upgrade Node Group │
              │ Rolling Upgrade    │
              └────────┬───────────┘
                       │
                       ▼
              ┌────────────────────┐
              │ Upgrade EKS        │
              │ Add-ons            │
              └────────┬───────────┘
                       │
                       ▼
              ┌────────────────────┐
              │ Application Tests  │
              └────────┬───────────┘
                       │
                       ▼
                ┌──────────────┐
                │ EKS 1.36     │
                │ Production   │
                └──────────────┘
```

---

# 📁 Repository Structure

```text
eks-zero-downtime-upgrade/
│
├── README.md
│
├── docs/
│   ├── architecture.md
│   ├── pre-upgrade-checklist.md
│   ├── upgrade-runbook.md
│   ├── rollback-plan.md
│   └── troubleshooting.md
│
├── kubernetes/
│   ├── deployment.yaml
│   ├── service.yaml
│   ├── pdb.yaml
│   ├── hpa.yaml
│   ├── namespace.yaml
│   └── probes.yaml
│
├── scripts/
│   ├── pre-upgrade-check.sh
│   ├── upgrade-control-plane.sh
│   ├── upgrade-nodegroup.sh
│   ├── post-upgrade-check.sh
│   └── health-check.sh
│
├── terraform/
│   ├── main.tf
│   ├── variables.tf
│   ├── outputs.tf
│   ├── providers.tf
│   └── eks.tf
│
└── monitoring/
    ├── prometheus-rules.yaml
    └── grafana-dashboard.json
```

---

# 🛠️ Prerequisites

Install:

* AWS CLI
* kubectl
* eksctl
* Terraform
* Helm
* Git

Verify:

```bash
aws --version
kubectl version --client
eksctl version
terraform version
helm version
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

# 🔍 1. Check Current EKS Version

Set environment variables:

```bash
export AWS_REGION=ap-south-1
export CLUSTER_NAME=my-eks-cluster
export TARGET_VERSION=1.36
```

Check cluster version:

```bash
aws eks describe-cluster \
  --name $CLUSTER_NAME \
  --region $AWS_REGION \
  --query 'cluster.version' \
  --output text
```

Check nodes:

```bash
kubectl get nodes -o wide
```

Example:

```text
NAME            STATUS   VERSION
ip-10-0-1-101   Ready    v1.35.x
ip-10-0-2-102   Ready    v1.35.x
ip-10-0-3-103   Ready    v1.35.x
```

---

# 🔎 2. Pre-Upgrade Health Check

Check all pods:

```bash
kubectl get pods -A
```

Check unhealthy workloads:

```bash
kubectl get pods -A \
  --field-selector=status.phase!=Running,status.phase!=Succeeded
```

Check deployments:

```bash
kubectl get deployments -A
```

Check StatefulSets:

```bash
kubectl get statefulsets -A
```

Check DaemonSets:

```bash
kubectl get daemonsets -A
```

Check events:

```bash
kubectl get events -A \
  --sort-by=.lastTimestamp
```

---

# 🧪 3. Check Deprecated APIs

Before upgrading, identify APIs that may be removed in the target Kubernetes version.

Use EKS Upgrade Insights:

```bash
aws eks describe-insight-findings \
  --cluster-name $CLUSTER_NAME \
  --region $AWS_REGION
```

Look for resources using deprecated APIs.

Examples:

```text
extensions/v1beta1
apps/v1beta1
batch/v1beta1
policy/v1beta1
```

Update manifests and controllers before the cluster upgrade.

---

# 📦 4. Check Application Replicas

```bash
kubectl get deployment -A
```

Example:

```text
NAME       READY   UP-TO-DATE   AVAILABLE
frontend   3/3     3            3
backend    3/3     3            3
```

For production workloads, use multiple replicas.

Example:

```yaml
spec:
  replicas: 3
```

Avoid relying on:

```yaml
replicas: 1
```

for critical services if zero downtime is required.

---

# 🔄 5. Configure Rolling Updates

Example Deployment:

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
        image: nginx:latest

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

Apply:

```bash
kubectl apply -f kubernetes/deployment.yaml
```

Verify:

```bash
kubectl rollout status \
  deployment/frontend \
  -n production
```

---

# 🛡️ 6. Configure PodDisruptionBudget

Create:

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

Apply:

```bash
kubectl apply -f kubernetes/pdb.yaml
```

Check:

```bash
kubectl get pdb -n production
```

Expected:

```text
NAME           MIN AVAILABLE   ALLOWED DISRUPTIONS
frontend-pdb   2               1
```

---

# 📊 7. Check Node Capacity

```bash
kubectl top nodes
```

Example:

```text
NAME       CPU    MEMORY
node-01    35%    42%
node-02    31%    38%
node-03    28%    40%
```

Check detailed resources:

```bash
kubectl describe nodes
```

Make sure the cluster has enough spare capacity to schedule replacement pods.

---

# 📈 8. Increase Managed Node Group Capacity

List node groups:

```bash
aws eks list-nodegroups \
  --cluster-name $CLUSTER_NAME \
  --region $AWS_REGION
```

Example:

```text
frontend-ng
backend-ng
```

Check:

```bash
aws eks describe-nodegroup \
  --cluster-name $CLUSTER_NAME \
  --nodegroup-name frontend-ng \
  --region $AWS_REGION
```

Temporarily increase capacity:

```bash
aws eks update-nodegroup-config \
  --cluster-name $CLUSTER_NAME \
  --nodegroup-name frontend-ng \
  --scaling-config \
    minSize=3,maxSize=6,desiredSize=4 \
  --region $AWS_REGION
```

This provides additional capacity during the rolling upgrade.

---

# 🚀 9. Upgrade EKS Control Plane

Upgrade one Kubernetes minor version at a time.

Example:

```text
1.35 → 1.36
```

Using AWS CLI:

```bash
aws eks update-cluster-version \
  --name $CLUSTER_NAME \
  --kubernetes-version $TARGET_VERSION \
  --region $AWS_REGION
```

Monitor:

```bash
aws eks describe-cluster \
  --name $CLUSTER_NAME \
  --region $AWS_REGION \
  --query 'cluster.status'
```

Wait for:

```text
ACTIVE
```

Verify:

```bash
aws eks describe-cluster \
  --name $CLUSTER_NAME \
  --region $AWS_REGION \
  --query 'cluster.version'
```

---

# 🖥️ 10. Upgrade Managed Node Group

List node groups:

```bash
aws eks list-nodegroups \
  --cluster-name $CLUSTER_NAME \
  --region $AWS_REGION
```

Upgrade:

```bash
aws eks update-nodegroup-version \
  --cluster-name $CLUSTER_NAME \
  --nodegroup-name frontend-ng \
  --kubernetes-version $TARGET_VERSION \
  --region $AWS_REGION
```

Monitor:

```bash
aws eks describe-nodegroup \
  --cluster-name $CLUSTER_NAME \
  --nodegroup-name frontend-ng \
  --region $AWS_REGION \
  --query 'nodegroup.status'
```

Expected:

```text
ACTIVE
```

---

# 🔄 11. What Happens During the Node Upgrade?

Conceptually:

```text
BEFORE

Node 1        Node 2        Node 3
  │             │             │
 Pod A         Pod B         Pod C


             ↓


CORDON NODE 1

Node 1        Node 2        Node 3
  X             │             │
                │             │


             ↓


DRAIN NODE 1

Pod A
  │
  └──────────────► New Node


             ↓


AFTER

Node 2        Node 3        New Node
  │             │             │
 Pod B         Pod C         Pod A
```

The process repeats for each node.

---

# 👀 12. Monitor Nodes

Use:

```bash
watch kubectl get nodes
```

Expected transition:

```text
v1.35
  ↓
cordon
  ↓
drain
  ↓
replacement
  ↓
v1.36
  ↓
Ready
```

---

# 👀 13. Monitor Pods

```bash
watch kubectl get pods -A -o wide
```

Check Pending pods:

```bash
kubectl get pods -A | grep Pending
```

If a pod is Pending:

```bash
kubectl describe pod <pod-name> \
  -n <namespace>
```

Common causes:

```text
Insufficient CPU
Insufficient memory
Node affinity
Pod anti-affinity
Taints
Tolerations
PVC topology
```

---

# 📡 14. Monitor Application Availability

Run:

```bash
kubectl get deployments -A
```

Example:

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

Check Services:

```bash
kubectl get svc -A
```

Check EndpointSlices:

```bash
kubectl get endpointslices -A
```

---

# ❤️ 15. External Health Test

For an application exposed through an ALB/NLB:

```bash
curl -I https://example.com
```

Continuous test:

```bash
while true; do
  curl -sk \
    -o /dev/null \
    -w "%{http_code}\n" \
    https://example.com

  sleep 1
done
```

Expected:

```text
200
200
200
200
200
...
```

Investigate any unexpected:

```text
503
502
504
```

---

# 🧩 16. Upgrade EKS Add-ons

List add-ons:

```bash
aws eks list-addons \
  --cluster-name $CLUSTER_NAME \
  --region $AWS_REGION
```

Typical add-ons:

```text
vpc-cni
coredns
kube-proxy
aws-ebs-csi-driver
```

Check an add-on:

```bash
aws eks describe-addon \
  --cluster-name $CLUSTER_NAME \
  --addon-name vpc-cni \
  --region $AWS_REGION
```

Upgrade each add-on to a version compatible with the target Kubernetes version.

---

# 🔧 17. Check Controllers

Verify:

```bash
kubectl get deployments -A
```

Important components include:

```text
AWS Load Balancer Controller
ExternalDNS
Cluster Autoscaler
Karpenter
Metrics Server
EBS CSI Driver
EFS CSI Driver
Argo CD
Prometheus
Grafana
cert-manager
Istio
OpenTelemetry
```

Check:

```bash
kubectl get pods -A
```

Make sure controllers are:

```text
Running
Ready
```

---

# 📊 18. Monitoring During Upgrade

Recommended monitoring stack:

```text
                    EKS
                     │
          ┌──────────┴──────────┐
          │                     │
       Metrics                 Logs
          │                     │
          ▼                     ▼
     Prometheus                Loki
          │                     │
          └──────────┬──────────┘
                     │
                     ▼
                  Grafana
```

Monitor:

* CPU
* Memory
* Pod restarts
* HTTP error rate
* HTTP latency
* Node availability
* Pod availability
* API server errors
* Application errors

Useful commands:

```bash
kubectl get events -A \
  --sort-by=.lastTimestamp
```

```bash
kubectl top nodes
```

```bash
kubectl top pods -A
```

---

# 🔐 19. GitOps / Argo CD Validation

If Argo CD manages the applications:

```bash
kubectl get applications \
  -n argocd
```

Check:

```bash
argocd app list
```

Verify:

```text
SYNC = Synced
HEALTH = Healthy
```

After the upgrade:

```bash
argocd app get <application>
```

The Kubernetes infrastructure version changes should not unexpectedly modify application manifests managed by GitOps.

---

# 🧪 20. Post-Upgrade Validation

Run:

```bash
kubectl get nodes -o wide
```

Expected:

```text
NAME       STATUS   VERSION
node-01    Ready    v1.36.x
node-02    Ready    v1.36.x
node-03    Ready    v1.36.x
```

Check pods:

```bash
kubectl get pods -A
```

Deployments:

```bash
kubectl get deployments -A
```

StatefulSets:

```bash
kubectl get statefulsets -A
```

DaemonSets:

```bash
kubectl get daemonsets -A
```

PDB:

```bash
kubectl get pdb -A
```

Events:

```bash
kubectl get events -A \
  --sort-by=.lastTimestamp
```

---

# 🧪 21. Application Smoke Test

Example:

```bash
curl -f https://example.com/health
```

API:

```bash
curl -f https://example.com/api/health
```

Expected:

```json
{
  "status": "UP"
}
```

Test:

```text
DNS
 ↓
Load Balancer
 ↓
Ingress
 ↓
Service
 ↓
Pod
 ↓
Application
 ↓
Database
```

---

# ↩️ 22. Rollback Strategy

## Important

An EKS Kubernetes control-plane upgrade is **not a normal application rollback**.

Therefore, prepare recovery before upgrading.

Recommended strategy:

```text
Application rollback
        +
Node-group recovery
        +
Data recovery
        +
Infrastructure recovery
```

Keep:

* Terraform state
* Kubernetes manifests
* Helm values
* Git commits
* Database backups
* EBS snapshots where applicable
* Application images
* Previous node-group configuration

For application changes:

```bash
kubectl rollout history deployment/frontend \
  -n production
```

Rollback application version:

```bash
kubectl rollout undo deployment/frontend \
  -n production
```

Verify:

```bash
kubectl rollout status deployment/frontend \
  -n production
```

---

# 🚨 23. Troubleshooting

## Pods stuck Pending

```bash
kubectl describe pod <pod> \
  -n <namespace>
```

Check:

```text
CPU
Memory
Affinity
Taints
Tolerations
PVC
Topology
```

---

## PDB blocks node drain

```bash
kubectl get pdb -A
```

Check:

```bash
kubectl describe pdb <pdb-name> \
  -n <namespace>
```

Do not immediately delete the PDB.

Instead verify:

```text
Application replicas
Available pods
Node capacity
Readiness probes
```

---

## Node NotReady

```bash
kubectl describe node <node-name>
```

Check:

```bash
kubectl get events \
  --sort-by=.lastTimestamp
```

---

## Pods CrashLoopBackOff

```bash
kubectl logs <pod-name> \
  -n <namespace>
```

Previous container:

```bash
kubectl logs <pod-name> \
  -n <namespace> \
  --previous
```

---

## Service Returns 503

Check:

```bash
kubectl get pods -n <namespace>
```

Then:

```bash
kubectl get svc -n <namespace>
```

Then:

```bash
kubectl get endpointslices \
  -n <namespace>
```

Verify readiness probes.

---

# ⚠️ Zero-Downtime Requirements

A zero-downtime EKS upgrade is not guaranteed simply because EKS performs a rolling update.

Your application should satisfy:

```text
                    Zero Downtime
                         │
       ┌─────────────────┼─────────────────┐
       │                 │                 │
       ▼                 ▼                 ▼
 Multiple            Readiness           PDB
 Replicas             Probes
       │                 │                 │
       └─────────────────┼─────────────────┘
                         │
                         ▼
                 Spare Capacity
                         │
                         ▼
                Rolling Node Update
```

Recommended:

```yaml
replicas: 3

strategy:
  rollingUpdate:
    maxUnavailable: 0
    maxSurge: 1
```

And:

```yaml
readinessProbe:
```

plus:

```yaml
PodDisruptionBudget
```

---

# 📋 Production Upgrade Checklist

## Pre-Upgrade

* [ ] Confirm current Kubernetes version
* [ ] Confirm target Kubernetes version
* [ ] Review EKS Upgrade Insights
* [ ] Check deprecated APIs
* [ ] Check application replicas
* [ ] Check readiness probes
* [ ] Check liveness probes
* [ ] Check PDBs
* [ ] Check node capacity
* [ ] Check node groups
* [ ] Check EKS add-ons
* [ ] Check controllers
* [ ] Verify backups
* [ ] Verify monitoring
* [ ] Verify application health

## Control Plane

* [ ] Upgrade one minor version
* [ ] Monitor EKS update
* [ ] Confirm ACTIVE
* [ ] Verify Kubernetes API

## Node Groups

* [ ] Increase capacity if required
* [ ] Upgrade managed node group
* [ ] Monitor node replacement
* [ ] Monitor pod scheduling
* [ ] Verify PDB behavior
* [ ] Verify application availability

## Add-ons

* [ ] VPC CNI
* [ ] CoreDNS
* [ ] kube-proxy
* [ ] EBS CSI
* [ ] EFS CSI if used
* [ ] AWS Load Balancer Controller
* [ ] Metrics Server
* [ ] Other controllers

## Post-Upgrade

* [ ] All nodes Ready
* [ ] All nodes target version
* [ ] All pods healthy
* [ ] No Pending pods
* [ ] No CrashLoopBackOff
* [ ] PDB healthy
* [ ] Services healthy
* [ ] Load Balancer healthy
* [ ] DNS working
* [ ] Application smoke tests pass
* [ ] Argo CD Synced/Healthy
* [ ] Prometheus healthy
* [ ] Grafana healthy
* [ ] Logs available
* [ ] Alerts normal

---

# 🤖 Automated Pre-Upgrade Check

Create:

```text
scripts/pre-upgrade-check.sh
```

Example:

```bash
#!/bin/bash

set -e

echo "======================================"
echo " EKS PRE-UPGRADE HEALTH CHECK"
echo "======================================"

echo
echo "1. Kubernetes Version"
kubectl version

echo
echo "2. Nodes"
kubectl get nodes -o wide

echo
echo "3. Pods"
kubectl get pods -A

echo
echo "4. Deployments"
kubectl get deployments -A

echo
echo "5. StatefulSets"
kubectl get statefulsets -A

echo
echo "6. DaemonSets"
kubectl get daemonsets -A

echo
echo "7. PDB"
kubectl get pdb -A

echo
echo "8. Pending Pods"
kubectl get pods -A | grep Pending || true

echo
echo "9. Unhealthy Pods"
kubectl get pods -A \
  | grep -E "CrashLoopBackOff|ImagePullBackOff|Error" \
  || true

echo
echo "10. Recent Events"
kubectl get events -A \
  --sort-by=.lastTimestamp \
  | tail -30

echo
echo "======================================"
echo " PRE-UPGRADE CHECK COMPLETE"
echo "======================================"
```

Make executable:

```bash
chmod +x scripts/pre-upgrade-check.sh
```

Run:

```bash
./scripts/pre-upgrade-check.sh
```

---

# 🤖 Post-Upgrade Check

Create:

```text
scripts/post-upgrade-check.sh
```

```bash
#!/bin/bash

set -e

echo "======================================"
echo " EKS POST-UPGRADE VALIDATION"
echo "======================================"

echo
echo "Nodes:"
kubectl get nodes -o wide

echo
echo "Pods:"
kubectl get pods -A

echo
echo "Deployments:"
kubectl get deployments -A

echo
echo "StatefulSets:"
kubectl get statefulsets -A

echo
echo "DaemonSets:"
kubectl get daemonsets -A

echo
echo "PDB:"
kubectl get pdb -A

echo
echo "Recent Events:"
kubectl get events -A \
  --sort-by=.lastTimestamp \
  | tail -30

echo
echo "Pending Pods:"
kubectl get pods -A | grep Pending || true

echo
echo "Problem Pods:"
kubectl get pods -A \
  | grep -E "CrashLoopBackOff|ImagePullBackOff|Error" \
  || true

echo
echo "======================================"
echo " POST-UPGRADE VALIDATION COMPLETE"
echo "======================================"
```

---

# 📚 Useful Kubernetes Commands

Get nodes:

```bash
kubectl get nodes
```

Get pods:

```bash
kubectl get pods -A
```

Get services:

```bash
kubectl get svc -A
```

Get deployments:

```bash
kubectl get deploy -A
```

Get PDB:

```bash
kubectl get pdb -A
```

Get events:

```bash
kubectl get events -A \
  --sort-by=.lastTimestamp
```

Resource usage:

```bash
kubectl top nodes
```

```bash
kubectl top pods -A
```

Node details:

```bash
kubectl describe node <node>
```

Pod details:

```bash
kubectl describe pod <pod> \
  -n <namespace>
```

---

# 🔗 Useful AWS Commands

Cluster information:

```bash
aws eks describe-cluster \
  --name $CLUSTER_NAME \
  --region $AWS_REGION
```

Node groups:

```bash
aws eks list-nodegroups \
  --cluster-name $CLUSTER_NAME \
  --region $AWS_REGION
```

Add-ons:

```bash
aws eks list-addons \
  --cluster-name $CLUSTER_NAME \
  --region $AWS_REGION
```

Cluster updates:

```bash
aws eks list-updates \
  --name $CLUSTER_NAME \
  --region $AWS_REGION
```

---

# 🧠 Key Production Lessons

### 1. Upgrade one minor version at a time

```text
1.34 → 1.35 → 1.36
```

Avoid jumping multiple unsupported minor versions.

### 2. Control plane first

```text
Control Plane
      ↓
Node Groups
      ↓
Add-ons
      ↓
Controllers
      ↓
Applications
```

### 3. PDB protects workloads

```text
3 replicas
     ↓
PDB minAvailable=2
     ↓
Only controlled disruption
```

### 4. Readiness probes prevent bad pods from receiving traffic

```text
Pod starts
   ↓
Readiness check
   ↓
Healthy?
  / \
 NO  YES
 │    │
No    Receive
traffic traffic
```

### 5. Spare capacity matters

If every node is already full, Kubernetes may not have enough room to reschedule workloads during node replacement.

---

# 🏁 Final State

The target production state is:

```text
                    AWS
                     │
                     ▼
                Amazon EKS
                     │
          ┌──────────┴──────────┐
          │                     │
          ▼                     ▼
   Control Plane          Managed Nodes
      1.36                  1.36
                              │
             ┌────────────────┼────────────────┐
             ▼                ▼                ▼
          Frontend          Backend          Worker
          Pod × 3           Pod × 3           Pod × N
             │                │                │
             └────────────────┼────────────────┘
                              │
                              ▼
                       Observability
                              │
                   ┌──────────┴──────────┐
                   ▼                     ▼
               Prometheus              Grafana
```

Application traffic continues while nodes are upgraded because workloads are distributed across multiple healthy nodes and protected by Kubernetes scheduling, readiness probes, rolling-update configuration, and PDBs.

---

# ⭐ Recommended GitHub Topics

```text
aws
amazon-eks
eks
kubernetes
k8s
devops
cloud
aws-devops
kubernetes-upgrade
eks-upgrade
zero-downtime
terraform
gitops
argocd
prometheus
grafana
```

---

# 📖 References

* AWS EKS documentation
* Kubernetes documentation
* EKS managed node group documentation
* Kubernetes PodDisruptionBudget documentation
* Kubernetes Deployment rolling-update documentation

---

## 👨‍💻 Author

**Venkata Ram Vanimina**

DevOps Engineer | AWS | Kubernetes | Terraform | Jenkins | Argo CD | Observability

---

## ⭐ If this project helped you

Consider giving the repository a ⭐ and sharing it with other DevOps and Kubernetes engineers.

```text
Learn → Build → Automate → Monitor → Upgrade Safely
```
