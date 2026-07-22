
# UPDATED LAB — Four EC2 VMs with Applications Installed

> **Important**
>
> This new section extends the original lab without deleting any original content.
> The original two-VM guide is preserved below for reference.
>
> The updated design creates **four Ubuntu EC2 VMs**:
>
> 1. **Public VM 1 — Nginx Web Server and SSH Jump Host**
> 2. **Public VM 2 — Apache Web Server**
> 3. **Private VM 1 — Tomcat Application Server**
> 4. **Private VM 2 — MySQL Database Server**
>
> Each Availability Zone contains one public subnet, one private subnet, one NAT Gateway,
> one public VM, and one private VM.

---

## Updated architecture summary

| VM | Availability Zone | Subnet | Sample private IP | Sample public IP | Installed application |
|---|---|---|---:|---:|---|
| Public VM 1 | `us-east-1a` | Public subnet 1 `10.0.1.0/24` | `10.0.1.10` | Assigned by AWS | Nginx + SSH jump host |
| Public VM 2 | `us-east-1b` | Public subnet 2 `10.0.2.0/24` | `10.0.2.10` | Assigned by AWS | Apache HTTP Server |
| Private VM 1 | `us-east-1a` | Private subnet 1 `10.0.3.0/24` | `10.0.3.10` | None | Tomcat application server |
| Private VM 2 | `us-east-1b` | Private subnet 2 `10.0.4.0/24` | `10.0.4.10` | None | MySQL database server |

> The private IP addresses above are configured as fixed private IPs in Terraform.
> Public IP addresses are dynamically assigned by AWS unless Elastic IPs are added.

---

## New beginner-friendly color architecture diagram

```mermaid
flowchart TB
    Laptop["💻 Administrator Laptop<br/>Example source: 83.24.100.50/32"]
    Internet["🌐 Internet<br/>0.0.0.0/0"]

    subgraph AWS["AWS Region: us-east-1"]
        subgraph VPC["VPC: 10.0.0.0/16"]
            IGW["Internet Gateway<br/>Attached to the VPC"]

            subgraph AZ1["Availability Zone: us-east-1a"]
                subgraph PUB1["Public Subnet 1<br/>10.0.1.0/24"]
                    WEB1["Public VM 1<br/>Nginx + Jump Host<br/>Private IP: 10.0.1.10<br/>Public IP: assigned by AWS<br/>Ports: 22 and 80"]
                    NAT1["NAT Gateway 1<br/>Private IP: AWS assigned<br/>Elastic IP: NAT-EIP-1"]
                end

                subgraph PRI1["Private Subnet 1<br/>10.0.3.0/24"]
                    APP1["Private VM 1<br/>Tomcat Application Server<br/>Private IP: 10.0.3.10<br/>Public IP: none<br/>Port: 8080"]
                end

                PRT1["Private Route Table 1<br/>10.0.0.0/16 → local<br/>0.0.0.0/0 → NAT Gateway 1"]
            end

            subgraph AZ2["Availability Zone: us-east-1b"]
                subgraph PUB2["Public Subnet 2<br/>10.0.2.0/24"]
                    WEB2["Public VM 2<br/>Apache Web Server<br/>Private IP: 10.0.2.10<br/>Public IP: assigned by AWS<br/>Ports: 22 and 80"]
                    NAT2["NAT Gateway 2<br/>Private IP: AWS assigned<br/>Elastic IP: NAT-EIP-2"]
                end

                subgraph PRI2["Private Subnet 2<br/>10.0.4.0/24"]
                    DB1["Private VM 2<br/>MySQL Database Server<br/>Private IP: 10.0.4.10<br/>Public IP: none<br/>Port: 3306"]
                end

                PRT2["Private Route Table 2<br/>10.0.0.0/16 → local<br/>0.0.0.0/0 → NAT Gateway 2"]
            end

            PUBRT["Shared Public Route Table<br/>10.0.0.0/16 → local<br/>0.0.0.0/0 → Internet Gateway"]
        end
    end

    Laptop -->|"SSH TCP/22"| WEB1
    Laptop -->|"SSH TCP/22"| WEB2
    Internet -->|"HTTP TCP/80"| WEB1
    Internet -->|"HTTP TCP/80"| WEB2

    WEB1 -->|"SSH TCP/22 using private IP"| APP1
    WEB1 -->|"SSH TCP/22 using private IP"| DB1

    WEB1 -->|"Default route"| PUBRT
    WEB2 -->|"Default route"| PUBRT
    NAT1 -->|"Default route"| PUBRT
    NAT2 -->|"Default route"| PUBRT
    PUBRT --> IGW
    IGW <--> Internet

    APP1 -->|"0.0.0.0/0"| PRT1
    PRT1 --> NAT1
    DB1 -->|"0.0.0.0/0"| PRT2
    PRT2 --> NAT2

    APP1 -->|"MySQL TCP/3306<br/>private VPC route"| DB1

    classDef outer fill:#fff8e1,stroke:#ff9900,stroke-width:3px,color:#111;
    classDef public fill:#e8f5e9,stroke:#2e7d32,stroke-width:3px,color:#111;
    classDef private fill:#e3f2fd,stroke:#1565c0,stroke-width:3px,color:#111;
    classDef compute fill:#ffffff,stroke:#455a64,stroke-width:2px,color:#111;
    classDef nat fill:#fff3e0,stroke:#ef6c00,stroke-width:3px,color:#111;
    classDef route fill:#f3e5f5,stroke:#7b1fa2,stroke-width:2px,color:#111;
    classDef gateway fill:#ede7f6,stroke:#5e35b1,stroke-width:3px,color:#111;
    classDef client fill:#fce4ec,stroke:#c2185b,stroke-width:2px,color:#111;

    class AWS,VPC outer;
    class PUB1,PUB2 public;
    class PRI1,PRI2 private;
    class WEB1,WEB2,APP1,DB1 compute;
    class NAT1,NAT2 nat;
    class PUBRT,PRT1,PRT2 route;
    class IGW,Internet gateway;
    class Laptop client;
```

---

## Route details explained simply

### Public subnet 1 and public subnet 2

Both public subnets use the shared public route table:

| Destination | Target | Meaning |
|---|---|---|
| `10.0.0.0/16` | `local` | Communicate with resources inside the VPC |
| `0.0.0.0/0` | Internet Gateway | Send internet traffic directly to the internet |

Traffic example:

```text
Internet user
    ↓ TCP/80
Internet Gateway
    ↓
Public route table
    ↓
Public VM 1 or Public VM 2
```

### Private subnet 1

| Destination | Target | Meaning |
|---|---|---|
| `10.0.0.0/16` | `local` | Communicate privately with all VPC resources |
| `0.0.0.0/0` | NAT Gateway 1 | Outbound internet through NAT Gateway 1 |

Traffic example:

```text
Private Tomcat VM 10.0.3.10
    ↓ apt update or curl
Private route table 1
    ↓
NAT Gateway 1
    ↓
Internet Gateway
    ↓
Internet
```

### Private subnet 2

| Destination | Target | Meaning |
|---|---|---|
| `10.0.0.0/16` | `local` | Communicate privately with all VPC resources |
| `0.0.0.0/0` | NAT Gateway 2 | Outbound internet through NAT Gateway 2 |

Traffic example:

```text
Private MySQL VM 10.0.4.10
    ↓ apt update
Private route table 2
    ↓
NAT Gateway 2
    ↓
Internet Gateway
    ↓
Internet
```

### Application-to-database traffic

The Tomcat VM connects to MySQL using the VPC-local route:

```text
Tomcat VM 10.0.3.10
    ↓ TCP/3306
VPC local route 10.0.0.0/16
    ↓
MySQL VM 10.0.4.10
```

The traffic does **not** pass through a NAT Gateway or Internet Gateway.

---

# Updated Terraform code

The existing `versions.tf` and most of `network.tf` can remain unchanged.

Replace or extend the following files for the four-VM design.

---

## Updated `variables.tf` additions

Add these variables to the existing `variables.tf` file:

```hcl
variable "public_instance_type" {
  description = "EC2 instance type used by both public web VMs"
  type        = string
  default     = "t3.micro"
}

variable "private_instance_type" {
  description = "EC2 instance type used by the private application and database VMs"
  type        = string
  default     = "t3.micro"
}

variable "public_vm_private_ips" {
  description = "Fixed private IP addresses for the two public VMs"
  type        = list(string)
  default     = ["10.0.1.10", "10.0.2.10"]

  validation {
    condition     = length(var.public_vm_private_ips) == 2
    error_message = "Provide exactly two public VM private IP addresses."
  }
}

variable "private_vm_private_ips" {
  description = "Fixed private IP addresses for the application and database VMs"
  type        = list(string)
  default     = ["10.0.3.10", "10.0.4.10"]

  validation {
    condition     = length(var.private_vm_private_ips) == 2
    error_message = "Provide exactly two private VM private IP addresses."
  }
}
```

> If the original `private_instance_type` variable already exists, keep only one copy.

---

## Replace `security.tf` with this four-VM version

```hcl
# ---------------------------------------------------------
# Public web VM security group
# Used by Public VM 1 and Public VM 2
# ---------------------------------------------------------

resource "aws_security_group" "public_web" {
  name        = "${var.project_name}-${var.environment}-public-web-sg"
  description = "Allow SSH from administrators and HTTP from the internet"
  vpc_id      = aws_vpc.this.id

  tags = {
    Name = "${var.project_name}-${var.environment}-public-web-sg"
  }
}

resource "aws_vpc_security_group_ingress_rule" "public_ssh" {
  for_each = toset(var.admin_cidr_blocks)

  security_group_id = aws_security_group.public_web.id
  description       = "SSH from an approved administrator public IP"

  cidr_ipv4   = each.value
  from_port   = 22
  to_port     = 22
  ip_protocol = "tcp"
}

resource "aws_vpc_security_group_ingress_rule" "public_http" {
  security_group_id = aws_security_group.public_web.id
  description       = "HTTP from the internet"

  cidr_ipv4   = "0.0.0.0/0"
  from_port   = 80
  to_port     = 80
  ip_protocol = "tcp"
}

resource "aws_vpc_security_group_egress_rule" "public_all_outbound" {
  security_group_id = aws_security_group.public_web.id
  description       = "Allow all outbound IPv4 traffic"

  cidr_ipv4   = "0.0.0.0/0"
  ip_protocol = "-1"
}

# ---------------------------------------------------------
# Private Tomcat application VM security group
# ---------------------------------------------------------

resource "aws_security_group" "private_app" {
  name        = "${var.project_name}-${var.environment}-private-app-sg"
  description = "Security group for the private Tomcat application VM"
  vpc_id      = aws_vpc.this.id

  tags = {
    Name = "${var.project_name}-${var.environment}-private-app-sg"
  }
}

resource "aws_vpc_security_group_ingress_rule" "private_app_ssh" {
  security_group_id = aws_security_group.private_app.id
  description       = "SSH from the public VM security group"

  referenced_security_group_id = aws_security_group.public_web.id
  from_port                    = 22
  to_port                      = 22
  ip_protocol                  = "tcp"
}

resource "aws_vpc_security_group_ingress_rule" "private_app_tomcat" {
  security_group_id = aws_security_group.private_app.id
  description       = "Tomcat access from the public web VMs"

  referenced_security_group_id = aws_security_group.public_web.id
  from_port                    = 8080
  to_port                      = 8080
  ip_protocol                  = "tcp"
}

resource "aws_vpc_security_group_egress_rule" "private_app_all_outbound" {
  security_group_id = aws_security_group.private_app.id
  description       = "Allow outbound traffic through NAT Gateway 1"

  cidr_ipv4   = "0.0.0.0/0"
  ip_protocol = "-1"
}

# ---------------------------------------------------------
# Private MySQL database VM security group
# ---------------------------------------------------------

resource "aws_security_group" "private_db" {
  name        = "${var.project_name}-${var.environment}-private-db-sg"
  description = "Security group for the private MySQL database VM"
  vpc_id      = aws_vpc.this.id

  tags = {
    Name = "${var.project_name}-${var.environment}-private-db-sg"
  }
}

resource "aws_vpc_security_group_ingress_rule" "private_db_ssh" {
  security_group_id = aws_security_group.private_db.id
  description       = "SSH from the public VM security group"

  referenced_security_group_id = aws_security_group.public_web.id
  from_port                    = 22
  to_port                      = 22
  ip_protocol                  = "tcp"
}

resource "aws_vpc_security_group_ingress_rule" "private_db_mysql" {
  security_group_id = aws_security_group.private_db.id
  description       = "MySQL only from the Tomcat application VM"

  referenced_security_group_id = aws_security_group.private_app.id
  from_port                    = 3306
  to_port                      = 3306
  ip_protocol                  = "tcp"
}

resource "aws_vpc_security_group_egress_rule" "private_db_all_outbound" {
  security_group_id = aws_security_group.private_db.id
  description       = "Allow outbound traffic through NAT Gateway 2"

  cidr_ipv4   = "0.0.0.0/0"
  ip_protocol = "-1"
}
```

---

## Replace `instances.tf` with this four-VM version

```hcl
# ---------------------------------------------------------
# Public VM 1: Nginx Web Server and SSH Jump Host
# Public Subnet 1 / us-east-1a
# ---------------------------------------------------------

resource "aws_instance" "public_nginx" {
  ami                         = var.ubuntu_ami_id
  instance_type               = var.public_instance_type
  subnet_id                   = aws_subnet.public[0].id
  private_ip                  = var.public_vm_private_ips[0]
  vpc_security_group_ids      = [aws_security_group.public_web.id]
  key_name                    = var.key_name
  associate_public_ip_address = true

  metadata_options {
    http_endpoint = "enabled"
    http_tokens   = "required"
  }

  root_block_device {
    volume_type           = "gp3"
    volume_size           = 10
    encrypted             = true
    delete_on_termination = true
  }

  user_data = <<-EOF
    #!/bin/bash
    set -euxo pipefail
    export DEBIAN_FRONTEND=noninteractive

    apt-get update -y
    apt-get install -y nginx curl unzip jq

    cat > /var/www/html/index.html <<'HTML'
    <!doctype html>
    <html lang="en">
      <head>
        <meta charset="utf-8">
        <title>Public Nginx VM</title>
      </head>
      <body>
        <h1>Public VM 1 - Nginx</h1>
        <p>Availability Zone: us-east-1a</p>
        <p>Subnet: 10.0.1.0/24</p>
        <p>Private IP: 10.0.1.10</p>
        <p>This VM also acts as the SSH jump host.</p>
      </body>
    </html>
    HTML

    systemctl enable nginx
    systemctl restart nginx
  EOF

  user_data_replace_on_change = true

  tags = {
    Name        = "${var.project_name}-${var.environment}-public-nginx"
    Role        = "public-nginx-jump-host"
    Application = "nginx"
  }

  depends_on = [
    aws_internet_gateway.this,
    aws_route_table_association.public
  ]
}

# ---------------------------------------------------------
# Public VM 2: Apache Web Server
# Public Subnet 2 / us-east-1b
# ---------------------------------------------------------

resource "aws_instance" "public_apache" {
  ami                         = var.ubuntu_ami_id
  instance_type               = var.public_instance_type
  subnet_id                   = aws_subnet.public[1].id
  private_ip                  = var.public_vm_private_ips[1]
  vpc_security_group_ids      = [aws_security_group.public_web.id]
  key_name                    = var.key_name
  associate_public_ip_address = true

  metadata_options {
    http_endpoint = "enabled"
    http_tokens   = "required"
  }

  root_block_device {
    volume_type           = "gp3"
    volume_size           = 10
    encrypted             = true
    delete_on_termination = true
  }

  user_data = <<-EOF
    #!/bin/bash
    set -euxo pipefail
    export DEBIAN_FRONTEND=noninteractive

    apt-get update -y
    apt-get install -y apache2 curl unzip jq

    cat > /var/www/html/index.html <<'HTML'
    <!doctype html>
    <html lang="en">
      <head>
        <meta charset="utf-8">
        <title>Public Apache VM</title>
      </head>
      <body>
        <h1>Public VM 2 - Apache</h1>
        <p>Availability Zone: us-east-1b</p>
        <p>Subnet: 10.0.2.0/24</p>
        <p>Private IP: 10.0.2.10</p>
      </body>
    </html>
    HTML

    systemctl enable apache2
    systemctl restart apache2
  EOF

  user_data_replace_on_change = true

  tags = {
    Name        = "${var.project_name}-${var.environment}-public-apache"
    Role        = "public-apache-web-server"
    Application = "apache2"
  }

  depends_on = [
    aws_internet_gateway.this,
    aws_route_table_association.public
  ]
}

# ---------------------------------------------------------
# Private VM 1: Tomcat Application Server
# Private Subnet 1 / us-east-1a
# ---------------------------------------------------------

resource "aws_instance" "private_tomcat" {
  ami                         = var.ubuntu_ami_id
  instance_type               = var.private_instance_type
  subnet_id                   = aws_subnet.private[0].id
  private_ip                  = var.private_vm_private_ips[0]
  vpc_security_group_ids      = [aws_security_group.private_app.id]
  key_name                    = var.key_name
  associate_public_ip_address = false

  metadata_options {
    http_endpoint = "enabled"
    http_tokens   = "required"
  }

  root_block_device {
    volume_type           = "gp3"
    volume_size           = 10
    encrypted             = true
    delete_on_termination = true
  }

  user_data = <<-EOF
    #!/bin/bash
    set -euxo pipefail
    export DEBIAN_FRONTEND=noninteractive

    apt-get update -y
    apt-get install -y tomcat10 curl default-jdk

    mkdir -p /var/lib/tomcat10/webapps/ROOT

    cat > /var/lib/tomcat10/webapps/ROOT/index.html <<'HTML'
    <!doctype html>
    <html lang="en">
      <head>
        <meta charset="utf-8">
        <title>Private Tomcat Application</title>
      </head>
      <body>
        <h1>Private VM 1 - Tomcat Application</h1>
        <p>Availability Zone: us-east-1a</p>
        <p>Subnet: 10.0.3.0/24</p>
        <p>Private IP: 10.0.3.10</p>
        <p>This server has no public IPv4 address.</p>
      </body>
    </html>
    HTML

    chown -R tomcat:tomcat /var/lib/tomcat10/webapps/ROOT
    systemctl enable tomcat10
    systemctl restart tomcat10
  EOF

  user_data_replace_on_change = true

  tags = {
    Name        = "${var.project_name}-${var.environment}-private-tomcat"
    Role        = "private-application-server"
    Application = "tomcat10"
  }

  depends_on = [
    aws_nat_gateway.this,
    aws_route_table_association.private
  ]
}

# ---------------------------------------------------------
# Private VM 2: MySQL Database Server
# Private Subnet 2 / us-east-1b
# ---------------------------------------------------------

resource "aws_instance" "private_mysql" {
  ami                         = var.ubuntu_ami_id
  instance_type               = var.private_instance_type
  subnet_id                   = aws_subnet.private[1].id
  private_ip                  = var.private_vm_private_ips[1]
  vpc_security_group_ids      = [aws_security_group.private_db.id]
  key_name                    = var.key_name
  associate_public_ip_address = false

  metadata_options {
    http_endpoint = "enabled"
    http_tokens   = "required"
  }

  root_block_device {
    volume_type           = "gp3"
    volume_size           = 15
    encrypted             = true
    delete_on_termination = true
  }

  user_data = <<-EOF
    #!/bin/bash
    set -euxo pipefail
    export DEBIAN_FRONTEND=noninteractive

    apt-get update -y
    apt-get install -y mysql-server curl

    sed -i 's/^bind-address.*/bind-address = 0.0.0.0/' \
      /etc/mysql/mysql.conf.d/mysqld.cnf

    systemctl enable mysql
    systemctl restart mysql

    mysql <<'SQL'
    CREATE DATABASE IF NOT EXISTS openhelpapp;
    CREATE USER IF NOT EXISTS 'appuser'@'10.0.%'
      IDENTIFIED BY 'ChangeMe-For-Lab-Only-123!';
    GRANT SELECT, INSERT, UPDATE, DELETE
      ON openhelpapp.*
      TO 'appuser'@'10.0.%';
    FLUSH PRIVILEGES;

    USE openhelpapp;

    CREATE TABLE IF NOT EXISTS application_status (
      id INT PRIMARY KEY AUTO_INCREMENT,
      component VARCHAR(100) NOT NULL,
      status VARCHAR(30) NOT NULL,
      created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP
    );

    INSERT INTO application_status (component, status)
    VALUES ('private-mysql-vm', 'READY');
    SQL
  EOF

  user_data_replace_on_change = true

  tags = {
    Name        = "${var.project_name}-${var.environment}-private-mysql"
    Role        = "private-database-server"
    Application = "mysql"
  }

  depends_on = [
    aws_nat_gateway.this,
    aws_route_table_association.private
  ]
}
```

> **Security note:** The MySQL password shown above is only for a temporary beginner lab.
> Do not store production passwords directly in Terraform user data.
> Use AWS Secrets Manager, Systems Manager Parameter Store, or another approved secret manager.

---

## Replace `outputs.tf` with this four-VM version

```hcl
output "vpc_id" {
  description = "ID of the VPC"
  value       = aws_vpc.this.id
}

output "public_subnet_ids" {
  description = "IDs of both public subnets"
  value       = aws_subnet.public[*].id
}

output "private_subnet_ids" {
  description = "IDs of both private subnets"
  value       = aws_subnet.private[*].id
}

output "public_nginx_public_ip" {
  description = "Public IPv4 address of Public VM 1"
  value       = aws_instance.public_nginx.public_ip
}

output "public_nginx_private_ip" {
  description = "Private IPv4 address of Public VM 1"
  value       = aws_instance.public_nginx.private_ip
}

output "public_apache_public_ip" {
  description = "Public IPv4 address of Public VM 2"
  value       = aws_instance.public_apache.public_ip
}

output "public_apache_private_ip" {
  description = "Private IPv4 address of Public VM 2"
  value       = aws_instance.public_apache.private_ip
}

output "private_tomcat_ip" {
  description = "Private IPv4 address of the Tomcat VM"
  value       = aws_instance.private_tomcat.private_ip
}

output "private_mysql_ip" {
  description = "Private IPv4 address of the MySQL VM"
  value       = aws_instance.private_mysql.private_ip
}

output "nat_gateway_public_ips" {
  description = "Elastic public IPv4 addresses of both NAT Gateways"
  value       = aws_eip.nat[*].public_ip
}

output "nginx_url" {
  description = "Public URL for the Nginx web server"
  value       = "http://${aws_instance.public_nginx.public_ip}"
}

output "apache_url" {
  description = "Public URL for the Apache web server"
  value       = "http://${aws_instance.public_apache.public_ip}"
}

output "ssh_public_nginx" {
  description = "SSH command for Public VM 1"
  value       = "ssh -i openhelp-key.pem ubuntu@${aws_instance.public_nginx.public_ip}"
}

output "ssh_public_apache" {
  description = "SSH command for Public VM 2"
  value       = "ssh -i openhelp-key.pem ubuntu@${aws_instance.public_apache.public_ip}"
}

output "ssh_private_tomcat" {
  description = "SSH to Tomcat through Public VM 1"
  value       = "ssh -i openhelp-key.pem -J ubuntu@${aws_instance.public_nginx.public_ip} ubuntu@${aws_instance.private_tomcat.private_ip}"
}

output "ssh_private_mysql" {
  description = "SSH to MySQL through Public VM 1"
  value       = "ssh -i openhelp-key.pem -J ubuntu@${aws_instance.public_nginx.public_ip} ubuntu@${aws_instance.private_mysql.private_ip}"
}
```

---

## Updated `terraform.tfvars`

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

public_vm_private_ips = [
  "10.0.1.10",
  "10.0.2.10"
]

private_vm_private_ips = [
  "10.0.3.10",
  "10.0.4.10"
]

ubuntu_ami_id = "ami-xxxxxxxxxxxxxxxxx"
key_name      = "openhelp-key"

public_instance_type  = "t3.micro"
private_instance_type = "t3.micro"

admin_cidr_blocks = [
  "83.24.100.50/32"
]
```

---

# Updated deployment steps

## Step 1 — Keep the existing network code

Keep the original:

- `versions.tf`
- VPC block
- Internet Gateway block
- Two public subnets
- Two private subnets
- Shared public route table
- Two NAT Gateways
- Two private route tables
- Route-table associations

The existing two-NAT design already provides one NAT Gateway per Availability Zone.

## Step 2 — Update variables

Add the four-VM private-IP variables and the public/private instance-type variables.

## Step 3 — Replace the security-group code

Use the updated `security.tf` from this section.

The security flow becomes:

```text
Laptop /32
    ├── SSH 22 → Public VM 1
    └── SSH 22 → Public VM 2

Internet
    ├── HTTP 80 → Public VM 1
    └── HTTP 80 → Public VM 2

Public VM security group
    ├── SSH 22 → Private Tomcat VM
    └── SSH 22 → Private MySQL VM

Public VM security group
    └── TCP 8080 → Private Tomcat VM

Private Tomcat security group
    └── TCP 3306 → Private MySQL VM
```

## Step 4 — Replace the instance code

Use the updated `instances.tf`.

Terraform creates:

```text
Public VM 1  → Nginx + jump host
Public VM 2  → Apache
Private VM 1 → Tomcat
Private VM 2 → MySQL
```

## Step 5 — Replace outputs

Use the updated `outputs.tf`.

## Step 6 — Format and validate

```bash
terraform fmt -recursive
terraform validate
```

Expected:

```text
Success! The configuration is valid.
```

## Step 7 — Review the plan

```bash
terraform plan -out=tfplan
```

Confirm that Terraform plans to create:

```text
4 EC2 instances
2 NAT Gateways
2 Elastic IPs for the NAT Gateways
2 public subnets
2 private subnets
3 security groups
1 Internet Gateway
1 public route table
2 private route tables
```

## Step 8 — Apply

```bash
terraform apply tfplan
```

## Step 9 — Display all important outputs

```bash
terraform output
```

Example:

```text
public_nginx_public_ip = "54.12.1.10"
public_apache_public_ip = "54.12.3.10"
private_tomcat_ip = "10.0.3.10"
private_mysql_ip = "10.0.4.10"
nat_gateway_public_ips = [
  "54.12.1.20",
  "54.12.3.20"
]
```

## Step 10 — Wait for cloud-init

Applications are installed through EC2 user data.

Wait approximately two to five minutes, then check:

```bash
ssh -i openhelp-key.pem \
  ubuntu@$(terraform output -raw public_nginx_public_ip) \
  "sudo cloud-init status --wait"
```

```bash
ssh -i openhelp-key.pem \
  ubuntu@$(terraform output -raw public_apache_public_ip) \
  "sudo cloud-init status --wait"
```

---

# Verify all four applications

## Verify Public VM 1 — Nginx

```bash
curl http://$(terraform output -raw public_nginx_public_ip)
```

Expected text:

```text
Public VM 1 - Nginx
```

Check service status:

```bash
ssh -i openhelp-key.pem \
  ubuntu@$(terraform output -raw public_nginx_public_ip) \
  "systemctl is-active nginx"
```

Expected:

```text
active
```

---

## Verify Public VM 2 — Apache

```bash
curl http://$(terraform output -raw public_apache_public_ip)
```

Expected text:

```text
Public VM 2 - Apache
```

Check service status:

```bash
ssh -i openhelp-key.pem \
  ubuntu@$(terraform output -raw public_apache_public_ip) \
  "systemctl is-active apache2"
```

Expected:

```text
active
```

---

## Connect to Private VM 1 — Tomcat

```bash
ssh -i openhelp-key.pem \
  -J ubuntu@$(terraform output -raw public_nginx_public_ip) \
  ubuntu@$(terraform output -raw private_tomcat_ip)
```

Check Tomcat:

```bash
systemctl status tomcat10 --no-pager
```

Test locally:

```bash
curl http://localhost:8080
```

Test Tomcat from Public VM 1:

```bash
ssh -i openhelp-key.pem \
  ubuntu@$(terraform output -raw public_nginx_public_ip) \
  "curl -s http://10.0.3.10:8080"
```

Expected text:

```text
Private VM 1 - Tomcat Application
```

---

## Connect to Private VM 2 — MySQL

```bash
ssh -i openhelp-key.pem \
  -J ubuntu@$(terraform output -raw public_nginx_public_ip) \
  ubuntu@$(terraform output -raw private_mysql_ip)
```

Check MySQL:

```bash
sudo systemctl status mysql --no-pager
```

Check the database and table:

```bash
sudo mysql -e "
SELECT * FROM openhelpapp.application_status;
"
```

Expected logical result:

```text
private-mysql-vm    READY
```

---

## Test Tomcat-to-MySQL network connectivity

From the private Tomcat VM:

```bash
nc -zv 10.0.4.10 3306
```

If `nc` is missing:

```bash
sudo apt-get update
sudo apt-get install -y netcat-openbsd
nc -zv 10.0.4.10 3306
```

Expected:

```text
Connection to 10.0.4.10 3306 port [tcp/mysql] succeeded!
```

Install the MySQL client on the Tomcat VM:

```bash
sudo apt-get install -y mysql-client
```

Connect to MySQL:

```bash
mysql \
  -h 10.0.4.10 \
  -u appuser \
  -p \
  openhelpapp
```

Enter the lab password from the MySQL user-data block.

Run:

```sql
SELECT * FROM application_status;
```

---

# Verify routing for each VM

## Public VM 1 route

```bash
ssh -i openhelp-key.pem \
  ubuntu@$(terraform output -raw public_nginx_public_ip) \
  "ip route"
```

The instance uses the public subnet route table:

```text
0.0.0.0/0 → Internet Gateway
```

## Public VM 2 route

```bash
ssh -i openhelp-key.pem \
  ubuntu@$(terraform output -raw public_apache_public_ip) \
  "ip route"
```

The instance uses the shared public route table.

## Private Tomcat outbound public IP

```bash
ssh -i openhelp-key.pem \
  -J ubuntu@$(terraform output -raw public_nginx_public_ip) \
  ubuntu@$(terraform output -raw private_tomcat_ip) \
  "curl -s https://checkip.amazonaws.com"
```

The returned public IP should match NAT Gateway 1:

```bash
terraform output -json nat_gateway_public_ips | jq -r '.[0]'
```

## Private MySQL outbound public IP

```bash
ssh -i openhelp-key.pem \
  -J ubuntu@$(terraform output -raw public_nginx_public_ip) \
  ubuntu@$(terraform output -raw private_mysql_ip) \
  "curl -s https://checkip.amazonaws.com"
```

The returned public IP should match NAT Gateway 2:

```bash
terraform output -json nat_gateway_public_ips | jq -r '.[1]'
```

---

# Updated Terraform dependency flow

```mermaid
flowchart TD
    VPC["VPC<br/>10.0.0.0/16"] --> IGW["Internet Gateway"]
    VPC --> PUB1["Public Subnet 1<br/>10.0.1.0/24"]
    VPC --> PUB2["Public Subnet 2<br/>10.0.2.0/24"]
    VPC --> PRI1["Private Subnet 1<br/>10.0.3.0/24"]
    VPC --> PRI2["Private Subnet 2<br/>10.0.4.0/24"]

    IGW --> PUBRT["Public Route Table<br/>0.0.0.0/0 → IGW"]
    PUB1 --> PUBRT
    PUB2 --> PUBRT

    PUB1 --> NAT1["NAT Gateway 1"]
    PUB2 --> NAT2["NAT Gateway 2"]

    NAT1 --> PRT1["Private Route Table 1<br/>0.0.0.0/0 → NAT 1"]
    NAT2 --> PRT2["Private Route Table 2<br/>0.0.0.0/0 → NAT 2"]

    PRI1 --> PRT1
    PRI2 --> PRT2

    PUBRT --> NGINX["Public VM 1<br/>Nginx + Jump Host"]
    PUBRT --> APACHE["Public VM 2<br/>Apache"]
    PRT1 --> TOMCAT["Private VM 1<br/>Tomcat"]
    PRT2 --> MYSQL["Private VM 2<br/>MySQL"]

    NGINX -->|"TCP/8080"| TOMCAT
    TOMCAT -->|"TCP/3306"| MYSQL

    classDef network fill:#e3f2fd,stroke:#1565c0,stroke-width:2px,color:#111;
    classDef nat fill:#fff3e0,stroke:#ef6c00,stroke-width:3px,color:#111;
    classDef public fill:#e8f5e9,stroke:#2e7d32,stroke-width:2px,color:#111;
    classDef private fill:#f3e5f5,stroke:#7b1fa2,stroke-width:2px,color:#111;
    classDef gateway fill:#fffde7,stroke:#f9a825,stroke-width:2px,color:#111;

    class VPC,PUB1,PUB2,PRI1,PRI2,PUBRT,PRT1,PRT2 network;
    class NAT1,NAT2 nat;
    class NGINX,APACHE public;
    class TOMCAT,MYSQL private;
    class IGW gateway;
```

---

# Updated troubleshooting

## Public website does not open

Check the public IP:

```bash
terraform output -raw public_nginx_public_ip
terraform output -raw public_apache_public_ip
```

Check TCP/80 in the public security group:

```bash
aws ec2 describe-security-groups \
  --region us-east-1 \
  --filters "Name=group-name,Values=openhelp-prod-public-web-sg" \
  --output json
```

Check cloud-init:

```bash
sudo cloud-init status --long
sudo tail -n 200 /var/log/cloud-init-output.log
```

Check the service:

```bash
sudo systemctl status nginx --no-pager
sudo systemctl status apache2 --no-pager
```

## Tomcat does not answer on port 8080

From Public VM 1:

```bash
curl -v http://10.0.3.10:8080
```

On the Tomcat VM:

```bash
sudo ss -lntp | grep 8080
sudo systemctl status tomcat10 --no-pager
sudo journalctl -u tomcat10 -n 100 --no-pager
```

Verify that the Tomcat security group allows TCP/8080 from the public web security group.

## MySQL connection is refused

On the MySQL VM:

```bash
sudo ss -lntp | grep 3306
sudo systemctl status mysql --no-pager
```

Confirm MySQL bind address:

```bash
sudo grep '^bind-address' /etc/mysql/mysql.conf.d/mysqld.cnf
```

Expected:

```text
bind-address = 0.0.0.0
```

Confirm the application user:

```bash
sudo mysql -e "
SELECT user, host
FROM mysql.user
WHERE user = 'appuser';
"
```

Confirm TCP/3306 is allowed only from the private application security group.

---

# Updated cost warning

This version creates more billable resources than the original lab:

| Resource | Quantity |
|---|---:|
| EC2 instances | 4 |
| NAT Gateways | 2 |
| NAT Gateway Elastic IPs | 2 |
| EBS root volumes | 4 |
| Public IPv4 addresses for public VMs | 2 |

Destroy the environment after testing:

```bash
terraform destroy
```

---

# Updated final resource summary

| Terraform resource | Quantity | Purpose |
|---|---:|---|
| `aws_vpc` | 1 | Main isolated AWS network |
| `aws_internet_gateway` | 1 | Internet access for public subnet resources |
| `aws_subnet.public` | 2 | Public subnet in each Availability Zone |
| `aws_subnet.private` | 2 | Private subnet in each Availability Zone |
| `aws_route_table.public` | 1 | Sends public subnet internet traffic to the IGW |
| `aws_route_table.private` | 2 | Sends each private subnet to its same-AZ NAT Gateway |
| `aws_eip.nat` | 2 | One Elastic IP for each NAT Gateway |
| `aws_nat_gateway.this` | 2 | Outbound internet for private VMs |
| `aws_security_group.public_web` | 1 | SSH from administrator and HTTP from internet |
| `aws_security_group.private_app` | 1 | Protects Tomcat application VM |
| `aws_security_group.private_db` | 1 | Protects MySQL database VM |
| `aws_instance.public_nginx` | 1 | Public Nginx server and SSH jump host |
| `aws_instance.public_apache` | 1 | Public Apache web server |
| `aws_instance.private_tomcat` | 1 | Private Tomcat application server |
| `aws_instance.private_mysql` | 1 | Private MySQL database server |

---

# Updated end-to-end checklist

- [ ] Existing VPC and subnet code retained
- [ ] Two public subnets created
- [ ] Two private subnets created
- [ ] One Internet Gateway attached
- [ ] Two NAT Gateways created
- [ ] Public route table points to the Internet Gateway
- [ ] Private route table 1 points to NAT Gateway 1
- [ ] Private route table 2 points to NAT Gateway 2
- [ ] Public VM 1 created at `10.0.1.10`
- [ ] Public VM 2 created at `10.0.2.10`
- [ ] Private Tomcat VM created at `10.0.3.10`
- [ ] Private MySQL VM created at `10.0.4.10`
- [ ] Nginx installed and active
- [ ] Apache installed and active
- [ ] Tomcat installed and active
- [ ] MySQL installed and active
- [ ] Public websites reachable over TCP/80
- [ ] Private VMs have no public IP
- [ ] Private Tomcat VM uses NAT Gateway 1
- [ ] Private MySQL VM uses NAT Gateway 2
- [ ] Tomcat can connect to MySQL on TCP/3306
- [ ] SSH to private VMs works through Public VM 1
- [ ] `terraform destroy` completed after the lab

---

# ORIGINAL GUIDE — Preserved Without Deleting Content

The complete original guide starts below.

---

# AWS VPC with Two Public Subnets, Two Private Subnets, Two NAT Gateways, Jump Host and Private Ubuntu VM

> **Beginner-friendly Terraform lab**
>
> This guide creates a complete AWS network with:
>
> - 1 VPC
> - 2 public subnets in two Availability Zones
> - 2 private subnets in two Availability Zones
> - 1 Internet Gateway
> - 2 public NAT Gateways, one in each public subnet, each with its own Elastic IP
> - 1 public Ubuntu jump host
> - 1 private Ubuntu web server
> - SSH access from your laptop through the jump host
> - Availability-Zone-local outbound internet access from each private subnet through its own NAT Gateway
> - No direct public access to the private VM

---

## 1. What we are building

The IP address plan used in this guide is:

| Resource | Availability Zone | CIDR/IP | Purpose |
|---|---|---:|---|
| VPC | Regional | `10.0.0.0/16` | Main private AWS network |
| Public subnet 1 | `us-east-1a` | `10.0.1.0/24` | Jump host and NAT Gateway 1 |
| Public subnet 2 | `us-east-1b` | `10.0.2.0/24` | NAT Gateway 2 and future public resources |
| Private subnet 1 | `us-east-1a` | `10.0.3.0/24` | Private Ubuntu web server |
| Private subnet 2 | `us-east-1b` | `10.0.4.0/24` | Future private workloads |
| Jump host | Public subnet 1 | Dynamic `10.0.1.x` + public IP | Entry point from your laptop |
| Private web VM | Private subnet 1 | Dynamic `10.0.3.x` | Not directly reachable from the internet |
| NAT Gateway 1 | Public subnet 1 | Dynamic `10.0.1.x` + Elastic IP 1 | Outbound internet for private subnet 1 |
| NAT Gateway 2 | Public subnet 2 | Dynamic `10.0.2.x` + Elastic IP 2 | Outbound internet for private subnet 2 |

A `/24` subnet contains 256 addresses. AWS reserves five addresses in every subnet, so 251 addresses are available for resources.

---

## 2. Simple high-availability architecture diagram

```mermaid
flowchart TB
    Laptop["💻 Laptop<br/>Source: YOUR_PUBLIC_IP/32"]
    Internet["🌐 Internet"]

    subgraph AWS["AWS Region: us-east-1"]
        subgraph VPC["VPC: 10.0.0.0/16"]

            IGW["Internet Gateway<br/>Attached to VPC"]

            subgraph AZ1["Availability Zone: us-east-1a"]
                PUB1["Public Subnet 1<br/>10.0.1.0/24"]
                NAT1["NAT Gateway 1<br/>Elastic IP 1<br/>Located in Public Subnet 1"]
                JUMP["Ubuntu Jump Host<br/>Private IP: 10.0.1.x<br/>Public IP: assigned by AWS"]
                PRI1["Private Subnet 1<br/>10.0.3.0/24"]
                WEB["Private Ubuntu Web VM<br/>Private IP: 10.0.3.x<br/>Public IP: none"]
                PRT1["Private Route Table 1<br/>0.0.0.0/0 → NAT Gateway 1"]

                PUB1 --- NAT1
                PUB1 --- JUMP
                PRI1 --- WEB
                PRI1 --- PRT1
            end

            subgraph AZ2["Availability Zone: us-east-1b"]
                PUB2["Public Subnet 2<br/>10.0.2.0/24"]
                NAT2["NAT Gateway 2<br/>Elastic IP 2<br/>Located in Public Subnet 2"]
                PRI2["Private Subnet 2<br/>10.0.4.0/24"]
                FUTURE["Future Private Workload<br/>Private IP: 10.0.4.x<br/>Public IP: none"]
                PRT2["Private Route Table 2<br/>0.0.0.0/0 → NAT Gateway 2"]

                PUB2 --- NAT2
                PRI2 --- FUTURE
                PRI2 --- PRT2
            end

            PUBLICRT["Public Route Table<br/>0.0.0.0/0 → Internet Gateway"]
            PUB1 --- PUBLICRT
            PUB2 --- PUBLICRT
        end
    end

    Laptop -->|"SSH TCP/22"| JUMP
    JUMP -->|"SSH using private IP"| WEB

    JUMP --> IGW
    NAT1 --> IGW
    NAT2 --> IGW
    IGW <--> Internet

    WEB -->|"Outbound internet"| NAT1
    FUTURE -->|"Outbound internet"| NAT2

    classDef outer fill:#fff7e6,stroke:#ff9900,stroke-width:3px,color:#111;
    classDef public fill:#e8f5e9,stroke:#2e7d32,stroke-width:3px,color:#111;
    classDef private fill:#e3f2fd,stroke:#1565c0,stroke-width:3px,color:#111;
    classDef nat fill:#fff3e0,stroke:#ef6c00,stroke-width:3px,color:#111;
    classDef route fill:#f3e5f5,stroke:#7b1fa2,stroke-width:2px,color:#111;
    classDef client fill:#fce4ec,stroke:#c2185b,stroke-width:2px,color:#111;
    classDef gateway fill:#fffde7,stroke:#f9a825,stroke-width:3px,color:#111;

    class AWS,VPC outer;
    class PUB1,PUB2,JUMP public;
    class PRI1,PRI2,WEB,FUTURE private;
    class NAT1,NAT2 nat;
    class PUBLICRT,PRT1,PRT2 route;
    class Laptop client;
    class IGW,Internet gateway;
```

### Architecture meaning

```text
Private subnet 1 in us-east-1a
        |
        +--> Private route table 1
                    |
                    +--> NAT Gateway 1 in public subnet 1

Private subnet 2 in us-east-1b
        |
        +--> Private route table 2
                    |
                    +--> NAT Gateway 2 in public subnet 2
```

Each private subnet uses the NAT Gateway in the **same Availability Zone**. This avoids depending on a NAT Gateway in another zone and provides better resilience.

---

## 3. Understand the traffic flow

### 3.1 Laptop to jump host

```text
Laptop
  |
  | SSH TCP/22
  | Source: YOUR_PUBLIC_IP/32
  v
Jump host public IP
```

The jump-host security group permits SSH only from your laptop's public IP.

### 3.2 Laptop to private VM

```text
Laptop
  |
  | SSH ProxyJump
  v
Public jump host
  |
  | SSH using 10.0.3.x
  v
Private Ubuntu VM
```

The private VM:

- Has no public IP.
- Does not accept SSH from the internet.
- Accepts SSH only from the jump-host security group.

### 3.3 Private workloads to the internet

Availability Zone 1:

```text
Private VM in 10.0.3.0/24
  |
  | Private route table 1: 0.0.0.0/0
  v
NAT Gateway 1 in 10.0.1.0/24
  |
  | Public route table: 0.0.0.0/0
  v
Internet Gateway
  |
  v
Internet
```

Availability Zone 2:

```text
Private workload in 10.0.4.0/24
  |
  | Private route table 2: 0.0.0.0/0
  v
NAT Gateway 2 in 10.0.2.0/24
  |
  | Public route table: 0.0.0.0/0
  v
Internet Gateway
  |
  v
Internet
```

The private VM can initiate connections such as:

```bash
sudo apt update
curl https://ubuntu.com
```

An internet client cannot initiate a connection back to the private VM through the NAT Gateway.

---

## 4. Public subnet versus private subnet

A subnet is not public merely because its name contains `public`.

A subnet is public when its associated route table contains:

```text
Destination: 0.0.0.0/0
Target:      Internet Gateway
```

A private subnet in this lab contains:

```text
Destination: 0.0.0.0/0
Target:      NAT Gateway
```

The private VM also has:

```hcl
associate_public_ip_address = false
```

These settings prevent direct public access.

---

## 5. Why each AWS component is required

### VPC

The VPC is your isolated virtual network in AWS.

```text
10.0.0.0/16
```

All four subnets are created inside this address range.

### Public subnets

Public subnets are used for resources that need direct internet routing, such as:

- Jump hosts
- Internet-facing load balancers
- Public NAT Gateways

### Private subnets

Private subnets are used for resources that should not be directly reachable from the internet, such as:

- Application servers
- Databases
- Internal services
- Kubernetes worker nodes

### Internet Gateway

The Internet Gateway is attached to the VPC. It is the route-table target used by resources that communicate directly with the internet.

### NAT Gateway

The NAT Gateway is placed in a public subnet and receives an Elastic IP.

It allows private instances to start outbound internet connections while preventing unsolicited inbound internet connections.

### Route tables

A route table tells AWS where network traffic must go.

Public route:

```text
0.0.0.0/0 → Internet Gateway
```

Private route:

```text
0.0.0.0/0 → NAT Gateway
```

AWS also automatically creates a local VPC route:

```text
10.0.0.0/16 → local
```

The local route allows resources in the VPC to communicate using private IP addresses, subject to security-group and network ACL rules.

### Security groups

Security groups are stateful virtual firewalls attached to network interfaces.

This guide creates:

1. A jump-host security group allowing TCP/22 only from your public `/32`.
2. A private-VM security group allowing TCP/22 only from the jump-host security group.

### EC2 key pair

The EC2 key pair allows SSH public-key authentication.

- AWS stores the public key.
- You keep the private `.pem` key.
- Do not upload the private key to GitHub.
- Do not copy it to the jump host when using `ProxyJump`.

---

## 6. Cost warning

This lab can create billable AWS resources, especially:

- Two NAT Gateways with hourly usage charges
- NAT Gateway data processing charges for traffic passing through each gateway
- EC2 instances
- Public IPv4 addresses
- EBS volumes

Destroy the lab when it is no longer required:

```bash
terraform destroy
```

---

## 7. Prerequisites

Install:

- Terraform
- AWS CLI
- OpenSSH client
- An AWS account
- AWS credentials with permission to create VPC and EC2 resources

Check the commands:

```bash
terraform version
aws --version
ssh -V
```

Configure the AWS CLI:

```bash
aws configure
```

Typical input:

```text
AWS Access Key ID: YOUR_ACCESS_KEY
AWS Secret Access Key: YOUR_SECRET_KEY
Default region name: us-east-1
Default output format: json
```

Verify the active identity:

```bash
aws sts get-caller-identity
```

---

## 8. Create the project directory

```bash
mkdir aws-private-vm-lab
cd aws-private-vm-lab
```

Create these files:

```text
aws-private-vm-lab/
├── versions.tf
├── variables.tf
├── network.tf
├── security.tf
├── instances.tf
├── outputs.tf
└── terraform.tfvars
```

Terraform reads all `.tf` files in the current directory as one configuration. The filenames are for organization; Terraform does not execute them one by one.

---

# Terraform configuration

## 9. `versions.tf`

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
  region = var.region

  default_tags {
    tags = {
      Project     = var.project_name
      Environment = var.environment
      ManagedBy   = "Terraform"
    }
  }
}
```

### Explanation

```hcl
terraform {
```

Starts the Terraform settings block.

```hcl
required_version = ">= 1.6.0"
```

Requires Terraform 1.6 or newer.

```hcl
required_providers {
```

Declares external providers used by the project.

```hcl
source = "hashicorp/aws"
```

Downloads the official HashiCorp AWS provider.

```hcl
version = "~> 6.0"
```

Allows compatible AWS provider 6.x releases but not 7.x.

```hcl
provider "aws" {
  region = var.region
}
```

Configures the AWS Region using a Terraform variable.

```hcl
default_tags {
```

Automatically applies common tags to supported AWS resources.

---

## 10. `variables.tf`

```hcl
variable "project_name" {
  description = "Project name used in resource names and tags"
  type        = string
  default     = "openhelp"
}

variable "environment" {
  description = "Deployment environment such as dev, test or prod"
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
  description = "Availability Zones used by public and private subnets"
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
  description = "Official Ubuntu 24.04 LTS AMI ID retrieved with AWS CLI"
  type        = string

  validation {
    condition     = can(regex("^ami-[0-9a-f]+$", var.ubuntu_ami_id))
    error_message = "ubuntu_ami_id must be an AMI ID such as ami-0123456789abcdef0."
  }
}

variable "key_name" {
  description = "Name of an existing EC2 key pair in the selected Region"
  type        = string
}

variable "jump_instance_type" {
  description = "EC2 instance type for the public jump host"
  type        = string
  default     = "t3.micro"
}

variable "private_instance_type" {
  description = "EC2 instance type for the private web server"
  type        = string
  default     = "t3.micro"
}

variable "admin_cidr_blocks" {
  description = "Public IPv4 CIDR blocks allowed to SSH to the jump host"
  type        = list(string)

  validation {
    condition = (
      length(var.admin_cidr_blocks) > 0 &&
      alltrue([
        for cidr in var.admin_cidr_blocks :
        cidr != "0.0.0.0/0" && can(cidrhost(cidr, 0))
      ])
    )

    error_message = "Use a valid restricted CIDR such as 83.24.100.50/32. Do not use 0.0.0.0/0."
  }
}
```

### Important variable concepts

A variable definition describes the expected input:

```hcl
variable "region" {
  type = string
}
```

The actual value is normally supplied in `terraform.tfvars`:

```hcl
region = "us-east-1"
```

### Why validate the AMI

```hcl
can(regex("^ami-[0-9a-f]+$", var.ubuntu_ami_id))
```

This checks that:

- The value begins with `ami-`.
- The remaining characters contain lowercase hexadecimal digits.
- Terraform shows an understandable error before contacting AWS if the format is invalid.

### Why reject `0.0.0.0/0`

```text
0.0.0.0/0
```

means every IPv4 address on the internet. Allowing SSH from this CIDR unnecessarily exposes port 22.

Your laptop IP should normally be written as:

```text
83.24.100.50/32
```

`/32` means one exact IPv4 address.

---

## 11. Find the official Ubuntu 24.04 AMI using AWS CLI

AMI IDs are Region-specific and are replaced as Canonical publishes updated images. Retrieve the current AMI instead of copying an old ID from another Region.

Canonical's AWS account ID is:

```text
099720109477
```

Run:

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
  --query 'sort_by(Images, &CreationDate)[-1].[ImageId,Name,CreationDate]' \
  --output table
```

### What each part means

| Part | Meaning |
|---|---|
| `describe-images` | Lists EC2 AMIs |
| `--region us-east-1` | Searches only in Northern Virginia |
| `--owners 099720109477` | Restricts results to Canonical |
| `ubuntu-noble-24.04` | Ubuntu 24.04 LTS, code name Noble |
| `amd64` | x86-64 architecture for instance families such as `t3` |
| `gp3` | Image using a GP3 EBS root volume |
| `state=available` | Returns images that can be launched |
| `sort_by(...CreationDate)[-1]` | Sorts by creation time and selects the newest image |

Return only the AMI ID:

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

Example format:

```text
ami-0123456789abcdef0
```

Use the actual value returned by your command.

Store it in a shell variable:

```bash
UBUNTU_AMI_ID=$(aws ec2 describe-images \
  --region us-east-1 \
  --owners 099720109477 \
  --filters \
    "Name=name,Values=ubuntu/images/hvm-ssd-gp3/ubuntu-noble-24.04-amd64-server-*" \
    "Name=architecture,Values=x86_64" \
    "Name=root-device-type,Values=ebs" \
    "Name=virtualization-type,Values=hvm" \
    "Name=state,Values=available" \
  --query 'sort_by(Images, &CreationDate)[-1].ImageId' \
  --output text)

echo "$UBUNTU_AMI_ID"
```

---

## 12. `network.tf`

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
    Name = "${var.project_name}-${var.environment}-public-${count.index + 1}"
    Tier = "public"
  }
}

# ---------------------------------------------------------
# Two private subnets
# ---------------------------------------------------------

resource "aws_subnet" "private" {
  count = length(var.private_subnet_cidrs)

  vpc_id                  = aws_vpc.this.id
  cidr_block              = var.private_subnet_cidrs[count.index]
  availability_zone       = var.availability_zones[count.index]
  map_public_ip_on_launch = false

  tags = {
    Name = "${var.project_name}-${var.environment}-private-${count.index + 1}"
    Tier = "private"
  }
}

# ---------------------------------------------------------
# One shared public route table
# Both public subnets route internet traffic to the IGW
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
# Two Elastic IPs: one for each NAT Gateway
# ---------------------------------------------------------

resource "aws_eip" "nat" {
  count = length(var.public_subnet_cidrs)

  domain = "vpc"

  tags = {
    Name = "${var.project_name}-${var.environment}-nat-eip-${count.index + 1}"
  }

  depends_on = [
    aws_internet_gateway.this
  ]
}

# ---------------------------------------------------------
# Two public NAT Gateways
# NAT 1 is placed in public subnet 1 / AZ 1
# NAT 2 is placed in public subnet 2 / AZ 2
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
# Two private route tables
# Each private subnet uses the NAT Gateway in the same AZ
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

---

## 13. Network block explanations

### `aws_vpc`

```hcl
resource "aws_vpc" "this" {
```

- `aws_vpc` is the Terraform resource type.
- `this` is Terraform's local name for the resource.
- Other resources reference it as `aws_vpc.this`.

```hcl
cidr_block = var.vpc_cidr
```

Assigns `10.0.0.0/16` to the VPC.

```hcl
enable_dns_support = true
```

Enables DNS resolution in the VPC.

```hcl
enable_dns_hostnames = true
```

Allows supported EC2 instances to receive DNS hostnames.

### `aws_internet_gateway`

```hcl
vpc_id = aws_vpc.this.id
```

An Internet Gateway must be attached to a VPC. Terraform obtains the newly created VPC ID automatically.

This reference also creates an implicit dependency:

```text
Create VPC first
        ↓
Create Internet Gateway
```

### Public subnet loop

```hcl
count = length(var.public_subnet_cidrs)
```

The list contains two CIDRs, so Terraform creates two public subnets.

During the first iteration:

```text
count.index = 0
CIDR        = 10.0.1.0/24
AZ          = us-east-1a
Name        = openhelp-prod-public-1
```

During the second iteration:

```text
count.index = 1
CIDR        = 10.0.2.0/24
AZ          = us-east-1b
Name        = openhelp-prod-public-2
```

```hcl
map_public_ip_on_launch = true
```

Requests automatic public IPv4 assignment for instances launched in the subnet when the instance configuration permits it.

### Private subnet loop

```hcl
map_public_ip_on_launch = false
```

Prevents subnet-level automatic public IPv4 assignment.

### Public route table

```hcl
route {
  cidr_block = "0.0.0.0/0"
  gateway_id = aws_internet_gateway.this.id
}
```

All IPv4 destinations not matched by a more specific route are sent to the Internet Gateway.

### Route-table association

Creating a route table is not enough. It must be associated with the intended subnet:

```hcl
subnet_id      = aws_subnet.public[count.index].id
route_table_id = aws_route_table.public.id
```

### Two Elastic IPs

```hcl
resource "aws_eip" "nat" {
  count  = length(var.public_subnet_cidrs)
  domain = "vpc"
}
```

There are two public-subnet CIDRs, so Terraform creates:

```text
aws_eip.nat[0] → Elastic IP for NAT Gateway 1
aws_eip.nat[1] → Elastic IP for NAT Gateway 2
```

Each public NAT Gateway requires its own Elastic IP.

### Two NAT Gateways

```hcl
resource "aws_nat_gateway" "this" {
  count = length(var.public_subnet_cidrs)

  allocation_id = aws_eip.nat[count.index].id
  subnet_id     = aws_subnet.public[count.index].id
}
```

Terraform pairs resources by index:

| Index | Public subnet | Availability Zone | Elastic IP | NAT Gateway |
|---:|---|---|---|---|
| `0` | `10.0.1.0/24` | `us-east-1a` | `aws_eip.nat[0]` | `aws_nat_gateway.this[0]` |
| `1` | `10.0.2.0/24` | `us-east-1b` | `aws_eip.nat[1]` | `aws_nat_gateway.this[1]` |

A public NAT Gateway must be located in a public subnet whose route table sends `0.0.0.0/0` to the Internet Gateway.

### Two private route tables

```hcl
resource "aws_route_table" "private" {
  count = length(var.private_subnet_cidrs)

  route {
    cidr_block     = "0.0.0.0/0"
    nat_gateway_id = aws_nat_gateway.this[count.index].id
  }
}
```

Terraform creates one private route table per private subnet:

```text
private route table 1 → NAT Gateway 1
private route table 2 → NAT Gateway 2
```

The matching index is important:

```text
private[0] in us-east-1a → nat[0] in us-east-1a
private[1] in us-east-1b → nat[1] in us-east-1b
```

Use `nat_gateway_id`, not `gateway_id`, for a NAT Gateway route. Using the wrong attribute can cause repeated Terraform plan differences.

---

## 14. `security.tf`

The following uses dedicated rule resources. This is clearer for lifecycle management than mixing every rule inside the security-group block.

```hcl
# ---------------------------------------------------------
# Jump-host security group
# ---------------------------------------------------------

resource "aws_security_group" "jump" {
  name        = "${var.project_name}-${var.environment}-jump-sg"
  description = "Security group for the public jump host"
  vpc_id      = aws_vpc.this.id

  tags = {
    Name = "${var.project_name}-${var.environment}-jump-sg"
  }
}

resource "aws_vpc_security_group_ingress_rule" "jump_ssh" {
  for_each = toset(var.admin_cidr_blocks)

  security_group_id = aws_security_group.jump.id
  description       = "SSH from an approved administrator public IP"

  cidr_ipv4   = each.value
  from_port   = 22
  to_port     = 22
  ip_protocol = "tcp"
}

resource "aws_vpc_security_group_egress_rule" "jump_all_outbound" {
  security_group_id = aws_security_group.jump.id
  description       = "Allow all outbound IPv4 traffic"

  cidr_ipv4   = "0.0.0.0/0"
  ip_protocol = "-1"
}

# ---------------------------------------------------------
# Private-VM security group
# ---------------------------------------------------------

resource "aws_security_group" "private_vm" {
  name        = "${var.project_name}-${var.environment}-private-vm-sg"
  description = "Security group for the private Ubuntu VM"
  vpc_id      = aws_vpc.this.id

  tags = {
    Name = "${var.project_name}-${var.environment}-private-vm-sg"
  }
}

resource "aws_vpc_security_group_ingress_rule" "private_ssh_from_jump" {
  security_group_id = aws_security_group.private_vm.id
  description       = "Allow SSH only from the jump-host security group"

  referenced_security_group_id = aws_security_group.jump.id
  from_port                    = 22
  to_port                      = 22
  ip_protocol                  = "tcp"
}

resource "aws_vpc_security_group_egress_rule" "private_all_outbound" {
  security_group_id = aws_security_group.private_vm.id
  description       = "Allow outbound IPv4 traffic through the NAT Gateway"

  cidr_ipv4   = "0.0.0.0/0"
  ip_protocol = "-1"
}
```

---

## 15. Security block explanations

### Jump-host inbound rule

```hcl
for_each = toset(var.admin_cidr_blocks)
```

Creates one ingress rule for every approved administrator CIDR.

```hcl
cidr_ipv4 = each.value
```

Uses the current CIDR from the set.

```hcl
from_port   = 22
to_port     = 22
ip_protocol = "tcp"
```

Allows only SSH.

### Private-VM inbound rule

```hcl
referenced_security_group_id = aws_security_group.jump.id
```

This does not allow every address in the public subnet. It allows traffic from network interfaces that use the jump-host security group.

This is better than hard-coding the jump host's private IP because the instance IP could change after replacement.

### Outbound rules

```hcl
cidr_ipv4   = "0.0.0.0/0"
ip_protocol = "-1"
```

Allows all outbound IPv4 protocols.

For the private VM, the security group allows the traffic, but the private route table determines that internet-bound traffic must travel through the NAT Gateway.

### Stateful behavior

Security groups are stateful. When an allowed outbound request is sent, its response traffic is automatically permitted. A separate inbound rule for the return packets is not required.

---

## 16. `instances.tf`

```hcl
# ---------------------------------------------------------
# Public Ubuntu jump host
# ---------------------------------------------------------

resource "aws_instance" "jump" {
  ami                         = var.ubuntu_ami_id
  instance_type               = var.jump_instance_type
  subnet_id                   = aws_subnet.public[0].id
  vpc_security_group_ids      = [aws_security_group.jump.id]
  key_name                    = var.key_name
  associate_public_ip_address = true

  metadata_options {
    http_endpoint = "enabled"
    http_tokens   = "required"
  }

  root_block_device {
    volume_type           = "gp3"
    volume_size           = 10
    encrypted             = true
    delete_on_termination = true
  }

  tags = {
    Name = "${var.project_name}-${var.environment}-jump"
    Role = "jump-host"
  }

  depends_on = [
    aws_internet_gateway.this,
    aws_route_table_association.public
  ]
}

# ---------------------------------------------------------
# Private Ubuntu web server
# ---------------------------------------------------------

resource "aws_instance" "web" {
  ami                         = var.ubuntu_ami_id
  instance_type               = var.private_instance_type
  subnet_id                   = aws_subnet.private[0].id
  vpc_security_group_ids      = [aws_security_group.private_vm.id]
  key_name                    = var.key_name
  associate_public_ip_address = false

  metadata_options {
    http_endpoint = "enabled"
    http_tokens   = "required"
  }

  root_block_device {
    volume_type           = "gp3"
    volume_size           = 10
    encrypted             = true
    delete_on_termination = true
  }

  user_data = <<-EOF
    #!/bin/bash
    set -euxo pipefail

    export DEBIAN_FRONTEND=noninteractive

    apt-get update -y
    apt-get install -y nginx curl

    cat > /var/www/html/index.html <<'HTML'
    <!doctype html>
    <html lang="en">
      <head>
        <meta charset="utf-8">
        <title>OpenHelp Private Web Server</title>
      </head>
      <body>
        <h1>OpenHelp Private Web Server</h1>
        <p>This Ubuntu VM is running in private subnet 10.0.3.0/24.</p>
        <p>It has no public IPv4 address.</p>
        <p>Its outbound internet traffic uses the NAT Gateway.</p>
      </body>
    </html>
    HTML

    systemctl enable nginx
    systemctl restart nginx
  EOF

  user_data_replace_on_change = true

  tags = {
    Name = "${var.project_name}-${var.environment}-private-web"
    Role = "private-web-server"
  }

  depends_on = [
    aws_nat_gateway.this,
    aws_route_table_association.private
  ]
}
```

---

## 17. Instance block explanations

### AMI

```hcl
ami = var.ubuntu_ami_id
```

Uses the Ubuntu 24.04 AMI ID copied from the AWS CLI result.

### Instance type

```hcl
instance_type = "t3.micro"
```

Controls the virtual CPU and memory capacity. The value is supplied through variables.

### Subnet placement

Jump host:

```hcl
subnet_id = aws_subnet.public[0].id
```

Private VM:

```hcl
subnet_id = aws_subnet.private[0].id
```

Index `[0]` selects the first subnet created by the `count` loop.

### Security groups

```hcl
vpc_security_group_ids = [aws_security_group.jump.id]
```

Attaches the jump-host firewall.

```hcl
vpc_security_group_ids = [aws_security_group.private_vm.id]
```

Attaches the private-VM firewall.

### Key pair

```hcl
key_name = var.key_name
```

Uses an EC2 key pair that already exists in `us-east-1`.

### Public IP behavior

Jump host:

```hcl
associate_public_ip_address = true
```

Private VM:

```hcl
associate_public_ip_address = false
```

The private VM therefore has no direct public address.

### Instance Metadata Service

```hcl
metadata_options {
  http_endpoint = "enabled"
  http_tokens   = "required"
}
```

Enables the EC2 Instance Metadata Service and requires IMDSv2 session tokens.

### Root disk

```hcl
root_block_device {
  volume_type           = "gp3"
  volume_size           = 10
  encrypted             = true
  delete_on_termination = true
}
```

Creates an encrypted 10-GiB GP3 EBS root volume and deletes it with the instance.

### `user_data`

The private VM runs the startup script during its first boot:

1. Updates Ubuntu package metadata.
2. Installs Nginx and curl.
3. Writes a basic web page.
4. Enables and starts Nginx.

This also tests NAT connectivity because `apt-get update` needs outbound internet and DNS access.

### `depends_on`

```hcl
depends_on = [
  aws_nat_gateway.this,
  aws_route_table_association.private
]
```

Makes the intended startup order explicit so the private VM is launched after both NAT Gateways and the private route-table associations are configured. The VM in private subnet 1 then uses NAT Gateway 1.

---

## 18. `outputs.tf`

```hcl
output "vpc_id" {
  description = "ID of the VPC"
  value       = aws_vpc.this.id
}

output "public_subnet_ids" {
  description = "IDs of both public subnets"
  value       = aws_subnet.public[*].id
}

output "private_subnet_ids" {
  description = "IDs of both private subnets"
  value       = aws_subnet.private[*].id
}

output "ubuntu_ami_id" {
  description = "Ubuntu AMI used for both EC2 instances"
  value       = var.ubuntu_ami_id
}

output "jump_public_ip" {
  description = "Public IPv4 address of the jump host"
  value       = aws_instance.jump.public_ip
}

output "jump_private_ip" {
  description = "Private IPv4 address of the jump host"
  value       = aws_instance.jump.private_ip
}

output "web_private_ip" {
  description = "Private IPv4 address of the private web VM"
  value       = aws_instance.web.private_ip
}

output "nat_gateway_public_ips" {
  description = "Elastic public IPv4 addresses used by both NAT Gateways"
  value       = aws_eip.nat[*].public_ip
}

output "nat_gateway_ids" {
  description = "IDs of both NAT Gateways"
  value       = aws_nat_gateway.this[*].id
}

output "ssh_to_jump" {
  description = "Template command for connecting to the jump host"
  value       = "ssh -i openhelp-key.pem ubuntu@${aws_instance.jump.public_ip}"
}

output "ssh_to_private_web" {
  description = "Template command for connecting to the private VM with ProxyJump"
  value       = "ssh -i openhelp-key.pem -J ubuntu@${aws_instance.jump.public_ip} ubuntu@${aws_instance.web.private_ip}"
}
```

### Why outputs are useful

Terraform outputs expose values that AWS assigns dynamically, including:

- Public IP of the jump host
- Private IP of the web server
- Elastic IPs of both NAT Gateways
- Subnet IDs

Read one value without quotes:

```bash
terraform output -raw jump_public_ip
```

---

## 19. Create the EC2 key pair

An EC2 key pair is Regional. Create it in the same Region as the instances:

for windows power shell use the below command

```bash
 cmd /c "aws ec2 create-key-pair --region us-east-1 --key-name openhelp-key --key-type rsa --key-format pem --query KeyMaterial --output text > openhelp-key-fixed.pem"
```

To delete key pair use the below command

```bash
 aws ec2 delete-key-pair --region us-east-1 --key-name openhelp-key
```

Foe linux:- 

```bash
aws ec2 create-key-pair \
  --region us-east-1 \
  --key-name openhelp-key \
  --query 'KeyMaterial' \
  --output text > openhelp-key.pem
```

Restrict the private-key permissions on Linux, macOS, Git Bash or WSL:

```bash
chmod 400 openhelp-key.pem
```

Verify it:

```bash
aws ec2 describe-key-pairs \
  --region us-east-1 \
  --key-names openhelp-key \
  --output table
```

Do not run `create-key-pair` again with the same name unless you have intentionally removed the original key pair.

Add this to `.gitignore`:

```gitignore
*.pem
*.tfstate
*.tfstate.*
.terraform/
.terraform.lock.hcl
crash.log
```

> Many teams commit `.terraform.lock.hcl` to version control to pin provider selections. For a personal lab, decide according to your workflow; never commit private keys or state containing sensitive values.

---

## 20. Find your laptop public IP

Run:

```bash
curl https://checkip.amazonaws.com
```

Example result:

```text
83.24.100.50
```

Convert it to a single-host CIDR:

```text
83.24.100.50/32
```

If your ISP changes your public IP, update `terraform.tfvars` and run `terraform apply` again.

---

## 21. `terraform.tfvars`

Replace the AMI ID and public IP with your real values:

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

# Replace this with the current value returned by the AWS CLI.
ubuntu_ami_id = "ami-xxxxxxxxxxxxxxxxx"

# This must be an existing EC2 key-pair name, not the .pem filename.
key_name = "openhelp-key"

jump_instance_type    = "t3.micro"
private_instance_type = "t3.micro"

# Replace this with your laptop's current public IPv4 address.
admin_cidr_blocks = [
  "83.24.100.50/32"
]
```

### Automatically replace the AMI value

```bash
UBUNTU_AMI_ID=$(aws ec2 describe-images \
  --region us-east-1 \
  --owners 099720109477 \
  --filters \
    "Name=name,Values=ubuntu/images/hvm-ssd-gp3/ubuntu-noble-24.04-amd64-server-*" \
    "Name=architecture,Values=x86_64" \
    "Name=root-device-type,Values=ebs" \
    "Name=virtualization-type,Values=hvm" \
    "Name=state,Values=available" \
  --query 'sort_by(Images, &CreationDate)[-1].ImageId' \
  --output text)

if [[ ! "$UBUNTU_AMI_ID" =~ ^ami-[0-9a-f]+$ ]]; then
  echo "ERROR: A valid Ubuntu AMI ID was not returned."
  exit 1
fi

sed -i \
  "s/^ubuntu_ami_id.*/ubuntu_ami_id = \"$UBUNTU_AMI_ID\"/" \
  terraform.tfvars

grep '^ubuntu_ami_id' terraform.tfvars
```

On macOS, the `sed` syntax may require:

```bash
sed -i '' \
  "s/^ubuntu_ami_id.*/ubuntu_ami_id = \"$UBUNTU_AMI_ID\"/" \
  terraform.tfvars
```

---

# Deploy the environment

## 22. Initialize Terraform

```bash
terraform init
```

This:

- Downloads the AWS provider.
- Initializes the working directory.
- Creates `.terraform/`.
- Creates or updates the dependency lock file.

---

## 23. Format the code

```bash
terraform fmt -recursive
```

This applies standard Terraform formatting.

Check formatting without changing files:

```bash
terraform fmt -check -recursive
```

---

## 24. Validate the configuration

```bash
terraform validate
```

Expected:

```text
Success! The configuration is valid.
```

Validation checks Terraform syntax and internal references. It does not guarantee that AWS will accept every value.

---

## 25. Review the execution plan

```bash
terraform plan -out=tfplan
```

Review the proposed resources carefully.

Apply the saved plan:

```bash
terraform apply tfplan
```

Alternatively:

```bash
terraform apply
```

Then enter:

```text
yes
```

---

## 26. Expected creation order

Terraform builds a dependency graph. The approximate order is:

```mermaid
flowchart TD
    VPC["VPC 10.0.0.0/16"] --> IGW["Internet Gateway"]
    VPC --> PUB["2 Public Subnets"]
    VPC --> PRI["2 Private Subnets"]

    IGW --> PRT["Public Route Table"]
    PUB --> PASSOC["2 Public Route Associations"]
    PRT --> PASSOC

    IGW --> EIPS["2 Elastic IPs"]
    EIPS --> NATS["2 NAT Gateways"]
    PUB --> NATS
    PASSOC --> NATS

    NATS --> PRIVRT["2 Private Route Tables"]
    PRI --> PRIASSOC["2 Private Route Associations"]
    PRIVRT --> PRIASSOC

    VPC --> JSG["Jump Security Group"]
    VPC --> WSG["Private VM Security Group"]
    JSG --> WSG

    PASSOC --> JUMP["Jump Host in Public Subnet 1"]
    JSG --> JUMP

    PRIASSOC --> WEB["Private Web VM in Private Subnet 1"]
    WSG --> WEB
    NATS --> WEB

    classDef network fill:#e3f2fd,stroke:#1565c0,stroke-width:2px,color:#111;
    classDef nat fill:#fff3e0,stroke:#ef6c00,stroke-width:3px,color:#111;
    classDef security fill:#fce4ec,stroke:#c2185b,stroke-width:2px,color:#111;
    classDef compute fill:#e8f5e9,stroke:#2e7d32,stroke-width:2px,color:#111;
    classDef gateway fill:#fffde7,stroke:#f9a825,stroke-width:2px,color:#111;

    class VPC,PUB,PRI,PRT,PASSOC,PRIVRT,PRIASSOC network;
    class EIPS,NATS nat;
    class JSG,WSG security;
    class JUMP,WEB compute;
    class IGW gateway;
```

screenshot from aws console

<img width="1175" height="464" alt="image" src="https://github.com/user-attachments/assets/823f2274-ea3e-461f-a467-35cd5bb8c286" />



---

## 27. View the outputs

```bash
terraform output
```

Example format:

```text
jump_public_ip          = "54.210.10.20"
jump_private_ip         = "10.0.1.25"
web_private_ip          = "10.0.3.80"
nat_gateway_public_ips  = ["3.90.50.40", "44.210.80.25"]
```

AWS chooses the actual host addresses dynamically.

---

# Connect and verify

## 28. Connect to the public jump host

Ubuntu cloud images use the user:

```text
ubuntu
```

Run:

```bash
ssh -i openhelp-key.pem ubuntu@$(terraform output -raw jump_public_ip)
```

On first connection, verify and accept the host fingerprint when appropriate.

---

## 29. Connect to the private VM using ProxyJump

Run from your laptop:

```bash
ssh -i openhelp-key.pem \
  -J ubuntu@$(terraform output -raw jump_public_ip) \
  ubuntu@$(terraform output -raw web_private_ip)
```

`-J` means use the specified machine as an SSH jump host.

The private key remains on your laptop. OpenSSH performs the authentication through the jump connection.

---

## 30. Optional SSH client configuration

Add this to:

```text
~/.ssh/config
```

```sshconfig
Host openhelp-jump
    HostName JUMP_PUBLIC_IP
    User ubuntu
    IdentityFile ~/.ssh/openhelp-key.pem
    IdentitiesOnly yes

Host openhelp-private
    HostName PRIVATE_VM_IP
    User ubuntu
    IdentityFile ~/.ssh/openhelp-key.pem
    IdentitiesOnly yes
    ProxyJump openhelp-jump
```

Replace:

- `JUMP_PUBLIC_IP` with `terraform output -raw jump_public_ip`
- `PRIVATE_VM_IP` with `terraform output -raw web_private_ip`

Connect with:

```bash
ssh openhelp-private
```

---

## 31. Verify that the private VM has no public IP

```bash
aws ec2 describe-instances \
  --region us-east-1 \
  --filters "Name=tag:Name,Values=openhelp-prod-private-web" \
  --query 'Reservations[].Instances[].{
    Name:Tags[?Key==`Name`]|[0].Value,
    State:State.Name,
    PrivateIP:PrivateIpAddress,
    PublicIP:PublicIpAddress,
    SubnetId:SubnetId,
    VpcId:VpcId
  }' \
  --output table
```

Expected meaning:

```text
PrivateIP = 10.0.3.x
PublicIP  = None
```

---

## 32. Verify internet access through the NAT Gateway

Connect to the private VM and run:

```bash
curl https://checkip.amazonaws.com
```

Display both NAT Gateway public IPs:

```bash
terraform output -json nat_gateway_public_ips
```

Because the current private web VM is deployed in private subnet 1 (`10.0.3.0/24`), its outbound address should match the **first** NAT Gateway Elastic IP.

You can display only the first value with:

```bash
terraform output -json nat_gateway_public_ips | jq -r '.[0]'
```

This demonstrates that private subnet 1 uses NAT Gateway 1 in the same Availability Zone.

Test DNS:

```bash
getent hosts ubuntu.com
```

Test HTTPS:

```bash
curl -I https://ubuntu.com
```

Test Ubuntu repositories:

```bash
sudo apt-get update
```

---

## 33. Verify Nginx

On the private VM:

```bash
systemctl status nginx --no-pager
```

Test the local page:

```bash
curl http://localhost
```

Expected text includes:

```text
OpenHelp Private Web Server
```

You can run the test from your laptop through SSH:

```bash
ssh -i openhelp-key.pem \
  -J ubuntu@$(terraform output -raw jump_public_ip) \
  ubuntu@$(terraform output -raw web_private_ip) \
  "curl -s http://localhost"
```

---


## 34. Verify the two NAT Gateway mappings

Display the NAT Gateways:

```bash
aws ec2 describe-nat-gateways \
  --region us-east-1 \
  --filter "Name=vpc-id,Values=$(terraform output -raw vpc_id)" \
  --query 'NatGateways[].{
    Name:Tags[?Key==`Name`]|[0].Value,
    State:State,
    SubnetId:SubnetId,
    NatGatewayId:NatGatewayId,
    PublicIP:NatGatewayAddresses[0].PublicIp,
    PrivateIP:NatGatewayAddresses[0].PrivateIp
  }' \
  --output table
```

Expected logical placement:

```text
openhelp-prod-nat-1 → public subnet 1 → us-east-1a
openhelp-prod-nat-2 → public subnet 2 → us-east-1b
```

Display Terraform's NAT outputs:

```bash
terraform output -json nat_gateway_ids
terraform output -json nat_gateway_public_ips
```

---

## 35. Verify the route tables

List route tables in the VPC:

```bash
aws ec2 describe-route-tables \
  --region us-east-1 \
  --filters "Name=vpc-id,Values=$(terraform output -raw vpc_id)" \
  --query 'RouteTables[].{
    RouteTableId:RouteTableId,
    Routes:Routes,
    Associations:Associations
  }'
```

The public route table should contain a route similar to:

```text
0.0.0.0/0 → igw-xxxxxxxx
```

The private route table should contain:

```text
0.0.0.0/0 → nat-xxxxxxxx
```

Both should also contain:

```text
10.0.0.0/16 → local
```

---

## 36. Verify security groups

```bash
aws ec2 describe-security-groups \
  --region us-east-1 \
  --filters \
    "Name=group-name,Values=openhelp-prod-jump-sg,openhelp-prod-private-vm-sg" \
  --output json
```

Expected design:

| Security group | Inbound | Outbound |
|---|---|---|
| Jump SG | TCP/22 from your `/32` | All IPv4 |
| Private VM SG | TCP/22 from Jump SG | All IPv4 |

---

# Troubleshooting

## 37. SSH to jump host times out

Check:

1. Your current public IP:

   ```bash
   curl https://checkip.amazonaws.com
   ```

2. `admin_cidr_blocks` contains that IP with `/32`.
3. The jump host has a public IP:

   ```bash
   terraform output -raw jump_public_ip
   ```

4. Public subnet route table has `0.0.0.0/0 → Internet Gateway`.
5. Jump security group allows TCP/22.
6. Local or corporate firewall permits outbound SSH.
7. The key pair matches the instance.

After changing your IP:

```bash
terraform apply
```

---

## 38. SSH through the jump host fails

Check direct jump-host access first:

```bash
ssh -i openhelp-key.pem ubuntu@$(terraform output -raw jump_public_ip)
```

Then verify the private address:

```bash
terraform output -raw web_private_ip
```

Check that the private security group references the jump security group.

Use verbose SSH output:

```bash
ssh -vvv \
  -i openhelp-key.pem \
  -J ubuntu@$(terraform output -raw jump_public_ip) \
  ubuntu@$(terraform output -raw web_private_ip)
```

Do not use `ec2-user` for Ubuntu. Use:

```text
ubuntu
```

---

## 39. Private VM cannot access the internet

Check:

1. Both NAT Gateway states:

   ```bash
   aws ec2 describe-nat-gateways \
     --region us-east-1 \
     --filter "Name=state,Values=available" \
     --output table
   ```

2. NAT Gateway 1 is in public subnet 1 and NAT Gateway 2 is in public subnet 2.
3. Each NAT Gateway has its own Elastic IP.
4. Both public subnets use the route table that points to the Internet Gateway.
5. Private route table 1 uses NAT Gateway 1.
6. Private route table 2 uses NAT Gateway 2.
7. Each private subnet is associated with its matching private route table.
7. The private security group permits outbound traffic.
8. VPC DNS support is enabled.

---

## 40. Nginx was not installed

Read cloud-init logs on the private VM:

```bash
sudo cloud-init status --long
```

```bash
sudo tail -n 200 /var/log/cloud-init-output.log
```

Check NAT access:

```bash
curl -I https://archive.ubuntu.com
```

Retry installation manually:

```bash
sudo apt-get update
sudo apt-get install -y nginx
```

---

## 41. AMI not found

AMI IDs are tied to a Region.

Confirm your Terraform Region:

```bash
grep '^region' terraform.tfvars
```

Confirm the AMI exists in that Region:

```bash
aws ec2 describe-images \
  --region us-east-1 \
  --image-ids YOUR_AMI_ID \
  --output table
```

Retrieve it again using the Canonical owner query from this guide.

---

## 42. Key pair not found

List key pairs:

```bash
aws ec2 describe-key-pairs \
  --region us-east-1 \
  --query 'KeyPairs[].KeyName' \
  --output table
```

The `key_name` value must be the AWS key-pair name:

```hcl
key_name = "openhelp-key"
```

It must not be:

```hcl
key_name = "openhelp-key.pem"
```

---

# Production considerations

## 43. Why this guide uses two NAT Gateways

This updated design uses one NAT Gateway per Availability Zone:

```text
us-east-1a:
Private subnet 1 → NAT Gateway 1 → Internet Gateway

us-east-1b:
Private subnet 2 → NAT Gateway 2 → Internet Gateway
```

Advantages:

- Each private subnet uses a NAT Gateway in the same Availability Zone.
- A failure affecting one NAT Gateway or Availability Zone does not automatically remove outbound internet access from the other Availability Zone.
- Cross-AZ NAT traffic is avoided when workloads use the matching route table.
- The layout follows the common production high-availability pattern.

Cost impact:

- Two NAT Gateways incur two hourly charges.
- Each NAT Gateway has data-processing charges.
- Each NAT Gateway uses a public IPv4 Elastic IP.

For a low-cost temporary lab, one NAT Gateway can be used, but it is less resilient.

---

## 44. Better alternatives to a permanent jump host

For production, consider AWS Systems Manager Session Manager because it can reduce the need for:

- A public bastion host
- Inbound SSH
- Long-lived SSH keys

This requires an IAM role, SSM Agent support, and suitable outbound connectivity or VPC endpoints.

The jump host is retained in this guide because it clearly demonstrates public/private subnet routing and SSH ProxyJump.

---

## 45. Additional hardening ideas

Consider these improvements for production:

- Use Session Manager rather than public SSH.
- Use one NAT Gateway per Availability Zone.
- Use VPC endpoints for services such as S3 and Systems Manager.
- Enable VPC Flow Logs.
- Use centralized remote Terraform state with locking.
- Restrict outbound security-group traffic where practical.
- Apply patching and vulnerability management.
- Use IAM roles rather than static AWS credentials on instances.
- Add monitoring and alerting.
- Add CloudTrail and AWS Config.
- Use a hardened or organization-approved AMI.
- Avoid storing secrets in user data or Terraform state.

---

# Clean up

## 46. Destroy the lab

Preview destruction:

```bash
terraform plan -destroy
```

Destroy:

```bash
terraform destroy
```

Enter:

```text
yes
```

Verify that NAT Gateways and EC2 instances are gone to avoid continuing charges.

The locally saved key remains on your laptop. Remove the AWS key pair when no longer required:

```bash
aws ec2 delete-key-pair \
  --region us-east-1 \
  --key-name openhelp-key
```

Remove the local private key only after confirming that it is no longer needed:

```bash
rm -f openhelp-key.pem
```

---

# Final resource summary

| Terraform resource | Quantity | Purpose |
|---|---:|---|
| `aws_vpc` | 1 | Main isolated network |
| `aws_internet_gateway` | 1 | Direct internet routing for public subnet resources |
| `aws_subnet.public` | 2 | Public networks in two AZs |
| `aws_subnet.private` | 2 | Private networks in two AZs |
| `aws_route_table.public` | 1 | Routes public traffic to IGW |
| `aws_route_table.private` | 2 | One private route table per Availability Zone |
| `aws_eip.nat` | 2 | One stable public IPv4 per NAT Gateway |
| `aws_nat_gateway` | 2 | One private-subnet egress gateway per Availability Zone |
| `aws_security_group.jump` | 1 | Protects jump host |
| `aws_security_group.private_vm` | 1 | Protects private VM |
| `aws_instance.jump` | 1 | Public SSH entry point |
| `aws_instance.web` | 1 | Private Ubuntu Nginx server |

---

# Reference documentation

The design and Terraform syntax in this guide were checked against the following primary documentation:

1. [Amazon VPC overview](https://docs.aws.amazon.com/vpc/latest/userguide/what-is-amazon-vpc.html)
2. [Internet Gateway and public/private subnet behavior](https://docs.aws.amazon.com/vpc/latest/userguide/VPC_Internet_Gateway.html)
3. [AWS route-table configuration](https://docs.aws.amazon.com/vpc/latest/userguide/VPC_Route_Tables.html)
4. [AWS routing options](https://docs.aws.amazon.com/vpc/latest/userguide/route-table-options.html)
5. [AWS NAT Gateways](https://docs.aws.amazon.com/vpc/latest/userguide/vpc-nat-gateway.html)
6. [AWS NAT Gateway basics and one-per-AZ resiliency guidance](https://docs.aws.amazon.com/vpc/latest/userguide/nat-gateway-basics.html)
7. [AWS example: private subnets with a NAT Gateway in each Availability Zone](https://docs.aws.amazon.com/vpc/latest/userguide/vpc-example-private-subnets-nat.html)
8. [AWS NAT Gateway use cases](https://docs.aws.amazon.com/vpc/latest/userguide/nat-gateway-scenarios.html)
9. [Connect to a Linux EC2 instance using SSH](https://docs.aws.amazon.com/AWSEC2/latest/UserGuide/connect-linux-inst-ssh.html)
10. [Canonical: Find Ubuntu images on AWS](https://documentation.ubuntu.com/aws/aws-how-to/instances/find-ubuntu-images/)
11. [Terraform AWS provider: VPC](https://registry.terraform.io/providers/hashicorp/aws/latest/docs/resources/vpc)
12. [Terraform AWS provider: Internet Gateway](https://registry.terraform.io/providers/hashicorp/aws/latest/docs/resources/internet_gateway)
13. [Terraform AWS provider: NAT Gateway](https://registry.terraform.io/providers/hashicorp/aws/latest/docs/resources/nat_gateway)
14. [Terraform AWS provider: Elastic IP](https://registry.terraform.io/providers/hashicorp/aws/latest/docs/resources/eip)
15. [Terraform AWS provider: Route table](https://registry.terraform.io/providers/hashicorp/aws/latest/docs/resources/route_table)
16. [Terraform AWS provider: EC2 instance](https://registry.terraform.io/providers/hashicorp/aws/latest/docs/resources/instance)
17. [Terraform AWS provider: Security group](https://registry.terraform.io/providers/hashicorp/aws/latest/docs/resources/security_group)

---

## End-to-end checklist

- [ ] AWS CLI installed
- [ ] Terraform installed
- [ ] AWS credentials configured
- [ ] `aws sts get-caller-identity` successful
- [ ] Official Ubuntu 24.04 AMI retrieved
- [ ] EC2 key pair created
- [ ] `.pem` permissions restricted
- [ ] Laptop public IP retrieved
- [ ] `/32` CIDR added to `terraform.tfvars`
- [ ] `terraform init` completed
- [ ] `terraform fmt` completed
- [ ] `terraform validate` successful
- [ ] `terraform plan` reviewed
- [ ] `terraform apply` successful
- [ ] Jump host reachable
- [ ] Private VM reachable through ProxyJump
- [ ] Private VM has no public IP
- [ ] Private subnet 1 outbound IP matches NAT Gateway 1 Elastic IP
- [ ] Private subnet 2 route points to NAT Gateway 2
- [ ] Nginx page works
- [ ] `terraform destroy` completed after testing
