# AWS EKS Kubernetes Architecture — Deployment Guide

> Deploy a production-ready Kubernetes cluster on AWS EKS with VPC networking, IAM roles, load balancing, and secure access — step by step.

---

## Table of Contents

1. [Architecture Overview](#architecture-overview)
2. [Prerequisites](#prerequisites)
3. [Infrastructure Setup](#infrastructure-setup)
4. [EKS Cluster Setup](#eks-cluster-setup)
5. [Networking & Security](#networking--security)
6. [IAM Roles & Policies](#iam-roles--policies)
7. [Deploy Your Application](#deploy-your-application)
8. [Accessing the Cluster](#accessing-the-cluster)
9. [Push Code to GitHub](#push-code-to-github)
10. [CI/CD Integration (Optional)](#cicd-integration-optional)
11. [Troubleshooting](#troubleshooting)
12. [Contributing](#contributing)
13. [License](#license)

---

## Architecture Overview

This project deploys a fully managed Kubernetes cluster using **Amazon EKS** inside a custom **VPC** with the following components:
![image](<AWS EKS Architecture.png>)

```
Internet
   │
   ▼
Internet Gateway (IGW)
   │
   ▼
Public Subnet (10.1.0.0/24)
   ├── Bastion Host / Jump Server (optional)
   └── Application Load Balancer (ALB)
         │
         ▼
   Private Subnet (10.0.2.0/24)
         ├── EKS Worker Nodes (Auto Scaling Group)
         ├── Amazon EKS (Kubernetes Control Plane)
         └── Kubernetes Pods
                │
                ▼
         NAT Gateway ──► Internet (outbound only)
```

**Key components:**

| Component | Purpose |
|---|---|
| VPC (10.0.0.0/16) | Isolated network for all resources |
| Public Subnet | Hosts ALB and optional Bastion |
| Private Subnet | Hosts EKS worker nodes and pods |
| Internet Gateway | Inbound/outbound internet for public subnet |
| NAT Gateway | Outbound internet for private subnet |
| Application Load Balancer | Routes HTTPS/HTTP traffic to pods |
| Amazon EKS | Managed Kubernetes control plane |
| Auto Scaling Group | Scales worker nodes automatically |
| IAM Roles | Cluster and node permissions |

---

## Prerequisites

Before you begin, make sure you have the following installed and configured:

### Required Tools

```bash
# AWS CLI
aws --version          # >= 2.x

# kubectl
kubectl version        # >= 1.28

# eksctl (recommended for EKS setup)
eksctl version         # >= 0.170

# Helm (for deploying apps)
helm version           # >= 3.x

# Docker (for building images)
docker --version       # >= 24.x

# Git
git --version
```

### Install AWS CLI

```bash
# macOS
brew install awscli

# Linux
curl "https://awscli.amazonaws.com/awscli-exe-linux-x86_64.zip" -o "awscliv2.zip"
unzip awscliv2.zip && sudo ./aws/install

# Windows (PowerShell)
winget install Amazon.AWSCLI
```

### Install kubectl

```bash
# macOS
brew install kubectl

# Linux
curl -LO "https://dl.k8s.io/release/$(curl -Ls https://dl.k8s.io/release/stable.txt)/bin/linux/amd64/kubectl"
chmod +x kubectl && sudo mv kubectl /usr/local/bin/

# Windows
winget install Kubernetes.kubectl
```

### Install eksctl

```bash
# macOS
brew tap weaveworks/tap && brew install weaveworks/tap/eksctl

## EKS Local Setup

[EKS Local Setup Guide](/EKS-local-Setup/)
```

### Configure AWS Credentials

```bash
aws configure
# AWS Access Key ID: <your-access-key>
# AWS Secret Access Key: <your-secret-key>
# Default region name: ap-south-1   (or your preferred region)
# Default output format: json

# Verify
aws sts get-caller-identity
```

---

## Infrastructure Setup

### Step 1 — Create the VPC

```bash
# Create VPC
- [create vpc terraform](./modules/vpc/)
```

### Step 2 — Create Subnets

```bash
# Public Subnet
aws ec2 create-subnet \
  --vpc-id <VPC_ID> \
  --cidr-block 10.1.0.0/24 \
  --availability-zone ap-south-1a \
  --tag-specifications 'ResourceType=subnet,Tags=[{Key=Name,Value=eks-public-subnet},{Key=kubernetes.io/role/elb,Value=1}]'

# Private Subnet
aws ec2 create-subnet \
  --vpc-id <VPC_ID> \
  --cidr-block 10.0.2.0/24 \
  --availability-zone ap-south-1a \
  --tag-specifications 'ResourceType=subnet,Tags=[{Key=Name,Value=eks-private-subnet},{Key=kubernetes.io/role/internal-elb,Value=1}]'
```

### Step 3 — Create Internet Gateway

```bash
# Create and attach IGW
aws ec2 create-internet-gateway \
  --tag-specifications 'ResourceType=internet-gateway,Tags=[{Key=Name,Value=eks-igw}]'

aws ec2 attach-internet-gateway \
  --internet-gateway-id <IGW_ID> \
  --vpc-id <VPC_ID>
```

### Step 4 — Create NAT Gateway

```bash
# Allocate Elastic IP for NAT Gateway
aws ec2 allocate-address --domain vpc

# Create NAT Gateway in the public subnet
aws ec2 create-nat-gateway \
  --subnet-id <PUBLIC_SUBNET_ID> \
  --allocation-id <ELASTIC_IP_ALLOCATION_ID> \
  --tag-specifications 'ResourceType=natgateway,Tags=[{Key=Name,Value=eks-nat-gw}]'
```

### Step 5 — Configure Route Tables

```bash
# --- Public Route Table ---
aws ec2 create-route-table \
  --vpc-id <VPC_ID> \
  --tag-specifications 'ResourceType=route-table,Tags=[{Key=Name,Value=eks-public-rt}]'

# Add route: 0.0.0.0/0 -> IGW
aws ec2 create-route \
  --route-table-id <PUBLIC_RT_ID> \
  --destination-cidr-block 0.0.0.0/0 \
  --gateway-id <IGW_ID>

# Associate with public subnet
aws ec2 associate-route-table \
  --route-table-id <PUBLIC_RT_ID> \
  --subnet-id <PUBLIC_SUBNET_ID>

# --- Private Route Table ---
aws ec2 create-route-table \
  --vpc-id <VPC_ID> \
  --tag-specifications 'ResourceType=route-table,Tags=[{Key=Name,Value=eks-private-rt}]'

# Add route: 0.0.0.0/0 -> NAT Gateway
aws ec2 create-route \
  --route-table-id <PRIVATE_RT_ID> \
  --destination-cidr-block 0.0.0.0/0 \
  --nat-gateway-id <NAT_GW_ID>

# Associate with private subnet
aws ec2 associate-route-table \
  --route-table-id <PRIVATE_RT_ID> \
  --subnet-id <PRIVATE_SUBNET_ID>
```

---

## EKS Cluster Setup

### Step 1 — Create IAM Role for Cluster

```bash
# Create trust policy
cat > eks-cluster-trust.json <<EOF
{
  "Version": "2012-10-17",
  "Statement": [{
    "Effect": "Allow",
    "Principal": { "Service": "eks.amazonaws.com" },
    "Action": "sts:AssumeRole"
  }]
}
EOF

# Create the role
aws iam create-role \
  --role-name EKSClusterRole \
  --assume-role-policy-document file://eks-cluster-trust.json

# Attach required policies
aws iam attach-role-policy \
  --role-name EKSClusterRole \
  --policy-arn arn:aws:iam::aws:policy/AmazonEKSClusterPolicy

aws iam attach-role-policy \
  --role-name EKSClusterRole \
  --policy-arn arn:aws:iam::aws:policy/AmazonEKSVPCResourceController
```

### Step 2 — Create the EKS Cluster

```bash
aws eks create-cluster \
  --name my-eks-cluster \
  --kubernetes-version 1.29 \
  --role-arn arn:aws:iam::<ACCOUNT_ID>:role/EKSClusterRole \
  --resources-vpc-config \
    subnetIds=<PUBLIC_SUBNET_ID>,<PRIVATE_SUBNET_ID>,\
    securityGroupIds=<CLUSTER_SG_ID>,\
    endpointPublicAccess=true,\
    endpointPrivateAccess=true

# Wait for cluster to become ACTIVE (takes ~10-15 minutes)
aws eks wait cluster-active --name my-eks-cluster

# Update kubeconfig
aws eks update-kubeconfig --name my-eks-cluster --region ap-south-1
```

### Step 3 — Create IAM Role for Worker Nodes

```bash
# Create trust policy
cat > eks-node-trust.json <<EOF
{
  "Version": "2012-10-17",
  "Statement": [{
    "Effect": "Allow",
    "Principal": { "Service": "ec2.amazonaws.com" },
    "Action": "sts:AssumeRole"
  }]
}
EOF

# Create the role
aws iam create-role \
  --role-name EKSNodeRole \
  --assume-role-policy-document file://eks-node-trust.json

# Attach all required node policies
aws iam attach-role-policy --role-name EKSNodeRole \
  --policy-arn arn:aws:iam::aws:policy/AmazonEKSWorkerNodePolicy

aws iam attach-role-policy --role-name EKSNodeRole \
  --policy-arn arn:aws:iam::aws:policy/AmazonEC2ContainerRegistryReadOnly

aws iam attach-role-policy --role-name EKSNodeRole \
  --policy-arn arn:aws:iam::aws:policy/AmazonEKS_CNI_Policy

aws iam attach-role-policy --role-name EKSNodeRole \
  --policy-arn arn:aws:iam::aws:policy/AmazonSSMManagedInstanceCore
```

### Step 4 — Create Node Group (Auto Scaling)

```bash
aws eks create-nodegroup \
  --cluster-name my-eks-cluster \
  --nodegroup-name my-node-group \
  --scaling-config minSize=1,maxSize=5,desiredSize=2 \
  --disk-size 20 \
  --subnets <PRIVATE_SUBNET_ID> \
  --instance-types t3.medium \
  --ami-type AL2_x86_64 \
  --node-role arn:aws:iam::<ACCOUNT_ID>:role/EKSNodeRole

# Wait for node group to be ACTIVE
aws eks wait nodegroup-active \
  --cluster-name my-eks-cluster \
  --nodegroup-name my-node-group

# Verify nodes are ready
kubectl get nodes
```

---

## Networking & Security

### Security Groups

```bash
# Create cluster security group
aws ec2 create-security-group \
  --group-name eks-cluster-sg \
  --description "EKS Cluster Security Group" \
  --vpc-id <VPC_ID>

# Allow HTTPS from ALB to cluster API
aws ec2 authorize-security-group-ingress \
  --group-id <CLUSTER_SG_ID> \
  --protocol tcp --port 443 \
  --source-group <ALB_SG_ID>

# Worker node security group — allow all from cluster SG
aws ec2 authorize-security-group-ingress \
  --group-id <NODE_SG_ID> \
  --protocol -1 \
  --source-group <CLUSTER_SG_ID>
```

### Network ACLs (NACLs)

NACLs act as a subnet-level firewall. For EKS, allow the following:

| Rule | Type | Port | Source | Allow/Deny |
|---|---|---|---|---|
| 100 | HTTPS | 443 | 0.0.0.0/0 | Allow |
| 110 | HTTP | 80 | 0.0.0.0/0 | Allow |
| 120 | Custom TCP | 1024-65535 | 0.0.0.0/0 | Allow |
| * | All Traffic | All | 0.0.0.0/0 | Deny |

### VPC Endpoints (Optional but Recommended)

Reduce data transfer costs and improve security by keeping traffic private:

```bash
# ECR API endpoint
aws ec2 create-vpc-endpoint \
  --vpc-id <VPC_ID> \
  --service-name com.amazonaws.ap-south-1.ecr.api \
  --vpc-endpoint-type Interface \
  --subnet-ids <PRIVATE_SUBNET_ID> \
  --security-group-ids <NODE_SG_ID>

# ECR Docker endpoint
aws ec2 create-vpc-endpoint \
  --vpc-id <VPC_ID> \
  --service-name com.amazonaws.ap-south-1.ecr.dkr \
  --vpc-endpoint-type Interface \
  --subnet-ids <PRIVATE_SUBNET_ID> \
  --security-group-ids <NODE_SG_ID>

# S3 Gateway endpoint
aws ec2 create-vpc-endpoint \
  --vpc-id <VPC_ID> \
  --service-name com.amazonaws.ap-south-1.s3 \
  --vpc-endpoint-type Gateway \
  --route-table-ids <PRIVATE_RT_ID>
```

---

## IAM Roles & Policies

### Summary of Required Roles

#### 1. IAM Role for Cluster (EKSClusterRole)

Used by the EKS control plane to manage AWS resources on your behalf.

| Policy | Purpose |
|---|---|
| AmazonEKSClusterPolicy | Core EKS cluster management |
| AmazonEKSVPCResourceController | Create/manage ENI, ELB, SG |

#### 2. IAM Role for Nodes (EKSNodeRole)

Used by EC2 worker nodes to interact with AWS services.

| Policy | Purpose |
|---|---|
| AmazonEKSWorkerNodePolicy | Node registration with EKS |
| AmazonEC2ContainerRegistryReadOnly | Pull images from ECR |
| AmazonEKS_CNI_Policy | Networking — ENI, IP allocation |
| AmazonSSMManagedInstanceCore | SSM session management |

---

## Deploy Your Application

### Step 1 — Build and Push Docker Image to ECR

```bash
# Create ECR repository
aws ecr create-repository --repository-name my-app --region ap-south-1

# Authenticate Docker to ECR
aws ecr get-login-password --region ap-south-1 | \
  docker login --username AWS \
  --password-stdin <ACCOUNT_ID>.dkr.ecr.ap-south-1.amazonaws.com

# Build your Docker image
docker build -t my-app .

# Tag and push
docker tag my-app:latest <ACCOUNT_ID>.dkr.ecr.ap-south-1.amazonaws.com/my-app:latest
docker push <ACCOUNT_ID>.dkr.ecr.ap-south-1.amazonaws.com/my-app:latest
```

### Step 2 — Create Kubernetes Deployment

Create a file `deployment.yaml`:

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: my-app
  namespace: default
spec:
  replicas: 2
  selector:
    matchLabels:
      app: my-app
  template:
    metadata:
      labels:
        app: my-app
    spec:
      containers:
        - name: my-app
          image: <ACCOUNT_ID>.dkr.ecr.ap-south-1.amazonaws.com/my-app:latest
          ports:
            - containerPort: 8080
          resources:
            requests:
              cpu: "100m"
              memory: "128Mi"
            limits:
              cpu: "500m"
              memory: "512Mi"
```

```bash
kubectl apply -f deployment.yaml
kubectl get pods
```

### Step 3 — Create a Service

Create a file `service.yaml`:

```yaml
apiVersion: v1
kind: Service
metadata:
  name: my-app-service
spec:
  selector:
    app: my-app
  ports:
    - protocol: TCP
      port: 80
      targetPort: 8080
  type: ClusterIP
```

```bash
kubectl apply -f service.yaml
```

### Step 4 — Configure Ingress / ALB

Install the AWS Load Balancer Controller:

```bash
helm repo add eks https://aws.github.io/eks-charts
helm repo update

helm install aws-load-balancer-controller eks/aws-load-balancer-controller \
  -n kube-system \
  --set clusterName=my-eks-cluster \
  --set serviceAccount.create=false \
  --set serviceAccount.name=aws-load-balancer-controller
```

Create a file `ingress.yaml`:

```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: my-app-ingress
  annotations:
    kubernetes.io/ingress.class: alb
    alb.ingress.kubernetes.io/scheme: internet-facing
    alb.ingress.kubernetes.io/target-type: ip
spec:
  rules:
    - http:
        paths:
          - path: /
            pathType: Prefix
            backend:
              service:
                name: my-app-service
                port:
                  number: 80
```

```bash
kubectl apply -f ingress.yaml

# Get the ALB DNS name
kubectl get ingress my-app-ingress
```

---

## Accessing the Cluster

### Option 1 — kubectl via Bastion Host / VPN

```bash
# SSH into Bastion Host
ssh -i your-key.pem ec2-user@<BASTION_PUBLIC_IP>

# From Bastion, configure kubectl
aws eks update-kubeconfig --name my-eks-cluster --region ap-south-1

# Verify access
kubectl get nodes
kubectl get pods -A
```

### Option 2 — HTTPS via Application Load Balancer

Once the ALB Ingress is created:

```bash
# Get the ALB DNS name
kubectl get ingress -n default

# Access the app
curl http://<ALB_DNS_NAME>

# Test specific endpoints
curl http://<ALB_DNS_NAME>/health
curl http://<ALB_DNS_NAME>/api/v1/
```

### Option 3 — Internal Services

For internal communication between pods:

```bash
# Access a service from within the cluster
kubectl exec -it <POD_NAME> -- curl http://my-app-service:80

# Port-forward for local testing
kubectl port-forward service/my-app-service 8080:80
# Then open http://localhost:8080 in your browser
```

---

## Push Code to GitHub

Follow these steps to push your project code (including this README and Kubernetes manifests) to GitHub.

### Step 1 — Initialize Git Repository

```bash
# Navigate to your project folder
cd my-eks-project

# Initialize git
git init

# Check status
git status
```

### Step 2 — Create a .gitignore File

```bash
cat > .gitignore <<EOF
# AWS credentials — NEVER commit these
.aws/
*.pem
*.key
secrets.yaml
*-secret.yaml

# Terraform state
*.tfstate
*.tfstate.backup
.terraform/

# Node modules
node_modules/

# Docker
.docker/

# OS files
.DS_Store
Thumbs.db
EOF
```

### Step 3 — Add and Commit Your Files

```bash
# Stage all files
git add .

# Review what's being committed
git status

# Create your first commit
git commit -m "Initial commit: AWS EKS architecture with Kubernetes manifests"
```

### Step 4 — Create a Repository on GitHub

1. Go to [github.com](https://github.com) and log in.
2. Click the **+** icon → **New repository**.
3. Set the repository name (e.g., `aws-eks-deployment`).
4. Choose **Public** or **Private**.
5. Do **not** initialize with README (you already have one).
6. Click **Create repository**.

### Step 5 — Add Remote and Push

```bash
# Add GitHub as remote origin
git remote add origin https://github.com/<YOUR_USERNAME>/aws-eks-deployment.git

# Rename branch to main (if needed)
git branch -M main

# Push your code
git push -u origin main
```

### Step 6 — Verify on GitHub

Open your browser and navigate to:

```
https://github.com/<YOUR_USERNAME>/aws-eks-deployment
```

You should see all your files and this README rendered on the repository homepage.

### Step 7 — Push Future Changes

```bash
# After making changes
git add .
git commit -m "feat: add HPA configuration for worker nodes"
git push origin main
```

---

## CI/CD Integration (Optional)

Automate your deployment using **GitHub Actions**.

Create the file `.github/workflows/deploy.yml`:

```yaml
name: Deploy to EKS

on:
  push:
    branches: [main]

jobs:
  deploy:
    runs-on: ubuntu-latest

    steps:
      - name: Checkout code
        uses: actions/checkout@v4

      - name: Configure AWS credentials
        uses: aws-actions/configure-aws-credentials@v4
        with:
          aws-access-key-id: ${{ secrets.AWS_ACCESS_KEY_ID }}
          aws-secret-access-key: ${{ secrets.AWS_SECRET_ACCESS_KEY }}
          aws-region: ap-south-1

      - name: Login to Amazon ECR
        uses: aws-actions/amazon-ecr-login@v2

      - name: Build and push Docker image
        run: |
          docker build -t ${{ secrets.ECR_REGISTRY }}/my-app:${{ github.sha }} .
          docker push ${{ secrets.ECR_REGISTRY }}/my-app:${{ github.sha }}

      - name: Update kubeconfig
        run: |
          aws eks update-kubeconfig --name my-eks-cluster --region ap-south-1

      - name: Deploy to EKS
        run: |
          kubectl set image deployment/my-app \
            my-app=${{ secrets.ECR_REGISTRY }}/my-app:${{ github.sha }}
          kubectl rollout status deployment/my-app
```

Add these GitHub Secrets in your repository settings:

| Secret | Value |
|---|---|
| `AWS_ACCESS_KEY_ID` | Your AWS access key |
| `AWS_SECRET_ACCESS_KEY` | Your AWS secret key |
| `ECR_REGISTRY` | `<ACCOUNT_ID>.dkr.ecr.ap-south-1.amazonaws.com` |

---

## Troubleshooting

### Nodes not joining the cluster

```bash
# Check node group status
aws eks describe-nodegroup \
  --cluster-name my-eks-cluster \
  --nodegroup-name my-node-group

# Verify IAM role is correct
aws iam get-role --role-name EKSNodeRole

# Check if aws-auth ConfigMap has the node role
kubectl describe configmap aws-auth -n kube-system
```

### Pods stuck in Pending state

```bash
# Describe the pod for events
kubectl describe pod <POD_NAME>

# Check node resources
kubectl top nodes

# Check if image can be pulled
kubectl get events --sort-by='.metadata.creationTimestamp'
```

### ALB not provisioning

```bash
# Check Load Balancer Controller logs
kubectl logs -n kube-system \
  -l app.kubernetes.io/name=aws-load-balancer-controller

# Verify ingress annotations
kubectl describe ingress my-app-ingress

# Check IAM permissions for LB controller
aws iam get-role --role-name AmazonEKSLoadBalancerControllerRole
```

### Cannot connect to cluster API

```bash
# Refresh kubeconfig
aws eks update-kubeconfig --name my-eks-cluster --region ap-south-1

# Test connectivity
kubectl cluster-info

# Check if security group allows 443 from your IP
aws ec2 describe-security-groups --group-ids <CLUSTER_SG_ID>
```

---

## Contributing

Contributions are welcome! Please follow these steps:

1. Fork the repository.
2. Create a feature branch: `git checkout -b feature/my-new-feature`
3. Commit your changes: `git commit -m "feat: add my new feature"`
4. Push to the branch: `git push origin feature/my-new-feature`
5. Open a Pull Request.

Please follow the [Conventional Commits](https://www.conventionalcommits.org/) specification for commit messages.

---

## License

This project is licensed under the MIT License. See the [LICENSE](LICENSE) file for details.

---

> **Security Note:** Never commit AWS credentials, `.pem` files, or Kubernetes secrets to your repository. Use GitHub Secrets, AWS Secrets Manager, or environment variables for sensitive values.
