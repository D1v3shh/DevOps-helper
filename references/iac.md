# Infrastructure as Code Reference

## Table of Contents
1. Core Concepts
2. Terraform — Complete Example
3. Ansible — Complete Example
4. Pulumi (TypeScript)
5. When to use which tool
6. File Structure

---

## 1. Core Concepts

**IaC** = define your infrastructure in code files, version-controlled like app code.

| Tool | Type | Language | State |
|---|---|---|---|
| Terraform | Provisioning | HCL | Remote state (S3, etc.) |
| Ansible | Configuration | YAML | Stateless |
| Pulumi | Provisioning | TS/Python/Go | Pulumi Cloud or S3 |
| CloudFormation | Provisioning | YAML/JSON | AWS-managed |

---

## 2. Terraform — AWS Example

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

  # Store state remotely so team can collaborate
  backend "s3" {
    bucket         = "my-terraform-state"
    key            = "production/terraform.tfstate"
    region         = "us-east-1"
    dynamodb_table = "terraform-locks"   # Prevents concurrent applies
    encrypt        = true
  }
}

provider "aws" {
  region = var.aws_region
}

# VPC
resource "aws_vpc" "main" {
  cidr_block           = "10.0.0.0/16"
  enable_dns_hostnames = true

  tags = {
    Name        = "${var.project_name}-vpc"
    Environment = var.environment
  }
}

# Subnet
resource "aws_subnet" "public" {
  count             = 2
  vpc_id            = aws_vpc.main.id
  cidr_block        = cidrsubnet(aws_vpc.main.cidr_block, 8, count.index)
  availability_zone = data.aws_availability_zones.available.names[count.index]

  tags = {
    Name = "${var.project_name}-public-${count.index}"
  }
}

# ECS Cluster
resource "aws_ecs_cluster" "main" {
  name = "${var.project_name}-cluster"
}
```

```hcl
# terraform/variables.tf
variable "aws_region" {
  description = "AWS region to deploy into"
  type        = string
  default     = "us-east-1"
}

variable "environment" {
  description = "Deployment environment (staging/production)"
  type        = string
  validation {
    condition     = contains(["staging", "production"], var.environment)
    error_message = "Environment must be 'staging' or 'production'."
  }
}

variable "project_name" {
  description = "Project identifier used in resource names"
  type        = string
}
```

```hcl
# terraform/outputs.tf
output "vpc_id" {
  description = "ID of the created VPC"
  value       = aws_vpc.main.id
}

output "cluster_arn" {
  description = "ARN of the ECS cluster"
  value       = aws_ecs_cluster.main.arn
}
```

**Terraform workflow:**
```bash
terraform init          # Download providers, configure backend
terraform plan          # Preview changes (always do this first!)
terraform apply         # Apply changes
terraform destroy       # Tear down all resources
```

---

## 3. Ansible — Server Configuration Example

```yaml
# ansible/playbooks/setup-webserver.yml
---
- name: Configure web server
  hosts: webservers
  become: true          # Run as sudo

  vars:
    app_user: deploy
    app_dir: /opt/myapp
    node_version: "20"

  tasks:
    - name: Update apt cache
      apt:
        update_cache: yes
        cache_valid_time: 3600

    - name: Install required packages
      apt:
        name:
          - nginx
          - git
          - curl
        state: present

    - name: Create app user
      user:
        name: "{{ app_user }}"
        shell: /bin/bash
        create_home: yes

    - name: Create app directory
      file:
        path: "{{ app_dir }}"
        state: directory
        owner: "{{ app_user }}"
        mode: '0755'

    - name: Copy nginx config
      template:
        src: templates/nginx.conf.j2
        dest: /etc/nginx/sites-available/myapp
        mode: '0644'
      notify: Reload nginx       # Trigger handler below

    - name: Enable nginx site
      file:
        src: /etc/nginx/sites-available/myapp
        dest: /etc/nginx/sites-enabled/myapp
        state: link

  handlers:
    - name: Reload nginx
      service:
        name: nginx
        state: reloaded
```

```ini
# ansible/inventory/production
[webservers]
web1.example.com ansible_user=ubuntu
web2.example.com ansible_user=ubuntu

[databases]
db1.example.com ansible_user=ubuntu
```

```bash
# Run the playbook
ansible-playbook -i inventory/production playbooks/setup-webserver.yml
```

---

## 4. File Structure

```
terraform/
├── main.tf              # Provider config, main resources
├── variables.tf         # Input variable definitions
├── outputs.tf           # Output values
├── data.tf              # Data sources (lookup existing resources)
├── locals.tf            # Local computed values
└── modules/
    ├── vpc/             # Reusable VPC module
    │   ├── main.tf
    │   ├── variables.tf
    │   └── outputs.tf
    └── ecs/             # Reusable ECS module

ansible/
├── inventory/
│   ├── staging          # Staging hosts
│   └── production       # Production hosts
├── playbooks/
│   ├── setup-webserver.yml
│   └── deploy-app.yml
├── roles/               # Reusable role bundles
│   └── nodejs/
│       ├── tasks/main.yml
│       ├── templates/
│       └── defaults/main.yml
├── group_vars/          # Variables per host group
│   └── all.yml
└── ansible.cfg          # Ansible configuration
```