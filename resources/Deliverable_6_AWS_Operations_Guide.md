# AWS Setup Guide for AgentCore demo test 1

## A Step-by-Step Manual for First-Time AWS Users

**Last Updated:** 2026-05-12  
**Region:** `us-east-1` (N. Virginia)  
**Starting Point:** Brand new AWS account, root user only, billing alarm already set

---

## What This Guide Does

This guide takes you from a brand-new AWS account to a fully running cloud environment for the AgentCore demo test 1 demo. It assumes you have never used AWS before. Every step is explained in plain English. No prior cloud experience required.

**What we will build (in order):**
1. A secure admin user (so you never use root again)
2. A virtual network (VPC) for your resources
3. A database (RDS PostgreSQL)
4. File storage (S3 buckets)
5. A server (EC2) to run your application
6. Logging setup (CloudWatch)
7. Model access for AI (Bedrock)
8. Agent deployment setup (AgentCore)

**Estimated time:** 2-3 hours (mostly waiting for AWS to create things)
**Estimated cost:** $50-80/month while running (see Cost Report)

---

## Before You Start

### What You Need
- An AWS account (already created)
- A billing alarm set to $50 (already done)
- Your computer with a terminal (Mac Terminal, Windows PowerShell, or Linux)
- AWS CLI installed (see Step 0)

### Important Concepts (Explained Simply)

| Term | What It Means | Think Of It As... |
|------|--------------|-------------------|
| **Region** | A physical AWS data center location | Choosing a city for your office |
| **VPC** | Your private network in the cloud | The walls and doors of your building |
| **EC2** | A virtual server (computer) | A computer that lives in AWS's building |
| **RDS** | A managed database | A filing cabinet that AWS maintains for you |
| **S3** | File storage | A giant hard drive in the cloud |
| **IAM** | User permissions and security | Keycards and access rules for your building |
| **CloudWatch** | Monitoring and logging | Security cameras and logbooks |
| **Bedrock** | AWS's AI model service | A library of AI brains you can rent |
| **AgentCore** | AWS's agent hosting service | A factory that runs your AI workers |

---

## Step 0: Install the AWS Command Line Tool

The AWS CLI is a program that lets you control AWS from your computer's terminal. You will use it throughout this guide.

### For Mac Users
```bash
# Open Terminal and run:
curl "https://awscli.amazonaws.com/AWSCLIV2.pkg" -o "AWSCLIV2.pkg"
sudo installer -pkg AWSCLIV2.pkg -target /
```

### For Windows Users
```powershell
# Open PowerShell and run:
msiexec.exe /i https://awscli.amazonaws.com/AWSCLIV2.msi
```

### For Linux Users
```bash
curl "https://awscli.amazonaws.com/awscli-exe-linux-x86_64.zip" -o "awscliv2.zip"
unzip awscliv2.zip
sudo ./aws/install
```

### Verify Installation
```bash
aws --version
```
You should see something like: `aws-cli/2.15.0`

### Login with Your Root Account
```bash
aws configure
```
It will ask for 4 things:
1. **AWS Access Key ID**: Go to AWS Console → Click your name (top right) → "Security credentials" → "Create access key" → Copy the Access Key ID
2. **AWS Secret Access Key**: Copy this from the same page (shown only once — save it!)
3. **Default region name**: Type `us-east-1`
4. **Default output format**: Type `json`

**Verify you are logged in:**
```bash
aws sts get-caller-identity
```
You should see your account number and "root" as the user.

---

## Step 1: Create an Admin User (Stop Using Root!)

**Why:** Using the root account for daily work is dangerous. If someone steals your root password, they own everything. An admin user has full permissions but is separate and safer.

### 1.1 Create the Admin User

```bash
# Run this in your terminal

# First, create the user
aws iam create-user --user-name demo-admin

# Attach the AdministratorAccess policy (full permissions for this demo)
aws iam attach-user-policy \
  --user-name demo-admin \
  --policy-arn arn:aws:iam::aws:policy/AdministratorAccess

# Create an access key for this user (save the output!)
aws iam create-access-key --user-name demo-admin
```

**IMPORTANT:** The output shows two values — `AccessKeyId` and `SecretAccessKey`. The Secret Access Key is shown ONLY NOW. Copy both to a safe place (a password manager or text file).

### 1.2 Switch to the Admin User

```bash
# Update your AWS CLI to use the admin user instead of root
aws configure set aws_access_key_id YOUR_ACCESS_KEY_ID_HERE
aws configure set aws_secret_access_key YOUR_SECRET_ACCESS_KEY_HERE
aws configure set region us-east-1
aws configure set output json
```

### 1.3 Verify
```bash
aws sts get-caller-identity
```
You should now see `"Arn": "...user/demo-admin"` instead of root.

**From now on, always use `demo-admin`. Never use the root account.**

---

## Step 2: Create a Virtual Network (VPC)

**Why:** Your EC2 server and database need a private network to communicate securely. A VPC is like creating your own private internet inside AWS.

**What this does:**
- Creates a private network (10.0.0.0/16) — this means 65,000 possible IP addresses
- Creates one public subnet (your server goes here — it can reach the internet)
- Creates one private subnet (your database goes here — hidden from the internet)
- Creates an internet gateway (the door to the internet for your public subnet)

### Run the VPC Script

```bash
# Create a folder for your scripts
mkdir -p ~/agentcore-demo-test1/infra/scripts
cd ~/agentcore-demo-test1/infra/scripts

# Save this as 00_create_vpc.sh and run it
```

**Script:** `00_create_vpc.sh`
```bash
#!/bin/bash
set -e  # Stop if any command fails

REGION="us-east-1"
PROJECT="agentcore-demo-test1"
ENV="demo"

echo "=== Step 2: Creating your virtual network ==="

# --- Create VPC (your private network) ---
echo "Creating VPC..."
VPC_ID=$(aws ec2 create-vpc \
  --cidr-block 10.0.0.0/16 \
  --tag-specifications "ResourceType=vpc,Tags=[{Key=Name,Value=${PROJECT}-vpc}]" \
  --query 'Vpc.VpcId' --output text)
echo "VPC created: ${VPC_ID}"

# --- Create Public Subnet (for your server) ---
echo "Creating public subnet..."
PUB_SUBNET=$(aws ec2 create-subnet \
  --vpc-id "${VPC_ID}" \
  --cidr-block 10.0.1.0/24 \
  --availability-zone "us-east-1a" \
  --tag-specifications "ResourceType=subnet,Tags=[{Key=Name,Value=${PROJECT}-public}]" \
  --query 'Subnet.SubnetId' --output text)
echo "Public subnet: ${PUB_SUBNET}"

# --- Create Private Subnet (for your database) ---
echo "Creating private subnet..."
PRIV_SUBNET=$(aws ec2 create-subnet \
  --vpc-id "${VPC_ID}" \
  --cidr-block 10.0.2.0/24 \
  --availability-zone "us-east-1a" \
  --tag-specifications "ResourceType=subnet,Tags=[{Key=Name,Value=${PROJECT}-private}]" \
  --query 'Subnet.SubnetId' --output text)
echo "Private subnet: ${PRIV_SUBNET}"

# --- Create Internet Gateway (door to the internet) ---
echo "Creating internet gateway..."
IGW=$(aws ec2 create-internet-gateway \
  --tag-specifications "ResourceType=internet-gateway,Tags=[{Key=Name,Value=${PROJECT}-igw}]" \
  --query 'InternetGateway.InternetGatewayId' --output text)
aws ec2 attach-internet-gateway --internet-gateway-id "${IGW}" --vpc-id "${VPC_ID}"
echo "Internet gateway: ${IGW}"

# --- Create Route Table (traffic rules for public subnet) ---
echo "Creating route table..."
RT=$(aws ec2 create-route-table \
  --vpc-id "${VPC_ID}" \
  --tag-specifications "ResourceType=route-table,Tags=[{Key=Name,Value=${PROJECT}-public-rt}]" \
  --query 'RouteTable.RouteTableId' --output text)
aws ec2 create-route --route-table-id "${RT}" --destination-cidr-block 0.0.0.0/0 --gateway-id "${IGW}"
aws ec2 associate-route-table --route-table-id "${RT}" --subnet-id "${PUB_SUBNET}"
echo "Route table: ${RT}"

# --- Enable public IP auto-assignment ---
aws ec2 modify-subnet-attribute --subnet-id "${PUB_SUBNET}" --map-public-ip-on-launch

# --- Save all IDs to a file for later steps ---
cat > .vpc_env <<EOF
VPC_ID=${VPC_ID}
PUB_SUBNET=${PUB_SUBNET}
PRIV_SUBNET=${PRIV_SUBNET}
IGW=${IGW}
REGION=${REGION}
EOF
echo "=== Network created! IDs saved to .vpc_env ==="
```

**Run it:**
```bash
chmod +x 00_create_vpc.sh
./00_create_vpc.sh
```

This takes about 1-2 minutes. You will see IDs printed — these are saved to `.vpc_env` automatically.

---

## Step 3: Create Security Groups (Firewall Rules)

**Why:** Security groups are like firewall rules. They control WHO can access your server and database.

**What we create:**
1. **EC2 Security Group**: Allows SSH (from YOUR computer only), HTTP (port 8000 for FastAPI), and port 3000 (for frontend)
2. **RDS Security Group**: Allows PostgreSQL access ONLY from your EC2 server

### Get Your Computer's IP Address
```bash
MY_IP=$(curl -s https://checkip.amazonaws.com)/32
echo "Your IP: ${MY_IP}"
```
Write this down — you will need it.

### Run the Security Group Script

**Script:** `01_create_security_groups.sh`
```bash
#!/bin/bash
set -e

REGION="us-east-1"
PROJECT="agentcore-demo-test1"

# Load VPC ID from previous step
source .vpc_env
MY_IP=$(curl -s https://checkip.amazonaws.com)/32

echo "=== Step 3: Creating firewall rules ==="

# --- EC2 Security Group (for your server) ---
echo "Creating EC2 security group..."
EC2_SG=$(aws ec2 create-security-group \
  --group-name "${PROJECT}-ec2-sg" \
  --description "Security group for AgentCore demo test 1 EC2 server" \
  --vpc-id "${VPC_ID}" \
  --query 'GroupId' --output text)

# Allow SSH from YOUR computer only (port 22)
aws ec2 authorize-security-group-ingress \
  --group-id "${EC2_SG}" \
  --protocol tcp --port 22 --cidr "${MY_IP}"

# Allow FastAPI backend (port 8000)
aws ec2 authorize-security-group-ingress \
  --group-id "${EC2_SG}" \
  --protocol tcp --port 8000 --cidr 0.0.0.0/0

# Allow Temporal Web UI (port 8081)
aws ec2 authorize-security-group-ingress \
  --group-id "${EC2_SG}" \
  --protocol tcp --port 8081 --cidr "${MY_IP}"

# Allow Next.js frontend (port 3000)
aws ec2 authorize-security-group-ingress \
  --group-id "${EC2_SG}" \
  --protocol tcp --port 3000 --cidr "${MY_IP}"

# Allow all outbound traffic
aws ec2 authorize-security-group-egress \
  --group-id "${EC2_SG}" \
  --protocol all --port all --cidr 0.0.0.0/0

echo "EC2 security group: ${EC2_SG}"

# --- RDS Security Group (for your database) ---
echo "Creating RDS security group..."
RDS_SG=$(aws ec2 create-security-group \
  --group-name "${PROJECT}-rds-sg" \
  --description "Security group for AgentCore demo test 1 database" \
  --vpc-id "${VPC_ID}" \
  --query 'GroupId' --output text)

# Allow PostgreSQL ONLY from EC2 security group
aws ec2 authorize-security-group-ingress \
  --group-id "${RDS_SG}" \
  --protocol tcp --port 5432 \
  --source-group "${EC2_SG}"

echo "RDS security group: ${RDS_SG}"

# --- Save IDs ---
cat >> .vpc_env <<EOF
EC2_SG=${EC2_SG}
RDS_SG=${RDS_SG}
MY_IP=${MY_IP}
EOF
echo "=== Firewall rules created! ==="
```

**Run it:**
```bash
chmod +x 01_create_security_groups.sh
./01_create_security_groups.sh
```

---

## Step 4: Create S3 Buckets (File Storage)

**Why:** You need places to store: uploaded audio files, generated BRD documents, and large temporary payloads.

**Script:** `02_create_s3_buckets.sh`
```bash
#!/bin/bash
set -e

REGION="us-east-1"
PROJECT="agentcore-demo-test1"
ACCOUNT_ID=$(aws sts get-caller-identity --query Account --output text)

echo "=== Step 4: Creating file storage buckets ==="

# --- Audio uploads bucket ---
echo "Creating audio-uploads bucket..."
aws s3api create-bucket \
  --bucket "${PROJECT}-audio-uploads-${ACCOUNT_ID}" \
  --region "${REGION}" \
  --create-bucket-configuration LocationConstraint="${REGION}"

# --- Artifacts bucket (BRDs, reports) ---
echo "Creating artifacts bucket..."
aws s3api create-bucket \
  --bucket "${PROJECT}-artifacts-${ACCOUNT_ID}" \
  --region "${REGION}" \
  --create-bucket-configuration LocationConstraint="${REGION}"

# --- Claim-check bucket (large temporary payloads) ---
echo "Creating claim-check bucket..."
aws s3api create-bucket \
  --bucket "${PROJECT}-claimcheck-${ACCOUNT_ID}" \
  --region "${REGION}" \
  --create-bucket-configuration LocationConstraint="${REGION}"

# Enable versioning on all buckets (safety feature)
for BUCKET in "${PROJECT}-audio-uploads-${ACCOUNT_ID}" \
              "${PROJECT}-artifacts-${ACCOUNT_ID}" \
              "${PROJECT}-claimcheck-${ACCOUNT_ID}"; do
  echo "Enabling versioning on ${BUCKET}..."
  aws s3api put-bucket-versioning \
    --bucket "${BUCKET}" \
    --versioning-configuration Status=Enabled
done

echo "=== Buckets created! ==="
```

**Run it:**
```bash
chmod +x 02_create_s3_buckets.sh
./02_create_s3_buckets.sh
```

---

## Step 4b: Create S3 Files File System (for Agent Persistent Storage)

**Why:** Your agents need to read and write large claim-check payloads. Instead of using boto3 S3 API inside each agent, we use **AgentCore Persistent Filesystems** — S3 Files mounts your claim-check bucket directly into the agent's microVM as a local filesystem. Agents use ordinary file operations (`open`, `read`, `write`) instead of network S3 calls.

**What this does:**
- Creates an S3 Files file system (logical NFS gateway for your S3 bucket)
- Creates an access point in your VPC (mount target)
- Updates your security group to allow NFS traffic (port 2049)
- Updates your IAM role with S3 Files permissions

### 4b.1 Create the S3 Files File System

```bash
#!/bin/bash
# 02b_create_s3_files.sh

set -e
REGION="us-east-1"
PROJECT="agentcore-demo-test1"
ACCOUNT_ID=$(aws sts get-caller-identity --query Account --output text)

echo "=== Step 4b: Creating S3 Files File System ==="

# Create the S3 Files file system (NFS gateway for the claim-check bucket)
FS_ARN=$(aws s3files create-file-system \
  --s3-bucket-arn "arn:aws:s3:::${PROJECT}-claimcheck-${ACCOUNT_ID}" \
  --region "${REGION}" \
  --tags Key=Project,Value="${PROJECT}" \
  --query 'fileSystemArn' --output text)

echo "S3 Files File System: ${FS_ARN}"
echo "Waiting for it to become available..."

# Poll until available (may take 2-3 minutes)
while true; do
  STATE=$(aws s3files describe-file-system \
    --file-system-id "${FS_ARN}" \
    --query 'LifeCycleState' --output text 2>/dev/null || echo "CREATING")
  echo "  Status: ${STATE}"
  if [ "${STATE}" = "AVAILABLE" ]; then
    break
  fi
  sleep 15
done

# Save
echo "FS_ARN=${FS_ARN}" >> .vpc_env
echo "=== S3 Files File System ready ==="
```

**Run it:**
```bash
chmod +x 02b_create_s3_files.sh
./02b_create_s3_files.sh
```

### 4b.2 Create an Access Point

```bash
#!/bin/bash
# 02c_create_access_point.sh

set -e
source .vpc_env

echo "=== Step 4b.2: Creating S3 Files Access Point ==="

AP_ARN=$(aws s3files create-access-point \
  --file-system-id "${FS_ARN}" \
  --subnet-id "${PUB_SUBNET}" \
  --vpc-id "${VPC_ID}" \
  --region "${REGION}" \
  --query 'AccessPointArn' --output text)

echo "Access Point: ${AP_ARN}"
echo "AP_ARN=${AP_ARN}" >> .vpc_env
echo "=== Access Point created ==="
```

**Run it:**
```bash
chmod +x 02c_create_access_point.sh
./02c_create_access_point.sh
```

### 4b.3 Update Security Group for NFS (Port 2049)

```bash
#!/bin/bash
# 02d_update_sg_s3files.sh

set -e
source .vpc_env

echo "=== Step 4b.3: Updating Security Group for S3 Files ==="

# Get mount target ID
MT_ID=$(aws s3files describe-mount-targets \
  --access-point-id "${AP_ARN}" \
  --query 'MountTargets[0].MountTargetId' --output text)

# Get mount target's security group
MT_SG=$(aws s3files describe-mount-target-security-groups \
  --mount-target-id "${MT_ID}" \
  --query 'SecurityGroups[0]' --output text)

# Allow EC2 to reach S3 Files on port 2049
aws ec2 authorize-security-group-egress \
  --group-id "${EC2_SG}" \
  --protocol tcp \
  --port 2049 \
  --destination-group "${MT_SG}" \
  --description "S3 Files NFS access for AgentCore"

echo "=== Security Group updated (port 2049) ==="
```

**Run it:**
```bash
chmod +x 02d_update_sg_s3files.sh
./02d_update_sg_s3files.sh
```

### 4b.4 Add S3 Files Permissions to IAM Role

```bash
#!/bin/bash
# 02e_update_iam_s3files.sh

set -e
source .vpc_env

echo "=== Step 4b.4: Adding S3 Files permissions to IAM Role ==="

# Get the current policy
current_policy=$(aws iam get-policy-version \
  --policy-arn "${POLICY_ARN}" \
  --version-id v1 \
  --query 'PolicyVersion.Document' --output text)

# Build S3 Files permission statement
cat > /tmp/s3files-statement.json <<EOF
{
  "Sid": "S3FilesAgentAccess",
  "Effect": "Allow",
  "Action": [
    "s3files:ClientMount",
    "s3files:ClientWrite",
    "s3files:GetAccessPoint"
  ],
  "Resource": "${FS_ARN}",
  "Condition": {
    "StringEquals": {
      "s3files:AccessPointArn": "${AP_ARN}"
    }
  }
}
EOF

echo "S3 Files permissions statement:"
cat /tmp/s3files-statement.json

echo ""
echo "=== IMPORTANT: Add the above statement to your IAM policy ==="
echo "Policy ARN: ${POLICY_ARN}"
echo "You can do this via AWS Console > IAM > Policies > agentcore-demo-test1-ec2-policy > Edit"
echo "Or ask your admin to run the update script."
```

**Run it:**
```bash
chmod +x 02e_update_iam_s3files.sh
./02e_update_iam_s3files.sh
```

---

## Step 5: Create the Database (RDS PostgreSQL)

**Why:** You need a database to store workflow states, chat messages, evidence packs, and HITL interactions.

**What we create:**
- A small PostgreSQL database (`db.t4g.micro` — cheapest option, suitable for demo)
- Two databases inside it: `temporal` (for Temporal workflow engine) and `agentcore_demo_test1` (for your app)
- Master password saved in AWS Secrets Manager (secure)

### 5.1 Save the Database Password Securely

```bash
# Generate a random password
DB_PASSWORD=$(openssl rand -base64 32)
echo "Database password: ${DB_PASSWORD}"
# Write this down! You will need it once during setup.

# Save it in AWS Secrets Manager
aws secretsmanager create-secret \
  --name "agentcore-demo-test1/db-password" \
  --description "Password for AgentCore demo test 1 demo database" \
  --secret-string "${DB_PASSWORD}"
```

### 5.2 Create the Database

**Script:** `03_create_rds.sh`
```bash
#!/bin/bash
set -e

REGION="us-east-1"
PROJECT="agentcore-demo-test1"

source .vpc_env

echo "=== Step 5: Creating PostgreSQL database ==="

# Get password from Secrets Manager
DB_PASSWORD=$(aws secretsmanager get-secret-value \
  --secret-id "agentcore-demo-test1/db-password" \
  --query SecretString --output text)

# --- Create DB Subnet Group (tells RDS which subnets to use) ---
echo "Creating DB subnet group..."
aws rds create-db-subnet-group \
  --db-subnet-group-name "${PROJECT}-db-subnet-group" \
  --db-subnet-group-description "Subnet group for AgentCore demo test 1 database" \
  --subnet-ids "[\"${PUB_SUBNET}\",\"${PRIV_SUBNET}\"]"

# --- Create the database ---
echo "Creating PostgreSQL database (this takes 10-15 minutes)..."
aws rds create-db-instance \
  --db-instance-identifier "${PROJECT}-db" \
  --db-instance-class db.t4g.micro \
  --engine postgres \
  --engine-version 15.7 \
  --allocated-storage 20 \
  --storage-type gp3 \
  --db-name "postgres" \
  --master-username "postgres" \
  --master-user-password "${DB_PASSWORD}" \
  --vpc-security-group-ids "${RDS_SG}" \
  --db-subnet-group-name "${PROJECT}-db-subnet-group" \
  --publicly-accessible false \
  --storage-encrypted \
  --backup-retention-period 7 \
  --deletion-protection false \
  --no-multi-az \
  --enable-performance-insights \
  --performance-insights-retention-period 7

echo "=== Database is being created. This takes 10-15 minutes. ==="
echo "Check status: aws rds describe-db-instances --db-instance-identifier ${PROJECT}-db"
```

**Run it:**
```bash
chmod +x 03_create_rds.sh
./03_create_rds.sh
```

**Wait for the database to be ready:**
```bash
# Run this until you see "available"
aws rds describe-db-instances \
  --db-instance-identifier agentcore-demo-test1-db \
  --query 'DBInstances[0].DBInstanceStatus'
```

### 5.3 Create the Application Database

Once the RDS instance is "available", create the second database:

```bash
# Get the database endpoint (address)
DB_ENDPOINT=$(aws rds describe-db-instances \
  --db-instance-identifier agentcore-demo-test1-db \
  --query 'DBInstances[0].Endpoint.Address' --output text)
echo "Database endpoint: ${DB_ENDPOINT}"

# Save it
cat >> .vpc_env <<EOF
DB_ENDPOINT=${DB_ENDPOINT}
DB_PASSWORD=${DB_PASSWORD}
EOF
```

---

## Step 6: Create an IAM Role for Your EC2 Server (with Least-Privilege Policies)

**Why:** Your EC2 server needs permission to talk to other AWS services (S3, RDS, Bedrock, CloudWatch). Instead of putting secret keys on the server, you attach an **IAM Role** — AWS handles the authentication automatically through **IMDS** (Instance Metadata Service). Your code never touches AWS credentials.

**How it works:** When your EC2 server (or a Docker container running on it) calls an AWS service, boto3 automatically fetches temporary credentials from IMDS. These credentials are rotated automatically by AWS. You write zero credential code.

**What we create:** A custom IAM role with specific, minimal permissions for each service your app actually uses. We do NOT use blanket AdministratorAccess.

### The Permission Map

| AWS Service | What Our App Does | Exact Permissions Needed |
|-------------|------------------|-------------------------|
| **S3** | Read/write audio files, BRDs, claim-checks | `s3:GetObject`, `s3:PutObject`, `s3:DeleteObject` on 3 specific buckets |
| **Bedrock** | Call Claude 3.5 Sonnet, AgentCore runtime | `bedrock:InvokeModel`, `bedrock-agentcore:InvokeAgentRuntime` |
| **CloudWatch Logs** | Write application logs | `logs:CreateLogGroup`, `logs:CreateLogStream`, `logs:PutLogEvents` on specific log groups |
| **CloudWatch Metrics** | Write OTel metrics (via ADOT) | `cloudwatch:PutMetricData` |
| **RDS** | Connect to PostgreSQL | `rds:DescribeDBInstances` (for endpoint discovery) |
| **Secrets Manager** | Read database password | `secretsmanager:GetSecretValue` on one specific secret |

### 6.1 Create the Custom Policy

**Script:** `04_create_iam_policy.sh`
```bash
#!/bin/bash
set -e

PROJECT="agentcore-demo-test1"
ACCOUNT_ID=$(aws sts get-caller-identity --query Account --output text)
REGION="us-east-1"

echo "=== Step 6.1: Creating custom least-privilege policy ==="

# Build the policy JSON with your actual resource ARNs
cat > /tmp/agentcore-demo-test1-policy.json <<EOF
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Sid": "S3Access",
      "Effect": "Allow",
      "Action": [
        "s3:GetObject",
        "s3:PutObject",
        "s3:DeleteObject",
        "s3:ListBucket"
      ],
      "Resource": [
        "arn:aws:s3:::${PROJECT}-audio-uploads-${ACCOUNT_ID}",
        "arn:aws:s3:::${PROJECT}-audio-uploads-${ACCOUNT_ID}/*",
        "arn:aws:s3:::${PROJECT}-artifacts-${ACCOUNT_ID}",
        "arn:aws:s3:::${PROJECT}-artifacts-${ACCOUNT_ID}/*",
        "arn:aws:s3:::${PROJECT}-claimcheck-${ACCOUNT_ID}",
        "arn:aws:s3:::${PROJECT}-claimcheck-${ACCOUNT_ID}/*"
      ]
    },
    {
      "Sid": "BedrockAccess",
      "Effect": "Allow",
      "Action": [
        "bedrock:InvokeModel",
        "bedrock:InvokeModelWithResponseStream",
        "bedrock:GetFoundationModel",
        "bedrock:ListFoundationModels",
        "bedrock-agentcore:InvokeAgentRuntime",
        "bedrock-agentcore:InvokeInlineAgentRuntime",
        "bedrock-agentcore:StartTaskInvocation",
        "bedrock-agentcore:GetTaskInvocation",
        "bedrock-agentcore:ListTaskInvocations"
      ],
      "Resource": "*"
    },
    {
      "Sid": "CloudWatchLogs",
      "Effect": "Allow",
      "Action": [
        "logs:CreateLogGroup",
        "logs:CreateLogStream",
        "logs:PutLogEvents",
        "logs:DescribeLogGroups",
        "logs:DescribeLogStreams"
      ],
      "Resource": [
        "arn:aws:logs:${REGION}:${ACCOUNT_ID}:log-group:/agentcore-demo-test1/agent-logs:*",
        "arn:aws:logs:${REGION}:${ACCOUNT_ID}:log-group:/agentcore-demo-test1/adot-collector:*",
        "arn:aws:logs:${REGION}:${ACCOUNT_ID}:log-group:/agentcore-demo-test1/fastapi:*",
        "arn:aws:logs:${REGION}:${ACCOUNT_ID}:log-group:/agentcore-demo-test1/temporal-worker:*",
        "arn:aws:logs:${REGION}:${ACCOUNT_ID}:log-group:/aws/bedrock-agentcore/runtimes/*:*"
      ]
    },
    {
      "Sid": "CloudWatchMetrics",
      "Effect": "Allow",
      "Action": [
        "cloudwatch:PutMetricData",
        "cloudwatch:PutMetricStream",
        "cloudwatch:GetMetricData"
      ],
      "Resource": "*",
      "Condition": {
        "StringEquals": {
          "cloudwatch:namespace": "DemoSDLC/Agent"
        }
      }
    },
    {
      "Sid": "RDSReadOnly",
      "Effect": "Allow",
      "Action": [
        "rds:DescribeDBInstances"
      ],
      "Resource": "arn:aws:rds:${REGION}:${ACCOUNT_ID}:db:${PROJECT}-db"
    },
    {
      "Sid": "SecretsManagerRead",
      "Effect": "Allow",
      "Action": [
        "secretsmanager:GetSecretValue"
      ],
      "Resource": "arn:aws:secretsmanager:${REGION}:${ACCOUNT_ID}:secret:agentcore-demo-test1/db-password-*"
    }
  ]
}
EOF

# Create the policy
POLICY_ARN=$(aws iam create-policy \
  --policy-name "${PROJECT}-ec2-policy" \
  --policy-description "Least-privilege policy for AgentCore demo test 1 EC2 server" \
  --policy-document file:///tmp/agentcore-demo-test1-policy.json \
  --query 'Policy.Arn' --output text)

echo "Custom policy created: ${POLICY_ARN}"

# Save for later
cat >> .vpc_env <<EOF
POLICY_ARN=${POLICY_ARN}
EOF
echo "=== Custom least-privilege policy created ==="
```

**Run it:**
```bash
chmod +x 04a_create_iam_policy.sh
./04a_create_iam_policy.sh
```

### 6.2 Create the Role and Attach the Policy

**Script:** `04b_create_iam_role.sh`
```bash
#!/bin/bash
set -e

PROJECT="agentcore-demo-test1"
source .vpc_env

echo "=== Step 6.2: Creating IAM role and attaching policy ==="

# --- Create trust policy (only EC2 can assume this role) ---
cat > /tmp/trust-policy.json <<'EOF'
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Principal": { "Service": "ec2.amazonaws.com" },
      "Action": "sts:AssumeRole"
    }
  ]
}
EOF

# --- Create the role ---
aws iam create-role \
  --role-name "${PROJECT}-ec2-role" \
  --assume-role-policy-document file:///tmp/trust-policy.json

# --- Attach our custom least-privilege policy ---
aws iam attach-role-policy \
  --role-name "${PROJECT}-ec2-role" \
  --policy-arn "${POLICY_ARN}"

# --- Create instance profile (this is what actually gets attached to EC2) ---
aws iam create-instance-profile --instance-profile-name "${PROJECT}-ec2-profile"
aws iam add-role-to-instance-profile \
  --instance-profile-name "${PROJECT}-ec2-profile" \
  --role-name "${PROJECT}-ec2-role"

echo "=== IAM role created and custom policy attached ==="
echo "Role: ${PROJECT}-ec2-role"
echo "Policy: ${POLICY_ARN}"
echo ""
echo "What this role CAN do:"
echo "  - Read/write ONLY your 3 S3 buckets (not all S3)"
echo "  - Invoke Bedrock models and AgentCore runtime"
echo "  - Write logs to 4 specific CloudWatch log groups"
echo "  - Write metrics to 'DemoSDLC/Agent' namespace only"
echo "  - Read the database password from Secrets Manager"
echo ""
echo "What this role CANNOT do:"
echo "  - Delete your RDS database"
echo "  - Access other people's S3 buckets"
echo "  - Create new IAM users or roles"
echo "  - See your AWS billing information"
echo "  - Access any other AWS account's resources"
```

**Run it:**
```bash
chmod +x 04b_create_iam_role.sh
./04b_create_iam_role.sh
```

**Run it:**
```bash
chmod +x 04_create_iam_roles.sh
./04_create_iam_roles.sh
```

---

## Step 7: Launch Your EC2 Server

**Why:** This is the main computer where your application will run. It hosts FastAPI, Temporal, and the frontend.

**What we create:**
- An EC2 instance (virtual server) running Amazon Linux 2023
- Instance type: `t3.large` (2 vCPUs, 8 GB RAM — sufficient for demo)
- Docker and docker-compose pre-installed
- Your IAM role attached (so it can talk to S3, Bedrock, etc.)

### 7.1 Create the EC2 Instance

**Script:** `05_launch_ec2.sh`
```bash
#!/bin/bash
set -e

REGION="us-east-1"
PROJECT="agentcore-demo-test1"

source .vpc_env

echo "=== Step 7: Launching EC2 server ==="

# --- Create a key pair (for SSH login) ---
echo "Creating SSH key pair..."
aws ec2 create-key-pair \
  --key-name "${PROJECT}-key" \
  --query 'KeyMaterial' --output text > "${PROJECT}-key.pem"
chmod 400 "${PROJECT}-key.pem"
echo "Key saved to: ${PROJECT}-key.pem (DO NOT LOSE THIS)"

# --- Launch the EC2 instance ---
echo "Launching EC2 instance (this takes 2-3 minutes)..."
INSTANCE_ID=$(aws ec2 run-instances \
  --image-id ami-0c02fb55956c7d316 \
  --instance-type t3.large \
  --key-name "${PROJECT}-key" \
  --security-group-ids "${EC2_SG}" \
  --subnet-id "${PUB_SUBNET}" \
  --iam-instance-profile Name="${PROJECT}-ec2-profile" \
  --block-device-mappings 'DeviceName=/dev/xvda,Ebs={VolumeSize=30,VolumeType=gp3}' \
  --tag-specifications "ResourceType=instance,Tags=[{Key=Name,Value=${PROJECT}-server}]" \
  --query 'Instances[0].InstanceId' --output text)

echo "Instance ID: ${INSTANCE_ID}"

# Wait for it to be running
echo "Waiting for instance to be ready..."
aws ec2 wait instance-running --instance-ids "${INSTANCE_ID}"

# Get the public IP
PUBLIC_IP=$(aws ec2 describe-instances \
  --instance-ids "${INSTANCE_ID}" \
  --query 'Reservations[0].Instances[0].PublicIpAddress' --output text)

echo "=== EC2 server is running! ==="
echo "Public IP: ${PUBLIC_IP}"
echo "SSH command: ssh -i ${PROJECT}-key.pem ec2-user@${PUBLIC_IP}"

# Save
cat >> .vpc_env <<EOF
INSTANCE_ID=${INSTANCE_ID}
PUBLIC_IP=${PUBLIC_IP}
EOF
```

**Run it:**
```bash
chmod +x 05_launch_ec2.sh
./05_launch_ec2.sh
```

### 7.2 Install Docker on the Server

Once the server is running, SSH into it and install Docker:

```bash
# Replace YOUR_IP with the Public IP from the previous step
ssh -i agentcore-demo-test1-key.pem ec2-user@YOUR_IP

# Inside the server, run these commands:
sudo yum update -y
sudo yum install -y docker
sudo systemctl start docker
sudo systemctl enable docker
sudo usermod -aG docker ec2-user

# Install docker-compose
sudo curl -L "https://github.com/docker/compose/releases/latest/download/docker-compose-$(uname -s)-$(uname -m)" \
  -o /usr/local/bin/docker-compose
sudo chmod +x /usr/local/bin/docker-compose

# Verify
docker --version
docker-compose --version

# Exit and log back in for group changes to take effect
exit
ssh -i agentcore-demo-test1-key.pem ec2-user@YOUR_IP

# Test docker without sudo
docker ps
```

### 7.3 Configure IMDSv2 (Instance Metadata Service) — Critical for Security

**What is IMDS?** IMDS is an AWS service built into every EC2 instance. It lives at a special IP address (`169.254.169.254`) inside your server. When your application (or a Docker container) uses boto3 without explicit credentials, boto3 automatically asks IMDS: "Hey, what credentials should I use?" IMDS replies with temporary credentials that are valid for a few hours. AWS rotates these automatically.

**Why IMDSv2?** Version 1 had a security vulnerability where attackers could steal credentials. Version 2 requires a session token, making it much safer. We enforce v2 only.

**How it works in practice:**
```
Your Python Code (in Docker container)
    |
    | import boto3
    | client = boto3.client("s3")   # NO credentials passed!
    v
IMDS (169.254.169.254) on EC2
    |
    | "Give me temporary credentials for this EC2's IAM Role"
    v
AWS STS (Security Token Service)
    |
    | Returns: AccessKeyId + SecretKey + SessionToken (valid ~6 hours)
    v
Your Python Code uses those credentials automatically
```

**This means:** Your `.env` file and your code contain ZERO AWS credentials. If someone steals your code, they cannot access your AWS account. If a container is compromised, the credentials expire within hours.

### 7.4 Enable IMDSv2 on Your EC2 Instance

```bash
# SSH into your server first
ssh -i agentcore-demo-test1-key.pem ec2-user@YOUR_PUBLIC_IP

# Check current IMDS version (should show "v2.0")
TOKEN=$(curl -X PUT "http://169.254.169.254/latest/api/token" -H "X-aws-ec2-metadata-token-ttl-seconds: 21600")
curl -H "X-aws-ec2-metadata-token: $TOKEN" http://169.254.169.254/latest/meta-data/iam/info

# From YOUR LOCAL MACHINE (not SSH), enforce IMDSv2:
aws ec2 modify-instance-metadata-options \
  --instance-id YOUR_INSTANCE_ID \
  --http-tokens required \
  --http-endpoint enabled \
  --http-put-response-hop-limit 2
```

**What `http-put-response-hop-limit 2` means:** The default is 1, which means only the EC2 host can reach IMDS. Setting it to 2 allows Docker containers (which are one network hop away) to also reach IMDS. This is required for containers to use the IAM Role.

### 7.5 Verify IMDS Works from Inside a Container

```bash
# SSH into your server
ssh -i agentcore-demo-test1-key.pem ec2-user@YOUR_PUBLIC_IP

# Run a test container that uses AWS credentials
docker run --rm -it amazonlinux:2023 bash

# Inside the container, run:
TOKEN=$(curl -X PUT "http://169.254.169.254/latest/api/token" \
  -H "X-aws-ec2-metadata-token-ttl-seconds: 21600" 2>/dev/null)
curl -H "X-aws-ec2-metadata-token: $TOKEN" \
  http://169.254.169.254/latest/meta-data/iam/security-credentials/

# You should see: "agentcore-demo-test1-ec2-role" — this proves IMDS is working

# Now test with AWS CLI:
yum install -y awscli
aws sts get-caller-identity

# You should see your account number and the role ARN:
# "Arn": "arn:aws:sts::YOUR_ACCOUNT:assumed-role/agentcore-demo-test1-ec2-role/..."

exit  # exit the container
```

### 7.6 How Your Application Code Uses IMDS

**You write NO credential code.** Here is what your Python code looks like:

```python
# backend/app/storage/s3_claim_check.py
import boto3

# NO access key, NO secret key, NO .env variable!
# boto3 automatically uses IMDS via the EC2 IAM Role
s3_client = boto3.client("s3", region_name="us-east-1")

# This works because IMDS provides temporary credentials automatically
def upload_to_s3(bucket, key, data):
    s3_client.put_object(Bucket=bucket, Key=key, Body=data)
```

```python
# backend/app/temporal/activities.py
import boto3

# Same here — zero credentials in code
bedrock_client = boto3.client("bedrock-agentcore-runtime", region_name="us-east-1")

def invoke_agent(agent_role, prompt, session_state):
    response = bedrock_client.invoke_agent_runtime(
        agentRuntimeArn=AGENT_RUNTIME_ARN_MAP[agent_role],
        sessionId=session_state["workflow_id"],
        qualifier="DEFAULT",
        inputText=prompt,
        sessionState=session_state,
    )
    return response
```

**What NEVER appears in your code:**
```python
# WRONG — NEVER do this:
client = boto3.client(
    "s3",
    aws_access_key_id="AKIA...",      # NEVER hardcode
    aws_secret_access_key="secret",    # NEVER hardcode
    region_name="us-east-1"
)

# WRONG — NEVER do this either:
AWS_ACCESS_KEY_ID = os.getenv("AWS_ACCESS_KEY_ID")     # NOT NEEDED
AWS_SECRET_ACCESS_KEY = os.getenv("AWS_SECRET_ACCESS_KEY")  # NOT NEEDED
```

### 7.7 IMDS Checklist

After completing this section, verify:
- [ ] `aws ec2 modify-instance-metadata-options` shows `"HttpTokens": "required"`
- [ ] Container içinden `aws sts get-caller-identity` rol ARN'i gösteriyor
- [ ] Kodda hiçbir AWS credential (access key, secret key) yok
- [ ] `.env` dosyasında AWS_ACCESS_KEY_ID veya AWS_SECRET_ACCESS_KEY yok
- [ ] Uygulama S3'e yazabiliyor (IMDS credential'ları ile)
- [ ] Uygulama Bedrock'a erişebiliyor (IMDS credential'ları ile)

---

## Step 8: Enable CloudWatch Log Groups

**Why:** You need separate log storage for each part of your application so you can debug issues.

**Script:** `06_create_cloudwatch_log_groups.sh`
```bash
#!/bin/bash
set -e

REGION="us-east-1"
PROJECT="agentcore-demo-test1"

echo "=== Step 8: Creating CloudWatch log groups (canonical names per PCD §10) ==="

# Four log groups. Per-agent ADOT routing is not used — every record carries
# demo.agent_id for filtering, and AWS auto-creates per-agent vended log groups
# under /aws/bedrock-agentcore/runtimes/<agent_id>-<random>-<qualifier>.
for LOG_GROUP in \
    "/agentcore-demo-test1/agent-logs" \
    "/agentcore-demo-test1/adot-collector" \
    "/agentcore-demo-test1/fastapi" \
    "/agentcore-demo-test1/temporal-worker"; do
  aws logs create-log-group --log-group-name "${LOG_GROUP}" \
    --region "${REGION}" 2>/dev/null || echo "Exists: ${LOG_GROUP}"
  aws logs put-retention-policy --log-group-name "${LOG_GROUP}" \
    --retention-in-days 7 --region "${REGION}"
done

# NOTE: AgentCore vended logs (/aws/bedrock-agentcore/runtimes/<id>) are
# auto-created by AWS on first invocation. Do NOT pre-create them.

echo "=== 4 log groups created with 7-day retention ==="
```

**Run it:**
```bash
chmod +x 06_create_cloudwatch_log_groups.sh
./06_create_cloudwatch_log_groups.sh
```

---

## Step 9: Enable Bedrock Model Access

**Why:** Your agents need access to AI models (Claude 3.5 Sonnet for reasoning, Amazon Transcribe for audio). AWS requires you to explicitly enable each model.

### 9.1 Which Services Need to Be Enabled?

AWS has two types of services:
1. **Auto-enabled** — Available immediately (EC2, S3, IAM, CloudWatch)
2. **Manual enable** — You must explicitly request access (Bedrock models, some regions)

For this demo, you need to manually enable:

| Service | Why It Needs Enabling | How to Enable |
|---------|----------------------|---------------|
| **Claude 3.5 Sonnet** (Bedrock) | Third-party model requires opt-in | Console or CLI |
| **Amazon Transcribe** | Part of Bedrock ecosystem | Console or CLI |
| **Titan Embeddings** (optional) | Foundation model | Console or CLI |

**You do NOT need to enable:** EC2, S3, RDS, IAM, CloudWatch, Secrets Manager — these work immediately.

### 9.2 Enable Models via AWS Console (Recommended for First-Time)

**Step-by-step with screenshots:**

1. **Go to AWS Console:** Open [https://console.aws.amazon.com](https://console.aws.amazon.com) in your browser
2. **Find Bedrock:** In the search bar at the top, type **"Bedrock"** and click on **"Amazon Bedrock"** under Services
3. **Navigate to Model Access:**
   - In the left sidebar, click **"Model access"** (under "Foundation models")
   - You will see a page saying "You don't have access to any models yet" — this is normal
4. **Enable Models:**
   - Click the orange button **"Manage model access"** (top right)
   - You will see a list of model providers (Anthropic, Amazon, Cohere, etc.)
   - Find **Anthropic** section and check the box for:
     - **Claude 3.5 Sonnet** — `anthropic.claude-sonnet-4-6`
   - Find **Amazon** section and check boxes for:
     - **Titan Text G1 - Express** (if available)
     - **Titan Embeddings G1 - Text** (if you plan to use embeddings)
   - Find **Amazon Transcribe** section and check the box
5. **Submit Request:**
   - Click **"Save changes"** at the bottom
   - AWS will show "Requesting model access..." 
   - Wait 10-30 seconds
   - Status will change to "Access granted" with a green checkmark
6. **Verify:**
   - Go back to **"Model access"** in the left sidebar
   - You should see Claude 3.5 Sonnet with "Access granted" status

### 9.3 Enable Models via AWS CLI (Alternative)

If you prefer the command line:

```bash
# Enable Claude 3.5 Sonnet
aws bedrock put-model-access-policy \
  --region us-east-1 \
  --model-access-policy '{
    "modelPrivileges": [
      {
        "modelId": "anthropic.claude-sonnet-4-6",
        "privilege": "INFERENCE"
      }
    ]
  }'

# Enable Titan Embeddings (optional)
aws bedrock put-model-access-policy \
  --region us-east-1 \
  --model-access-policy '{
    "modelPrivileges": [
      {
        "modelId": "amazon.titan-embed-text-v2:0",
        "privilege": "INFERENCE"
      }
    ]
  }'

# Enable Amazon Transcribe (via console, no CLI for this)
# Go to Console > Transcribe > "Get started" > Click "Enable"
```

### 9.4 Verify Access

**Via Console:**
- Go to **Amazon Bedrock** > **Model access**
- Look for green checkmarks next to Claude 3.5 Sonnet

**Via CLI:**
```bash
# List all enabled models
aws bedrock get-model-access-policy --region us-east-1

# Check specifically for Claude 3.5 Sonnet
aws bedrock list-foundation-models \
  --region us-east-1 \
  --by-output-modality TEXT \
  --by-inference-type ON_DEMAND \
  --query 'modelSummaries[?modelId==`anthropic.claude-sonnet-4-6`].{ID:modelId, Name:modelName, Status:modelLifecycle.status}'

# Expected output:
# [
#   {
#     "ID": "anthropic.claude-sonnet-4-6",
#     "Name": "Claude 3.5 Sonnet",
#     "Status": "ACTIVE"
#   }
# ]
```

### 9.5 Test the Model Actually Works

```bash
# SSH into your EC2 server
ssh -i agentcore-demo-test1-key.pem ec2-user@YOUR_PUBLIC_IP

# Run a quick test (this uses IMDS credentials automatically!)
python3 << 'PYEOF'
import boto3
import json

client = boto3.client("bedrock-runtime", region_name="us-east-1")

response = client.invoke_model(
    modelId="anthropic.claude-sonnet-4-6",
    body=json.dumps({
        "anthropic_version": "bedrock-2023-05-31",
        "max_tokens": 100,
        "messages": [{"role": "user", "content": "Say hello in one word"}]
    })
)

result = json.loads(response["body"].read())
print("Response:", result["content"][0]["text"])
PYEOF
```

If you see "Hello" (or similar), everything is working — your IAM Role, IMDS, and Bedrock model access are all configured correctly.

---

## Step 10: Install the AgentCore CLI

**Why:** AgentCore CLI (`agentcore`) is the tool that deploys your agent code to AWS as S3 zip packages.

### 10.1 Install on Your EC2 Server

```bash
# SSH into your server
ssh -i agentcore-demo-test1-key.pem ec2-user@YOUR_PUBLIC_IP

# Install the AgentCore starter toolkit (inside .venv)
uv venv .venv && source .venv/bin/activate
uv pip install bedrock-agentcore-starter-toolkit

# Verify
agentcore --version
```

### 10.2 Verify AgentCore Service is Available

```bash
aws service-quotas list-service-quotas \
  --service-code bedrock-agentcore-runtime \
  --region us-east-1 \
  --query "Quotas[*].{Name:QuotaName,Value:Value}"
```

You should see quotas for AgentCore Runtime. If you see an error, wait a few minutes and try again.

---

## Step 11: Teardown (When You Are Done)

When you want to stop spending money, run this script to delete everything:

**Script:** `99_teardown.sh`
```bash
#!/bin/bash
set -e

REGION="us-east-1"
PROJECT="agentcore-demo-test1"
ACCOUNT_ID=$(aws sts get-caller-identity --query Account --output text)

echo "=== TEARDOWN: Deleting all resources ==="
read -p "Are you sure? This deletes EVERYTHING. Type 'yes' to confirm: " CONFIRM
if [ "${CONFIRM}" != "yes" ]; then
  echo "Cancelled."
  exit 1
fi

# Delete EC2 instance
source .vpc_env 2>/dev/null || true
if [ -n "${INSTANCE_ID:-}" ]; then
  echo "Deleting EC2 instance ${INSTANCE_ID}..."
  aws ec2 terminate-instances --instance-ids "${INSTANCE_ID}"
  aws ec2 wait instance-terminated --instance-ids "${INSTANCE_ID}"
fi

# Delete RDS database
echo "Deleting RDS database..."
aws rds delete-db-instance \
  --db-instance-identifier "${PROJECT}-db" \
  --skip-final-snapshot --delete-automated-backups 2>/dev/null || true

# Wait for RDS deletion
aws rds wait db-instance-deleted \
  --db-instance-identifier "${PROJECT}-db" 2>/dev/null || true

# Delete S3 buckets
echo "Deleting S3 buckets..."
for BUCKET in "${PROJECT}-audio-uploads-${ACCOUNT_ID}" \
              "${PROJECT}-artifacts-${ACCOUNT_ID}" \
              "${PROJECT}-claimcheck-${ACCOUNT_ID}"; do
  aws s3 rb "s3://${BUCKET}" --force 2>/dev/null || echo "Bucket ${BUCKET} not found or already empty"
done

# Delete IAM role
echo "Deleting IAM role..."
aws iam remove-role-from-instance-profile \
  --instance-profile-name "${PROJECT}-ec2-profile" \
  --role-name "${PROJECT}-ec2-role" 2>/dev/null || true
aws iam delete-instance-profile \
  --instance-profile-name "${PROJECT}-ec2-profile" 2>/dev/null || true
for POLICY in AmazonS3FullAccess AmazonBedrockFullAccess CloudWatchLogsFullAccess \
              AmazonRDSFullAccess SecretsManagerReadWrite CloudWatchFullAccess; do
  aws iam detach-role-policy \
    --role-name "${PROJECT}-ec2-role" \
    --policy-arn "arn:aws:iam::aws:policy/${POLICY}" 2>/dev/null || true
done
aws iam delete-role --role-name "${PROJECT}-ec2-role" 2>/dev/null || true

# Delete secrets
aws secretsmanager delete-secret \
  --secret-id "agentcore-demo-test1/db-password" \
  --force-delete-without-recovery 2>/dev/null || true

echo "=== Teardown complete! ==="
echo "NOTE: VPC, subnets, and security groups were not deleted to avoid dependency issues."
echo "To delete them manually, go to AWS Console > VPC."
```

---

## Quick Reference Card

| Task | Command |
|------|---------|
| Check who you are | `aws sts get-caller-identity` |
| SSH into server | `ssh -i agentcore-demo-test1-key.pem ec2-user@YOUR_IP` |
| Check EC2 status | `aws ec2 describe-instances --instance-ids YOUR_INSTANCE_ID` |
| Check RDS status | `aws rds describe-db-instances --db-instance-identifier agentcore-demo-test1-db` |
| List S3 buckets | `aws s3 ls` |
| View CloudWatch logs | Go to Console > CloudWatch > Log groups |
| Check Bedrock models | `aws bedrock list-foundation-models --region us-east-1` |
| Check billing | Go to Console > Billing > Bills |

---

## Adding a New Developer to the Project

**Scenario:** A new developer joins your team. They need to access the EC2 server, view logs, and deploy code. Here is exactly what to set up.

### What the New Developer Gets

| Access Type | What They Can Do | Setup Required |
|-------------|-----------------|----------------|
| **SSH to EC2** | Access the server, view logs, restart containers | Create SSH key pair + update security group |
| **AWS Console (read-only)** | View resources, check logs, monitor costs | Create IAM user with read-only policy |
| **Database (read-only)** | Query data for debugging | Share connection string (via jump host) |
| **Git repository** | Pull code, create branches | Add to GitHub/GitLab project |

### Step D1: Create an IAM User for the Developer

```bash
# Set the developer's name
DEV_NAME="developer-name"  # Change this!
PROJECT="agentcore-demo-test1"

echo "=== Creating IAM user for ${DEV_NAME} ==="

# Create the user
aws iam create-user --user-name "${PROJECT}-${DEV_NAME}"

# Create a login profile for Console access (they will change password on first login)
aws iam create-login-profile \
  --user-name "${PROJECT}-${DEV_NAME}" \
  --password "TempPass123!ChangeMe" \
  --password-reset-required

# Create access key for CLI usage
aws iam create-access-key --user-name "${PROJECT}-${DEV_NAME}"
# SAVE THE OUTPUT — SecretAccessKey is shown only once!
```

### Step D2: Attach Read-Only Policies

```bash
# Policy 1: View EC2 instances (cannot create/delete)
aws iam attach-user-policy \
  --user-name "${PROJECT}-${DEV_NAME}" \
  --policy-arn arn:aws:iam::aws:policy/AmazonEC2ReadOnlyAccess

# Policy 2: View CloudWatch logs and metrics
aws iam attach-user-policy \
  --user-name "${PROJECT}-${DEV_NAME}" \
  --policy-arn arn:aws:iam::aws:policy/CloudWatchReadOnlyAccess

# Policy 3: View S3 buckets (read-only for project buckets only — see custom policy below)
cat > /tmp/dev-s3-policy.json <<EOF
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Action": ["s3:GetObject", "s3:ListBucket"],
      "Resource": [
        "arn:aws:s3:::${PROJECT}-audio-uploads-*",
        "arn:aws:s3:::${PROJECT}-audio-uploads-*/*",
        "arn:aws:s3:::${PROJECT}-artifacts-*",
        "arn:aws:s3:::${PROJECT}-artifacts-*/*",
        "arn:aws:s3:::${PROJECT}-claimcheck-*",
        "arn:aws:s3:::${PROJECT}-claimcheck-*/*"
      ]
    }
  ]
}
EOF
aws iam put-user-policy \
  --user-name "${PROJECT}-${DEV_NAME}" \
  --policy-name "S3ProjectReadOnly" \
  --policy-document file:///tmp/dev-s3-policy.json

# Policy 4: View RDS (describe only, cannot modify)
aws iam attach-user-policy \
  --user-name "${PROJECT}-${DEV_NAME}" \
  --policy-arn arn:aws:iam::aws:policy/AmazonRDSReadOnlyAccess

# Policy 5: View Bedrock models (invoke for testing)
aws iam attach-user-policy \
  --user-name "${PROJECT}-${DEV_NAME}" \
  --policy-arn arn:aws:iam::aws:policy/AmazonBedrockReadOnly

echo "=== Read-only policies attached ==="
```

### Step D3: Grant SSH Access to EC2

The developer needs to connect to the EC2 server. You have two options:

**Option A: Share the existing key (simpler, less secure)**
```bash
# Send the .pem file securely (e.g., password manager, encrypted email)
# Developer uses:
# ssh -i agentcore-demo-test1-key.pem ec2-user@YOUR_PUBLIC_IP
```

**Option B: Create a separate key for the developer (recommended)**
```bash
# Generate a new key pair
aws ec2 create-key-pair \
  --key-name "${PROJECT}-${DEV_NAME}-key" \
  --query 'KeyMaterial' --output text > "${PROJECT}-${DEV_NAME}-key.pem"
chmod 400 "${PROJECT}-${DEV_NAME}-key.pem"

# Send this .pem file to the developer securely
# Developer uses:
# ssh -i agentcore-demo-test1-developer-name-key.pem ec2-user@YOUR_PUBLIC_IP
```

### Step D4: Update Security Group for Developer's IP

The EC2 security group only allows SSH from YOUR IP. Add the developer's IP:

```bash
# Get the developer's IP address (ask them to run this and tell you the result)
# They should run: curl https://checkip.amazonaws.com

# Add their IP to the security group
aws ec2 authorize-security-group-ingress \
  --group-id "${EC2_SG}" \
  --protocol tcp --port 22 \
  --cidr "DEVELOPER_IP/32" \
  --description "SSH access for ${DEV_NAME}"
```

### Step D5: What to Tell the New Developer

Send them this checklist:

```
Welcome! Here is everything you need:

1. AWS CLI Setup:
   - Install AWS CLI (see Step 0 in the setup guide)
   - Configure with your access key:
     aws configure
     Access Key ID: [from Step D1 output]
     Secret Access Key: [from Step D1 output]
     Region: us-east-1
     Output: json

2. SSH Access:
   - Save the .pem file to ~/.ssh/ and chmod 400 it
   - Connect: ssh -i agentcore-demo-test1-key.pem ec2-user@[PUBLIC_IP]

3. View Logs:
   - SSH into the server
   - docker logs [container-name]
   - Or use AWS Console > CloudWatch > Log groups

4. Database (read-only):
   - The database is NOT publicly accessible
   - To query: SSH into EC2, then:
     psql postgresql://postgres:[PASSWORD]@[DB_ENDPOINT]:5432/agentcore_demo_test1
   - (Ask the admin for the password — it's in AWS Secrets Manager)

5. Code Repository:
   - Git clone the project repository
   - Follow the DEMO_GELISTIRME_KILAVUZU.md for development flow

6. Important Rules:
   - NEVER put AWS credentials in code or .env files
   - The server uses IMDS — credentials are automatic
   - If you need to deploy agents, ask the admin (requires higher permissions)
   - Always use the demo-admin IAM user for CLI, never root
```

### Step D6: If the Developer Needs Deploy Access

If the developer needs to deploy agents or modify infrastructure:

```bash
# Add Bedrock full access (for agent deployment)
aws iam attach-user-policy \
  --user-name "${PROJECT}-${DEV_NAME}" \
  --policy-arn arn:aws:iam::aws:policy/AmazonBedrockFullAccess

# Add EC2 limited access (start/stop/reboot instances only)
cat > /tmp/dev-ec2-limited.json <<EOF
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Action": [
        "ec2:StartInstances",
        "ec2:StopInstances",
        "ec2:RebootInstances",
        "ec2:DescribeInstances"
      ],
      "Resource": "arn:aws:ec2:*:*:instance/*",
      "Condition": {
        "StringEquals": {
          "ec2:ResourceTag/Name": "${PROJECT}-server"
        }
      }
    }
  ]
}
EOF
aws iam put-user-policy \
  --user-name "${PROJECT}-${DEV_NAME}" \
  --policy-name "EC2LimitedAccess" \
  --policy-document file:///tmp/dev-ec2-limited.json
```

### Step D7: Removing a Developer

When someone leaves the project:

```bash
DEV_NAME="developer-name"
PROJECT="agentcore-demo-test1"

# Detach all policies
aws iam detach-user-policy --user-name "${PROJECT}-${DEV_NAME}" \
  --policy-arn arn:aws:iam::aws:policy/AmazonEC2ReadOnlyAccess 2>/dev/null || true
aws iam detach-user-policy --user-name "${PROJECT}-${DEV_NAME}" \
  --policy-arn arn:aws:iam::aws:policy/CloudWatchReadOnlyAccess 2>/dev/null || true

# Delete inline policies
aws iam delete-user-policy --user-name "${PROJECT}-${DEV_NAME}" \
  --policy-name "S3ProjectReadOnly" 2>/dev/null || true
aws iam delete-user-policy --user-name "${PROJECT}-${DEV_NAME}" \
  --policy-name "EC2LimitedAccess" 2>/dev/null || true

# Delete access keys
for KEY in $(aws iam list-access-keys --user-name "${PROJECT}-${DEV_NAME}" \
  --query 'AccessKeyMetadata[*].AccessKeyId' --output text); do
  aws iam delete-access-key --user-name "${PROJECT}-${DEV_NAME}" --access-key-id "${KEY}"
done

# Delete login profile
aws iam delete-login-profile --user-name "${PROJECT}-${DEV_NAME}" 2>/dev/null || true

# Delete the SSH key from AWS (they might still have the .pem file locally)
aws ec2 delete-key-pair --key-name "${PROJECT}-${DEV_NAME}-key" 2>/dev/null || true

# Finally, delete the user
aws iam delete-user --user-name "${PROJECT}-${DEV_NAME}"

echo "Developer ${DEV_NAME} removed successfully"
```

---

## Troubleshooting

| Problem | Likely Cause | Solution |
|---------|-------------|----------|
| "AccessDenied" errors | Wrong IAM user | Run `aws sts get-caller-identity` — should show `demo-admin` |
| RDS creation hangs | VPC or security group missing | Check `.vpc_env` has all IDs, re-run Step 3 |
| Cannot SSH to EC2 | Wrong IP in security group | Re-run Step 3 with `MY_IP=$(curl -s https://checkip.amazonaws.com)/32` |
| Bedrock model not found | Model not enabled | Go to Console > Bedrock > Model access and enable it |
| S3 bucket name taken | Someone else has that name | Bucket names are globally unique — the script uses your account ID to make it unique |
| Docker not found on EC2 | Installation failed | SSH in and run `sudo yum install -y docker` manually |

---

*End of AWS Setup Guide*
