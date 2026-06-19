# Terraform Reference

## Table of Contents
1. Core Concepts
2. Project Structure & State
3. Resources, Variables, Outputs
4. Modules
5. Workspaces & Environments
6. Provisioners & Data Sources
7. Terraform Commands
8. Best Practices
9. File Structure

---

## 1. Core Concepts

**Terraform** = declarative Infrastructure as Code. You describe the desired end state;
Terraform figures out how to get there (create/update/destroy resources).

```
.tf files (desired state) ──[terraform plan]──> Diff vs current state
                                                        ↓
                                              [terraform apply] → Real infrastructure
                                                        ↓
                                              terraform.tfstate (tracks what exists)
```

**Core workflow:**
```
write code → init → plan → apply → (later) destroy
```

---

## 2. Project Structure & State

```hcl
# terraform/main.tf
terraform {
  required_version = ">= 1.6"

  required_providers {
    aws = {
      source  = "hashicorp/aws"
      version = "~> 5.0"
    }
  }

  # Remote state — required for team collaboration
  backend "s3" {
    bucket         = "my-terraform-state-bucket"
    key            = "production/terraform.tfstate"
    region         = "us-east-1"
    dynamodb_table = "terraform-locks"   # Prevents two people applying at once
    encrypt        = true
  }
}

provider "aws" {
  region = var.aws_region
}
```

⚠️ **Never commit `terraform.tfstate` to Git** — it can contain secrets in plain text.
Always use a remote backend (S3, Terraform Cloud, GCS, Azure Storage) for team projects.

---

## 3. Resources, Variables, Outputs

```hcl
# terraform/variables.tf
variable "aws_region" {
  description = "AWS region to deploy into"
  type        = string
  default     = "us-east-1"
}

variable "environment" {
  description = "Deployment environment"
  type        = string
  validation {
    condition     = contains(["staging", "production"], var.environment)
    error_message = "Environment must be 'staging' or 'production'."
  }
}

variable "instance_count" {
  description = "Number of EC2 instances"
  type        = number
  default     = 2
}

variable "tags" {
  description = "Common resource tags"
  type        = map(string)
  default     = {}
}
```

```hcl
# terraform/main.tf (resources)
resource "aws_vpc" "main" {
  cidr_block           = "10.0.0.0/16"
  enable_dns_hostnames = true

  tags = merge(var.tags, {
    Name = "${var.environment}-vpc"
  })
}

resource "aws_instance" "web" {
  count         = var.instance_count          # Creates N instances
  ami           = data.aws_ami.ubuntu.id
  instance_type = "t3.micro"
  subnet_id     = aws_subnet.public[count.index].id

  tags = {
    Name = "${var.environment}-web-${count.index}"
  }

  lifecycle {
    create_before_destroy = true               # Avoid downtime on replace
  }
}
```

```hcl
# terraform/outputs.tf
output "vpc_id" {
  description = "ID of the created VPC"
  value       = aws_vpc.main.id
}

output "instance_ips" {
  description = "Public IPs of web instances"
  value       = aws_instance.web[*].public_ip   # Splat expression for all instances
}

output "db_password" {
  value     = aws_db_instance.main.password
  sensitive = true                               # Hides value in CLI output
}
```

---

## 4. Modules

Modules are reusable, parameterized groups of resources — like functions for infrastructure.

```hcl
# terraform/modules/vpc/main.tf
variable "cidr_block" { type = string }
variable "name"       { type = string }

resource "aws_vpc" "this" {
  cidr_block = var.cidr_block
  tags       = { Name = var.name }
}

output "vpc_id" {
  value = aws_vpc.this.id
}
```

```hcl
# terraform/main.tf (using the module)
module "vpc" {
  source     = "./modules/vpc"
  cidr_block = "10.0.0.0/16"
  name       = "${var.environment}-vpc"
}

resource "aws_subnet" "public" {
  vpc_id     = module.vpc.vpc_id        # Reference module output
  cidr_block = "10.0.1.0/24"
}
```

You can also pull modules from the public registry:
```hcl
module "vpc" {
  source  = "terraform-aws-modules/vpc/aws"
  version = "5.1.0"

  name = "my-vpc"
  cidr = "10.0.0.0/16"
}
```

---

## 5. Workspaces & Environments

**Option A — Workspaces** (same code, isolated state per environment):
```bash
terraform workspace new staging
terraform workspace new production
terraform workspace select staging
terraform workspace list
```

**Option B — Separate directories** (more explicit, recommended for prod):
```
environments/
├── staging/
│   ├── main.tf
│   └── terraform.tfvars
└── production/
    ├── main.tf
    └── terraform.tfvars
```

```bash
# terraform.tfvars (staging)
environment     = "staging"
instance_count  = 1
aws_region      = "us-east-1"
```

```bash
terraform apply -var-file="terraform.tfvars"
```

---

## 6. Provisioners & Data Sources

**Data sources** — read existing infrastructure not managed by this config:
```hcl
data "aws_ami" "ubuntu" {
  most_recent = true
  owners      = ["099720109477"]   # Canonical

  filter {
    name   = "name"
    values = ["ubuntu/images/hvm-ssd/ubuntu-jammy-22.04-amd64-server-*"]
  }
}

data "aws_availability_zones" "available" {
  state = "available"
}
```

**Provisioners** — run scripts after resource creation (use sparingly; prefer Ansible/cloud-init):
```hcl
resource "aws_instance" "web" {
  ami           = data.aws_ami.ubuntu.id
  instance_type = "t3.micro"

  provisioner "remote-exec" {
    inline = [
      "sudo apt update",
      "sudo apt install -y nginx"
    ]

    connection {
      type = "ssh"
      user = "ubuntu"
      host = self.public_ip
    }
  }
}
```

---

## 7. Terraform Commands

```bash
terraform init                       # Download providers + configure backend
terraform fmt                        # Auto-format .tf files
terraform validate                   # Check syntax without contacting cloud
terraform plan                       # Preview changes (always run before apply)
terraform plan -out=tfplan           # Save plan to a file
terraform apply                      # Apply changes (prompts for confirmation)
terraform apply tfplan               # Apply a saved plan (no prompt)
terraform apply -auto-approve        # Skip confirmation (use carefully, e.g. in CI)
terraform destroy                    # Tear down all resources
terraform destroy -target=aws_instance.web   # Destroy a specific resource

terraform state list                 # List all resources in state
terraform state show aws_vpc.main    # Show details of one resource
terraform import aws_vpc.main vpc-0123456  # Bring existing resource under management

terraform output                     # Show all outputs
terraform output vpc_id              # Show one output
terraform graph | dot -Tpng > graph.png   # Visualize resource dependencies
```

---

## 8. Best Practices

⚠️ **Common mistakes to avoid:**
- Committing `terraform.tfstate` or `*.tfvars` (secrets) to Git
- Running `terraform apply` without reviewing `terraform plan` first
- Hardcoding values instead of using variables
- Not pinning provider versions (`~> 5.0` not just `aws`)
- Sharing local state files across a team (use remote backend + locking)
- Using `count` when `for_each` is clearer (for_each uses keys, avoiding reordering issues)

✅ **Good practices:**
- One remote state per environment (staging ≠ production state)
- Use `terraform fmt` and `terraform validate` in CI before merge
- Tag every resource (`Environment`, `Project`, `ManagedBy = "terraform"`)
- Use `sensitive = true` for outputs containing secrets
- Run `terraform plan` in CI on every PR; only `apply` on merge to main

---

## 9. File Structure

```
terraform/
├── main.tf                    # Provider config + primary resources
├── variables.tf                # Input variable definitions
├── outputs.tf                  # Output values
├── data.tf                     # Data source lookups
├── locals.tf                   # Computed local values
├── versions.tf                 # Terraform + provider version constraints
├── terraform.tfvars.example     # Template (commit this, not the real .tfvars)
├── .gitignore                   # Ignore .terraform/, *.tfstate, *.tfvars
├── modules/
│   ├── vpc/
│   │   ├── main.tf
│   │   ├── variables.tf
│   │   └── outputs.tf
│   ├── ecs/
│   └── rds/
└── environments/
    ├── staging/
    │   ├── main.tf             # Calls modules with staging values
    │   └── terraform.tfvars
    └── production/
        ├── main.tf
        └── terraform.tfvars
```  
