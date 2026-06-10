# Terraform and AWS CLI Installation Guide for Windows 10

## Overview

This guide explains how to install:

- Terraform
- AWS CLI

on Windows 10 for AWS, Terraform, Kubernetes, and DevOps projects.

---

# Part 1: Install Terraform

## Architecture

```text
Windows 10
    |
    +--> Download Terraform ZIP
    |
    +--> Extract terraform.exe
    |
    +--> Add to PATH
    |
    +--> Verify Installation
```

---

## Step 1: Download Terraform

Official Download:

https://developer.hashicorp.com/terraform/downloads

Download:

```text
Windows AMD64 (64-bit)
terraform_<version>_windows_amd64.zip
```

---

## Step 2: Extract ZIP

Create:

```text
C:\Terraform
```

Extract:

```text
terraform.exe
```

Result:

```text
C:\Terraform
└── terraform.exe
```

---

## Step 3: Add Terraform to PATH

Open:

```text
Edit the system environment variables
Press Windows Key
Search for Environment Variables
Select Edit the system environment variables
```

Navigate:

```text
Environment Variables
→ System Variables
→ Path
→ Edit
→ New
```

Add:

```text
C:\Terraform
```

Save and close.

---

## Step 4: Verify Terraform

Open PowerShell:

```powershell
terraform -version
```

Example:

```text
Terraform v1.13.1
on windows_amd64
```

Check executable:

```powershell
where terraform
```

Example:

```text
C:\Terraform\terraform.exe
```

---

# Part 2: Install AWS CLI

## Architecture

```text
Windows 10
    |
    +--> Download AWSCLIV2.msi
    |
    +--> Install MSI
    |
    +--> Configure Credentials
    |
    +--> Verify Access
```

---

## Step 1: Download AWS CLI

Official Download:

https://awscli.amazonaws.com/AWSCLIV2.msi

Documentation:

https://docs.aws.amazon.com/cli/latest/userguide/getting-started-install.html

---

## Step 2: Install AWS CLI

Run:

```text
AWSCLIV2.msi
```

Wizard:

```text
Next
Next
Install
Finish
```

Default installation:

```text
C:\Program Files\Amazon\AWSCLIV2\
```

---

## Step 3: Verify AWS CLI

Open PowerShell:

```powershell
aws --version
```

Example:

```text
aws-cli/2.x.x Python/3.x Windows/10 exe/AMD64
```

---

## Step 4: Create AWS Access Keys

AWS Console:

https://console.aws.amazon.com/

Navigate:

```text
IAM
→ Users
→ Select User
→ Security Credentials
→ Create Access Key
```

Save:

```text
Access Key ID
Secret Access Key
```

---

## Step 5: Configure AWS CLI

```powershell
aws configure
```

Example:

```text
AWS Access Key ID [None]:
AKIAxxxxxxxxxxxxxxxx

AWS Secret Access Key [None]:
xxxxxxxxxxxxxxxxxxxxxxxx

Default region name [None]:
us-east-1

Default output format [None]:
json
```

---

## Step 6: Verify Authentication

```powershell
aws sts get-caller-identity
```

Example:

```json
{
  "UserId": "AIDAxxxxxxxx",
  "Account": "123456789012",
  "Arn": "arn:aws:iam::123456789012:user/admin"
}
```

---

## Useful AWS CLI Commands

### List S3 Buckets

```powershell
aws s3 ls
```

### List EC2 Instances

```powershell
aws ec2 describe-instances
```

### List VPCs

```powershell
aws ec2 describe-vpcs
```

### List EKS Clusters

```powershell
aws eks list-clusters
```

### Show Current Identity

```powershell
aws sts get-caller-identity
```

---

# Verify Complete DevOps Workstation

```powershell
terraform -version
aws --version
```

Expected:

```text
Terraform v1.x.x
aws-cli/2.x.x
```

---

# Troubleshooting

## terraform not recognized

Verify:

```powershell
where terraform
```

Check PATH contains:

```text
C:\Terraform
```

---

## aws not recognized

Verify:

```powershell
where aws
```

Example:

```text
C:\Program Files\Amazon\AWSCLIV2\aws.exe
```

---

## Invalid AWS Credentials

Test:

```powershell
aws sts get-caller-identity
```

Reconfigure:

```powershell
aws configure
```
