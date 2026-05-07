# iac-swarm ![Terraform](https://img.shields.io/badge/Terraform-1.x-purple?logo=terraform) ![Terragrunt](https://img.shields.io/badge/Terragrunt-latest-blue?logo=gruntwork) ![AWS](https://img.shields.io/badge/AWS-EC2%20%7C%20SSM-orange?logo=amazon-aws) ![License](https://img.shields.io/badge/License-MIT-green)

A real-world infrastructure repository for the presentation: **"EC2 Looked Normal, But Silently Failed: Investigating Hidden Issues Behind SSM"**

This repo demonstrates how a silent provisioning failure was investigated and fixed — caused by an **invalid SSM Parameter Store path** in an IAM policy.

> Presentation slides: [EC2 Looked Normal, But Silently Failed: Investigating Hidden Issues Behind SSM - Presentation](https://sintiasnn.github.io/assets/slides/EC2%20Looked%20Normal,%20But%20Silently%20Failed_%20Investigating%20Hidden%20Issues%20Behind%20SSM.pdf)

---

## Table of Contents

- [Overview](#overview)
- [Project Structure](#project-structure)
- [The Bug: SSM Path Mismatch](#the-bug-ssm-path-mismatch)
- [Investigation Steps](#investigation-steps)
- [Architecture](#architecture)
- [SSM Parameter Store](#ssm-parameter-store)
- [Getting Started](#getting-started)
- [Important Notes](#important-notes)
- [References](#references)

---

## Overview

**What this repo demonstrates:**

- How Docker Swarm provisioning can fail silently with no explicit error
- How AWS SSM Parameter Store is used as a coordination layer between EC2 nodes
- How a small IAM path misconfiguration causes a worker to never join the swarm
- How to investigate and fix the root cause using logs, SSH, and AWS CLI

**The scenario:**

A Docker Swarm worker was provisioned successfully — EC2 was running, health checks passed — but the service showed `0/1` replicas. No error was thrown. The worker had silently failed to join the swarm.

---

## Project Structure

```
iac-swarm/
├── infra/terragrunt/
│   ├── modules/                  # Reusable Terraform modules
│   │   ├── ec2/                  # EC2 + IAM roles + SSM policy
│   │   │   └── templates/
│   │   │       ├── user-data-master.sh       # Init swarm, store token to SSM
│   │   │       ├── user-data-master-join.sh  # Join as additional manager
│   │   │       ├── user-data-worker.sh       # Get token from SSM, join swarm
│   │   │       └── user-data-nginx.sh        # Setup nginx reverse proxy
│   │   ├── vpc/
│   │   ├── security-group/
│   │   └── keypair/
│   │
│   ├── environments/             # Per-environment Terragrunt configs
│   │   ├── dev/
│   │   │   ├── ec2-worker/       ← misconfigured config (already fixed)
│   │   │   ├── vpc/
│   │   │   ├── security-groups/
│   │   │   └── keypair/
│   │   ├── staging/
│   │   └── prod/
│   │
│   └── shared/                   # Shared configs across environments
│       ├── ec2/                  # Bastion, master, nginx, monitoring
│       ├── vpc-base/
│       ├── ecr/
│       ├── elastic-ip/
│       ├── s3/
│       └── variables/
│           └── ec2/variables.yaml
│
└── docker/
    └── _stacks_/                 # Local Docker Compose stacks
        ├── postgres.yaml
        ├── redis.yaml
        ├── minio.yaml
        ├── clickhouse.yaml
        ├── mailpit.yaml
        └── instrumentation.yaml
```

---

## The Bug: SSM Path Mismatch

This is the core of the presentation — a one-line misconfiguration that caused a silent failure.

### What was wrong

In `environments/dev/ec2-worker/terragrunt.hcl`, the IAM policy was scoped to:

```hcl
ssm_parameter_paths = ["/lgtm/swarm/*"]
```

But in `user-data-worker.sh`, the script reads from:

```bash
aws ssm get-parameter --name "/$PROJECT/swarm/$CLUSTER_IDENTIFIER/worker-token"
# where $PROJECT = "swarm-iac" (from root.hcl)
```

This generated the actual SSM path: `/swarm-iac/swarm/app-cluster/worker-token`

The IAM policy allowed `/lgtm/swarm/*` but the script accessed `/swarm-iac/swarm/*` — **access denied, silently**.

### The fix

```hcl
ssm_parameter_paths = ["/swarm-iac/swarm/*"]
```

Applied to: `environments/dev/ec2-worker`, `environments/staging/ec2-worker`, and all instances in `shared/ec2`.

---

## Investigation Steps

How the root cause was found — following the 5-step process from the presentation:

**Step 1: Check Docker Swarm cluster**
```sh
ssh ubuntu@<manager-ip> "docker service ls"
# → some services showing 0/1 replicas
```

**Step 2: Check provisioning logs**
```sh
ssh ubuntu@<worker-ip> "cat /var/log/user-data.log"
# → "Failed to retrieve swarm information... on attempt N"
```

**Step 3: SSH into the worker**
```sh
ssh ubuntu@<worker-ip> "docker ps"
# → no containers running
```

**Step 4: Trace the SSM parameter**
```sh
aws ssm get-parameter --name "/swarm-iac/swarm/app-cluster/worker-token" --with-decryption
# → AccessDeniedException
```

**Step 5: Trace via Terraform source code**
```sh
# Found in environments/dev/ec2-worker/terragrunt.hcl:
ssm_parameter_paths = ["/lgtm/swarm/*"]   # ← wrong prefix
# But root.hcl defines:
project_name = "swarm-iac"                 # ← actual prefix used by user-data
```

---

## Architecture

```
AWS Docker Swarm Cluster
├── Manager node  — initializes swarm, stores join token to SSM
└── Worker node   — reads join token from SSM, joins swarm
```

Inter-node communication uses **AWS SSM Parameter Store** as a token relay:

```
Manager → aws ssm put-parameter /swarm-iac/swarm/<cluster>/worker-token
Worker  → aws ssm get-parameter /swarm-iac/swarm/<cluster>/worker-token → docker swarm join
```

---

## SSM Parameter Store

The IAM policy in the `ec2` module grants access to a specific path:

```hcl
ssm_parameter_paths = ["/swarm-iac/swarm/*"]
```

Which generates the following IAM policy:

```json
{
  "Effect": "Allow",
  "Action": [
    "ssm:GetParameter",
    "ssm:GetParameters",
    "ssm:GetParametersByPath",
    "ssm:PutParameter"
  ],
  "Resource": "arn:aws:ssm:<region>:<account>:parameter/swarm-iac/swarm/*"
}
```

Parameters in use:

| Parameter | Created by | Read by |
|-----------|------------|---------|
| `/swarm-iac/swarm/<cluster>/worker-token` | Manager (init) | Worker |
| `/swarm-iac/swarm/<cluster>/master-ip` | Manager (init) | Worker |

---

## Getting Started

### Prerequisites

- [Terraform](https://developer.hashicorp.com/terraform/install) >= 1.0
- [Terragrunt](https://terragrunt.gruntwork.io/docs/getting-started/install/) >= 0.50
- AWS CLI configured
- SSH key at `~/.ssh/id_rsa`

### Provisioning order

```sh
# 1. Shared infrastructure
cd infra/terragrunt/shared/vpc-base && terragrunt apply
cd infra/terragrunt/shared/elastic-ip && terragrunt apply

# 2. Environment (example: dev)
cd infra/terragrunt/environments/dev/vpc && terragrunt apply
cd infra/terragrunt/environments/dev/security-groups && terragrunt apply
cd infra/terragrunt/environments/dev/keypair && terragrunt apply

# 3. EC2 instances (manager first, then worker)
cd infra/terragrunt/shared/ec2 && terragrunt apply
cd infra/terragrunt/environments/dev/ec2-worker && terragrunt apply
```

### Post-provision validation

```sh
# Check provisioning logs on worker
ssh ubuntu@<worker-ip> "tail -f /var/log/user-data.log"

# Verify SSM parameter is available
aws ssm get-parameter --name "/swarm-iac/swarm/app-cluster/worker-token" --with-decryption

# Check swarm status from manager
ssh ubuntu@<manager-ip> "docker node ls"
```

### Docker Stacks (Local)

```sh
# Deploy a stack (example: postgres)
docker stack deploy -c docker/_stacks_/postgres.yaml postgres

# Or use compose directly
docker-compose -f docker/_stacks_/postgres.yaml up -d
```

Available stacks: `postgres`, `redis`, `minio`, `clickhouse`, `mailpit`, `instrumentation`.

---

## Important Notes

**This repo is based on a real production incident.**

- ❌ Do not apply to real AWS without understanding the consequences
- ❌ SSH keys and AWS credentials are not included
- ⚠️ The bug (`/lgtm/swarm/*`) is preserved in git history for reference — the fix is on `main`
- ✅ All `ssm_parameter_paths` now correctly use `/swarm-iac/swarm/*`

**Key lesson:** Provisioning automation can fail silently. Always validate post-provision, not just the Terraform apply output.

---

## Author

**Ni Putu Sintia Wati**
- GitHub: [@sintiasnn](https://github.com/sintiasnn)

---

## References

- [AWS SSM Parameter Store Documentation](https://docs.aws.amazon.com/systems-manager/latest/userguide/systems-manager-parameter-store.html)
- [Terraform AWS Provider — IAM Policy](https://registry.terraform.io/providers/hashicorp/aws/latest/docs/resources/iam_role_policy)
- [Terragrunt Documentation](https://terragrunt.gruntwork.io/docs/)
- [Docker Swarm Documentation](https://docs.docker.com/engine/swarm/)

## Acknowledgments

This repository is based on the monorepo template by [Zero One Group](https://github.com/zero-one-group/monorepo/tree/main).

---

