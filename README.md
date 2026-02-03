# AWS & Terraform — EC2 Setup Guide

A documentation covering AWS fundamentals, Terraform basics, and two ways to spin up an EC2 instance.

---

## 1. AWS Core Concepts

| Concept | Description |
|---|---|
| **Region** | Physical geographic area (e.g., `us-east-1`). Choose based on latency & compliance. |
| **Availability Zone (AZ)** | Isolated data center(s) within a Region. Use multiple AZs for high availability. |
| **VPC** | Virtual Private Cloud — your isolated network inside AWS. |
| **Subnet** | A range of IPs inside a VPC. Can be *public* or *private*. |
| **Security Group** | Stateful firewall that controls inbound/outbound traffic to your resources. |
| **IAM** | Identity & Access Management — controls *who* can do *what* in your account. |
| **EC2** | Elastic Compute Cloud — virtual machines you rent by the hour/second. |
| **AMI** | Amazon Machine Image — a pre-built OS snapshot used to launch EC2 instances. |
| **Key Pair** | SSH key pair for secure access to Linux EC2 instances. |

---

## 2. Terraform Basics

Terraform is an **Infrastructure as Code (IaC)** tool that lets you define and manage cloud resources via plain-text `.tf` files.

### Key Ideas

- **Declarative** — You describe the *desired state*; Terraform figures out *how* to get there.
- **State** — Terraform tracks real infrastructure in a `terraform.tfstate` file.
- **Provider** — A plugin that talks to a cloud API (e.g., `hashicorp/aws`).

### Core Commands

```bash
terraform init      # Download providers & initialize the working directory
terraform plan      # Preview changes before applying
terraform apply     # Create / update resources
terraform destroy   # Tear down all managed resources
```

### Project Structure

```
project/
├── main.tf          # Resource definitions
├── variables.tf     # Input variables
├── outputs.tf       # Outputs (e.g., public IP)
└── terraform.tfvars # Variable values
```

---

## 3. Manual EC2 Creation (AWS Console)

1. Sign in → open the **EC2** console.
2. Click **Launch Instances**.
3. **Name** your instance (e.g., `my-web-server`).
4. Choose an **AMI** (e.g., *Amazon Linux 2023*).
5. Pick an **Instance Type** (e.g., `t3.micro` for free-tier eligible).
6. **Key Pair** — select an existing key or create a new one (`.pem` / `.ppk`).
7. **Network Settings**
   - Select or create a **VPC** and **Subnet**.
   - Enable *Auto-assign Public IP* if you need direct internet access.
   - Attach or create a **Security Group** (open port 22 for SSH, 80/443 for HTTP/S).
8. **Storage** — keep the default 8 GB gp3 EBS volume (or adjust).
9. Review and click **Launch Instance**.
10. Connect via SSH:
    ```bash
    ssh -i my-key.pem ec2-user@<public-ip>
    ```

---

## 4. Terraform EC2 Creation

### `variables.tf`
```hcl
variable "ami_id" {
  default = "ami-0abcdef1234567890"   # Replace with a valid AMI for your region
}

variable "instance_type" {
  default = "t3.micro"
}

variable "key_name" {
  default = "my-key"                  # Name of your existing EC2 Key Pair
}
```

### `main.tf`
```hcl
provider "aws" {
  region = "us-east-1"
}

# Security Group — allow SSH and HTTP
resource "aws_security_group" "web_sg" {
  name        = "web-sg"
  description = "Allow SSH and HTTP"

  ingress {
    from_port   = 22
    to_port     = 22
    protocol    = "tcp"
    cidr_blocks = ["0.0.0.0/0"]   # Restrict to your IP in production
  }

  ingress {
    from_port   = 80
    to_port     = 80
    protocol    = "tcp"
    cidr_blocks = ["0.0.0.0/0"]
  }

  egress {
    from_port   = 0
    to_port     = 0
    protocol    = "-1"
    cidr_blocks = ["0.0.0.0/0"]
  }
}

# EC2 Instance
resource "aws_instance" "web_server" {
  ami                    = var.ami_id
  instance_type          = var.instance_type
  key_name               = var.key_name
  vpc_security_group_ids = [aws_security_group.web_sg.id]

  tags = {
    Name = "my-web-server"
  }
}
```

### `outputs.tf`
```hcl
output "public_ip" {
  value = aws_instance.web_server.public_ip
}
```

### Deploy
```bash
terraform init
terraform plan          # Review the planned changes
terraform apply         # Confirm with "yes" to provision
```

After `apply`, Terraform prints the **public IP** — use it to SSH in or hit your app.

### Tear Down
```bash
terraform destroy       # Stops and terminates all resources managed here
```

---

## Quick Comparison

| Step | Console (Manual) | Terraform (IaC) |
|---|---|---|
| Repeatability | ✗ Manual each time | ✓ Run `apply` anywhere |
| Version Control | ✗ Not trackable | ✓ Commit `.tf` files to Git |
| Speed (repeat) | Slow | Fast |
| Audit Trail | CloudTrail only | Code history in Git |
| Team Collaboration | Error-prone | Single source of truth |

---

> **Tip:** Always store your Terraform state remotely (e.g., S3 + DynamoDB) when working in a team.
