# Cloud Platforms Reference

## Table of Contents
1. Core Concepts
2. AWS — Key Services
3. GCP — Key Services
4. Azure — Key Services
5. Choosing a Cloud
6. Common Patterns (VPC, IAM, Storage)
7. File Structure

---

## 1. Core Concepts

**Cloud computing** = rent compute, storage, and networking on-demand instead of owning hardware.

```
On-Premise:  You manage everything (hardware → OS → runtime → app)
IaaS:        Cloud manages hardware; you manage OS → runtime → app  (EC2, GCE)
PaaS:        Cloud manages hardware + OS; you manage app            (App Engine, Heroku)
SaaS:        Cloud manages everything; you just use the product     (Gmail, Salesforce)
```

---

## 2. AWS — Key Services

| Category | Service | Purpose |
|---|---|---|
| Compute | EC2 | Virtual machines |
| Compute | ECS / EKS | Docker containers / Kubernetes |
| Compute | Lambda | Serverless functions |
| Storage | S3 | Object storage (files, backups, static sites) |
| Storage | EBS | Block storage (attached to EC2) |
| Database | RDS | Managed PostgreSQL / MySQL / etc. |
| Database | DynamoDB | Serverless NoSQL |
| Networking | VPC | Private network |
| Networking | Route 53 | DNS |
| Networking | ALB / NLB | Load balancers |
| Security | IAM | Identity & access management |
| Security | Secrets Manager | Store secrets securely |
| CI/CD | CodePipeline | Managed CI/CD |
| Registry | ECR | Docker image registry |

### Minimal AWS Setup (Terraform)

```hcl
# terraform/modules/aws-infra/main.tf

# S3 bucket for app assets
resource "aws_s3_bucket" "assets" {
  bucket = "${var.project_name}-assets-${var.environment}"

  tags = {
    Environment = var.environment
  }
}

# Block all public access by default
resource "aws_s3_bucket_public_access_block" "assets" {
  bucket                  = aws_s3_bucket.assets.id
  block_public_acls       = true
  block_public_policy     = true
  ignore_public_acls      = true
  restrict_public_buckets = true
}

# IAM role for ECS task
resource "aws_iam_role" "ecs_task" {
  name = "${var.project_name}-ecs-task-role"

  assume_role_policy = jsonencode({
    Version = "2012-10-17"
    Statement = [{
      Action    = "sts:AssumeRole"
      Effect    = "Allow"
      Principal = { Service = "ecs-tasks.amazonaws.com" }
    }]
  })
}

# Allow ECS task to read from Secrets Manager
resource "aws_iam_role_policy_attachment" "ecs_secrets" {
  role       = aws_iam_role.ecs_task.name
  policy_arn = "arn:aws:iam::aws:policy/SecretsManagerReadWrite"
}
```

### AWS CLI Essentials

```bash
# Configure credentials
aws configure

# S3
aws s3 ls s3://my-bucket
aws s3 cp file.txt s3://my-bucket/
aws s3 sync ./dist s3://my-bucket/

# EC2
aws ec2 describe-instances --filters "Name=tag:Name,Values=my-server"

# ECS
aws ecs list-clusters
aws ecs describe-services --cluster my-cluster --services my-service
aws ecs update-service --cluster my-cluster --service my-service --force-new-deployment

# Secrets Manager
aws secretsmanager get-secret-value --secret-id my-app/production/db-password
```

⚠️ **IAM Best Practices:**
- Never use root account for daily work
- Grant least privilege (only what's needed)
- Use roles for services, not access keys where possible
- Rotate access keys regularly
- Enable MFA on all accounts

---

## 3. GCP — Key Services

| Category | Service | AWS Equivalent |
|---|---|---|
| Compute | Compute Engine | EC2 |
| Containers | GKE | EKS |
| Serverless | Cloud Run | Lambda (but containerized) |
| Storage | Cloud Storage | S3 |
| Database | Cloud SQL | RDS |
| Database | Firestore | DynamoDB |
| Networking | Cloud VPC | VPC |
| DNS | Cloud DNS | Route 53 |
| Registry | Artifact Registry | ECR |
| CI/CD | Cloud Build | CodePipeline |

```bash
# GCP CLI (gcloud) essentials
gcloud auth login
gcloud config set project my-project-id

# Cloud Run deploy
gcloud run deploy my-app \
  --image gcr.io/my-project/my-app:v1.0.0 \
  --platform managed \
  --region us-central1 \
  --allow-unauthenticated

# GKE cluster
gcloud container clusters create my-cluster \
  --region us-central1 \
  --num-nodes 3

gcloud container clusters get-credentials my-cluster --region us-central1
```

---

## 4. Azure — Key Services

| Category | Service | AWS Equivalent |
|---|---|---|
| Compute | Azure VMs | EC2 |
| Containers | AKS | EKS |
| Serverless | Azure Functions | Lambda |
| Storage | Blob Storage | S3 |
| Database | Azure SQL | RDS |
| Registry | ACR | ECR |
| CI/CD | Azure DevOps | CodePipeline |
| Secrets | Key Vault | Secrets Manager |

```bash
# Azure CLI essentials
az login
az account set --subscription "My Subscription"

# AKS cluster
az aks create \
  --resource-group myResourceGroup \
  --name myAKSCluster \
  --node-count 3 \
  --generate-ssh-keys

az aks get-credentials --resource-group myResourceGroup --name myAKSCluster
```

---

## 5. Common Patterns

### VPC Design (any cloud)
```
VPC: 10.0.0.0/16
├── Public Subnets (10.0.1.0/24, 10.0.2.0/24)   ← Load balancers, NAT gateway
│     Internet Gateway attached
└── Private Subnets (10.0.10.0/24, 10.0.11.0/24) ← App servers, databases
      No direct internet access (goes via NAT)
```

### Multi-region setup
```
Primary Region (us-east-1)     ← Active traffic
Secondary Region (eu-west-1)   ← Standby / DR
          ↑
    Route 53 health checks route traffic on failure
```

---

## 6. File Structure

```
terraform/
├── environments/
│   ├── staging/
│   │   ├── main.tf
│   │   └── terraform.tfvars     # staging-specific values
│   └── production/
│       ├── main.tf
│       └── terraform.tfvars
└── modules/
    ├── aws-vpc/
    ├── aws-ecs/
    ├── aws-rds/
    └── aws-iam/
```