# Developer Onboarding Guide

## AgentCore demo test 1 Project — New Team Member Setup

**Last Updated:** 2026-05-12  
**Region:** `us-east-1`  
**Time to Complete:** 30-45 minutes  
**Who This Is For:** New developers joining the project

---

## What You Will Have After This Guide

- [ ] SSH access to the EC2 server
- [ ] AWS CLI configured with your own credentials
- [ ] Ability to view logs and monitor the system
- [ ] Local development environment ready
- [ ] Understanding of how IMDS works (no AWS keys in code!)

**You will NOT have:**
- Root AWS access
- Ability to delete the database or infrastructure
- AWS credentials in your code or .env files (intentionally!)

---

## Development Environment Rules (MANDATORY)

Before you write any code, understand these rules. They apply to EVERYONE.

### Python: Use `uv` + `.venv` — NEVER Global `pip install`

```bash
# WRONG — never do this
pip install requests

# CORRECT — always do this
cd your-project-directory
uv venv .venv
source .venv/bin/activate  # Linux/Mac
# .venv\Scripts\activate    # Windows
uv pip install requests
```

- The ONLY exception is `RUN pip install` inside a Dockerfile (container-level isolation).
- The EC2 server has a shared `.venv` at `/opt/agentcore-demo-test1/.venv`.
- For your local machine, create your own `.venv` per project.

### Node.js: Use Local `npm install` — NEVER `npm install -g`

```bash
# WRONG — never do this
npm install -g typescript

# CORRECT — always do this
cd frontend/
npm install typescript        # installs into ./node_modules
npx typescript --version       # runs the local binary
```

### Code Comments: English Only

All comments, docstrings, and string literals in code MUST be in English. No exceptions.

```python
# CORRECT
# This function builds the V5 payload with all 8 blocks
def build_payload():
    pass

# WRONG — do not write comments in other languages
# Bu fonksiyon V5 payload olusturur
def build_payload():
    pass
```

---

## Step 1: Get Your Access Credentials (Ask Your Admin)

Your project admin should provide you with:

| Item | What It Looks Like | How You Get It |
|------|-------------------|----------------|
| **AWS Access Key ID** | `AKIA...` (20 chars) | Admin creates IAM user for you |
| **AWS Secret Access Key** | `wJalrXUtnFEMI...` (40 chars) | Shown once by admin, save immediately! |
| **SSH private key (.pem)** | `agentcore-demo-test1-yourname-key.pem` | Admin generates and sends securely |
| **EC2 Public IP** | `3.81.123.45` | Admin shares from `.vpc_env` file |
| **Database password** | (random string) | Admin reads from AWS Secrets Manager |

**If you don't have these, stop here and ask your admin.** They should follow the "Adding a New Developer" section in the AWS Setup Guide.

---

## Step 2: Install Required Tools on Your Computer

### 2.1 AWS CLI

**Mac:**
```bash
curl "https://awscli.amazonaws.com/AWSCLIV2.pkg" -o "AWSCLIV2.pkg"
sudo installer -pkg AWSCLIV2.pkg -target /
```

**Windows:**
```powershell
msiexec.exe /i https://awscli.amazonaws.com/AWSCLIV2.msi
```

**Linux:**
```bash
curl "https://awscli.amazonaws.com/awscli-exe-linux-x86_64.zip" -o "awscliv2.zip"
unzip awscliv2.zip && sudo ./aws/install
```

**Verify:**
```bash
aws --version
# Should show: aws-cli/2.x.x
```

### 2.2 Configure AWS CLI (Your Credentials)

```bash
aws configure
```

Enter these values when prompted:
- **AWS Access Key ID:** Paste the one your admin gave you
- **AWS Secret Access Key:** Paste the one your admin gave you
- **Default region name:** `us-east-1`
- **Default output format:** `json`

**Verify it works:**
```bash
aws sts get-caller-identity
```

You should see something like:
```json
{
    "UserId": "AIDAXXXXXXXXXXXXXXXX",
    "Account": "123456789012",
    "Arn": "arn:aws:iam::123456789012:user/agentcore-demo-test1-your-name"
}
```

**Important:** Make sure it shows your name, NOT "root". If it shows root, something is wrong — contact your admin.

### 2.3 Docker Desktop (for local testing)

Download from [https://www.docker.com/products/docker-desktop](https://www.docker.com/products/docker-desktop)

**Verify:**
```bash
docker --version
docker-compose --version
```

### 2.4 Python 3.11+

```bash
# Mac
brew install python@3.11

# Windows: Download from python.org

# Verify
python3 --version  # Should show 3.11.x or higher
```

### 2.5 Node.js 20+ (per PCD §15.13)

```bash
# Mac
brew install node@20

# Verify
node --version  # Should show v20.x.x or higher
npm --version
```

### 2.6 Git

```bash
git --version  # Should show 2.x.x
```

---

## Step 3: Set Up SSH Access to the EC2 Server

### 3.1 Save Your SSH Key

```bash
# Create .ssh directory if it doesn't exist
mkdir -p ~/.ssh
chmod 700 ~/.ssh

# Move the .pem file your admin gave you
mv ~/Downloads/agentcore-demo-test1-yourname-key.pem ~/.ssh/
chmod 400 ~/.ssh/agentcore-demo-test1-yourname-key.pem
```

**Why chmod 400?** This makes the key readable only by you. SSH will refuse to use it otherwise.

### 3.2 Add SSH Config (Optional but Recommended)

Create `~/.ssh/config`:

```bash
cat >> ~/.ssh/config << 'EOF'
Host agentcore-demo-test1
    HostName 3.81.123.45        # REPLACE with your actual EC2 Public IP
    User ec2-user
    IdentityFile ~/.ssh/agentcore-demo-test1-yourname-key.pem
    StrictHostKeyChecking accept-new
EOF

chmod 600 ~/.ssh/config
```

Now you can simply type:
```bash
ssh agentcore-demo-test1
```

Instead of the full command.

### 3.3 Test SSH Connection

```bash
ssh -i ~/.ssh/agentcore-demo-test1-yourname-key.pem ec2-user@3.81.123.45
```

**First time only:** It will ask "Are you sure you want to continue connecting?" Type `yes`.

You should see the Amazon Linux welcome message:
```
   ,     #_
   ~\_  ####_
  ~~  \_#####\
  ~~     \###|
  ~~       \#/ ___
   ~       V~' '->
    ~         /
    ~~       _/
     ~      _/
__   \    /~      __
  \  |     /     /
   | |    '     /
   \ \     \   /
    ~      ~~~

[ec2-user@ip-10-0-1-123 ~]$
```

**Success!** You are now inside the EC2 server. Type `exit` to return to your computer.

---

## Step 4: Understand How Authentication Works (Important!)

### The Golden Rule

**Your code and .env files contain ZERO AWS credentials.** The EC2 server gets its credentials automatically from AWS through a system called IMDS.

### How It Works

```
Your Laptop (SSH)          EC2 Server                  AWS Cloud
    |                           |                           |
    |  ssh ec2-user@...         |                           |
    |-------------------------->|                           |
    |                           |   boto3.client("s3")      |
    |                           |   (no credentials!)       |
    |                           |-------------------------->|
    |                           |   "Who am I?"             |
    |                           |                           |
    |                           |<--------------------------|
    |                           |   "You are agentcore-demo-test1-ec2- |
    |                           |    role. Here are your    |
    |                           |    temporary credentials" |
    |                           |                           |
    |                           |   Temporary creds valid   |
    |                           |   for ~6 hours, auto-     |
    |                           |   rotated by AWS          |
```

### What This Means for You

**Your Python code looks like this:**
```python
# CORRECT — No credentials needed!
import boto3
s3 = boto3.client("s3", region_name="us-east-1")
s3.list_buckets()
```

**NOT like this:**
```python
# WRONG — Never do this!
import boto3
s3 = boto3.client(
    "s3",
    aws_access_key_id="AKIA...",      # NEVER
    aws_secret_access_key="secret",    # NEVER
    region_name="us-east-1"
)
```

**Your .env file looks like this:**
```bash
# backend/.env — NO AWS credentials!
DATABASE_URL=postgresql://postgres:****@agentcore-demo-test1-db.***.us-east-1.rds.amazonaws.com:5432/agentcore_demo_test1
TEMPORAL_HOST=temporal:7233
S3_AUDIO_BUCKET=agentcore-demo-test1-audio-uploads-123456789
S3_ARTIFACTS_BUCKET=agentcore-demo-test1-artifacts-123456789
# ... but NO AWS_ACCESS_KEY_ID or AWS_SECRET_ACCESS_KEY!
```

### Verify IMDS Is Working

```bash
# SSH into the server
ssh agentcore-demo-test1

# Check which role the server has
curl -s http://169.254.169.254/latest/meta-data/iam/info

# You should see:
# {
#   "Code" : "Success",
#   "LastUpdated" : "2026-05-12T10:00:00Z",
#   "InstanceProfileArn" : "arn:aws:iam::123456789012:instance-profile/agentcore-demo-test1-ec2-profile",
#   "InstanceProfileId" : "AIPA..."
# }
```

---

## Step 5: Clone the Project Repositories (Polyrepo per PCD §5)

The project uses a **4-repo polyrepo** strategy (PCD D4 / REPO_STRATEGY.md). Clone all four into a single parent directory:

```bash
# On your local machine
mkdir -p ~/projects/agentcore-demo-test1
cd ~/projects/agentcore-demo-test1

GITHUB_USER="${GITHUB_USER:-ugocen}"
for REPO in backend frontend infra docs; do
    git clone "git@github.com:${GITHUB_USER}/agentcore-demo-test1-${REPO}.git"
done

# Verify
ls -la
```

You should see four sibling repositories:

```
agentcore-demo-test1/
├── agentcore-demo-test1-backend/        # FastAPI + Temporal + 3 agents (S3 ZIP)
├── agentcore-demo-test1-frontend/       # Next.js 14+ + CopilotKit
├── agentcore-demo-test1-infra/          # AWS shell scripts + ADOT config
└── agentcore-demo-test1-docs/           # All deliverables, prompts, guides
```

**Note:** This is NOT a monorepo — each repo is independent with its own `.git`, its own CI/CD, and its own deployment cadence. See `REPO_STRATEGY.md` for the rationale.

---

## Step 6: View Logs (Your Primary Debugging Tool)

### 6.1 View Container Logs (via SSH)

```bash
# SSH into the server
ssh agentcore-demo-test1

# List running containers
docker ps

# View logs for a specific container
docker logs fastapi-container        # FastAPI backend
docker logs temporal-worker          # Temporal worker
docker logs temporal-server          # Temporal server
docker logs nextjs-frontend          # Frontend

# Follow logs in real-time (like tail -f)
docker logs -f fastapi-container

# View last 100 lines
docker logs --tail 100 fastapi-container
```

### 6.2 View CloudWatch Logs (via AWS CLI from your laptop)

```bash
# On your local machine

# List log groups
aws logs describe-log-groups \
  --log-group-name-prefix "/aws/" \
  --query 'logGroups[*].logGroupName'

# View recent logs from FastAPI
aws logs tail "/agentcore-demo-test1/fastapi" --follow

# View AgentCore logs
aws logs tail "/aws/bedrock-agentcore/runtimes" --log-stream-name-prefix agent --follow

# View Temporal worker logs
aws logs tail "/aws/temporal/worker" --follow
```

### 6.3 View CloudWatch Logs (via Browser)

1. Go to [AWS Console](https://console.aws.amazon.com)
2. Search for **"CloudWatch"** and click it
3. In the left sidebar, click **"Logs"** > **"Log groups"**
4. Click on any log group to see its logs

### 6.4 View Temporal Web UI

1. Open your browser
2. Go to: `http://YOUR_EC2_IP:8081`
3. You will see the Temporal dashboard with running workflows

---

## Step 7: Access the Database (Read-Only)

The database is NOT publicly accessible (security). You access it through the EC2 server.

```bash
# SSH into the server
ssh agentcore-demo-test1

# Install PostgreSQL client (if not already installed)
sudo dnf install -y postgresql15

# Connect to the database (ask admin for the password)
psql postgresql://postgres:DATABASE_PASSWORD@agentcore-demo-test1-db.XXXXXXX.us-east-1.rds.amazonaws.com:5432/agentcore_demo_test1

# Common queries you might run:
\dt                    # List tables
SELECT * FROM workflow_states LIMIT 10;     # View recent workflows
SELECT * FROM chat_messages LIMIT 10;       # View chat messages
SELECT * FROM evidence_packs;               # View evidence packs
\q                     # Quit
```

**Alternative: Port forwarding (run queries from your local machine)**

```bash
# On your local machine — this creates a tunnel
ssh -N -L 5433:agentcore-demo-test1-db.XXXXXXX.us-east-1.rds.amazonaws.com:5432 agentcore-demo-test1

# Leave this running. In another terminal:
psql postgresql://postgres:DATABASE_PASSWORD@localhost:5433/agentcore_demo_test1
```

---

## Step 8: Your Development Workflow

### 8.1 Daily Workflow

```bash
# 1. Pull latest code
git pull origin main

# 2. Make your changes locally
# Edit files in your IDE (VS Code, PyCharm, etc.)

# 3. Test locally (if you have Docker running)
cd backend && docker build -t agentcore-demo-test1-backend .

# 4. Push your branch
git checkout -b feature/your-feature-name
git add .
git commit -m "Add: your feature description"
git push origin feature/your-feature-name

# 5. Create a Pull Request on GitHub/GitLab
# Admin reviews and deploys
```

### 8.2 Deploying to the Server (Admin Required)

As a developer with read-only AWS access, you **cannot** deploy directly. Here is the flow:

1. You push your code to a branch
2. You create a Pull Request
3. Admin reviews and merges
4. Admin SSHs into the server and runs:
   ```bash
   cd /opt/agentcore-demo-test1
   git pull origin main
   docker-compose down
   docker-compose up -d --build
   ```

**If you have deploy access** (admin granted additional permissions):
```bash
# SSH into the server
ssh agentcore-demo-test1
cd /opt/agentcore-demo-test1
git pull origin main
docker-compose down
docker-compose up -d --build

# Verify all containers are running
docker ps
```

---

## Step 9: Common Tasks Reference

### Check if the Server Is Running

```bash
# SSH and check containers
ssh agentcore-demo-test1 "docker ps --format 'table {{.Names}}\t{{.Status}}'"
```

### Restart a Service

```bash
# SSH and restart
ssh agentcore-demo-test1 "docker restart fastapi-container"
```

### Check Disk Space

```bash
ssh agentcore-demo-test1 "df -h"
```

### Check Memory Usage

```bash
ssh agentcore-demo-test1 "free -h"
```

### View Environment Variables on the Server

```bash
ssh agentcore-demo-test1 "cat /opt/agentcore-demo-test1/.env"
```

### Check Bedrock Model Access

```bash
# From your laptop
aws bedrock list-foundation-models \
  --query 'modelSummaries[?modelId==`anthropic.claude-sonnet-4-6`].modelId'
```

---

## Step 10: What to Do When Things Go Wrong

### "Permission Denied" when running AWS commands

**Cause:** Your IAM user doesn't have permission for that action.  
**Fix:** Ask your admin to attach the required policy. Do NOT use the root account.

### "Connection Refused" when SSHing

**Causes:**
- Your IP changed (ISPs rotate IPs). Ask admin to update the security group.
- EC2 is stopped. Ask admin to start it.
- Wrong .pem file. Make sure you're using YOUR key, not someone else's.

### "NoCredentialsError" in Python code

**Cause:** You're running code on your laptop, not on the EC2 server.  
**Fix:** IMDS only works on EC2. For local development, ask admin for temporary credentials OR use the AWS CLI profile you configured in Step 2.

### Containers keep crashing

```bash
# SSH and check
ssh agentcore-demo-test1
docker ps -a              # See which ones exited
docker logs container-name # See why it crashed
docker-compose logs       # See all logs
```

### Database connection fails

**Causes:**
- RDS security group doesn't allow your EC2. Ask admin.
- Wrong password. Ask admin to check Secrets Manager.
- RDS is still starting. Wait 2-3 minutes and retry.

---

## Security Rules You Must Follow

| Rule | Why | Violation Consequence |
|------|-----|----------------------|
| **Never share your .pem file** | Anyone with the key can access the server | Immediate access revocation |
| **Never put AWS keys in code** | IMDS handles this automatically | Code leak = AWS breach |
| **Never use the root account** | Root has unlimited power | Account compromise |
| **Never commit .env files** | They contain database passwords | Credential leak |
| **Always use your IAM user** | Auditable, revocable | Security incident |
| **Report lost credentials immediately** | Admin can revoke access quickly | Minimize damage |

---

## Quick Reference Card

| I Want To... | Command |
|-------------|---------|
| SSH to server | `ssh agentcore-demo-test1` |
| View FastAPI logs | `docker logs -f fastapi-container` |
| View all containers | `docker ps` |
| Restart everything | `docker-compose down && docker-compose up -d` |
| View CloudWatch logs | `aws logs tail "/agentcore-demo-test1/fastapi" --follow` |
| Check database | `psql $DATABASE_URL` (on EC2) |
| Check Bedrock models | `aws bedrock list-foundation-models` |
| Check my AWS identity | `aws sts get-caller-identity` |
| Check disk space | `df -h` (on EC2) |
| Check memory | `free -h` (on EC2) |

---

## Who to Contact

| Issue | Contact | How |
|-------|---------|-----|
| Lost SSH key | Admin | Slack/email |
| Need AWS permissions | Admin | Create ticket |
| Server is down | Admin | Pager/Slack |
| Billing question | Admin | Email |
| Code questions | Tech Lead | Slack channel |
| AWS Console login issues | Admin | Slack/email |

---

*Welcome to the team! If anything in this guide is unclear, ask your admin — this document is updated regularly based on feedback.*
