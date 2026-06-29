# dwe_dbt_kg — Infrastructure Template

Pulumi IaC template for deploying the [dbt Knowledge Graph](https://github.com/ponderdwe/dbt-kg) stack on AWS.

Provisions: ALB (HTTP→HTTPS) + Auto Scaling Group (single-instance) + EC2 + Route53 CNAME + optional EFS persistent volume for FalkorDB.

## Stack

| Component | Details |
|-----------|---------|
| **FalkorDB** | Graph DB + vector + full-text indexes |
| **FastAPI** | dbt manifest upload, embedding rebuild, chat backend |
| **Streamlit** | Password-protected chat UI |
| **ALB** | HTTP→HTTPS redirect, HTTPS termination (ACM), configurable internal/internet-facing |
| **ASG** | min=max=1, rolling instance refresh on deploy |
| **EFS** (optional) | Persistent FalkorDB data volume — survives instance replacements |

## Prerequisites

- Pulumi CLI + S3 state backend
- AWS credentials with EC2 / EFS / ALB / Route53 / Secrets Manager / IAM permissions
- Secrets loaded into AWS Secrets Manager (see [Required Secrets](#required-secrets))

## Deploy

```bash
pulumi stack init dev
pulumi config set environment dev
pulumi config set git_branch main
pulumi config set secret_id my-dbt-kg-secrets
pulumi up
```

## Required Secrets

Store these as a single JSON object in AWS Secrets Manager:

| Key | Description |
|-----|-------------|
| `VPC_ID` | VPC where infrastructure is deployed |
| `ALB_SUBNET_IDS` | JSON array of subnet IDs for the ALB (e.g. `["subnet-aaa","subnet-bbb"]`) |
| `KEY_NAME` | EC2 key pair name |
| `EC2_SECURITY_GROUP_ID` | Extra SG to attach to the EC2 instance |
| `LB_SECURITY_GROUP_ID` | SG to attach to the ALB |
| `ALB_INTERNAL` | `"true"` for VPC-only ALB, `"false"` for internet-facing |
| `ROUTE53_ZONE_ID` | Route53 hosted zone ID |
| `DNS_NAME` | Public DNS name (e.g. `dbt-kg.myorg.com`) |
| `ACM_CERTIFICATE_ARN` | ACM certificate ARN covering `DNS_NAME` |
| `git_deploy_token` | GitHub / GitLab token for EC2 to clone the repo |
| `git_deploy_username` | Username paired with the deploy token |
| `LLM_MODEL_ID` | LLM in `provider:model` format (e.g. `bedrock:anthropic.claude-sonnet-4-20250514-v1:0`) |
| `SECRET_KEY` | FastAPI session middleware secret (`openssl rand -hex 32`) |
| `FAST_API_ACCESS_SECRET_TOKEN` | Shared token between Streamlit and FastAPI |
| `STREAMLIT_PASSWORD` | Password shown on the Streamlit login screen |
| `POSTGRES_PASSWORD` | PostgreSQL password for the LangGraph chat-history checkpointer |

## Optional Secrets

| Key | Description |
|-----|-------------|
| `EBS_FALKORDB_STORAGE_PERSIST` | Set to `"true"` to mount an EFS filesystem at `/mnt/falkordb-data` — FalkorDB graph data persists across instance replacements |
| `GRAPH_USER` | FalkorDB username (default: `default`) |
| `GRAPH_PASSWORD` | FalkorDB password |
| `AWS_DEFAULT_REGION` | AWS region (default: `us-east-1`) |
| `HOSTED_ZONE_ID_EC2_DNS` | Route53 zone ID for the EC2 A record |
| `EC2_DNS` | DNS name for the EC2 A record (private IP for internal ALB, public IP for internet-facing) |
| `LLM_API_KEY` | API key for non-Bedrock LLM providers |
| `BEDROCK_RERANKER_MODEL_ARN` | Full ARN of the Bedrock reranker (default: Cohere Rerank v3.5 in deployment region) |
| `SPLIT_EMBEDDINGS` | Set to `"true"` to split large node text into chunks stored in a separate `dbt_graph_chunks` FalkorDB graph (recommended when using Bedrock Titan which has an 8192-token limit) |

## Persistent FalkorDB Storage (EFS)

By default, FalkorDB data is stored on the EC2 root volume and lost on instance replacement. To persist it:

1. Add `EBS_FALKORDB_STORAGE_PERSIST = "true"` to your Secrets Manager secret.
2. Run `pulumi up`.

Pulumi will:
- Create an EFS filesystem (encrypted)
- Create mount targets in each subnet listed in `ALB_SUBNET_IDS`
- Create a dedicated security group allowing NFS (port 2049) from the EC2 security group
- Mount the filesystem at `/mnt/falkordb-data` on boot (via `/etc/fstab`)
- Bind-mount `/mnt/falkordb-data` into the FalkorDB container

The EFS DNS name is stable — the new instance after an instance refresh automatically reconnects to the same filesystem.

## CI Templates

The `ci-templates/` directory contains reusable CI snippets for GitHub Actions and GitLab CI.

Both templates:
- Read `asg_name` from `pulumi stack output` (avoids tag-based ASG lookup ambiguity)
- Use AWS SSM Run Command (`AWS-RunShellScript`) to trigger a rolling redeploy on the EC2 instance
- Export `HOME=/root` before running shell commands via SSM (SSM runs as root without a HOME env var)

## Pulumi Outputs

| Output | Description |
|--------|-------------|
| `alb_dns` | ALB DNS name |
| `url` | Public HTTPS URL |
| `asg_name` | Auto Scaling Group name (used by CI to target SSM) |
| `environment` | Stack environment name |
| `falkordb_efs_id` | EFS filesystem ID (only present when `EBS_FALKORDB_STORAGE_PERSIST=true`) |
