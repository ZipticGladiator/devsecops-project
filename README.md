# DevSecOps Infrastructure Security Project

This repository contains a comprehensive implementation of a **Security-as-Code** infrastructure project. The project demonstrates how to build, secure, and monitor a Kubernetes-based application environment on AWS using modern DevSecOps principles and automation tools.

## 🎯 Project Goal
The primary goal of this project is to build a secure, automated infrastructure for an Amazon EKS (Elastic Kubernetes Service) cluster. Rather than applying security as an afterthought, this project integrates security gates at every stage of the infrastructure lifecycle—from writing Terraform code to runtime monitoring.

## 🏆 Expected Outcome
By transitioning from a manual setup to an automated infrastructure model, this project achieves:
- **Repeatable & Auditable Infrastructure:** Using Terraform to define architecture as code.
- **Shift-Left Security:** Catching misconfigurations early using static analysis (IaC scanning) and container image scanning before deployment.
- **Hardened Environments:** Implementing Identity and Access Management via IRSA (IAM Roles for Service Accounts) and zero-trust internal network communications via Default Deny policies.
- **Continuous Protection:** Real-time threat detection and runtime monitoring of the Kubernetes cluster.

## 🛠️ Tools & Technologies Used
### **Infrastructure & Orchestration**
- **AWS** (Cloud Provider) - Utilizing EKS, VPC, IAM, and KMS for secure resource provisioning.
- **Terraform** - Infrastructure as Code (IaC) tool used to deploy the VPC and EKS cluster.
- **Kubernetes** - Container orchestration configured with Calico for strict Network Policies.

### **Security Toolchain**
- **Checkov** - Static analysis tool used to scan Terraform code for misconfigurations prior to deployment (e.g., ensuring strict KMS encryption and public API limits).
- **Trivy** - Image vulnerability scanner integrated into the CI/CD pipeline to catch CVEs before containers are pushed to ECR.
- **Falco** - Cloud-native runtime security tool running as a DaemonSet in EKS to monitor system calls and alert on suspicious container activities (e.g., unexpected shell executions).
- **AWS Secrets Manager** - Integrated with the Secrets Store CSI Driver to securely mount secrets into Kubernetes pods without hardcoding sensitive data.

## 🚀 Implementation Phases

### Phase 1: Infrastructure as Code (IaC)
- Provisioning a secure VPC architecture (Private & Public Subnets, NAT Gateways).
- Provisioning an Amazon EKS cluster using `terraform-aws-modules`.
- Scanning the `main.tf` configuration with **Checkov** to identify supply chain and logic vulnerabilities before `terraform apply`.

### Phase 2: Securing the Cluster
- **Identity:** Enabling OIDC and IRSA to assign granular IAM permissions specifically to Pods rather than exposing the underlying EC2 nodes to broad access.
- **Encryption:** Enabling KMS to encrypt Kubernetes Secrets natively at rest.
- **Network Boundaries:** Disabling open network traffic across the cluster by creating a `default-deny-all` Kubernetes NetworkPolicy.

### Phase 3: The DevSecOps Pipeline
- Integrating **Trivy** scanning into the CI pipeline (e.g., GitHub Actions) to fail builds if HIGH or CRITICAL severity vulnerabilities are found.
- Managing application secrets dynamically through AWS Secrets Manager rather than environment variables.

### Phase 4: Runtime Security and Monitoring
- Deploying **Falco** via Helm to actively monitor container processes and file system modifications (e.g., unexpected writes to `/etc`).
- Forwarding application and security logs to central aggregating services for auditing.