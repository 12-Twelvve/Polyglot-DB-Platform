# Phase 1 — Terraform AWS Foundation

Provisions the entire AWS infrastructure for the Polyglot DB Platform.
Nothing is created via the console. Everything is reproducible and version-controlled.

## What gets provisioned

| Resource | Module | Notes |
|---|---|---|
| VPC | `modules/vpc` | 3 AZs, public + private subnets, NAT GW |
| EKS 1.30 | `modules/eks` | Managed node groups, IRSA, Karpenter-ready |
| RDS Postgres 16 | `modules/rds` | Multi-AZ, encrypted, Parameter Group |
| RDS MySQL 8 | `modules/rds` | Multi-AZ, encrypted, slow query log |
| ElastiCache Redis 7 | `modules/elasticache` | Cluster mode, TLS, auth token |
| DocumentDB 5 | `modules/documentdb` | 3-node cluster, TLS enforced |
| IAM + IRSA | `modules/iam` | Pod-level AWS permissions, no static keys |
| S3 + DynamoDB | `modules/s3-state` | Remote state backend with locking |

## Prerequisites

```bash
# Terraform >= 1.7
brew install terraform

# AWS CLI v2 configured
aws configure sso  # or export AWS_PROFILE=...

# kubectl
brew install kubectl

# For drift detection locally
pip install checkov   # optional but useful
```

## Bootstrap (first time only)

The remote state backend must exist before anything else. Bootstrap it once:

```bash
cd environments/bootstrap
terraform init
terraform apply
# outputs: bucket name + dynamodb table name
```

Then paste those values into `environments/*/backend.tf`.

## Usage

```bash
# Plan
cd environments/dev
terraform init
terraform plan -out=tfplan

# Apply
terraform apply tfplan

# After EKS is up — update kubeconfig
aws eks update-kubeconfig --region us-east-1 --name $(terraform output -raw eks_cluster_name)

# Destroy (be careful in prod)
terraform destroy
```

## Environments

```
environments/
  dev/       # cheap: single-AZ, t3.medium nodes, no Multi-AZ RDS
  staging/   # mirrors prod topology, smaller instances
  prod/      # full Multi-AZ, bigger instances, deletion protection
```

## Drift detection

A nightly GitHub Actions job runs `terraform plan` against prod.
If any diff is detected, the next deployment is blocked and Slack is notified.
See `.github/workflows/drift-detection.yml`.

## Module inputs/outputs

Each module has its own `README.md`, `variables.tf`, and `outputs.tf`.
Outputs are wired between modules in each environment's `main.tf`.

## Cost estimate (prod, us-east-1, ~2026 prices)

| Resource | $/month (approx) |
|---|---|
| EKS cluster | $73 |
| 3× m5.large nodes | $312 |
| RDS Postgres Multi-AZ db.t3.medium | $98 |
| RDS MySQL Multi-AZ db.t3.medium | $98 |
| ElastiCache cache.t3.medium | $48 |
| DocumentDB db.t3.medium ×3 | $180 |
| NAT Gateways ×3 | $99 |
| **Total** | **~$908/month** |

Dev environment is ~$180/month with single-AZ + smaller instances.