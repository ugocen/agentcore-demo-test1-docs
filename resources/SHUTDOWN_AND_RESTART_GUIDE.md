# Shutdown & Restart Guide — agentcore-demo-test1

> Last updated: 2026-05-19  
> Shut down by: cost-minimisation pass while the platform is idle.

---

## What Was Shut Down

| Resource | ID / Name | Type | State after shutdown | Monthly saving |
|---|---|---|---|---|
| EC2 instance | `i-0974a815b81d0c028` | `t3.large` | **stopped** (EBS preserved) | ~$60 |
| RDS instance | `agentcore-demo-test1-db` | `db.t4g.micro` / PostgreSQL 15 | **stopped** (data preserved) | ~$12 |

### What is still running (low / zero cost)

| Resource | Notes |
|---|---|
| 4 × AgentCore Runtime agents | Serverless — no cost when idle (no invocations). Left in `READY` state. |
| EBS volume `vol-09d8488e8b86d3fb1` | 20 GB gp3, attached to EC2 — ~$1.6/mo regardless of EC2 state |
| 6 × S3 buckets | Storage-only cost, pennies per month |
| VPC, Security Groups, Subnets | Zero cost |
| Temporal Cloud namespace | Billed by Temporal separately; no AWS cost |

**Remaining AWS cost while idle: ~$1.6/month (EBS only).**

---

## ⚠️ Important Notes Before Restarting

1. **RDS auto-restart**: AWS automatically restarts a stopped RDS instance after **7 days**. If you intend to leave it off for more than 7 days, you must stop it again each week — or snapshot it and delete the instance entirely.
2. **EC2 public IP**: The instance has no Elastic IP. After restarting it will get a **new public IP**. Access is via **SSM only**, so this does not matter for connectivity.
3. **Temporal workflows**: Any workflows that were in-flight when EC2 stopped will be in a *stuck / no-poller* state in Temporal Cloud. They will resume automatically once the Temporal worker is running again on EC2. No workflow data is lost.
4. **AgentCore DRY_RUN flag**: The EC2 `.env` defaults `AGENTCORE_DRY_RUN=true`. Set it to `false` before starting the worker if you want agents to actually be invoked.

---

## How to Restart

### Step 1 — Start the RDS instance

```bash
aws rds start-db-instance \
  --region us-east-1 \
  --db-instance-identifier agentcore-demo-test1-db
```

Wait until status is `available` (~2–3 minutes):

```bash
aws rds describe-db-instances \
  --region us-east-1 \
  --db-instance-identifier agentcore-demo-test1-db \
  --query "DBInstances[0].DBInstanceStatus"
```

### Step 2 — Start the EC2 instance

```bash
aws ec2 start-instances \
  --region us-east-1 \
  --instance-ids i-0974a815b81d0c028
```

Wait until state is `running`:

```bash
aws ec2 describe-instances \
  --region us-east-1 \
  --instance-ids i-0974a815b81d0c028 \
  --query "Reservations[0].Instances[0].State.Name"
```

### Step 3 — Connect to EC2 via SSM

```bash
aws ssm start-session \
  --region us-east-1 \
  --target i-0974a815b81d0c028
```

Once inside the session, switch to the app user and go to the project directory:

```bash
sudo su - ec2-user
cd ~/agentcore-demo-test1-backend   # adjust if path differs
```

### Step 4 — Verify / update the .env file

Key variables to check before starting:

```bash
cat .env | grep -E "AGENTCORE_DRY_RUN|AGENTCORE_ARN|TEMPORAL|DB_"
```

Set dry-run to false if you want real agent invocations:

```bash
# Edit .env
sed -i 's/AGENTCORE_DRY_RUN=true/AGENTCORE_DRY_RUN=false/' .env
```

The four `AGENTCORE_ARN_*` values must be present and must also be **explicitly listed** in the `environment:` block of the `temporal-worker` service in `docker-compose.yml` — they are not picked up automatically from `.env`.

Current ARNs (as of 2026-05-19):

| Agent | ARN |
|---|---|
| agent_1_transcriber | `arn:aws:bedrock-agentcore:us-east-1:122524101917:runtime/agent_1_transcriber-mjSXuCC153` |
| agent_2_drafter | `arn:aws:bedrock-agentcore:us-east-1:122524101917:runtime/agent_2_drafter-XIh3stAQFo` |
| agent_3_reviewer | `arn:aws:bedrock-agentcore:us-east-1:122524101917:runtime/agent_3_reviewer-BFKEb76k4q` |
| agent_4_goals_collector | `arn:aws:bedrock-agentcore:us-east-1:122524101917:runtime/agent_4_goals_collector-PP2yUS5n0E` |

### Step 5 — Start the Temporal worker (docker compose)

```bash
docker compose pull          # optional: pull latest images
docker compose up -d
docker compose logs -f temporal-worker
```

The worker should log a successful connection to Temporal Cloud within ~10 seconds.

### Step 6 — Verify everything is healthy

```bash
# Worker process running
docker compose ps

# No errors in last 50 log lines
docker compose logs --tail=50 temporal-worker

# RDS reachable from EC2
psql "$DATABASE_URL" -c "SELECT 1;"

# Temporal Cloud — check for active pollers (from local machine)
temporal workflow list --namespace <your-namespace>
```

---

## If You Need to Sync New Code to EC2

Git pull is blocked (private repo, no SSH key on EC2). Use the S3-staging approach:

```bash
# On your local machine — from the backend repo root
tar czf /tmp/backend.tar.gz --exclude='.git' --exclude='node_modules' --exclude='__pycache__' .
aws s3 cp /tmp/backend.tar.gz s3://agentcore-deploy-122524101917/backend.tar.gz

# On EC2 (via SSM)
cd ~
aws s3 cp s3://agentcore-deploy-122524101917/backend.tar.gz /tmp/backend.tar.gz
tar xzf /tmp/backend.tar.gz -C ~/agentcore-demo-test1-backend
```

---

## AgentCore Agents — Re-deployment (if needed)

The 4 agents are still deployed and in `READY` state; no action is required. If you ever delete and need to redeploy:

```bash
# From backend repo root (with correct AWS credentials)
cd agents/agent_1_transcriber
python deploy.py   # or equivalent deploy script

# Repeat for agents 2, 3, 4
```

After redeployment, update the ARNs in `.env` and `docker-compose.yml` on EC2 — they change with each deployment.

---

## Estimated Monthly Cost Summary

| State | EC2 | RDS | EBS | AgentCore | Total |
|---|---|---|---|---|---|
| **Idle (current)** | $0 | $0 | ~$1.6 | $0 | **~$1.6/mo** |
| **Running** | ~$60 | ~$12 | ~$1.6 | $0 + invocation | **~$74/mo** |
