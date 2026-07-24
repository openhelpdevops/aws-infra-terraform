# Day 4 Extension: HA VPC + Bastion + Jenkins/SonarQube + Private EKS + Two Terraform State Buckets

> **Beginner-friendly, production-style Terraform extension**
>
> This guide extends the existing Day 3 VPC project without redesigning its network.
> It keeps:
>
> - 1 VPC: `10.0.0.0/16`
> - 2 public subnets across `us-east-1a` and `us-east-1b`
> - 2 private subnets across `us-east-1a` and `us-east-1b`
> - 1 Internet Gateway
> - 2 NAT Gateways, one per Availability Zone
> - 1 public route table
> - 2 private route tables, each using the NAT Gateway in the same AZ
>
> It adds:
>
> - Public EC2 instance 1: bastion/administration host with AWS CLI, `kubectl`, Helm and Terraform
> - Public EC2 instance 2: Jenkins and SonarQube tools host
> - Private Amazon EKS cluster
> - 4 EKS managed worker nodes distributed across both private subnets
> - Separate IAM roles for the bastion, Jenkins/SonarQube host, EKS cluster and EKS nodes
> - Two encrypted, versioned S3 buckets:
>   - One for VPC/EC2 Terraform state
>   - One for EKS Terraform state
> - S3 state locking using Terraform's native `use_lockfile = true`
> - KMS encryption for both state buckets
>
> **Important:** Terraform backend buckets must exist before Terraform can initialize a configuration that uses them. Therefore, a small `bootstrap-state` configuration creates the two buckets first. The bootstrap configuration keeps a small local state file. The main VPC/EC2 and EKS configurations then use the two remote S3 backends.

---

## 1. Final architecture

<img width="1535" height="1024" alt="ChatGPT Image Jul 23, 2026, 06_19_36 PM" src="https://github.com/user-attachments/assets/bf46d650-0039-4a64-809f-371935f9c929" />

---

## 2. What is created

| Component | Count | Placement |
|---|---:|---|
| VPC | 1 | Regional |
| Public subnet | 2 | One per AZ |
| Private subnet | 2 | One per AZ |
| Internet Gateway | 1 | Attached to VPC |
| NAT Gateway | 2 | One per public subnet/AZ |
| Bastion EC2 | 1 | Public subnet 1 |
| Jenkins/SonarQube EC2 | 1 | Public subnet 2 |
| EKS control plane | 1 | AWS managed |
| EKS managed node group | 1 | Both private subnets |
| EKS worker EC2 instances | 4 desired | Private subnets |
| S3 backend buckets | 2 | EC2 state and EKS state |
| KMS keys for state | 2 | One per state bucket |

### Total virtual machines visible in your AWS account

```text
2 public standalone EC2 instances
+ 4 private EKS worker EC2 instances
= 6 EC2 virtual machines
```

The EKS control plane is managed by AWS and does not appear as EC2 instances in your account.

---

## 3. Recommended repository structure

```text
aws-infra-terraform/
├── bootstrap-state/
│   ├── versions.tf
│   ├── variables.tf
│   ├── main.tf
│   ├── outputs.tf
│   └── terraform.tfvars
│
├── ec2-infra/
│   ├── backend.tf
│   ├── versions.tf
│   ├── variables.tf
│   ├── network.tf
│   ├── security.tf
│   ├── iam.tf
│   ├── instances.tf
│   ├── outputs.tf
│   └── terraform.tfvars
│
└── eks-infra/
    ├── backend.tf
    ├── versions.tf
    ├── variables.tf
    ├── data.tf
    ├── iam.tf
    ├── eks.tf
    ├── outputs.tf
    └── terraform.tfvars
```

The existing Day 3 files become the `ec2-infra` configuration. The network logic remains the same. Replace the old private web VM with the second public tools VM and add the new files shown below.

---

# PART A — Bootstrap the two state buckets

## 4. `bootstrap-state/versions.tf`

```hcl
terraform {
  required_version = ">= 1.10.0"

  required_providers {
    aws = {
      source  = "hashicorp/aws"
      version = "~> 6.0"
    }
  }
}

provider "aws" {
  region = var.region

  default_tags {
    tags = {
      Project     = var.project_name
      Environment = var.environment
      ManagedBy   = "Terraform"
      Layer       = "state-bootstrap"
    }
  }
}
```

`use_lockfile = true` requires a modern Terraform release. Terraform 1.10 or newer is used here.

---

## 5. `bootstrap-state/variables.tf`

```hcl
variable "project_name" {
  description = "Project name used in resource names"
  type        = string
  default     = "openhelp"
}

variable "environment" {
  description = "Deployment environment"
  type        = string
  default     = "prod"
}

variable "region" {
  description = "AWS Region"
  type        = string
  default     = "us-east-1"
}

variable "ec2_state_bucket_name" {
  description = "Globally unique S3 bucket name for VPC and EC2 state"
  type        = string
}

variable "eks_state_bucket_name" {
  description = "Globally unique S3 bucket name for EKS state"
  type        = string
}
```

S3 bucket names are globally unique. Replace the example names with names unique to your AWS account.

---

## 6. `bootstrap-state/main.tf`

```hcl
# ---------------------------------------------------------
# KMS key for EC2/VPC Terraform state
# ---------------------------------------------------------
resource "aws_kms_key" "ec2_state" {
  description             = "KMS key for EC2 and VPC Terraform state"
  deletion_window_in_days = 7
  enable_key_rotation     = true

  tags = {
    Name = "${var.project_name}-${var.environment}-ec2-state-kms"
  }
}

resource "aws_kms_alias" "ec2_state" {
  name          = "alias/${var.project_name}-${var.environment}-ec2-state"
  target_key_id = aws_kms_key.ec2_state.key_id
}

# ---------------------------------------------------------
# KMS key for EKS Terraform state
# ---------------------------------------------------------
resource "aws_kms_key" "eks_state" {
  description             = "KMS key for EKS Terraform state"
  deletion_window_in_days = 7
  enable_key_rotation     = true

  tags = {
    Name = "${var.project_name}-${var.environment}-eks-state-kms"
  }
}

resource "aws_kms_alias" "eks_state" {
  name          = "alias/${var.project_name}-${var.environment}-eks-state"
  target_key_id = aws_kms_key.eks_state.key_id
}

# ---------------------------------------------------------
# Bucket 1: EC2/VPC Terraform state
# ---------------------------------------------------------
resource "aws_s3_bucket" "ec2_state" {
  bucket = var.ec2_state_bucket_name

  lifecycle {
    prevent_destroy = true
  }

  tags = {
    Name    = var.ec2_state_bucket_name
    Purpose = "EC2 and VPC Terraform state"
  }
}

resource "aws_s3_bucket_versioning" "ec2_state" {
  bucket = aws_s3_bucket.ec2_state.id

  versioning_configuration {
    status = "Enabled"
  }
}

resource "aws_s3_bucket_server_side_encryption_configuration" "ec2_state" {
  bucket = aws_s3_bucket.ec2_state.id

  rule {
    apply_server_side_encryption_by_default {
      sse_algorithm     = "aws:kms"
      kms_master_key_id = aws_kms_key.ec2_state.arn
    }

    bucket_key_enabled = true
  }
}

resource "aws_s3_bucket_public_access_block" "ec2_state" {
  bucket = aws_s3_bucket.ec2_state.id

  block_public_acls       = true
  block_public_policy     = true
  ignore_public_acls      = true
  restrict_public_buckets = true
}

resource "aws_s3_bucket_ownership_controls" "ec2_state" {
  bucket = aws_s3_bucket.ec2_state.id

  rule {
    object_ownership = "BucketOwnerEnforced"
  }
}

resource "aws_s3_bucket_policy" "ec2_state" {
  bucket = aws_s3_bucket.ec2_state.id

  policy = jsonencode({
    Version = "2012-10-17"
    Statement = [
      {
        Sid       = "DenyInsecureTransport"
        Effect    = "Deny"
        Principal = "*"
        Action    = "s3:*"
        Resource = [
          aws_s3_bucket.ec2_state.arn,
          "${aws_s3_bucket.ec2_state.arn}/*"
        ]
        Condition = {
          Bool = {
            "aws:SecureTransport" = "false"
          }
        }
      }
    ]
  })
}

# ---------------------------------------------------------
# Bucket 2: EKS Terraform state
# ---------------------------------------------------------
resource "aws_s3_bucket" "eks_state" {
  bucket = var.eks_state_bucket_name

  lifecycle {
    prevent_destroy = true
  }

  tags = {
    Name    = var.eks_state_bucket_name
    Purpose = "EKS Terraform state"
  }
}

resource "aws_s3_bucket_versioning" "eks_state" {
  bucket = aws_s3_bucket.eks_state.id

  versioning_configuration {
    status = "Enabled"
  }
}

resource "aws_s3_bucket_server_side_encryption_configuration" "eks_state" {
  bucket = aws_s3_bucket.eks_state.id

  rule {
    apply_server_side_encryption_by_default {
      sse_algorithm     = "aws:kms"
      kms_master_key_id = aws_kms_key.eks_state.arn
    }

    bucket_key_enabled = true
  }
}

resource "aws_s3_bucket_public_access_block" "eks_state" {
  bucket = aws_s3_bucket.eks_state.id

  block_public_acls       = true
  block_public_policy     = true
  ignore_public_acls      = true
  restrict_public_buckets = true
}

resource "aws_s3_bucket_ownership_controls" "eks_state" {
  bucket = aws_s3_bucket.eks_state.id

  rule {
    object_ownership = "BucketOwnerEnforced"
  }
}

resource "aws_s3_bucket_policy" "eks_state" {
  bucket = aws_s3_bucket.eks_state.id

  policy = jsonencode({
    Version = "2012-10-17"
    Statement = [
      {
        Sid       = "DenyInsecureTransport"
        Effect    = "Deny"
        Principal = "*"
        Action    = "s3:*"
        Resource = [
          aws_s3_bucket.eks_state.arn,
          "${aws_s3_bucket.eks_state.arn}/*"
        ]
        Condition = {
          Bool = {
            "aws:SecureTransport" = "false"
          }
        }
      }
    ]
  })
}
```

### Why `prevent_destroy` is used

A backend bucket contains Terraform's record of the infrastructure. Accidental deletion can make management and recovery much harder. To intentionally remove a state bucket, first remove or temporarily change `prevent_destroy`, after safely backing up and migrating the state.

---

## 7. `bootstrap-state/outputs.tf`

```hcl
output "ec2_state_bucket_name" {
  description = "S3 bucket used by the EC2/VPC Terraform backend"
  value       = aws_s3_bucket.ec2_state.id
}

output "ec2_state_kms_key_arn" {
  description = "KMS key ARN for the EC2/VPC state bucket"
  value       = aws_kms_key.ec2_state.arn
}

output "eks_state_bucket_name" {
  description = "S3 bucket used by the EKS Terraform backend"
  value       = aws_s3_bucket.eks_state.id
}

output "eks_state_kms_key_arn" {
  description = "KMS key ARN for the EKS state bucket"
  value       = aws_kms_key.eks_state.arn
}
```

---

## 8. `bootstrap-state/terraform.tfvars`

```hcl
project_name = "openhelp"
environment  = "prod"
region       = "us-east-1"

# These names must be globally unique.
# Add your AWS account ID or another unique suffix.
ec2_state_bucket_name = "openhelp-prod-123456789012-ec2-tfstate"
eks_state_bucket_name = "openhelp-prod-123456789012-eks-tfstate"
```

Replace `123456789012` with your AWS account ID:

```bash
aws sts get-caller-identity --query Account --output text
```

---

# PART B — Existing VPC and public EC2 infrastructure

## 9. `ec2-infra/backend.tf`

Backend blocks cannot use normal Terraform variables. Put the real bucket and KMS key values directly in this file, or supply them through `terraform init -backend-config`.

```hcl
terraform {
  backend "s3" {
    bucket       = "openhelp-prod-123456789012-ec2-tfstate"
    key          = "ec2/terraform.tfstate"
    region       = "us-east-1"
    encrypt      = true
    kms_key_id   = "alias/openhelp-prod-ec2-state"
    use_lockfile = true
  }
}
```

The final object path is:

```text
s3://openhelp-prod-123456789012-ec2-tfstate/ec2/terraform.tfstate
```

---

## 10. `ec2-infra/versions.tf`

```hcl
terraform {
  required_version = ">= 1.10.0"

  required_providers {
    aws = {
      source  = "hashicorp/aws"
      version = "~> 6.0"
    }
  }
}

provider "aws" {
  region = var.region

  default_tags {
    tags = {
      Project     = var.project_name
      Environment = var.environment
      ManagedBy   = "Terraform"
      Layer       = "ec2-infra"
    }
  }
}
```

---

## 11. Replace `ec2-infra/variables.tf`

```hcl
variable "project_name" {
  description = "Project name used in resource names and tags"
  type        = string
  default     = "openhelp"
}

variable "environment" {
  description = "Deployment environment"
  type        = string
  default     = "prod"
}

variable "region" {
  description = "AWS Region"
  type        = string
  default     = "us-east-1"
}

variable "vpc_cidr" {
  description = "IPv4 CIDR block assigned to the VPC"
  type        = string
  default     = "10.0.0.0/16"
}

variable "availability_zones" {
  description = "Exactly two Availability Zones"
  type        = list(string)
  default     = ["us-east-1a", "us-east-1b"]

  validation {
    condition     = length(var.availability_zones) == 2
    error_message = "Provide exactly two Availability Zones."
  }
}

variable "public_subnet_cidrs" {
  description = "CIDR blocks for the two public subnets"
  type        = list(string)
  default     = ["10.0.1.0/24", "10.0.2.0/24"]

  validation {
    condition     = length(var.public_subnet_cidrs) == 2
    error_message = "Provide exactly two public subnet CIDR blocks."
  }
}

variable "private_subnet_cidrs" {
  description = "CIDR blocks for the two private subnets"
  type        = list(string)
  default     = ["10.0.3.0/24", "10.0.4.0/24"]

  validation {
    condition     = length(var.private_subnet_cidrs) == 2
    error_message = "Provide exactly two private subnet CIDR blocks."
  }
}

variable "ubuntu_ami_id" {
  description = "Official Ubuntu 24.04 LTS AMI ID for the selected Region"
  type        = string

  validation {
    condition     = can(regex("^ami-[0-9a-f]+$", var.ubuntu_ami_id))
    error_message = "ubuntu_ami_id must be an AMI ID such as ami-0123456789abcdef0."
  }
}

variable "key_name" {
  description = "Existing EC2 key-pair name in the selected Region"
  type        = string
}

variable "admin_cidr_blocks" {
  description = "Approved administrator public IPv4 CIDRs"
  type        = list(string)

  validation {
    condition = (
      length(var.admin_cidr_blocks) > 0 &&
      alltrue([
        for cidr in var.admin_cidr_blocks :
        cidr != "0.0.0.0/0" && can(cidrhost(cidr, 0))
      ])
    )

    error_message = "Use restricted valid CIDRs such as 83.24.100.50/32. Do not use 0.0.0.0/0."
  }
}

variable "bastion_instance_type" {
  description = "Instance type for the bastion/kubectl host"
  type        = string
  default     = "t3.small"
}

variable "tools_instance_type" {
  description = "Instance type for Jenkins and SonarQube"
  type        = string
  default     = "t3.large"
}

variable "bastion_root_volume_size" {
  description = "Bastion root EBS volume size in GiB"
  type        = number
  default     = 20
}

variable "tools_root_volume_size" {
  description = "Jenkins/SonarQube root EBS volume size in GiB"
  type        = number
  default     = 50
}

variable "eks_cluster_name" {
  description = "Name of the EKS cluster that the EC2 IAM policies may access"
  type        = string
  default     = "openhelp-prod-eks"
}
```

---

## 12. Keep `ec2-infra/network.tf`, with these EKS subnet tags

The original Day 3 network logic is correct for two public subnets, two private subnets and two same-AZ NAT Gateways. Use the complete version below so EKS and the AWS Load Balancer Controller can identify the subnets.

```hcl
# ---------------------------------------------------------
# VPC
# ---------------------------------------------------------
resource "aws_vpc" "this" {
  cidr_block           = var.vpc_cidr
  enable_dns_support   = true
  enable_dns_hostnames = true

  tags = {
    Name = "${var.project_name}-${var.environment}-vpc"
  }
}

# ---------------------------------------------------------
# Internet Gateway
# ---------------------------------------------------------
resource "aws_internet_gateway" "this" {
  vpc_id = aws_vpc.this.id

  tags = {
    Name = "${var.project_name}-${var.environment}-igw"
  }
}

# ---------------------------------------------------------
# Two public subnets
# ---------------------------------------------------------
resource "aws_subnet" "public" {
  count = length(var.public_subnet_cidrs)

  vpc_id                  = aws_vpc.this.id
  cidr_block              = var.public_subnet_cidrs[count.index]
  availability_zone       = var.availability_zones[count.index]
  map_public_ip_on_launch = true

  tags = {
    Name                     = "${var.project_name}-${var.environment}-public-${count.index + 1}"
    Tier                     = "public"
    "kubernetes.io/role/elb" = "1"
  }
}

# ---------------------------------------------------------
# Two private subnets for EKS nodes
# ---------------------------------------------------------
resource "aws_subnet" "private" {
  count = length(var.private_subnet_cidrs)

  vpc_id                  = aws_vpc.this.id
  cidr_block              = var.private_subnet_cidrs[count.index]
  availability_zone       = var.availability_zones[count.index]
  map_public_ip_on_launch = false

  tags = {
    Name                              = "${var.project_name}-${var.environment}-private-${count.index + 1}"
    Tier                              = "private"
    "kubernetes.io/role/internal-elb" = "1"
  }
}

# ---------------------------------------------------------
# Shared public route table
# ---------------------------------------------------------
resource "aws_route_table" "public" {
  vpc_id = aws_vpc.this.id

  route {
    cidr_block = "0.0.0.0/0"
    gateway_id = aws_internet_gateway.this.id
  }

  tags = {
    Name = "${var.project_name}-${var.environment}-public-rt"
  }
}

resource "aws_route_table_association" "public" {
  count = length(aws_subnet.public)

  subnet_id      = aws_subnet.public[count.index].id
  route_table_id = aws_route_table.public.id
}

# ---------------------------------------------------------
# Two Elastic IPs
# ---------------------------------------------------------
resource "aws_eip" "nat" {
  count = length(var.public_subnet_cidrs)

  domain = "vpc"

  tags = {
    Name = "${var.project_name}-${var.environment}-nat-eip-${count.index + 1}"
  }

  depends_on = [aws_internet_gateway.this]
}

# ---------------------------------------------------------
# Two NAT Gateways
# ---------------------------------------------------------
resource "aws_nat_gateway" "this" {
  count = length(var.public_subnet_cidrs)

  allocation_id = aws_eip.nat[count.index].id
  subnet_id     = aws_subnet.public[count.index].id

  tags = {
    Name = "${var.project_name}-${var.environment}-nat-${count.index + 1}"
  }

  depends_on = [
    aws_internet_gateway.this,
    aws_route_table_association.public
  ]
}

# ---------------------------------------------------------
# Two private route tables; each uses its same-AZ NAT
# ---------------------------------------------------------
resource "aws_route_table" "private" {
  count = length(var.private_subnet_cidrs)

  vpc_id = aws_vpc.this.id

  route {
    cidr_block     = "0.0.0.0/0"
    nat_gateway_id = aws_nat_gateway.this[count.index].id
  }

  tags = {
    Name = "${var.project_name}-${var.environment}-private-rt-${count.index + 1}"
  }
}

resource "aws_route_table_association" "private" {
  count = length(aws_subnet.private)

  subnet_id      = aws_subnet.private[count.index].id
  route_table_id = aws_route_table.private[count.index].id
}
```

### Important subnet behavior

```text
Private subnet 1 (us-east-1a) → Private route table 1 → NAT Gateway 1
Private subnet 2 (us-east-1b) → Private route table 2 → NAT Gateway 2
```

The EKS nodes do not receive public IP addresses. They download images and updates through their same-AZ NAT Gateway.

---

## 13. Replace `ec2-infra/security.tf`

```hcl
# ---------------------------------------------------------
# Bastion security group
# ---------------------------------------------------------
resource "aws_security_group" "bastion" {
  name        = "${var.project_name}-${var.environment}-bastion-sg"
  description = "Security group for bastion and Kubernetes administration"
  vpc_id      = aws_vpc.this.id

  tags = {
    Name = "${var.project_name}-${var.environment}-bastion-sg"
  }
}

resource "aws_vpc_security_group_ingress_rule" "bastion_ssh" {
  for_each = toset(var.admin_cidr_blocks)

  security_group_id = aws_security_group.bastion.id
  description       = "SSH from approved administrator IP"
  cidr_ipv4         = each.value
  from_port         = 22
  to_port           = 22
  ip_protocol       = "tcp"
}

resource "aws_vpc_security_group_egress_rule" "bastion_all_outbound" {
  security_group_id = aws_security_group.bastion.id
  description       = "Allow outbound traffic"
  cidr_ipv4         = "0.0.0.0/0"
  ip_protocol       = "-1"
}

# ---------------------------------------------------------
# Jenkins and SonarQube security group
# ---------------------------------------------------------
resource "aws_security_group" "tools" {
  name        = "${var.project_name}-${var.environment}-tools-sg"
  description = "Security group for Jenkins and SonarQube"
  vpc_id      = aws_vpc.this.id

  tags = {
    Name = "${var.project_name}-${var.environment}-tools-sg"
  }
}

resource "aws_vpc_security_group_ingress_rule" "tools_ssh" {
  for_each = toset(var.admin_cidr_blocks)

  security_group_id = aws_security_group.tools.id
  description       = "SSH from approved administrator IP"
  cidr_ipv4         = each.value
  from_port         = 22
  to_port           = 22
  ip_protocol       = "tcp"
}

resource "aws_vpc_security_group_ingress_rule" "jenkins_web" {
  for_each = toset(var.admin_cidr_blocks)

  security_group_id = aws_security_group.tools.id
  description       = "Jenkins web interface from approved administrator IP"
  cidr_ipv4         = each.value
  from_port         = 8080
  to_port           = 8080
  ip_protocol       = "tcp"
}

resource "aws_vpc_security_group_ingress_rule" "sonarqube_web" {
  for_each = toset(var.admin_cidr_blocks)

  security_group_id = aws_security_group.tools.id
  description       = "SonarQube web interface from approved administrator IP"
  cidr_ipv4         = each.value
  from_port         = 9000
  to_port           = 9000
  ip_protocol       = "tcp"
}

resource "aws_vpc_security_group_egress_rule" "tools_all_outbound" {
  security_group_id = aws_security_group.tools.id
  description       = "Allow package downloads, Git access and AWS APIs"
  cidr_ipv4         = "0.0.0.0/0"
  ip_protocol       = "-1"
}
```

> For a real production environment, place Jenkins and SonarQube behind an HTTPS ALB, use Route 53 and ACM, and avoid exposing ports `8080` and `9000` directly. This restricted-IP design is suitable for a controlled lab.

---

## 14. Add `ec2-infra/iam.tf`

The bastion needs permission to describe EKS and build a kubeconfig. Jenkins needs EKS access for deployment and ECR access for container images.

### Beginner-friendly order used in this file

The blocks below are arranged in the same order as the IAM story, you may refer the below image to understand it better.

```text
1. Find the AWS account and partition
2. Build the exact EKS cluster ARN
3. Write the shared trust rule: EC2 may assume the role
4. Create the bastion role and connect the trust rule to it
5. Give the bastion its job permissions
6. Put the bastion role in an instance profile
7. Create the tools role and connect the same trust rule to it
8. Give the tools role its job permissions and instance profile
```

<img width="1536" height="1024" alt="ChatGPT Image Jul 24, 2026, 05_04_03 PM" src="https://github.com/user-attachments/assets/3ea3d1a3-7959-4000-adf1-8b442e5cef7b" />



The most important connection is:

```hcl
assume_role_policy = data.aws_iam_policy_document.ec2_assume_role.json
```

That line attaches the trust rule created earlier to the IAM role.

```hcl
# =========================================================
# STEP 1: Find the current AWS account and partition
# Example account ID: 123456789012
# Example partition:  aws
# =========================================================
data "aws_caller_identity" "current" {}

data "aws_partition" "current" {}

# =========================================================
# STEP 2: Build the exact ARN of the EKS cluster
# =========================================================
locals {
  eks_cluster_arn = "arn:${data.aws_partition.current.partition}:eks:${var.region}:${data.aws_caller_identity.current.account_id}:cluster/${var.eks_cluster_name}"
}

# =========================================================
# STEP 3: Write one shared trust rule
# This does not create an employee/role.
# It only says: EC2 is allowed to assume roles that use it.
# =========================================================
data "aws_iam_policy_document" "ec2_assume_role" {
  statement {
    effect  = "Allow"
    actions = ["sts:AssumeRole"]

    principals {
      type        = "Service"
      identifiers = ["ec2.amazonaws.com"]
    }
  }
}

# ---------------------------------------------------------
# BASTION IAM FLOW
# ---------------------------------------------------------

# =========================================================
# STEP 4: Create the bastion IAM role
# The assume_role_policy line connects STEP 3 to this role.
# =========================================================
resource "aws_iam_role" "bastion" {
  name               = "${var.project_name}-${var.environment}-bastion-role"
  assume_role_policy = data.aws_iam_policy_document.ec2_assume_role.json
}

# =========================================================
# STEP 5A: Give the bastion permission to describe EKS
# This is an inline policy stored directly inside the role.
# =========================================================
resource "aws_iam_role_policy" "bastion_eks" {
  name = "${var.project_name}-${var.environment}-bastion-eks"
  role = aws_iam_role.bastion.id

  policy = jsonencode({
    Version = "2012-10-17"
    Statement = [
      {
        Sid      = "DescribeCluster"
        Effect   = "Allow"
        Action   = ["eks:DescribeCluster"]
        Resource = local.eks_cluster_arn
      }
    ]
  })
}

# =========================================================
# STEP 5B: Attach the AWS-managed SSM policy to the bastion
# =========================================================
resource "aws_iam_role_policy_attachment" "bastion_ssm" {
  role       = aws_iam_role.bastion.name
  policy_arn = "arn:${data.aws_partition.current.partition}:iam::aws:policy/AmazonSSMManagedInstanceCore"
}

# =========================================================
# STEP 6: Put the bastion role in an instance profile
# The EC2 instance receives this profile later in instances.tf.
# =========================================================
resource "aws_iam_instance_profile" "bastion" {
  name = "${var.project_name}-${var.environment}-bastion-profile"
  role = aws_iam_role.bastion.name
}

# ---------------------------------------------------------
# JENKINS/SONARQUBE TOOLS IAM FLOW
# ---------------------------------------------------------

# =========================================================
# STEP 7: Create the tools IAM role
# It reuses the same EC2 trust rule from STEP 3.
# =========================================================
resource "aws_iam_role" "tools" {
  name               = "${var.project_name}-${var.environment}-tools-role"
  assume_role_policy = data.aws_iam_policy_document.ec2_assume_role.json
}

# =========================================================
# STEP 8A: Give the tools role permission to describe EKS
# =========================================================
resource "aws_iam_role_policy" "tools_eks" {
  name = "${var.project_name}-${var.environment}-tools-eks"
  role = aws_iam_role.tools.id

  policy = jsonencode({
    Version = "2012-10-17"
    Statement = [
      {
        Sid      = "DescribeCluster"
        Effect   = "Allow"
        Action   = ["eks:DescribeCluster"]
        Resource = local.eks_cluster_arn
      }
    ]
  })
}

# =========================================================
# STEP 8B: Attach ECR PowerUser permission to the tools role
# Jenkins can push and pull container images.
# =========================================================
resource "aws_iam_role_policy_attachment" "tools_ecr" {
  role       = aws_iam_role.tools.name
  policy_arn = "arn:${data.aws_partition.current.partition}:iam::aws:policy/AmazonEC2ContainerRegistryPowerUser"
}

# =========================================================
# STEP 8C: Attach SSM permission to the tools role
# =========================================================
resource "aws_iam_role_policy_attachment" "tools_ssm" {
  role       = aws_iam_role.tools.name
  policy_arn = "arn:${data.aws_partition.current.partition}:iam::aws:policy/AmazonSSMManagedInstanceCore"
}

# =========================================================
# STEP 8D: Put the tools role in an instance profile
# =========================================================
resource "aws_iam_instance_profile" "tools" {
  name = "${var.project_name}-${var.environment}-tools-profile"
  role = aws_iam_role.tools.name
}
```

### Simple correlation map

```text
Shared EC2 trust rule
aws_iam_policy_document.ec2_assume_role
               |
               | .json is assigned to assume_role_policy
               +--------------------------+
               |                          |
               v                          v
aws_iam_role.bastion            aws_iam_role.tools
       |                                  |
       | permissions                      | permissions
       v                                  v
Describe EKS + SSM              Describe EKS + ECR + SSM
       |                                  |
       v                                  v
bastion instance profile        tools instance profile
       |                                  |
       v                                  v
Bastion EC2                     Jenkins/SonarQube EC2
```

IAM permission to call `eks:DescribeCluster` is only the AWS API part. Kubernetes authorization is configured later using EKS access entries.

## 15. Replace `ec2-infra/instances.tf`

This creates exactly two standalone EC2 instances, both in public subnets.

```hcl
# ---------------------------------------------------------
# Public EC2 1: Bastion, kubectl and Terraform host
# ---------------------------------------------------------
resource "aws_instance" "bastion" {
  ami                         = var.ubuntu_ami_id
  instance_type               = var.bastion_instance_type
  subnet_id                   = aws_subnet.public[0].id
  vpc_security_group_ids      = [aws_security_group.bastion.id]
  key_name                    = var.key_name
  associate_public_ip_address = true
  iam_instance_profile        = aws_iam_instance_profile.bastion.name

  metadata_options {
    http_endpoint = "enabled"
    http_tokens   = "required"
  }

  root_block_device {
    volume_type           = "gp3"
    volume_size           = var.bastion_root_volume_size
    encrypted             = true
    delete_on_termination = true
  }

  user_data = <<-EOF
    #!/bin/bash
    set -euxo pipefail

    export DEBIAN_FRONTEND=noninteractive

    apt-get update -y
    apt-get install -y \
      unzip \
      curl \
      jq \
      git \
      ca-certificates \
      apt-transport-https \
      gnupg \
      lsb-release

    # AWS CLI v2
    cd /tmp
    curl -fsSLo awscliv2.zip https://awscli.amazonaws.com/awscli-exe-linux-x86_64.zip
    unzip -q awscliv2.zip
    ./aws/install --update

    # kubectl: use the current stable release
    KUBECTL_VERSION="$(curl -fsSL https://dl.k8s.io/release/stable.txt)"
    curl -fsSLo /usr/local/bin/kubectl \
      "https://dl.k8s.io/release/$${KUBECTL_VERSION}/bin/linux/amd64/kubectl"
    chmod +x /usr/local/bin/kubectl

    # Helm 3
    curl -fsSL https://raw.githubusercontent.com/helm/helm/main/scripts/get-helm-3 | bash

    # Terraform from HashiCorp's APT repository
    curl -fsSL https://apt.releases.hashicorp.com/gpg \
      | gpg --dearmor -o /usr/share/keyrings/hashicorp-archive-keyring.gpg

    echo "deb [signed-by=/usr/share/keyrings/hashicorp-archive-keyring.gpg] \
    https://apt.releases.hashicorp.com $(lsb_release -cs) main" \
      > /etc/apt/sources.list.d/hashicorp.list

    apt-get update -y
    apt-get install -y terraform

    cat >/etc/profile.d/openhelp-tools.sh <<'PROFILE'
    export AWS_REGION="us-east-1"
    export AWS_DEFAULT_REGION="us-east-1"
    PROFILE

    chmod 644 /etc/profile.d/openhelp-tools.sh
  EOF

  user_data_replace_on_change = true

  tags = {
    Name = "${var.project_name}-${var.environment}-bastion"
    Role = "bastion-kubectl"
  }

  depends_on = [
    aws_internet_gateway.this,
    aws_route_table_association.public
  ]
}

# ---------------------------------------------------------
# Public EC2 2: Jenkins and SonarQube host
# ---------------------------------------------------------
resource "aws_instance" "tools" {
  ami                         = var.ubuntu_ami_id
  instance_type               = var.tools_instance_type
  subnet_id                   = aws_subnet.public[1].id
  vpc_security_group_ids      = [aws_security_group.tools.id]
  key_name                    = var.key_name
  associate_public_ip_address = true
  iam_instance_profile        = aws_iam_instance_profile.tools.name

  metadata_options {
    http_endpoint = "enabled"
    http_tokens   = "required"
  }

  root_block_device {
    volume_type           = "gp3"
    volume_size           = var.tools_root_volume_size
    encrypted             = true
    delete_on_termination = true
  }

  user_data = <<-EOF
    #!/bin/bash
    set -euxo pipefail

    export DEBIAN_FRONTEND=noninteractive

    apt-get update -y
    apt-get install -y \
      unzip \
      curl \
      jq \
      git \
      ca-certificates \
      fontconfig \
      openjdk-17-jre \
      docker.io

    systemctl enable --now docker

    # SonarQube/Elasticsearch Linux prerequisites
    cat >/etc/sysctl.d/99-sonarqube.conf <<'SYSCTL'
    vm.max_map_count=524288
    fs.file-max=131072
    SYSCTL
    sysctl --system

    cat >/etc/security/limits.d/99-sonarqube.conf <<'LIMITS'
    * soft nofile 131072
    * hard nofile 131072
    * soft nproc 8192
    * hard nproc 8192
    LIMITS

    # AWS CLI v2
    cd /tmp
    curl -fsSLo awscliv2.zip https://awscli.amazonaws.com/awscli-exe-linux-x86_64.zip
    unzip -q awscliv2.zip
    ./aws/install --update

    # kubectl
    KUBECTL_VERSION="$(curl -fsSL https://dl.k8s.io/release/stable.txt)"
    curl -fsSLo /usr/local/bin/kubectl \
      "https://dl.k8s.io/release/$${KUBECTL_VERSION}/bin/linux/amd64/kubectl"
    chmod +x /usr/local/bin/kubectl

    # Jenkins repository and package
    install -m 0755 -d /etc/apt/keyrings
    curl -fsSL https://pkg.jenkins.io/debian-stable/jenkins.io-2026.key \
      -o /etc/apt/keyrings/jenkins-keyring.asc

    echo "deb [signed-by=/etc/apt/keyrings/jenkins-keyring.asc] \
    https://pkg.jenkins.io/debian-stable binary/" \
      > /etc/apt/sources.list.d/jenkins.list

    apt-get update -y
    apt-get install -y jenkins
    usermod -aG docker jenkins
    systemctl enable --now jenkins

    # SonarQube lab container.
    # For production, use PostgreSQL and persistent storage.
    docker volume create sonarqube_data
    docker volume create sonarqube_logs
    docker volume create sonarqube_extensions

    docker run -d \
      --name sonarqube \
      --restart unless-stopped \
      -p 9000:9000 \
      -v sonarqube_data:/opt/sonarqube/data \
      -v sonarqube_logs:/opt/sonarqube/logs \
      -v sonarqube_extensions:/opt/sonarqube/extensions \
      sonarqube:lts-community
  EOF

  user_data_replace_on_change = true

  tags = {
    Name = "${var.project_name}-${var.environment}-jenkins-sonarqube"
    Role = "ci-code-quality"
  }

  depends_on = [
    aws_internet_gateway.this,
    aws_route_table_association.public
  ]
}
```

### Important SonarQube note

Running SonarQube using its embedded database/container arrangement is for learning only. A production SonarQube deployment should use:

- A supported external PostgreSQL database
- Persistent and backed-up storage
- HTTPS through an ALB or reverse proxy
- Secrets managed outside the script
- Monitoring and patching
- A larger instance selected from SonarQube sizing requirements

---

## 16. Replace `ec2-infra/outputs.tf`

The EKS project reads several of these outputs through `terraform_remote_state`.

```hcl
output "vpc_id" {
  description = "VPC ID"
  value       = aws_vpc.this.id
}

output "vpc_cidr" {
  description = "VPC IPv4 CIDR"
  value       = aws_vpc.this.cidr_block
}

output "public_subnet_ids" {
  description = "Two public subnet IDs"
  value       = aws_subnet.public[*].id
}

output "private_subnet_ids" {
  description = "Two private subnet IDs used by EKS"
  value       = aws_subnet.private[*].id
}

output "bastion_public_ip" {
  description = "Public IPv4 address of the bastion"
  value       = aws_instance.bastion.public_ip
}

output "bastion_private_ip" {
  description = "Private IPv4 address of the bastion"
  value       = aws_instance.bastion.private_ip
}

output "tools_public_ip" {
  description = "Public IPv4 address of Jenkins/SonarQube"
  value       = aws_instance.tools.public_ip
}

output "tools_private_ip" {
  description = "Private IPv4 address of Jenkins/SonarQube"
  value       = aws_instance.tools.private_ip
}

output "bastion_role_arn" {
  description = "IAM role ARN used by the bastion"
  value       = aws_iam_role.bastion.arn
}

output "tools_role_arn" {
  description = "IAM role ARN used by Jenkins/SonarQube"
  value       = aws_iam_role.tools.arn
}

output "nat_gateway_ids" {
  description = "NAT Gateway IDs"
  value       = aws_nat_gateway.this[*].id
}

output "nat_gateway_public_ips" {
  description = "NAT Gateway Elastic IPs"
  value       = aws_eip.nat[*].public_ip
}

output "ssh_to_bastion" {
  description = "Example SSH command"
  value       = "ssh -i openhelp-key.pem ubuntu@${aws_instance.bastion.public_ip}"
}

output "ssh_to_tools" {
  description = "Example SSH command"
  value       = "ssh -i openhelp-key.pem ubuntu@${aws_instance.tools.public_ip}"
}

output "jenkins_url" {
  description = "Jenkins lab URL"
  value       = "http://${aws_instance.tools.public_ip}:8080"
}

output "sonarqube_url" {
  description = "SonarQube lab URL"
  value       = "http://${aws_instance.tools.public_ip}:9000"
}
```

---

## 17. `ec2-infra/terraform.tfvars`

```hcl
project_name = "openhelp"
environment  = "prod"
region       = "us-east-1"

vpc_cidr = "10.0.0.0/16"

availability_zones = [
  "us-east-1a",
  "us-east-1b"
]

public_subnet_cidrs = [
  "10.0.1.0/24",
  "10.0.2.0/24"
]

private_subnet_cidrs = [
  "10.0.3.0/24",
  "10.0.4.0/24"
]

# Replace this with the current Canonical Ubuntu 24.04 AMI in us-east-1.
ubuntu_ami_id = "ami-REPLACE_ME"

key_name = "openhelp-key"

# Replace with your public IP/32.
admin_cidr_blocks = [
  "83.24.100.50/32"
]

bastion_instance_type = "t3.small"
tools_instance_type   = "t3.large"

bastion_root_volume_size = 20
tools_root_volume_size   = 50

eks_cluster_name = "openhelp-prod-eks"
```

Find the current Ubuntu AMI:

```bash
aws ec2 describe-images \
  --region us-east-1 \
  --owners 099720109477 \
  --filters \
    "Name=name,Values=ubuntu/images/hvm-ssd-gp3/ubuntu-noble-24.04-amd64-server-*" \
    "Name=architecture,Values=x86_64" \
    "Name=root-device-type,Values=ebs" \
    "Name=virtualization-type,Values=hvm" \
    "Name=state,Values=available" \
  --query 'sort_by(Images, &CreationDate)[-1].ImageId' \
  --output text
```

---

# PART C — Private EKS infrastructure

## 18. `eks-infra/backend.tf`

```hcl
terraform {
  backend "s3" {
    bucket       = "openhelp-prod-123456789012-eks-tfstate"
    key          = "eks/terraform.tfstate"
    region       = "us-east-1"
    encrypt      = true
    kms_key_id   = "alias/openhelp-prod-eks-state"
    use_lockfile = true
  }
}
```

The EKS state is separate from the VPC/EC2 state:

```text
s3://openhelp-prod-123456789012-eks-tfstate/eks/terraform.tfstate
```

---

## 19. `eks-infra/versions.tf`

```hcl
terraform {
  required_version = ">= 1.10.0"

  required_providers {
    aws = {
      source  = "hashicorp/aws"
      version = "~> 6.0"
    }
    tls = {
      source  = "hashicorp/tls"
      version = "~> 4.0"
    }
  }
}

provider "aws" {
  region = var.region

  default_tags {
    tags = {
      Project     = var.project_name
      Environment = var.environment
      ManagedBy   = "Terraform"
      Layer       = "eks-infra"
    }
  }
}
```

---

## 20. `eks-infra/variables.tf`

```hcl
variable "project_name" {
  description = "Project name"
  type        = string
  default     = "openhelp"
}

variable "environment" {
  description = "Deployment environment"
  type        = string
  default     = "prod"
}

variable "region" {
  description = "AWS Region"
  type        = string
  default     = "us-east-1"
}

variable "cluster_name" {
  description = "EKS cluster name"
  type        = string
  default     = "openhelp-prod-eks"
}

variable "kubernetes_version" {
  description = "Supported EKS Kubernetes version"
  type        = string
  default     = "1.36"
}

variable "ec2_state_bucket_name" {
  description = "Bucket containing VPC/EC2 Terraform state"
  type        = string
}

variable "ec2_state_key" {
  description = "S3 object key of VPC/EC2 Terraform state"
  type        = string
  default     = "ec2/terraform.tfstate"
}

variable "ec2_state_kms_key_id" {
  description = "KMS key or alias used by the VPC/EC2 backend"
  type        = string
  default     = "alias/openhelp-prod-ec2-state"
}

variable "admin_cidr_blocks" {
  description = "CIDRs allowed to reach the public EKS API endpoint"
  type        = list(string)

  validation {
    condition = (
      length(var.admin_cidr_blocks) > 0 &&
      alltrue([
        for cidr in var.admin_cidr_blocks :
        cidr != "0.0.0.0/0" && can(cidrhost(cidr, 0))
      ])
    )
    error_message = "Use restricted valid CIDRs; do not expose the EKS endpoint to 0.0.0.0/0."
  }
}

variable "node_instance_types" {
  description = "Allowed EC2 instance types for the managed node group"
  type        = list(string)
  default     = ["t3.medium"]
}

variable "node_desired_size" {
  description = "Desired EKS worker-node count"
  type        = number
  default     = 4
}

variable "node_min_size" {
  description = "Minimum EKS worker-node count"
  type        = number
  default     = 2
}

variable "node_max_size" {
  description = "Maximum EKS worker-node count"
  type        = number
  default     = 6
}

variable "node_disk_size" {
  description = "Worker-node root volume size in GiB"
  type        = number
  default     = 30
}
```

> The example uses Kubernetes `1.36`, which was in Amazon EKS standard support when this guide was prepared in July 2026. Verify support again before applying because EKS-supported versions change over time.

---

## 21. `eks-infra/data.tf`

This configuration reads the VPC and EC2 outputs from the first S3 bucket.

```hcl
data "terraform_remote_state" "ec2" {
  backend = "s3"

  config = {
    bucket       = var.ec2_state_bucket_name
    key          = var.ec2_state_key
    region       = var.region
    encrypt      = true
    kms_key_id   = var.ec2_state_kms_key_id
    use_lockfile = true
  }
}

data "aws_caller_identity" "current" {}

data "aws_partition" "current" {}

locals {
  vpc_id             = data.terraform_remote_state.ec2.outputs.vpc_id
  private_subnet_ids = data.terraform_remote_state.ec2.outputs.private_subnet_ids
  bastion_role_arn   = data.terraform_remote_state.ec2.outputs.bastion_role_arn
  tools_role_arn     = data.terraform_remote_state.ec2.outputs.tools_role_arn
}
```

### Why remote state is needed

The EKS configuration is stored in a different state file, but it needs the VPC ID and private subnet IDs created by the EC2/VPC configuration:

```text
EC2/VPC state bucket
        |
        | terraform_remote_state
        v
EKS configuration receives:
- vpc_id
- private_subnet_ids
- bastion_role_arn
- tools_role_arn
```

---

## 22. `eks-infra/iam.tf`

This IAM file is also arranged in beginner-friendly dependency order:

```text
EKS service trust rule → EKS control-plane role → Cluster policy
EC2 service trust rule → EKS node role → Worker, CNI and ECR policies
```

The trust policy always comes before the role that references it.

```hcl
# ---------------------------------------------------------
# EKS CONTROL-PLANE IAM FLOW
# ---------------------------------------------------------

# =========================================================
# STEP 1: Write the trust rule for the EKS service
# This rule says: eks.amazonaws.com may assume the role.
# =========================================================
data "aws_iam_policy_document" "eks_cluster_assume_role" {
  statement {
    effect  = "Allow"
    actions = ["sts:AssumeRole"]

    principals {
      type        = "Service"
      identifiers = ["eks.amazonaws.com"]
    }
  }
}

# =========================================================
# STEP 2: Create the EKS control-plane role
# This line connects the trust rule above to the role:
# assume_role_policy = data.aws_iam_policy_document...
# =========================================================
resource "aws_iam_role" "eks_cluster" {
  name               = "${var.cluster_name}-cluster-role"
  assume_role_policy = data.aws_iam_policy_document.eks_cluster_assume_role.json
}

# =========================================================
# STEP 3: Give the control-plane role its AWS permissions
# =========================================================
resource "aws_iam_role_policy_attachment" "eks_cluster_policy" {
  role       = aws_iam_role.eks_cluster.name
  policy_arn = "arn:${data.aws_partition.current.partition}:iam::aws:policy/AmazonEKSClusterPolicy"
}

# ---------------------------------------------------------
# EKS MANAGED-NODE IAM FLOW
# ---------------------------------------------------------

# =========================================================
# STEP 4: Write the trust rule for EC2 worker nodes
# This rule says: ec2.amazonaws.com may assume the node role.
# =========================================================
data "aws_iam_policy_document" "eks_node_assume_role" {
  statement {
    effect  = "Allow"
    actions = ["sts:AssumeRole"]

    principals {
      type        = "Service"
      identifiers = ["ec2.amazonaws.com"]
    }
  }
}

# =========================================================
# STEP 5: Create the EKS worker-node role
# This line connects the EC2 trust rule above to the role.
# =========================================================
resource "aws_iam_role" "eks_nodes" {
  name               = "${var.cluster_name}-node-role"
  assume_role_policy = data.aws_iam_policy_document.eks_node_assume_role.json
}

# =========================================================
# STEP 6A: Allow worker nodes to communicate with EKS
# =========================================================
resource "aws_iam_role_policy_attachment" "eks_worker_node" {
  role       = aws_iam_role.eks_nodes.name
  policy_arn = "arn:${data.aws_partition.current.partition}:iam::aws:policy/AmazonEKSWorkerNodePolicy"
}

# =========================================================
# STEP 6B: Allow the VPC CNI plugin to manage pod networking
# =========================================================
resource "aws_iam_role_policy_attachment" "eks_cni" {
  role       = aws_iam_role.eks_nodes.name
  policy_arn = "arn:${data.aws_partition.current.partition}:iam::aws:policy/AmazonEKS_CNI_Policy"
}

# =========================================================
# STEP 6C: Allow worker nodes to pull images from ECR
# =========================================================
resource "aws_iam_role_policy_attachment" "eks_ecr_read_only" {
  role       = aws_iam_role.eks_nodes.name
  policy_arn = "arn:${data.aws_partition.current.partition}:iam::aws:policy/AmazonEC2ContainerRegistryPullOnly"
}
```

### Simple correlation map

```text
Trust rule: EKS service
        |
        | attached through assume_role_policy
        v
EKS control-plane role
        |
        v
AmazonEKSClusterPolicy

Trust rule: EC2 service
        |
        | attached through assume_role_policy
        v
EKS worker-node role
        |
        +--> AmazonEKSWorkerNodePolicy
        +--> AmazonEKS_CNI_Policy
        +--> AmazonEC2ContainerRegistryPullOnly
```

## 23. `eks-infra/eks.tf`

```hcl
# ---------------------------------------------------------
# KMS key for Kubernetes secret encryption
# ---------------------------------------------------------
resource "aws_kms_key" "eks_secrets" {
  description             = "KMS key for ${var.cluster_name} Kubernetes secrets"
  deletion_window_in_days = 7
  enable_key_rotation     = true

  tags = {
    Name = "${var.cluster_name}-secrets-kms"
  }
}

resource "aws_kms_alias" "eks_secrets" {
  name          = "alias/${var.cluster_name}-secrets"
  target_key_id = aws_kms_key.eks_secrets.key_id
}

# ---------------------------------------------------------
# CloudWatch log group
# ---------------------------------------------------------
resource "aws_cloudwatch_log_group" "eks" {
  name              = "/aws/eks/${var.cluster_name}/cluster"
  retention_in_days = 30

  tags = {
    Name = "${var.cluster_name}-control-plane-logs"
  }
}

# ---------------------------------------------------------
# EKS cluster
# ---------------------------------------------------------
resource "aws_eks_cluster" "this" {
  name     = var.cluster_name
  role_arn = aws_iam_role.eks_cluster.arn
  version  = var.kubernetes_version

  access_config {
    authentication_mode                         = "API_AND_CONFIG_MAP"
    bootstrap_cluster_creator_admin_permissions = true
  }

  vpc_config {
    subnet_ids              = local.private_subnet_ids
    endpoint_private_access = true
    endpoint_public_access  = true
    public_access_cidrs     = var.admin_cidr_blocks
  }

  encryption_config {
    provider {
      key_arn = aws_kms_key.eks_secrets.arn
    }

    resources = ["secrets"]
  }

  enabled_cluster_log_types = [
    "api",
    "audit",
    "authenticator",
    "controllerManager",
    "scheduler"
  ]

  depends_on = [
    aws_iam_role_policy_attachment.eks_cluster_policy,
    aws_cloudwatch_log_group.eks
  ]

  tags = {
    Name = var.cluster_name
  }
}

# ---------------------------------------------------------
# Managed node group: four desired private workers
# ---------------------------------------------------------
resource "aws_eks_node_group" "general" {
  cluster_name    = aws_eks_cluster.this.name
  node_group_name = "${var.cluster_name}-general"
  node_role_arn   = aws_iam_role.eks_nodes.arn
  subnet_ids      = local.private_subnet_ids

  instance_types = var.node_instance_types
  capacity_type  = "ON_DEMAND"
  disk_size      = var.node_disk_size

  scaling_config {
    desired_size = var.node_desired_size
    min_size     = var.node_min_size
    max_size     = var.node_max_size
  }

  update_config {
    max_unavailable_percentage = 25
  }

  labels = {
    workload = "general"
  }

  depends_on = [
    aws_iam_role_policy_attachment.eks_worker_node,
    aws_iam_role_policy_attachment.eks_cni,
    aws_iam_role_policy_attachment.eks_ecr_read_only
  ]

  tags = {
    Name = "${var.cluster_name}-general"
  }
}

# ---------------------------------------------------------
# Grant bastion cluster-admin access
# ---------------------------------------------------------
resource "aws_eks_access_entry" "bastion" {
  cluster_name  = aws_eks_cluster.this.name
  principal_arn = local.bastion_role_arn
  type          = "STANDARD"
}

resource "aws_eks_access_policy_association" "bastion_admin" {
  cluster_name  = aws_eks_cluster.this.name
  principal_arn = local.bastion_role_arn

  policy_arn = "arn:${data.aws_partition.current.partition}:eks::aws:cluster-access-policy/AmazonEKSClusterAdminPolicy"

  access_scope {
    type = "cluster"
  }

  depends_on = [aws_eks_access_entry.bastion]
}

# ---------------------------------------------------------
# Grant Jenkins tools role deployment access
# This lab uses cluster-admin for simplicity.
# Use namespace-scoped access in production.
# ---------------------------------------------------------
resource "aws_eks_access_entry" "tools" {
  cluster_name  = aws_eks_cluster.this.name
  principal_arn = local.tools_role_arn
  type          = "STANDARD"
}

resource "aws_eks_access_policy_association" "tools_admin" {
  cluster_name  = aws_eks_cluster.this.name
  principal_arn = local.tools_role_arn

  policy_arn = "arn:${data.aws_partition.current.partition}:eks::aws:cluster-access-policy/AmazonEKSClusterAdminPolicy"

  access_scope {
    type = "cluster"
  }

  depends_on = [aws_eks_access_entry.tools]
}

# ---------------------------------------------------------
# Managed EKS add-ons
# ---------------------------------------------------------
resource "aws_eks_addon" "vpc_cni" {
  cluster_name                = aws_eks_cluster.this.name
  addon_name                  = "vpc-cni"
  resolve_conflicts_on_create = "OVERWRITE"
  resolve_conflicts_on_update = "PRESERVE"

  depends_on = [aws_eks_node_group.general]
}

resource "aws_eks_addon" "coredns" {
  cluster_name                = aws_eks_cluster.this.name
  addon_name                  = "coredns"
  resolve_conflicts_on_create = "OVERWRITE"
  resolve_conflicts_on_update = "PRESERVE"

  depends_on = [aws_eks_node_group.general]
}

resource "aws_eks_addon" "kube_proxy" {
  cluster_name                = aws_eks_cluster.this.name
  addon_name                  = "kube-proxy"
  resolve_conflicts_on_create = "OVERWRITE"
  resolve_conflicts_on_update = "PRESERVE"

  depends_on = [aws_eks_node_group.general]
}
```

### EKS endpoint design

```hcl
endpoint_private_access = true
endpoint_public_access  = true
public_access_cidrs     = var.admin_cidr_blocks
```

This allows:

- Private VPC communication with the EKS API
- Initial administrator access from an approved public `/32`
- No unrestricted `0.0.0.0/0` endpoint exposure

After confirming private administration through the bastion, you can make the cluster private-only:

```hcl
endpoint_private_access = true
endpoint_public_access  = false
```

Do that only after verifying your management path, because Terraform and `kubectl` must then run from inside the VPC or through an appropriate private network.

---

## 24. `eks-infra/outputs.tf`

```hcl
output "cluster_name" {
  description = "EKS cluster name"
  value       = aws_eks_cluster.this.name
}

output "cluster_arn" {
  description = "EKS cluster ARN"
  value       = aws_eks_cluster.this.arn
}

output "cluster_endpoint" {
  description = "EKS API endpoint"
  value       = aws_eks_cluster.this.endpoint
}

output "cluster_security_group_id" {
  description = "EKS-created cluster security group"
  value       = aws_eks_cluster.this.vpc_config[0].cluster_security_group_id
}

output "node_group_name" {
  description = "Managed node-group name"
  value       = aws_eks_node_group.general.node_group_name
}

output "node_role_arn" {
  description = "EKS worker-node IAM role"
  value       = aws_iam_role.eks_nodes.arn
}

output "private_subnet_ids" {
  description = "Private subnets used by EKS"
  value       = local.private_subnet_ids
}

output "update_kubeconfig_command" {
  description = "Command to configure kubectl"
  value       = "aws eks update-kubeconfig --region ${var.region} --name ${aws_eks_cluster.this.name}"
}
```

---

## 25. `eks-infra/terraform.tfvars`

```hcl
project_name = "openhelp"
environment  = "prod"
region       = "us-east-1"

cluster_name       = "openhelp-prod-eks"
kubernetes_version = "1.36"

ec2_state_bucket_name = "openhelp-prod-123456789012-ec2-tfstate"
ec2_state_key         = "ec2/terraform.tfstate"
ec2_state_kms_key_id  = "alias/openhelp-prod-ec2-state"

# Replace this with your current public IP/32.
admin_cidr_blocks = [
  "83.24.100.50/32"
]

node_instance_types = ["t3.medium"]
node_desired_size   = 4
node_min_size       = 2
node_max_size       = 6
node_disk_size      = 30
```

---

# PART D — Deployment procedure

## 26. Create the EC2 key pair

### PowerShell-safe command

```powershell
cmd /c "aws ec2 create-key-pair --region us-east-1 --key-name openhelp-key --key-type rsa --key-format pem --query KeyMaterial --output text > openhelp-key.pem"
```

Check that the file is not empty:

```powershell
Get-Item .\openhelp-key.pem
Get-Content .\openhelp-key.pem -TotalCount 2
```

The first line should resemble:

```text
-----BEGIN RSA PRIVATE KEY-----
```

Never commit the PEM file.

---

## 27. Deploy state buckets first

```bash
cd bootstrap-state
terraform init
terraform fmt -recursive
terraform validate
terraform plan
terraform apply
```

Record the outputs:

```bash
terraform output
```

Verify:

```bash
aws s3api get-bucket-versioning \
  --bucket openhelp-prod-123456789012-ec2-tfstate

aws s3api get-bucket-versioning \
  --bucket openhelp-prod-123456789012-eks-tfstate
```

---

## 28. Deploy VPC and the two public EC2 instances

Edit:

- `ec2-infra/backend.tf`
- `ec2-infra/terraform.tfvars`

Then run:

```bash
cd ../ec2-infra
terraform init
terraform fmt -recursive
terraform validate
terraform plan
terraform apply
```

Expected important outputs:

```text
bastion_public_ip
tools_public_ip
jenkins_url
sonarqube_url
private_subnet_ids
bastion_role_arn
tools_role_arn
```

Check EC2:

```bash
aws ec2 describe-instances \
  --region us-east-1 \
  --filters "Name=tag:Project,Values=openhelp" "Name=instance-state-name,Values=running" \
  --query 'Reservations[].Instances[].{Name:Tags[?Key==`Name`]|[0].Value,PrivateIP:PrivateIpAddress,PublicIP:PublicIpAddress,Subnet:SubnetId,Type:InstanceType}' \
  --output table
```

---

## 29. Check bastion installation

```bash
ssh -i openhelp-key.pem ubuntu@BASTION_PUBLIC_IP
```

Run:

```bash
aws --version
kubectl version --client
helm version
terraform version
aws sts get-caller-identity
```

Because the EC2 instance profile supplies temporary credentials, do not run `aws configure` with long-lived access keys on the bastion.

---

## 30. Check Jenkins and SonarQube

SSH:

```bash
ssh -i openhelp-key.pem ubuntu@TOOLS_PUBLIC_IP
```

Check cloud-init:

```bash
sudo cloud-init status --wait
sudo tail -n 100 /var/log/cloud-init-output.log
```

Check services:

```bash
sudo systemctl status jenkins --no-pager
sudo docker ps
```

Get the initial Jenkins administrator password:

```bash
sudo cat /var/lib/jenkins/secrets/initialAdminPassword
```

Open from an approved IP:

```text
http://TOOLS_PUBLIC_IP:8080
http://TOOLS_PUBLIC_IP:9000
```

Default SonarQube initial credentials are commonly:

```text
Username: admin
Password: admin
```

Change the password immediately.

---

## 31. Deploy EKS second

Edit:

- `eks-infra/backend.tf`
- `eks-infra/terraform.tfvars`

Then run:

```bash
cd ../eks-infra
terraform init
terraform fmt -recursive
terraform validate
terraform plan
terraform apply
```

The EKS configuration reads its VPC and private subnet IDs from the EC2 remote state. Therefore, the EC2/VPC deployment must complete first.

---

## 32. Connect to EKS from the bastion

```bash
ssh -i openhelp-key.pem ubuntu@BASTION_PUBLIC_IP
```

Configure kubeconfig:

```bash
aws eks update-kubeconfig \
  --region us-east-1 \
  --name openhelp-prod-eks
```

Verify:

```bash
kubectl cluster-info
kubectl get nodes -o wide
kubectl get pods -A
```

Expected node count:

```text
4 Ready worker nodes
```

The nodes should have private `10.0.3.x` or `10.0.4.x` addresses and no public IPv4 addresses.

---

## 33. Connect to EKS from Jenkins

On the tools host:

```bash
aws eks update-kubeconfig \
  --region us-east-1 \
  --name openhelp-prod-eks
```

Test as the Jenkins user:

```bash
sudo -u jenkins -H bash -c '
  mkdir -p ~/.kube
  aws eks update-kubeconfig \
    --region us-east-1 \
    --name openhelp-prod-eks
  kubectl get nodes
'
```

The EC2 tools role has both:

1. AWS permission to call `eks:DescribeCluster`
2. An EKS access entry and access-policy association

Both layers are required.

---

# PART E — Validation and troubleshooting

## 34. Validate route-table design

Public route table:

```bash
aws ec2 describe-route-tables \
  --region us-east-1 \
  --filters "Name=tag:Name,Values=openhelp-prod-public-rt" \
  --output table
```

Expected route:

```text
0.0.0.0/0 → igw-...
```

Private route tables:

```bash
aws ec2 describe-route-tables \
  --region us-east-1 \
  --filters "Name=tag:Name,Values=openhelp-prod-private-rt-*" \
  --output table
```

Expected:

```text
private-rt-1 → nat-1
private-rt-2 → nat-2
```

---

## 35. Validate remote state

EC2/VPC state:

```bash
aws s3 ls \
  s3://openhelp-prod-123456789012-ec2-tfstate/ec2/
```

EKS state:

```bash
aws s3 ls \
  s3://openhelp-prod-123456789012-eks-tfstate/eks/
```

Expected objects:

```text
ec2/terraform.tfstate
eks/terraform.tfstate
```

During an active Terraform operation, native S3 locking can create a lock object associated with the state key.

---

## 36. Common errors

### Error: backend bucket does not exist

Cause:

```text
terraform init was run before bootstrap-state/terraform apply
```

Fix:

```bash
cd bootstrap-state
terraform apply
```

Then retry the main configuration.

---

### Error: AccessDenied reading EC2 remote state

The identity running EKS Terraform needs:

- `s3:GetObject` on the EC2 state object
- `s3:ListBucket` on the EC2 state bucket
- `kms:Decrypt` for the EC2 state KMS key

Run Terraform using an administrator role for the lab, or create a restricted CI role with these permissions.

---

### Error: EKS nodes do not join

Check:

```bash
aws eks describe-nodegroup \
  --cluster-name openhelp-prod-eks \
  --nodegroup-name openhelp-prod-eks-general \
  --region us-east-1
```

Common causes:

- Private route table is not associated
- NAT Gateway is not available
- Node IAM policies are missing
- Unsupported Kubernetes version
- EC2 capacity unavailable for the chosen instance type

---

### Jenkins or SonarQube page does not open

Check that:

- Your current public IP still matches `admin_cidr_blocks`
- Cloud-init completed
- The security group allows your `/32`
- Jenkins is listening on 8080
- SonarQube container is listening on 9000

Commands:

```bash
sudo ss -lntp | grep -E ':8080|:9000'
sudo systemctl status jenkins
sudo docker logs --tail 100 sonarqube
```

---

### SonarQube container restarts repeatedly

SonarQube can need additional memory and kernel settings. Use at least `t3.large` for this combined lab host. Check:

```bash
free -h
sudo docker logs sonarqube
```

For a serious environment, separate Jenkins and SonarQube onto different systems or run them on appropriately designed Kubernetes workloads with persistent storage.

---

# PART F — Security and production improvements

## 37. Security improvements already included

- SSH, Jenkins and SonarQube ingress restricted to approved CIDRs
- No `0.0.0.0/0` administrative access
- EBS root volumes encrypted
- IMDSv2 required
- EKS workers placed only in private subnets
- Two same-AZ NAT Gateways
- EKS secret encryption using KMS
- EKS control-plane logging enabled
- S3 state buckets:
  - Versioned
  - KMS encrypted
  - Public access blocked
  - TLS enforced
  - Protected from accidental Terraform destruction
- S3 state locking enabled
- EC2 instance profiles used instead of static AWS keys

---

## 38. Recommended future production changes

1. Set the EKS public endpoint to `false` after private access is tested.
2. Use AWS Systems Manager Session Manager instead of public SSH where possible.
3. Move Jenkins and SonarQube to private subnets.
4. Publish tools through an HTTPS internal or internet-facing ALB as appropriate.
5. Use ACM certificates and Route 53 records.
6. Use an external PostgreSQL database for SonarQube.
7. Put Jenkins data and SonarQube data on separately managed persistent volumes.
8. Replace the Jenkins cluster-admin EKS policy with namespace-scoped permissions.
9. Use EKS Pod Identity or IRSA for Kubernetes workloads accessing AWS.
10. Add VPC endpoints for S3, ECR API, ECR DKR, STS and CloudWatch to reduce NAT traffic.
11. Add ECR repositories and image scanning.
12. Add the AWS Load Balancer Controller and ExternalDNS.
13. Add cluster autoscaling with Karpenter or Cluster Autoscaler.
14. Add CloudWatch alarms, GuardDuty, Security Hub and AWS Config.
15. Add backup and restore procedures before treating the environment as production.

---

# PART G — `.gitignore`

## 39. Root `.gitignore`

```gitignore
# Terraform working directories
**/.terraform/*

# Terraform state
*.tfstate
*.tfstate.*
*.tfstate.backup

# Terraform plans
*.tfplan
tfplan

# Crash logs
crash.log
crash.*.log

# Sensitive variable files
*.auto.tfvars
*.auto.tfvars.json
secret.tfvars

# SSH private keys
*.pem
*.ppk
id_rsa
id_ed25519

# Local override files
override.tf
override.tf.json
*_override.tf
*_override.tf.json

# Editor and OS files
.vscode/
.idea/
.DS_Store
Thumbs.db
```

Commit `.terraform.lock.hcl` so provider selections remain reproducible. Do not commit state, PEM files or secret variable files.

---

# PART H — Execution summary

## 40. Correct deployment order

```mermaid
flowchart LR
    A["1. bootstrap-state<br/>Create 2 S3 buckets + 2 KMS keys"] --> B["2. ec2-infra<br/>Create VPC, subnets, NATs,<br/>bastion and tools EC2"]
    B --> C["3. eks-infra<br/>Read EC2 state and create<br/>private EKS + 4 workers"]
    C --> D["4. Bastion<br/>update-kubeconfig and verify nodes"]
    D --> E["5. Jenkins<br/>Configure pipeline and deploy"]

    classDef bootstrap fill:#fee2e2,stroke:#dc2626,color:#111827,stroke-width:2px;
    classDef infra fill:#dbeafe,stroke:#2563eb,color:#111827,stroke-width:2px;
    classDef eks fill:#fef3c7,stroke:#d97706,color:#111827,stroke-width:2px;
    classDef ops fill:#dcfce7,stroke:#16a34a,color:#111827,stroke-width:2px;

    class A bootstrap;
    class B infra;
    class C eks;
    class D,E ops;
```

Commands:

```bash
# 1. State buckets
cd bootstrap-state
terraform init
terraform apply

# 2. Network and two public EC2 instances
cd ../ec2-infra
terraform init
terraform apply

# 3. Private EKS and four worker nodes
cd ../eks-infra
terraform init
terraform apply

# 4. Configure kubectl from bastion
aws eks update-kubeconfig \
  --region us-east-1 \
  --name openhelp-prod-eks

kubectl get nodes -o wide
```

---

# PART I — Destruction order

## 41. Destroy safely

Destroy EKS first:

```bash
cd eks-infra
terraform destroy
```

Destroy EC2/VPC second:

```bash
cd ../ec2-infra
terraform destroy
```

Do not immediately destroy the backend buckets. They contain state history. If you intentionally want to remove them:

1. Back up the state objects.
2. Confirm no infrastructure remains.
3. Remove object versions and lock objects.
4. Remove or change `prevent_destroy`.
5. Destroy the bootstrap configuration.

NAT Gateways and public IPv4 addresses incur costs, so verify that they are removed after the lab.

---

## 42. Final design explanation for interviews

> We created a highly available VPC across two Availability Zones with two public and two private subnets. Each private subnet has a same-AZ NAT Gateway for resilient outbound internet access. The public layer contains a restricted bastion host for Terraform, AWS CLI, kubectl and Helm, plus a separate Jenkins and SonarQube host for CI and code-quality analysis. The EKS control plane is AWS-managed, while four managed worker nodes run only in the private subnets. We separate Terraform state into two encrypted and versioned S3 buckets: one for VPC/EC2 resources and one for EKS. The EKS configuration reads network outputs from the EC2 state using `terraform_remote_state`, which keeps the infrastructure layers independent while allowing safe dependency sharing. IAM instance profiles avoid long-lived AWS keys, EKS access entries authorize the bastion and Jenkins roles, and KMS protects both Terraform state and Kubernetes secrets.
