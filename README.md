# Deploying 2048 on Amazon EKS with AWS Fargate & ALB

A hands-on Kubernetes and AWS EKS project where I deployed the **2048 web application** on **Amazon EKS using AWS Fargate**, exposed it through a Kubernetes Service, and configured an **AWS Application Load Balancer (ALB)** using the **AWS Load Balancer Controller**.

This project focuses not only on deployment, but also on understanding **how Kubernetes networking, Services, Fargate, IAM/IRSA, TargetGroups, ALB health checks, and troubleshooting work together**.

---

## Project Overview

The objective of this project was to:

* Create an Amazon EKS cluster
* Run Kubernetes workloads on AWS Fargate
* Create a dedicated namespace for the application
* Deploy the 2048 application using Kubernetes Deployment
* Expose the application using a Kubernetes Service
* Configure IAM OIDC / IRSA
* Install AWS Load Balancer Controller using Helm
* Create an internet-facing Application Load Balancer
* Configure ALB target registration using Pod IPs
* Troubleshoot and resolve an ALB `502 Bad Gateway`
* Verify connectivity from:

  * Pod → Service
  * Service → Application
  * ALB → Pod
  * Browser → ALB

---

# Architecture

```text
                         Internet
                            │
                            │ HTTP :80
                            ▼
              ┌─────────────────────────────┐
              │     AWS Application LB      │
              │      Internet-facing        │
              └──────────────┬──────────────┘
                             │
                             │ HTTP :80
                             ▼
              ┌─────────────────────────────┐
              │ Kubernetes Ingress          │
              │ ingressClass: alb           │
              └──────────────┬──────────────┘
                             │
                             ▼
              ┌─────────────────────────────┐
              │ TargetGroupBinding          │
              │ Target Type: IP             │
              └──────────────┬──────────────┘
                             │
                ┌────────────┼────────────┐
                │            │            │
                ▼            ▼            ▼
          ┌──────────┐ ┌──────────┐ ┌──────────┐
          │ 2048 Pod │ │ 2048 Pod │ │ 2048 Pod │
          │  :80     │ │  :80     │ │  :80     │
          └──────────┘ └──────────┘ └──────────┘
                │            │            │
                └────────────┼────────────┘
                             │
                    Amazon EKS / Fargate
```

### AWS/Kubernetes components

```text
AWS
│
├── VPC
│   ├── Subnet - ap-south-1a
│   ├── Subnet - ap-south-1b
│   └── Subnet - ap-south-1c
│
└── EKS Cluster
    │
    ├── Fargate
    │   ├── kube-system workloads
    │   └── game-2048 workloads
    │
    ├── AWS Load Balancer Controller
    │
    └── Kubernetes
        ├── Namespace
        ├── Deployment
        ├── Pods
        ├── Service
        └── Ingress
```

---

# Technologies Used

| Technology                   | Purpose                                       |
| ---------------------------- | --------------------------------------------- |
| AWS EKS                      | Managed Kubernetes control plane              |
| AWS Fargate                  | Serverless compute for Kubernetes Pods        |
| Kubernetes                   | Container orchestration                       |
| AWS ALB                      | Internet-facing application load balancer     |
| AWS Load Balancer Controller | Creates/manages ALB from Kubernetes resources |
| Helm                         | Deploying the AWS Load Balancer Controller    |
| IAM                          | AWS permissions                               |
| IAM OIDC                     | Connect Kubernetes ServiceAccount with IAM    |
| IRSA                         | IAM Roles for Service Accounts                |
| kubectl                      | Kubernetes management                         |
| eksctl                       | EKS cluster/Fargate management                |
| AWS CLI                      | AWS resource management                       |
| curl                         | Connectivity testing                          |

---

# Repository Structure

```text
eks-2048-fargate-alb/
│
├── README.md
│
├── deployment.yml
├── service.yml
├── ingress.yml
│
├── iam-policy.json
│
└── docs/
    ├── architecture.md
    ├── troubleshooting.md
    └── interview-preparation.md
```

---

# Prerequisites

Install:

```bash
aws
kubectl
eksctl
helm
```

Verify:

```bash
aws --version
kubectl version --client
eksctl version
helm version
```

Verify AWS credentials:

```bash
aws sts get-caller-identity
```

---

# Create the EKS Cluster

The cluster was created in:

```text
Region: ap-south-1
Cluster: eks-devops-2048
```

Example:

```bash
eksctl create cluster \
  --name eks-devops-2048 \
  --region ap-south-1 \
  --fargate
```

Verify:

```bash
eksctl get cluster
```

Expected:

```text
NAME             REGION       EKSCTL CREATED
eks-devops-2048  ap-south-1   True
```

---

# Verify Kubernetes Access

Check the current context:

```bash
kubectl config current-context
```

Check nodes:

```bash
kubectl get nodes
```

The cluster uses Fargate, so the nodes appear as Fargate-backed nodes.

Example:

```text
fargate-ip-192-168-xxx-xxx.ap-south-1.compute.internal
```

Check namespaces:

```bash
kubectl get namespaces
```

Check system Pods:

```bash
kubectl get pods -A
```

---

# Create a Dedicated Fargate Profile

The application was deployed into:

```text
game-2048
```

Create the namespace-specific Fargate profile:

```bash
eksctl create fargateprofile \
  --cluster eks-devops-2048 \
  --region ap-south-1 \
  --name fp-game-2048 \
  --namespace game-2048
```

Verify:

```bash
eksctl get fargateprofile \
  --cluster eks-devops-2048 \
  --region ap-south-1
```

Expected:

```text
fp-game-2048    game-2048    ACTIVE
```

### Why?

Fargate profiles tell EKS:

> "Pods matching this namespace should run on Fargate."

Therefore, when a Pod is created in `game-2048`, EKS can schedule it onto Fargate.

---

# Create the Application Namespace

```bash
kubectl create namespace game-2048
```

Verify:

```bash
kubectl get namespaces
```

---

# Deploy the 2048 Application

The application Deployment runs three replicas.

Example:

```yaml
apiVersion: apps/v1
kind: Deployment

metadata:
  name: app-2048
  namespace: game-2048

spec:
  replicas: 3

  selector:
    matchLabels:
      app.kubernetes.io/name: app-2048

  template:
    metadata:
      labels:
        app.kubernetes.io/name: app-2048

    spec:
      containers:
        - name: app-2048
          image: public.ecr.aws/l6m2t8p7/docker-2048:latest
          ports:
            - containerPort: 80
```

Apply:

```bash
kubectl apply -f deployment.yml
```

Verify:

```bash
kubectl get deployments -n game-2048
```

```bash
kubectl get pods -n game-2048
```

Expected:

```text
app-2048-xxxxx   1/1   Running
app-2048-xxxxx   1/1   Running
app-2048-xxxxx   1/1   Running
```

---

# Kubernetes Deployment vs Container Port

The Deployment contains:

```yaml
containerPort: 80
```

This describes the port the application container is expected to use.

However, Kubernetes does **not automatically redirect traffic** based on `containerPort`.

The actual traffic path is controlled by the Service.

---

# Create the Kubernetes Service

The Service exposes the Pods internally.

Example:

```yaml
apiVersion: v1
kind: Service

metadata:
  name: app-2048
  namespace: game-2048

spec:
  selector:
    app.kubernetes.io/name: app-2048

  ports:
    - port: 80
      targetPort: 80

  type: NodePort
```

Apply:

```bash
kubectl apply -f service.yml
```

Verify:

```bash
kubectl get svc -n game-2048
```

---

# Understanding the Ports

This was one of the most important troubleshooting lessons in the project.

```text
Client
  │
  │ Service port 80
  ▼
Service :80
  │
  │ targetPort 80
  ▼
Pod :80
```

### `port`

The port exposed by the Kubernetes Service.

```yaml
port: 80
```

### `targetPort`

The port where the Service sends traffic inside the selected Pod.

```yaml
targetPort: 80
```

### `containerPort`

Declared in the container specification:

```yaml
containerPort: 80
```

It documents the expected application port but does not itself create the networking connection.

---

# Verify Service Endpoints

```bash
kubectl get endpoints -n game-2048
```

On newer Kubernetes versions, EndpointSlice is preferred:

```bash
kubectl get endpointslices -n game-2048
```

The Service should point to the Pod IPs.

Example:

```text
192.168.xxx.xxx:80
192.168.xxx.xxx:80
192.168.xxx.xxx:80
```

---

# Test Service Connectivity

Run a temporary curl Pod:

```bash
kubectl run test-curl \
  -n game-2048 \
  --image=curlimages/curl \
  --rm -it \
  --restart=Never \
  -- curl -v http://app-2048.game-2048.svc.cluster.local:80/
```

Expected:

```text
HTTP/1.1 200 OK
Server: nginx
```

This confirms:

```text
Pod
 ↓
Kubernetes DNS
 ↓
Service
 ↓
Target Pod
 ↓
Nginx
 ↓
HTTP 200
```

---

# Configure IAM OIDC

The AWS Load Balancer Controller needs permission to interact with AWS APIs.

Associate the cluster with an IAM OIDC provider:

```bash
eksctl utils associate-iam-oidc-provider \
  --cluster eks-devops-2048 \
  --region ap-south-1 \
  --approve
```

Verify:

```bash
aws eks describe-cluster \
  --name eks-devops-2048 \
  --region ap-south-1 \
  --query "cluster.identity.oidc.issuer" \
  --output text
```

---

# AWS Load Balancer Controller IAM Policy

Download the controller IAM policy:

```bash
curl -o iam-policy.json \
https://raw.githubusercontent.com/kubernetes-sigs/aws-load-balancer-controller/v2.13.4/docs/install/iam_policy.json
```

Create the IAM policy if it does not already exist:

```bash
aws iam create-policy \
  --policy-name AWSLoadBalancerControllerIAMPolicy \
  --policy-document file://iam-policy.json
```

If AWS returns:

```text
EntityAlreadyExists
```

the policy already exists.

Verify:

```bash
aws iam get-policy \
  --policy-arn arn:aws:iam::<ACCOUNT_ID>:policy/AWSLoadBalancerControllerIAMPolicy
```

---

# IAM Role + ServiceAccount

The AWS Load Balancer Controller uses a Kubernetes ServiceAccount:

```text
aws-load-balancer-controller
```

The ServiceAccount contains an IAM role annotation:

```yaml
eks.amazonaws.com/role-arn:
  arn:aws:iam::<ACCOUNT_ID>:role/<ROLE>
```

This is the basis of **IRSA — IAM Roles for Service Accounts**.

The trust policy allows:

```text
Kubernetes ServiceAccount
        ↓
OIDC Provider
        ↓
STS
        ↓
IAM Role
        ↓
AWS API permissions
```

Verify:

```bash
kubectl get serviceaccount \
  aws-load-balancer-controller \
  -n kube-system \
  -o yaml
```

---

# Install AWS Load Balancer Controller Using Helm

Add the EKS Helm repository:

```bash
helm repo add eks https://aws.github.io/eks-charts
```

Update:

```bash
helm repo update
```

Search:

```bash
helm search repo eks/aws-load-balancer-controller
```

Install:

```bash
helm upgrade --install aws-load-balancer-controller \
  eks/aws-load-balancer-controller \
  -n kube-system \
  --set clusterName=eks-devops-2048 \
  --set region=ap-south-1 \
  --set serviceAccount.create=false \
  --set serviceAccount.name=aws-load-balancer-controller
```

Verify:

```bash
helm list -n kube-system
```

Then:

```bash
kubectl get pods -n kube-system \
  -l app.kubernetes.io/name=aws-load-balancer-controller
```

Expected:

```text
1/1 Running
1/1 Running
```

---

# Important Troubleshooting: Controller CrashLoopBackOff

During this project, the AWS Load Balancer Controller initially entered:

```text
CrashLoopBackOff
```

The logs showed:

```text
failed to get VPC ID:
failed to fetch VPC ID from instance metadata
```

This was particularly important because the controller was running on **Fargate**.

The controller attempted to obtain the VPC ID through EC2 instance metadata.

The final working deployment used the cluster/region configuration so the controller could operate correctly without depending on EC2 metadata.

Check logs with:

```bash
kubectl logs \
  <controller-pod> \
  -n kube-system \
  --previous
```

Always check logs before changing random configuration.

---

# Create the ALB Ingress

The Ingress tells the AWS Load Balancer Controller to create an AWS Application Load Balancer.

Example:

```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress

metadata:
  name: app-2048
  namespace: game-2048

  annotations:
    alb.ingress.kubernetes.io/scheme: internet-facing
    alb.ingress.kubernetes.io/target-type: ip

spec:
  ingressClassName: alb

  rules:
    - http:
        paths:
          - path: /
            pathType: Prefix

            backend:
              service:
                name: app-2048
                port:
                  number: 80
```

Apply:

```bash
kubectl apply -f ingress.yml
```

---

# Verify the Ingress

```bash
kubectl get ingress -n game-2048
```

Expected:

```text
NAME       CLASS   HOSTS   ADDRESS
app-2048   alb     *       k8s-game2048-....elb.amazonaws.com
```

Describe it:

```bash
kubectl describe ingress app-2048 -n game-2048
```

Look for:

```text
SuccessfullyReconciled
```

---

# TargetGroupBinding

The AWS Load Balancer Controller creates a TargetGroupBinding.

Verify:

```bash
kubectl get targetgroupbindings -n game-2048
```

Example:

```text
NAME
k8s-game2048-app2048-xxxxxxxx
```

This connects:

```text
AWS Target Group
        ↕
Kubernetes Service
```

Because the Ingress uses:

```yaml
alb.ingress.kubernetes.io/target-type: ip
```

the ALB target group contains the **Pod IP addresses**.

Example:

```text
192.168.140.58
192.168.161.89
192.168.167.222
```

---

# Major Troubleshooting: 502 Bad Gateway

Initially, accessing the ALB returned:

```text
502 Bad Gateway
```

Instead of assuming the ALB itself was broken, the investigation followed the traffic path.

## Step 1 — Check ALB target health

```bash
aws elbv2 describe-target-health \
  --target-group-arn <TARGET_GROUP_ARN> \
  --region ap-south-1
```

Initial result:

```text
unhealthy
Target.FailedHealthChecks
```

This showed that the ALB could not successfully communicate with the backend targets.

---

## Step 2 — Check the Kubernetes Service

```bash
kubectl get svc app-2048 \
  -n game-2048 \
  -o yaml
```

The important configuration was:

```yaml
port: 80
targetPort: 8080
```

But the application was actually listening on:

```text
80
```

Therefore:

```text
ALB
 ↓
Pod :8080
 ↓
Nothing listening
 ↓
Health check fails
 ↓
502
```

---

# 🛠️ Root Cause

The Service was forwarding traffic to the wrong port.

Incorrect:

```yaml
ports:
  - port: 80
    targetPort: 8080
```

The application was listening on port `80`.

Therefore the Service was corrected to:

```yaml
ports:
  - port: 80
    targetPort: 80
```

Apply:

```bash
kubectl apply -f service.yml
```

---

# Validate the Fix

First test inside Kubernetes:

```bash
kubectl run test-curl \
  -n game-2048 \
  --image=curlimages/curl \
  --rm -it \
  --restart=Never \
  -- curl -v http://app-2048.game-2048.svc.cluster.local:80/
```

Successful response:

```text
HTTP/1.1 200 OK
Server: nginx
```

Then check the ALB:

```bash
aws elbv2 describe-target-health \
  --target-group-arn <TARGET_GROUP_ARN> \
  --region ap-south-1
```

Final result:

```text
192.168.xxx.xxx   80   healthy
192.168.xxx.xxx   80   healthy
192.168.xxx.xxx   80   healthy
```

Finally:

```bash
curl http://<ALB-DNS-NAME>
```

Returned:

```text
<!DOCTYPE html>
<html>
...
<title>2048</title>
```

The application was then accessible through the ALB/browser.

---

# What I Learned From the 502 Issue

The important lesson was not simply:

> "Change targetPort from 8080 to 80."

The deeper troubleshooting approach was:

```text
Browser
   ↓
ALB
   ↓
Target Group
   ↓
Pod IP
   ↓
Service
   ↓
Pod
   ↓
Application
```

When something fails, test each layer independently.

### Troubleshooting commands

```bash
kubectl get pods
```

```bash
kubectl get svc
```

```bash
kubectl get endpointslices
```

```bash
kubectl describe ingress
```

```bash
kubectl get targetgroupbindings
```

```bash
aws elbv2 describe-target-health
```

```bash
curl <ALB-DNS>
```

This is a useful production-style troubleshooting pattern.

---

# Interview Explanation

## 60–90 Second Project Explanation

> "I built and deployed a 2048 application on Amazon EKS using AWS Fargate. I created a dedicated Fargate profile for the application namespace and deployed three replicas using a Kubernetes Deployment. I exposed those Pods through a Kubernetes Service and then configured the AWS Load Balancer Controller using Helm.
>
> I configured IAM OIDC and IRSA so that the controller could securely obtain AWS permissions through a Kubernetes ServiceAccount. I then created an ALB Ingress with an internet-facing scheme and IP target type. The controller automatically created the Application Load Balancer, target group and TargetGroupBinding, registering the Pod IPs as targets.
>
> During the implementation I faced a 502 Bad Gateway issue. I traced the request path from the ALB to the target group and found that all targets were unhealthy. I then tested the application from inside the cluster using a temporary curl Pod and identified that the Kubernetes Service was forwarding traffic to port 8080 while the application was actually listening on port 80. I corrected the Service targetPort to 80, after which the internal Service returned HTTP 200, the ALB targets became healthy, and the application became accessible through the ALB DNS name.
>
> This project helped me understand EKS, Fargate scheduling, Kubernetes Services, Ingress, IAM/IRSA, AWS Load Balancer Controller, ALB target groups and practical Kubernetes networking troubleshooting."

---

# Common Interview Questions

## Q1. Why did you use Fargate?

Fargate provides serverless compute for Kubernetes Pods.

Instead of managing EC2 worker nodes, AWS manages the underlying compute infrastructure.

The application Pods were scheduled onto Fargate based on the Fargate profile.

---

## Q2. Why did you create a Fargate profile?

A Fargate profile defines which Pods should run on Fargate.

In this project:

```text
Namespace: game-2048
```

was associated with:

```text
fp-game-2048
```

Therefore Pods created in that namespace could be scheduled onto Fargate.

---

## Q3. What is the AWS Load Balancer Controller?

It is a Kubernetes controller that watches Kubernetes resources such as Ingress and creates/configures AWS load-balancing resources.

In this project:

```text
Kubernetes Ingress
        ↓
AWS Load Balancer Controller
        ↓
AWS Application Load Balancer
        ↓
Target Group
        ↓
Pod IPs
```

---

## Q4. Why did you need IAM OIDC?

The AWS Load Balancer Controller needs AWS API permissions.

OIDC allows Kubernetes identities to be associated with AWS IAM roles.

This enables IRSA:

```text
ServiceAccount
      ↓
OIDC
      ↓
IAM Role
      ↓
AWS API
```

This avoids giving broad AWS permissions to every Pod.

---

## Q5. Why use IP target type?

The Ingress contained:

```yaml
alb.ingress.kubernetes.io/target-type: ip
```

This means the ALB target group registers the **Pod IP addresses directly**.

So the traffic path is approximately:

```text
ALB
 ↓
Pod IP
 ↓
Application
```

rather than relying on EC2 worker-node IPs.

---

## Q6. What is TargetGroupBinding?

TargetGroupBinding is a custom resource used by the AWS Load Balancer Controller to associate an AWS Target Group with Kubernetes workloads/services.

It provides the connection between:

```text
AWS Target Group
        ↕
Kubernetes Service
```

---

## Q7. What caused your 502?

The ALB targets were unhealthy.

I verified this using:

```bash
aws elbv2 describe-target-health
```

The targets showed:

```text
Target.FailedHealthChecks
```

I then tested the Kubernetes Service from inside the cluster.

The Service was forwarding traffic to port `8080`, but the application was listening on port `80`.

After changing:

```yaml
targetPort: 8080
```

to:

```yaml
targetPort: 80
```

the Service returned HTTP 200 and the ALB targets became healthy.

---

## Q8. Does containerPort control traffic?

Not by itself.

For example:

```yaml
containerPort: 80
```

is primarily declarative metadata describing the port the container listens on.

The Service controls how Kubernetes routes traffic:

```yaml
port: 80
targetPort: 80
```

The important traffic mapping is:

```text
Service port
     ↓
targetPort
     ↓
Container application port
```

---

# Useful Verification Commands

### Cluster

```bash
eksctl get cluster
```

```bash
kubectl get nodes
```

### Fargate

```bash
eksctl get fargateprofile \
  --cluster eks-devops-2048 \
  --region ap-south-1
```

### Application

```bash
kubectl get deployment -n game-2048
```

```bash
kubectl get pods -n game-2048 -o wide
```

### Service

```bash
kubectl get svc -n game-2048
```

```bash
kubectl get endpointslices -n game-2048
```

### Ingress

```bash
kubectl get ingress -n game-2048
```

```bash
kubectl describe ingress app-2048 -n game-2048
```

### TargetGroupBinding

```bash
kubectl get targetgroupbindings -n game-2048
```

### ALB target health

```bash
aws elbv2 describe-target-health \
  --target-group-arn <TARGET_GROUP_ARN> \
  --region ap-south-1
```

### Controller

```bash
kubectl get pods -n kube-system \
  -l app.kubernetes.io/name=aws-load-balancer-controller
```

```bash
kubectl logs \
  -n kube-system \
  <controller-pod>
```

---

# Cleanup

Delete the application resources:

```bash
kubectl delete -f ingress.yml
kubectl delete -f service.yml
kubectl delete -f deployment.yml
```

Delete the Fargate profile:

```bash
eksctl delete fargateprofile \
  --cluster eks-devops-2048 \
  --name fp-game-2048 \
  --region ap-south-1
```

Remove the Helm release:

```bash
helm uninstall aws-load-balancer-controller \
  -n kube-system
```

Finally delete the EKS cluster:

```bash
eksctl delete cluster \
  --name eks-devops-2048 \
  --region ap-south-1
```

Verify:

```bash
eksctl get cluster
```

---

# Key Concepts Demonstrated

This project demonstrates practical knowledge of:

* Amazon EKS
* AWS Fargate
* Kubernetes Deployments
* Kubernetes Services
* Kubernetes DNS
* EndpointSlices
* Kubernetes Ingress
* AWS Application Load Balancer
* AWS Load Balancer Controller
* Target Groups
* TargetGroupBinding
* IAM
* IAM OIDC
* IRSA
* Helm
* AWS CLI
* kubectl
* Kubernetes networking
* ALB health checks
* Application troubleshooting
* Service port mapping
* Production-style debugging

---

# Key Takeaway

The main goal of this project was not simply to deploy the 2048 game.

The important learning was understanding the complete traffic and infrastructure flow:

```text
User
 │
 ▼
AWS ALB
 │
 ▼
Target Group
 │
 ▼
Pod IP
 │
 ▼
Kubernetes Service
 │
 ▼
Application Container
 │
 ▼
Nginx / 2048
```

And the AWS/Kubernetes control flow:

```text
Kubernetes Ingress
        │
        ▼
AWS Load Balancer Controller
        │
        ▼
AWS APIs
        │
        ▼
Application Load Balancer
        │
        ▼
Target Group
        │
        ▼
Pod IPs
```

The most valuable part of the project was troubleshooting the complete request path instead of treating the ALB `502` as an isolated AWS problem.

---

# Project Outcome

Successfully deployed the 2048 application on Amazon EKS using AWS Fargate and exposed it through an internet-facing AWS Application Load Balancer.

The project also included hands-on troubleshooting of:

* AWS Load Balancer Controller CrashLoopBackOff
* Fargate scheduling
* IAM/OIDC/IRSA configuration
* Kubernetes Service routing
* ALB target health
* HTTP 502 Bad Gateway
* Service `targetPort` mismatch
* Internal cluster connectivity
* ALB-to-Pod connectivity

This project can be reproduced from scratch using the manifests and commands documented in this repository.
