# Infrastructure & Cost Report

## AgentCore demo test 1 — AWS Infrastructure

**Version:** 2.0 (V5)  
**Date:** 2026-05-12  
**Region:** `us-east-1` (N. Virginia)  
**Cost Basis:** On-Demand pricing (no reserved instances)  
**Billing Alarm:** $50/month  

---

## What This Report Covers

This document lists every AWS resource created by the setup guide, explains what it does, and estimates how much it costs. All prices are for `us-east-1` and assume the demo runs continuously. If you shut down resources when not in use, costs will be lower.

**Total estimated cost: $55-75/month** while running continuously.

---

## Resource List and Costs

### 1. EC2 — Virtual Server

| Attribute | Value |
|-----------|-------|
| **What it is** | The main computer that runs your application (FastAPI, Temporal, Frontend) |
| **Instance type** | `t3.large` — 2 vCPUs, 8 GB RAM |
| **OS** | Amazon Linux 2023 |
| **Storage** | 30 GB gp3 (general purpose SSD) |
| **Network** | Public subnet with Elastic IP |
| **Why this size** | Sufficient to run 5 Docker containers simultaneously |

| Cost Component | Price | Monthly |
|---------------|-------|---------|
| t3.large (on-demand) | $0.0832/hour | ~$60.74 |
| 30 GB gp3 storage | $0.08/GB/month | ~$2.40 |
| Data transfer out (estimated 10 GB) | $0.09/GB | ~$0.90 |
| **EC2 Total** | | **~$64.04/month** |

**Money-saving tip:** When not using the demo, stop the instance:
```bash
aws ec2 stop-instances --instance-ids YOUR_INSTANCE_ID
```
Stopped instances cost only for storage (~$2.40/month). Start again with:
```bash
aws ec2 start-instances --instance-ids YOUR_INSTANCE_ID
```

---

### 2. RDS — PostgreSQL Database

| Attribute | Value |
|-----------|-------|
| **What it is** | A managed database that stores workflow states, chat messages, and evidence packs |
| **Engine** | PostgreSQL 15.7 |
| **Instance type** | `db.t4g.micro` — 1 vCPU, 1 GB RAM (smallest available) |
| **Storage** | 20 GB gp3 |
| **Deployment** | Single AZ (no standby — acceptable for demo) |
| **Databases** | 2 inside: `temporal` (workflow engine) + `agentcore_demo_test1` (your app) |
| **Backup** | 7 days automatic backup |
| **Public access** | No — only accessible from your EC2 server |

| Cost Component | Price | Monthly |
|---------------|-------|---------|
| db.t4g.micro (on-demand) | $0.012/hour | ~$8.76 |
| 20 GB gp3 storage | $0.115/GB/month | ~$2.30 |
| Backup storage (estimated 5 GB) | $0.095/GB/month | ~$0.48 |
| **RDS Total** | | **~$11.54/month** |

**Money-saving tip:** RDS cannot be "stopped" like EC2 for long periods. For extended non-use, take a final snapshot and delete the instance:
```bash
aws rds delete-db-instance --db-instance-identifier agentcore-demo-test1-db \
  --final-db-snapshot-identifier "agentcore-demo-test1-final-snapshot"
```
Restore later:
```bash
aws rds restore-db-instance-from-db-snapshot \
  --db-instance-identifier agentcore-demo-test1-db \
  --db-snapshot-identifier "agentcore-demo-test1-final-snapshot"
```

---

### 3. S3 — File Storage

| Attribute | Value |
|-----------|-------|
| **What it is** | Three separate buckets for storing files |
| **Bucket 1** | `agentcore-demo-test1-audio-uploads-ACCOUNTID` — Uploaded audio files |
| **Bucket 2** | `agentcore-demo-test1-artifacts-ACCOUNTID` — Generated BRDs, reports |
| **Bucket 3** | `agentcore-demo-test1-claimcheck-ACCOUNTID` — Large temporary payloads |
| **Features** | Versioning enabled, encryption enabled |

| Cost Component | Price | Monthly |
|---------------|-------|---------|
| Storage (estimated 5 GB total) | $0.023/GB/month | ~$0.12 |
| PUT requests (estimated 1,000) | $0.005/1,000 requests | ~$0.01 |
| GET requests (estimated 10,000) | $0.0004/1,000 requests | ~$0.01 |
| **S3 Total** | | **~$0.14/month** |

S3 is extremely cheap for small amounts of data. Even with 50 GB of audio files, it would only be ~$1.15/month.

---

### 4. CloudWatch — Logging and Monitoring

| Attribute | Value |
|-----------|-------|
| **What it is** | Stores application logs and metrics from all services |
| **Log groups** | 4 groups (AgentCore, Temporal, FastAPI, OTel Collector) |
| **Retention** | 7 days (to save money) |

| Cost Component | Price | Monthly |
|---------------|-------|---------|
| Log ingestion (estimated 1 GB/month) | $0.50/GB | ~$0.50 |
| Log storage (7-day retention, minimal) | $0.03/GB/month | ~$0.03 |
| CloudWatch metrics (basic, included) | Free | $0 |
| **CloudWatch Total** | | **~$0.53/month** |

---

### 5. Bedrock — AI Model Usage

| Attribute | Value |
|-----------|-------|
| **What it is** | Pay-per-use AI model inference |
| **Models used** | Claude 3.5 Sonnet, Amazon Transcribe |
| **Pricing model** | Pay per token (input + output) + per minute of audio |

| Cost Component | Price | Per Demo Run |
|---------------|-------|--------------|
| Claude 3.5 Sonnet input tokens (est. 50K) | $0.003/1K tokens | ~$0.15 |
| Claude 3.5 Sonnet output tokens (est. 10K) | $0.015/1K tokens | ~$0.15 |
| Amazon Transcribe (est. 30 min audio) | $0.024/minute | ~$0.72 |
| **Bedrock per run** | | **~$1.02/run** |

| Monthly Scenarios | Cost |
|-------------------|------|
| Light usage (10 runs/month) | ~$10.20 |
| Medium usage (50 runs/month) | ~$51.00 |
| Heavy usage (100 runs/month) | ~$102.00 |

**Note:** Bedrock is the most variable cost. It depends entirely on how often you run the demo.

---

### 6. Data Transfer

| Attribute | Value |
|-----------|-------|
| **What it is** | Cost for data moving between AWS services and out to the internet |
| **Within same region** | Free (EC2 to RDS, EC2 to S3 — all free) |
| **Out to internet** | Charged per GB |

| Cost Component | Price | Monthly |
|---------------|-------|---------|
| Data transfer out (estimated 10 GB) | $0.09/GB | ~$0.90 |
| Data transfer within region | Free | $0 |
| **Data Transfer Total** | | **~$0.90/month** |

---

### 7. Secrets Manager

| Attribute | Value |
|-----------|-------|
| **What it is** | Securely stores your database password |
| **Secrets** | 1 secret (database password) |

| Cost Component | Price | Monthly |
|---------------|-------|---------|
| 1 secret | $0.40/secret/month | ~$0.40 |
| API calls (minimal) | $0.05/10,000 calls | ~$0.01 |
| **Secrets Manager Total** | | **~$0.41/month** |

---

## Total Monthly Cost Summary

### Fixed Costs (Run 24/7)

| Resource | Monthly Cost |
|----------|-------------|
| EC2 (t3.large, 30 GB) | ~$64.04 |
| RDS (db.t4g.micro, 20 GB) | ~$11.54 |
| S3 (5 GB storage) | ~$0.14 |
| CloudWatch (1 GB logs) | ~$0.53 |
| Secrets Manager (1 secret) | ~$0.41 |
| Data transfer | ~$0.90 |
| **Fixed Total** | **~$77.56/month** |

### Variable Costs (Depend on Usage)

| Usage Level | Bedrock Cost | Total Monthly |
|-------------|-------------|---------------|
| Light (10 runs/month) | ~$10.20 | **~$87.76** |
| Medium (50 runs/month) | ~$51.00 | **~$128.56** |
| Heavy (100 runs/month) | ~$102.00 | **~$179.56** |
| **Stopped EC2 (storage only)** | — | **~$16.29** |

### Cost-Saving Scenarios

| Scenario | Monthly Cost | How |
|----------|-------------|-----|
| **Running continuously, light usage** | ~$88 | As configured |
| **Running continuously, medium usage** | ~$129 | As configured |
| **Stop EC2 nights/weekends (12h/day)** | ~$46 | Stop instance 12 hours daily |
| **Stop EC2 when not in use** | ~$16 | Pay only for RDS + storage |
| **Full teardown** | ~$0 | Run `99_teardown.sh` |

---

## Cost Monitoring

### Set Up a Billing Alarm

If you have not already set one up, create a billing alarm:

```bash
# Create a billing alarm at $50
aws cloudwatch put-metric-alarm \
  --alarm-name "agentcore-demo-test1-monthly-budget" \
  --alarm-description "Alert when estimated monthly charges exceed $50" \
  --namespace AWS/Billing \
  --metric-name EstimatedCharges \
  --statistic Maximum \
  --period 86400 \
  --evaluation-periods 1 \
  --threshold 50 \
  --comparison-operator GreaterThanThreshold \
  --alarm-actions "arn:aws:sns:us-east-1:YOUR_ACCOUNT_ID:default"
```

### Check Your Bill Anytime

```bash
# Check current month's estimated charges
aws cloudwatch get-metric-statistics \
  --namespace AWS/Billing \
  --metric-name EstimatedCharges \
  --start-time $(date -u -d '1 day ago' +%Y-%m-%dT%H:%M:%SZ) \
  --end-time $(date -u +%Y-%m-%dT%H:%M:%SZ) \
  --period 86400 \
  --statistics Maximum
```

Or go to: AWS Console → Billing → Bills

---

## Resource Inventory

| # | Resource | AWS Service | Purpose | Monthly Cost |
|---|----------|-------------|---------|-------------|
| 1 | EC2 `t3.large` | EC2 | Main application server | ~$60.74 |
| 2 | 30 GB gp3 volume | EBS | Server storage | ~$2.40 |
| 3 | `db.t4g.micro` | RDS | PostgreSQL database | ~$8.76 |
| 4 | 20 GB gp3 storage | RDS | Database storage | ~$2.30 |
| 5 | `audio-uploads` bucket | S3 | Audio file storage | ~$0.05 |
| 6 | `artifacts` bucket | S3 | Document storage | ~$0.05 |
| 7 | `claimcheck` bucket | S3 | Temporary payload storage | ~$0.04 |
| 8 | CloudWatch log groups | CloudWatch | Application logging | ~$0.53 |
| 9 | Database password secret | Secrets Manager | Secure credential storage | ~$0.41 |
| 10 | `demo-admin` IAM user | IAM | Admin access | Free |
| 11 | `agentcore-demo-test1-ec2-role` | IAM | EC2 permissions | Free |
| 12 | VPC + subnets + security groups | VPC | Network infrastructure | Free |
| 13 | Internet gateway | VPC | Internet access | Free |
| 14 | Bedrock model access | Bedrock | AI model usage | Per-use |

---

## Price References

All prices verified from AWS official pricing pages:
- EC2: https://aws.amazon.com/ec2/pricing/on-demand/
- RDS: https://aws.amazon.com/rds/postgresql/pricing/
- S3: https://aws.amazon.com/s3/pricing/
- CloudWatch: https://aws.amazon.com/cloudwatch/pricing/
- Bedrock: https://aws.amazon.com/bedrock/pricing/
- Data transfer: https://aws.amazon.com/ec2/pricing/on-demand/#Data_Transfer
- Secrets Manager: https://aws.amazon.com/secrets-manager/pricing/

---

*End of Infrastructure & Cost Report*
