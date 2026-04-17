# End-to-End DevSecOps Project Guide

This guide provides step-by-step instructions on how to build, secure, and monitor a Kubernetes-based application environment on AWS from scratch. By following this tutorial, you will implement a **Security-as-Code** infrastructure model.

---

## 🛠️ Prerequisites
Before starting, ensure you have the following tools installed on your local machine:
*   [AWS CLI](https://aws.amazon.com/cli/) (Authenticated with appropriate IAM privileges)
*   [Terraform](https://www.terraform.io/downloads)
*   [kubectl](https://kubernetes.io/docs/tasks/tools/)
*   [Helm](https://helm.sh/docs/intro/install/)
*   [Checkov](https://www.checkov.io/)

---

## Step 1: Infrastructure as Code (IaC) with Terraform
We start by defining our foundation. Security must be baked into the Terraform code before deployment.

### 1.1 Create the configuration
Create a file named `main.tf` and paste the following base code to provision a VPC with private subnets and a hardened EKS cluster:

```hcl
terraform {
  required_providers {
    aws = {
      source  = "hashicorp/aws"
      version = "~> 5.0"
    }
  }
}

provider "aws" {
  region = "us-east-1"
}

module "vpc" {
  source  = "terraform-aws-modules/vpc/aws"
  version = "5.0.0"

  name = "devsecops-vpc"
  cidr = "10.0.0.0/16"

  azs             = ["us-east-1a", "us-east-1b"]
  private_subnets = ["10.0.1.0/24", "10.0.2.0/24"]
  public_subnets  = ["10.0.101.0/24", "10.0.102.0/24"]

  enable_nat_gateway = true
  single_nat_gateway = true
}

module "eks" {
  source  = "terraform-aws-modules/eks/aws"
  version = "19.21.0"

  cluster_name    = "secure-cluster"
  cluster_version = "1.31"

  vpc_id     = module.vpc.vpc_id
  subnet_ids = module.vpc.private_subnets

  # Security Hardening
  enable_irsa = true

  cluster_endpoint_private_access = true
  cluster_endpoint_public_access  = true # Set to false in high-security production

  create_kms_key = true
  cluster_encryption_config = {
    resources = ["secrets"]
  }

  eks_managed_node_groups = {
    green = {
      min_size     = 1
      max_size     = 3
      desired_size = 2
      instance_types = ["t3.medium"]
    }
  }
}
```

### 1.2 Run Static Security Analysis
Before deploying, we shift-left by scanning our Terraform code for misconfigurations.
```bash
terraform init
checkov -d . --framework terraform
```
*(Analyze the output and fix any HIGH findings before continuing).*

---

## Step 2: Deploying the Cluster
Once your code passes the security scan, provision the infrastructure. This takes roughly 15-20 minutes.
```bash
terraform apply -auto-approve
```

Once provisioning is complete, link your local terminal to the new cluster:
```bash
aws eks update-kubeconfig --region us-east-1 --name secure-cluster
```

---

## Step 3: Zero-Trust Network Policies
By default, all pods in Kubernetes can talk to each other. We will implement a "Default Deny" network policy.

Create a file called `default-deny-all.yaml`:
```yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: default-deny-all
spec:
  podSelector: {}
  policyTypes:
  - Ingress
  - Egress
```

Apply the policy to the cluster:
```bash
kubectl apply -f default-deny-all.yaml
```

---

## Step 4: Integrations in the CI/CD Pipeline
We want to actively scan any Docker images for vulnerabilities (CVEs) before they reach the cluster.

Create a `.github/workflows/devsecops-pipeline.yml` file to automate CI/CD scanning using **Trivy**:
```yaml
name: DevSecOps Pipeline

on:
  push:
    branches: [ "main" ]

jobs:
  security-scan:
    name: Security Scans
    runs-on: ubuntu-latest
    steps:
      - name: Checkout code
        uses: actions/checkout@v4

      - name: Run Checkov to scan Terraform
        uses: bridgecrewio/checkov-action@master
        with:
          directory: ./
          framework: terraform
          soft_fail: true

      - name: Run Trivy vulnerability scanner
        uses: aquasecurity/trivy-action@master
        with:
          image-ref: 'nginx:1.18' # Replace with your application image
          format: 'table'
          exit-code: '1' # Fails build on CRITICAL/HIGH 
          ignore-unfixed: true
          vuln-type: 'os,library'
          severity: 'CRITICAL,HIGH'
```

---

## Step 5: Runtime Threat Monitoring
Security doesn't end after deployment. We use **Falco** to monitor system calls dynamically for active threats (like arbitrary shells spawning heavily inside containers).

Deploy Falco using Helm:
```bash
helm repo add falcosecurity https://falcosecurity.github.io/charts
helm repo update
helm install falco falcosecurity/falco --namespace falco --create-namespace
```

You can verify the daemonset is running with:
```bash
kubectl get pods -n falco
```

---

## Conclusion & Teardown
By completing these steps, you have successfully deployed a secure end-to-end Kubernetes environment governed by strict Infrastructure-as-Code checks, deep image scanning, explicit internal network boundaries, and dynamic runtime protection.

**Don't forget to tear down the environment to avoid recurring AWS charges!**
```bash
terraform destroy --auto-approve
```
