# OpenHelp Terraform Modular AWS Infrastructure - Deep Dive

## Architecture Overview

```text
Developer
    |
terraform init
terraform plan
terraform apply
    |
    v
+----------------------+
| Root Module          |
| main.tf              |
+----------+-----------+
           |
   +-------+--------+----------------+------------------+
   |                |                |                  |
   v                v                v                  v
VPC Module      IAM Module     SG Module      EC2 Module
   |                |                |              |
   +----------------+----------------+--------------+
                            |
                            v
                     AWS Infrastructure
```

## Execution Flow

Terraform executes in dependency order:

1. versions.tf
2. providers.tf
3. variables.tf
4. main.tf
5. VPC Module
6. IAM Module
7. Security Group Module
8. EC2 Module
9. outputs.tf

---

## Project Tree

```text
openhelp-terraform-modular-aws-jumphost/
├── versions.tf
├── providers.tf
├── main.tf
├── variables.tf
├── outputs.tf
├── terraform.tfvars.example
├── modules/
│   ├── vpc/
│   ├── iam/
│   ├── security-group/
│   └── ec2-jumphost/
└── scripts/
```

# Root Files

## versions.tf

Purpose:

- Defines Terraform version
- Defines AWS provider version
- Configures S3 backend

Execution:

```text
terraform init
    |
    +--> downloads providers
    +--> connects to backend
```

---

## providers.tf

Purpose:

```hcl
provider "aws" {
  region = var.region
}
```

Creates AWS session.

---

## variables.tf

Purpose:

Central location for all configurable values.

Examples:

- Region
- VPC CIDR
- Subnet CIDRs
- Instance type
- Key pair
- IAM role names

---

## main.tf

Main orchestrator.

Calls:

```text
module.vpc
module.security_group
module.iam
module.ec2_jumphost
```

No infrastructure should be created directly here except module calls.

---

## outputs.tf

Displays useful information:

```text
VPC ID
Subnet IDs
Security Group ID
Instance ID
Public IP
```

# Module Correlation

```text
VPC Module
     |
     +--> creates VPC
     +--> creates Subnets
     +--> creates Route Tables
     |
     v
Security Group Module
     |
     v
EC2 Module
     ^
     |
IAM Module
```

EC2 depends on:

- VPC
- Subnet
- Security Group
- IAM Instance Profile

# VPC Module

Creates:

```text
VPC
Internet Gateway
Public Subnet 1
Public Subnet 2
Private Subnet 1
Private Subnet 2
Public Route Table
Private Route Table
```

Network Architecture

```text
10.0.0.0/16
|
+-- Public Subnet 1  (10.0.1.0/24)
|
+-- Public Subnet 2  (10.0.0.0/24)
|
+-- Private Subnet 1 (10.0.2.0/24)
|
+-- Private Subnet 2 (10.0.3.0/24)
```

Traffic Flow

```text
Internet
   |
Internet Gateway
   |
Public Route Table
   |
Public Subnets
```

# IAM Module

Creates:

```text
IAM Role
IAM Instance Profile
Custom EKS Policy
Managed Policy Attachments
```

Purpose:

Allows EC2 to access:

- EKS
- EC2
- CloudFormation
- IAM APIs

Flow:

```text
EC2
 |
IAM Instance Profile
 |
IAM Role
 |
AWS APIs
```

# Security Group Module

Creates:

```text
22
80
443
8080
9000
9090
3306
```

Architecture:

```text
Internet
   |
Security Group
   |
EC2
```

# EC2 Jumphost Module

Creates:

```text
EC2 Instance
Root EBS Volume
User Data Script
```

Dependencies:

```text
VPC
Subnet
Security Group
IAM Role
```

Boot Sequence:

```text
EC2 Launch
    |
Cloud Init
    |
install-tools.sh
    |
Install Software
```

# install-tools.sh Execution Flow

```text
OS Update
   |
Java
   |
NodeJS
   |
Jenkins
   |
Terraform
   |
Ansible
   |
Docker
   |
kubectl
   |
eksctl
   |
Helm
   |
SonarQube
   |
AWS CLI
```

# Complete Network Architecture

```text
                    Internet
                        |
                 Internet Gateway
                        |
                Public Route Table
                        |
        +---------------+--------------+
        |                              |
 Public Subnet 1                Public Subnet 2
        |
        |
   EC2 Jumphost
        |
 IAM Role + SG
        |
 AWS Services
```

# Terraform Lifecycle

## Initialize

```bash
terraform init
```

## Validate

```bash
terraform validate
```

## Plan

```bash
terraform plan
```

## Apply

```bash
terraform apply
```

## Destroy

```bash
terraform destroy
```

# Interview Questions

## Why use modules?

- Reusability
- Maintainability
- Standardization

## Why use IAM Instance Profile?

Allows EC2 to access AWS securely without access keys.

## Why use Public and Private Subnets?

Public:

- Internet accessible

Private:

- Internal workloads

## Why use User Data?

Automates software installation during instance startup.

# Summary

This project follows enterprise Terraform practices:

- Modular design
- Remote state
- IAM separation
- Reusable infrastructure
- Automated provisioning
- DevOps tooling bootstrap
