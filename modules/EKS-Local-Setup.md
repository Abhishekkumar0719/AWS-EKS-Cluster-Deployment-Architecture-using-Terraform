# 🖥️ How to Connect AWS EKS Cluster to Your Local Machine

> A step-by-step guide to configure `kubectl` on your local machine to communicate with your AWS EKS cluster.

---

## Step 1 — Install AWS CLI

Download and install the AWS CLI on your local machine.

```bash
# ──────────────────────────────────────────────
# macOS — install via Homebrew
# ──────────────────────────────────────────────
brew install awscli

# ──────────────────────────────────────────────
# Linux — download and install manually
# ──────────────────────────────────────────────
curl "https://awscli.amazonaws.com/awscli-exe-linux-x86_64.zip" -o "awscliv2.zip"
unzip awscliv2.zip
sudo ./aws/install

# ──────────────────────────────────────────────
# Windows — install via winget (PowerShell)
# ──────────────────────────────────────────────
winget install Amazon.AWSCLI
```

---

## Step 2 — Check AWS CLI Version (Verify Installation)

Make sure AWS CLI is installed correctly and is up to date (version 2.x recommended).

```bash
# ──────────────────────────────────────────────
# Check the currently installed AWS CLI version
# Expected output: aws-cli/2.x.x ...
# ──────────────────────────────────────────────
aws --version

# ──────────────────────────────────────────────
# If outdated, upgrade AWS CLI on macOS
# ──────────────────────────────────────────────
brew upgrade awscli

# ──────────────────────────────────────────────
# If outdated, upgrade AWS CLI on Linux
# ──────────────────────────────────────────────
curl "https://awscli.amazonaws.com/awscli-exe-linux-x86_64.zip" -o "awscliv2.zip"
unzip -o awscliv2.zip
sudo ./aws/install --update
```

---

## Step 3 — Check AWS Credentials Are Configured

Verify that your AWS Access Key and Secret Key are saved on your machine.

```bash
# ──────────────────────────────────────────────
# View the stored AWS credentials file
# This shows: aws_access_key_id & aws_secret_access_key
# File location: ~/.aws/credentials
# ──────────────────────────────────────────────
cat ~/.aws/credentials

# ──────────────────────────────────────────────
# Expected output format:
# [default]
# aws_access_key_id     = AKIAIOSFODNN7EXAMPLE
# aws_secret_access_key = wJalrXUtnFEMI/K7MDENG/bPxRfiCYEXAMPLEKEY
# ──────────────────────────────────────────────

# ──────────────────────────────────────────────
# Also check the AWS config file (region & output format)
# ──────────────────────────────────────────────
cat ~/.aws/config

# ──────────────────────────────────────────────
# If credentials are NOT set, configure them now
# You will be prompted for: Access Key, Secret Key, Region, Output format
# ──────────────────────────────────────────────
aws configure

# ──────────────────────────────────────────────
# Verify credentials are working by checking your AWS identity
# Should return: Account ID, User ARN, and IAM username
# ──────────────────────────────────────────────
aws sts get-caller-identity
```

---

## Step 4 — Connect kubectl to Your EKS Cluster

This command downloads the kubeconfig for your EKS cluster and merges it into your local `~/.kube/config` file.

```bash
# ──────────────────────────────────────────────
# Update local kubeconfig to connect to your EKS cluster
# --region  : AWS region where your cluster is deployed
# --name    : The name of your EKS cluster
# This command writes the cluster connection info to ~/.kube/config
# ──────────────────────────────────────────────
aws eks update-kubeconfig --region us-west-2 --name my-eks-cluster

# ──────────────────────────────────────────────
# Expected output:
# Added new context arn:aws:eks:us-west-2:<account-id>:cluster/my-eks-cluster to ~/.kube/config
# ──────────────────────────────────────────────

# ──────────────────────────────────────────────
# Optional: Use a specific AWS profile if you manage multiple accounts
# Replace "my-profile" with your profile name from ~/.aws/credentials
# ──────────────────────────────────────────────
aws eks update-kubeconfig --region us-west-2 --name my-eks-cluster --profile my-profile
```

---

## Step 5 — View the Full kubeconfig File

Inspect the full kubeconfig to confirm the cluster, user, and context were added correctly.

```bash
# ──────────────────────────────────────────────
# Display the entire kubeconfig file
# Shows all: clusters, users, and contexts configured on your machine
# Useful to verify the EKS cluster entry was added
# ──────────────────────────────────────────────
kubectl config view

# ──────────────────────────────────────────────
# View the raw kubeconfig file directly (unmasked)
# ──────────────────────────────────────────────
cat ~/.kube/config

# ──────────────────────────────────────────────
# List only the cluster names in your kubeconfig
# ──────────────────────────────────────────────
kubectl config get-clusters

# ──────────────────────────────────────────────
# List all users in your kubeconfig
# ──────────────────────────────────────────────
kubectl config get-users
```

---

## Step 6 — Check the Active Context (Which Cluster You Are Using)

The context tells kubectl which cluster and namespace to communicate with.

```bash
# ──────────────────────────────────────────────
# Show the currently active context (active cluster connection)
# This is the cluster all kubectl commands will run against
# ──────────────────────────────────────────────
kubectl config current-context

# ──────────────────────────────────────────────
# Expected output (EKS context format):
# arn:aws:eks:us-west-2:<account-id>:cluster/my-eks-cluster
# ──────────────────────────────────────────────

# ──────────────────────────────────────────────
# List ALL available contexts (if you have multiple clusters)
# The active context has an asterisk (*) next to it
# ──────────────────────────────────────────────
kubectl config get-contexts

# ──────────────────────────────────────────────
# Switch to a different context (if you manage multiple clusters)
# Replace <context-name> with the name from get-contexts output
# ──────────────────────────────────────────────
kubectl config use-context <context-name>

# ──────────────────────────────────────────────
# Verify the cluster is reachable and get cluster info
# ──────────────────────────────────────────────
kubectl cluster-info
```

---

## Step 7 — Check Service Accounts in the Cluster

Service Accounts are Kubernetes identities used by pods to interact with the Kubernetes API.

```bash
# ──────────────────────────────────────────────
# List service accounts across ALL namespaces
# -A flag = --all-namespaces
# Shows: NAMESPACE, NAME, SECRETS, AGE
# ──────────────────────────────────────────────
kubectl get sa -A

# ──────────────────────────────────────────────
# List service accounts ONLY in the kube-system namespace
# kube-system contains core Kubernetes system components
# e.g. coredns, aws-node, kube-proxy service accounts
# ──────────────────────────────────────────────
kubectl get sa -n kube-system

# ──────────────────────────────────────────────
# List service accounts in the default namespace
# ──────────────────────────────────────────────
kubectl get sa -n default

# ──────────────────────────────────────────────
# Get detailed info about a specific service account
# Replace <sa-name> with the actual service account name
# ──────────────────────────────────────────────
kubectl describe sa <sa-name> -n kube-system

# ──────────────────────────────────────────────
# Bonus: Check all nodes are Ready in your EKS cluster
# Confirms the worker nodes joined successfully
# ──────────────────────────────────────────────
kubectl get nodes

# ──────────────────────────────────────────────
# Bonus: Check all pods running across all namespaces
# ──────────────────────────────────────────────
kubectl get pods -A
```

---

## Quick Reference Summary

| Step | Command | Purpose |
|---|---|---|
| 1 | `brew install awscli` | Install AWS CLI |
| 2 | `aws --version` | Verify AWS CLI version |
| 3 | `cat ~/.aws/credentials` | Check AWS credentials |
| 3 | `aws sts get-caller-identity` | Verify credentials work |
| 4 | `aws eks update-kubeconfig --region us-west-2 --name my-eks-cluster` | Connect kubectl to EKS |
| 5 | `kubectl config view` | View full kubeconfig |
| 6 | `kubectl config current-context` | Check active cluster context |
| 7 | `kubectl get sa -A` | List all service accounts |
| 7 | `kubectl get sa -n kube-system` | List kube-system service accounts |

---

> **Tip:** If you get an `Unauthorized` error, make sure the IAM user/role you're using has the `eks:DescribeCluster` permission and is listed in the cluster's `aws-auth` ConfigMap.

```bash
# Check the aws-auth ConfigMap (maps IAM users/roles to Kubernetes RBAC)
kubectl describe configmap aws-auth -n kube-system
```
