# Prompt 00: Infrastructure Bootstrap — Phase 0 (AgentCore demo test 1)

**MODE: INTERACTIVE — User runs commands, AI guides and verifies**

## Reference Documents (READ THESE FIRST)

Before executing any steps, read the following documents in `resources/` for architectural context and decision rationale:

| Priority | Document | Why Read It |
|----------|----------|-------------|
| **PRIMARY** | `resources/Deliverable_1_Infrastructure_Cost_Report.md` | AWS resource sizing, cost estimates, region selection. Read Section A for the exact shell scripts this prompt is based on. |
| **PRIMARY** | `resources/Deliverable_6_AWS_Operations_Guide.md` | IAM setup, prerequisite checklist, operational commands. Read Section 2 for the AWS account preparation steps. |
| **REFERENCE** | `resources/Deliverable_0_PROJECT_CONTEXT.md` | Architecture decisions, topology diagram, non-functional requirements. Read Sections 1-3 for the "why" behind every infrastructure choice. |
| **REFERENCE** | `resources/ARCHITECTURE_DATAFLOW_GUIDE.md` | Complete system architecture with all 21 interactions mapped. Skim Layer 1 (Network) and Layer 2 (Compute) for the EC2/RDS/S3 topology. |

> **How to find these documents:** They are in the `resources/` folder of the `agentcore-demo-test1-docs` repo (or the `resources/` folder if docs are copied locally). All prompt and deliverable files follow the naming convention: `Prompt_NN_Descriptive_Name.md` and `Deliverable_N_Descriptive_Name.md`.

---

In this phase, YOU (the AI) guide the user through AWS setup. You NEVER execute commands yourself.
You provide ONE command at a time. The user runs it in their terminal and shares the output.
You verify the output, then provide the next command. This teaches the user AWS.

**CRITICAL RULES (NON-NEGOTIABLE):**
- **Python:** NEVER use global `pip install`. ALWAYS use `uv` with `.venv` (`uv venv .venv && source .venv/bin/activate && uv pip install ...`). The ONLY exception is `RUN pip install` inside a Dockerfile.
- **Node.js:** NEVER use `npm install -g`. ALWAYS use local `npm install` (or `pnpm install`, `yarn install`) into `./node_modules`.
- **All code comments MUST be in English only.** No other language in comments, docstrings, or string literals.
- **INTERACTIVE MODE:** Provide ONE command per step. Wait for user output. Do NOT batch commands.


**Target Models:** Gemini Flash, Claude Sonnet (LOW-THINKING)
**Execution Model:** INTERACTIVE — User executes, AI verifies
**Scope:** Guide user through ALL AWS infrastructure setup for AgentCore demo test 1
**Output:** Running EC2 instance with Docker, S3 buckets, RDS, IAM, S3 Files
**Platform:** macOS (zsh) at `/Users/ugurgocen/projects/agentcore-demo-test1/`
**GitHub:** SSH key configured, username `ugocen`
**Region:** `us-east-1` (override via `AWS_DEFAULT_REGION` env var)
**Tags:** `Project=agentcore-demo-test1`, `Environment=demo`

---

## EXECUTION RULES (READ BEFORE PROCEEDING)

1. **SEQUENTIAL ONLY**: Execute Step 1 fully, verify, then Step 2, verify. NEVER skip ahead.
2. **IDEMPOTENCY**: Every script must be safe to run twice. Use `describe`/`list` to check existence before creation.
3. **FAILURE HANDLING**: If ANY step fails, STOP immediately. Report the EXACT error message. Do NOT proceed.
4. **CHECKPOINT PATTERN**: At each CHECKPOINT, run ALL verification commands. Only proceed if ALL pass.
5. **VERIFICATION**: After every step, run the specified verification command and confirm output.
6. **FILE PATHS**: All scripts written to `infra/scripts/`. Env files written to `infra/scripts/`.

---

## PRE-REQUISITE VERIFICATION

Run these commands BEFORE starting. If any fail, STOP.

```bash
# Verify AWS CLI is installed and configured
aws sts get-caller-identity
# VERIFY: Returns your Account ID, UserId, and Arn. If not, STOP.

# Verify region
REGION="${AWS_DEFAULT_REGION:-us-east-1}"
echo "Region: $REGION"
# VERIFY: Region is set correctly.
```

---

## PRE-STEP: Verify Environment

The project directory and GitHub repos are already set up manually. Before we begin AWS infrastructure, confirm everything is ready.

**Tell me: "Ready" after running these:**

```bash
# 1. Confirm project directory exists
cd /Users/ugurgocen/projects/agentcore-demo-test1 && pwd && ls -la
# EXPECTED: 4 dirs (backend, frontend, infra, docs), .code-workspace, .gitignore

# 2. Confirm SSH to GitHub
ssh -T git@github.com
# EXPECTED: "Hi ugocen! You've successfully authenticated..."

# 3. Confirm git remotes
cd agentcore-demo-test1-backend && git remote -v && cd ..
cd agentcore-demo-test1-frontend && git remote -v && cd ..
cd agentcore-demo-test1-infra && git remote -v && cd ..
cd agentcore-demo-test1-docs && git remote -v && cd ..
# EXPECTED: origin → git@github.com:ugocen/REPO_NAME.git
```

**I verify: All 4 repos accessible on GitHub before we proceed.**

**CHECKPOINT PRE:**
- [ ] Project directory at `/Users/ugurgocen/projects/agentcore-demo-test1/`
- [ ] 4 GitHub repos exist and are accessible via SSH
- [ ] 4 local directories with `git init` + SSH remotes
- [ ] `.code-workspace` file present
- [ ] You confirmed: "Ready"
- [ ] I verified all 4 repos on GitHub

---

## STEP 1: VPC and Networking

### 1.1 Create the script directory

```bash
mkdir -p infra/scripts
```

### 1.2 Create `infra/scripts/00_create_vpc.sh`

Write the following file EXACTLY:

```bash
#!/bin/bash
set -euo pipefail

REGION="${AWS_DEFAULT_REGION:-us-east-1}"
PROJECT="agentcore-demo-test1"
ENV="demo"

echo "=== Creating VPC ==="

# Check if VPC already exists (idempotent)
EXISTING_VPC=$(aws ec2 describe-vpcs --filters "Name=tag:Name,Values=${PROJECT}-vpc" --query 'Vpcs[0].VpcId' --output text --region $REGION 2>/dev/null || echo "None")

if [ "$EXISTING_VPC" != "None" ] && [ "$EXISTING_VPC" != "null" ] && [ -n "$EXISTING_VPC" ]; then
    echo "VPC already exists: $EXISTING_VPC"
    VPC_ID=$EXISTING_VPC
else
    VPC_ID=$(aws ec2 create-vpc \
        --cidr-block 10.0.0.0/16 \
        --tag-specifications "ResourceType=vpc,Tags=[{Key=Name,Value=${PROJECT}-vpc},{Key=Project,Value=${PROJECT}},{Key=Environment,Value=${ENV}}]" \
        --query 'Vpc.VpcId' --output text --region $REGION)
    aws ec2 modify-vpc-attribute --vpc-id $VPC_ID --enable-dns-hostnames --region $REGION
    echo "Created VPC: $VPC_ID"
fi

# Save VPC_ID for later scripts
echo "VPC_ID=$VPC_ID" > infra/scripts/.vpc_env
echo "Done: VPC=$VPC_ID"
```

### 1.3 Create `infra/scripts/01_create_security_groups.sh`

Write the following file EXACTLY:

```bash
#!/bin/bash
set -euo pipefail

REGION="${AWS_DEFAULT_REGION:-us-east-1}"
PROJECT="agentcore-demo-test1"
ENV="demo"

# Source VPC_ID from previous script
if [ -f infra/scripts/.vpc_env ]; then
    source infra/scripts/.vpc_env
else
    echo "ERROR: .vpc_env not found. Run 00_create_vpc.sh first."
    exit 1
fi

echo "=== Creating Security Groups ==="

# --- EC2 Security Group ---
EXISTING_EC2_SG=$(aws ec2 describe-security-groups \
    --filters "Name=tag:Name,Values=${PROJECT}-ec2-sg" "Name=vpc-id,Values=${VPC_ID}" \
    --query 'SecurityGroups[0].GroupId' --output text --region $REGION 2>/dev/null || echo "None")

if [ "$EXISTING_EC2_SG" != "None" ] && [ "$EXISTING_EC2_SG" != "null" ] && [ -n "$EXISTING_EC2_SG" ]; then
    echo "EC2 Security Group already exists: $EXISTING_EC2_SG"
    EC2_SG_ID=$EXISTING_EC2_SG
else
    EC2_SG_ID=$(aws ec2 create-security-group \
        --group-name "${PROJECT}-ec2-sg" \
        --description "EC2 security group for ${PROJECT}" \
        --vpc-id $VPC_ID \
        --tag-specifications "ResourceType=security-group,Tags=[{Key=Name,Value=${PROJECT}-ec2-sg},{Key=Project,Value=${PROJECT}},{Key=Environment,Value=${ENV}}]" \
        --query 'GroupId' --output text --region $REGION)
    echo "Created EC2 Security Group: $EC2_SG_ID"
fi

# Get your public IP for SSH ingress
MY_IP=$(curl -s https://checkip.amazonaws.com)

# Add SSH ingress (port 22) from your IP only — idempotent by ignoring AlreadyExists errors
aws ec2 authorize-security-group-ingress \
    --group-id $EC2_SG_ID \
    --protocol tcp --port 22 --cidr "${MY_IP}/32" \
    --region $REGION 2>/dev/null || echo "SSH ingress rule already exists or updated"

# Add remaining ingress rules
for PORT in 80 443 3000 8000 7233 8081; do
    aws ec2 authorize-security-group-ingress \
        --group-id $EC2_SG_ID \
        --protocol tcp --port $PORT --cidr 0.0.0.0/0 \
        --region $REGION 2>/dev/null || echo "Port $PORT ingress rule already exists"
done

# --- RDS Security Group ---
EXISTING_RDS_SG=$(aws ec2 describe-security-groups \
    --filters "Name=tag:Name,Values=${PROJECT}-rds-sg" "Name=vpc-id,Values=${VPC_ID}" \
    --query 'SecurityGroups[0].GroupId' --output text --region $REGION 2>/dev/null || echo "None")

if [ "$EXISTING_RDS_SG" != "None" ] && [ "$EXISTING_RDS_SG" != "null" ] && [ -n "$EXISTING_RDS_SG" ]; then
    echo "RDS Security Group already exists: $EXISTING_RDS_SG"
    RDS_SG_ID=$EXISTING_RDS_SG
else
    RDS_SG_ID=$(aws ec2 create-security-group \
        --group-name "${PROJECT}-rds-sg" \
        --description "RDS security group for ${PROJECT}" \
        --vpc-id $VPC_ID \
        --tag-specifications "ResourceType=security-group,Tags=[{Key=Name,Value=${PROJECT}-rds-sg},{Key=Project,Value=${PROJECT}},{Key=Environment,Value=${ENV}}]" \
        --query 'GroupId' --output text --region $REGION)
    echo "Created RDS Security Group: $RDS_SG_ID"
fi

# Add PostgreSQL ingress from EC2 security group — idempotent
aws ec2 authorize-security-group-ingress \
    --group-id $RDS_SG_ID \
    --protocol tcp --port 5432 \
    --source-group $EC2_SG_ID \
    --region $REGION 2>/dev/null || echo "RDS ingress rule already exists"

# Save both SG IDs
echo "EC2_SG_ID=$EC2_SG_ID" >> infra/scripts/.vpc_env
echo "RDS_SG_ID=$RDS_SG_ID" >> infra/scripts/.vpc_env
echo "Done: EC2_SG=$EC2_SG_ID, RDS_SG=$RDS_SG_ID"
```

### 1.4 Make scripts executable and run them

```bash
chmod +x infra/scripts/00_create_vpc.sh
chmod +x infra/scripts/01_create_security_groups.sh
./infra/scripts/00_create_vpc.sh
./infra/scripts/01_create_security_groups.sh
```

**VERIFICATION after Step 1:**

```bash
# Verify .vpc_env exists and has VPC_ID
cat infra/scripts/.vpc_env
# VERIFY: Contains VPC_ID, EC2_SG_ID, RDS_SG_ID lines

# Verify VPC
aws ec2 describe-vpcs --filters "Name=tag:Name,Values=agentcore-demo-test1-vpc" --query 'Vpcs[0].VpcId' --output text
# VERIFY: Returns a VPC ID like vpc-xxxxxxxx (NOT "None" or "null")

# Verify Security Groups
aws ec2 describe-security-groups --filters "Name=tag:Project,Values=agentcore-demo-test1" --query 'SecurityGroups[*].[GroupId,GroupName]' --output table
# VERIFY: At least 2 rows returned (EC2 SG + RDS SG)
```

---

## CHECKPOINT 1

Run ALL of these. ALL must pass.

```bash
# CHECK 1A: VPC exists
VPC_CHECK=$(aws ec2 describe-vpcs --filters "Name=tag:Name,Values=agentcore-demo-test1-vpc" --query 'Vpcs[0].VpcId' --output text)
echo "VPC_CHECK=$VPC_CHECK"
# PASS CRITERION: NOT "None", NOT "null", NOT empty string

# CHECK 1B: Security Groups exist
SG_CHECK=$(aws ec2 describe-security-groups --filters "Name=tag:Project,Values=agentcore-demo-test1" --query 'length(SecurityGroups)' --output text)
echo "SG_COUNT=$SG_CHECK"
# PASS CRITERION: Returns "2" or greater

# CHECK 1C: .vpc_env file exists
ls -la infra/scripts/.vpc_env
# PASS CRITERION: File exists
```

**IF ANY CHECK FAILS**: STOP. Do not proceed. Report the exact error.
**IF ALL PASS**: Proceed to Step 2.

---

## STEP 2: S3 Buckets

### 2.1 Create `infra/scripts/02_create_s3_buckets.sh`

Write the following file EXACTLY:

```bash
#!/bin/bash
set -euo pipefail

REGION="${AWS_DEFAULT_REGION:-us-east-1}"
PROJECT="agentcore-demo-test1"
ENV="demo"

# Get AWS Account ID
ACCOUNT_ID=$(aws sts get-caller-identity --query Account --output text)
echo "Account ID: $ACCOUNT_ID"

BUCKETS=(
    "${PROJECT}-audio-uploads-${ACCOUNT_ID}"
    "${PROJECT}-artifacts-${ACCOUNT_ID}"
    "${PROJECT}-claimcheck-${ACCOUNT_ID}"
    "${PROJECT}-code-${ACCOUNT_ID}"
)

echo "=== Creating S3 Buckets ==="

for BUCKET in "${BUCKETS[@]}"; do
    echo "Processing bucket: $BUCKET"
    
    # Check if bucket exists
    if aws s3api head-bucket --bucket "$BUCKET" --region $REGION 2>/dev/null; then
        echo "  Bucket already exists: $BUCKET"
    else
        if [ "$REGION" == "us-east-1" ]; then
            aws s3api create-bucket \
                --bucket "$BUCKET" \
                --region $REGION
        else
            aws s3api create-bucket \
                --bucket "$BUCKET" \
                --region $REGION \
                --create-bucket-configuration LocationConstraint=$REGION
        fi
        echo "  Created bucket: $BUCKET"
    fi
    
    # Enable versioning (idempotent — safe to run multiple times)
    aws s3api put-bucket-versioning \
        --bucket "$BUCKET" \
        --versioning-configuration Status=Enabled \
        --region $REGION
    echo "  Versioning enabled: $BUCKET"
    
    # Enable SSE-S3 encryption
    aws s3api put-bucket-encryption \
        --bucket "$BUCKET" \
        --server-side-encryption-configuration '{
            "Rules": [{"ApplyServerSideEncryptionByDefault": {"SSEAlgorithm": "AES256"}}]
        }' \
        --region $REGION
    echo "  Encryption enabled: $BUCKET"
    
    # Add lifecycle rule: expire non-current versions after 30 days
    aws s3api put-bucket-lifecycle-configuration \
        --bucket "$BUCKET" \
        --lifecycle-configuration '{
            "Rules": [{
                "ID": "expire-old-versions",
                "Status": "Enabled",
                "NoncurrentVersionExpiration": {"NoncurrentDays": 30},
                "Filter": {"Prefix": ""}
            }]
        }' \
        --region $REGION 2>/dev/null || echo "  Lifecycle rule already set or applied"
    echo "  Lifecycle rule applied: $BUCKET"
    
    # Tag the bucket
    aws s3api put-bucket-tagging \
        --bucket "$BUCKET" \
        --tagging "TagSet=[{Key=Project,Value=${PROJECT}},{Key=Environment,Value=${ENV}}]" \
        --region $REGION
    echo "  Tagged: $BUCKET"
done

# Save bucket names for later scripts (canonical per PCD §3 D11)
BUCKET_LIST="${BUCKETS[*]}"
{
    echo "BUCKETS=$BUCKET_LIST"
    echo "S3_ACCOUNT_ID=$ACCOUNT_ID"
    echo "S3_BUCKET_AUDIO=${PROJECT}-audio-uploads-${ACCOUNT_ID}"
    echo "S3_BUCKET_ARTIFACTS=${PROJECT}-artifacts-${ACCOUNT_ID}"
    echo "S3_BUCKET_CLAIMCHECK=${PROJECT}-claimcheck-${ACCOUNT_ID}"
    echo "S3_BUCKET_CODE=${PROJECT}-code-${ACCOUNT_ID}"
} > infra/scripts/.s3_env
echo "Done: Created/verified 4 S3 buckets"
```

### 2.3 Create Amazon Transcribe Data Access Role

Amazon Transcribe needs its own IAM role to read audio files from S3 and write transcription results back. This is a **separate role** from the EC2 instance role.

```bash
#!/bin/bash
set -euo pipefail

REGION="${AWS_DEFAULT_REGION:-us-east-1}"
ACCOUNT_ID=$(aws sts get-caller-identity --query Account --output text)
PROJECT="agentcore-demo-test1"
ENV="demo"

echo "=== Creating Amazon Transcribe Data Access Role ==="

ROLE_NAME="AmazonTranscribeDataAccessRole"

# Create trust policy for Transcribe service
cat > /tmp/transcribe-trust-policy.json << 'EOF'
{
    "Version": "2012-10-17",
    "Statement": [
        {
            "Effect": "Allow",
            "Principal": {
                "Service": "transcribe.amazonaws.com"
            },
            "Action": "sts:AssumeRole"
        }
    ]
}
EOF

# Create the role (idempotent — safe to run multiple times)
if aws iam get-role --role-name "$ROLE_NAME" --region $REGION >/dev/null 2>&1; then
    echo "Role already exists: $ROLE_NAME"
else
    aws iam create-role \
        --role-name "$ROLE_NAME" \
        --assume-role-policy-document file:///tmp/transcribe-trust-policy.json \
        --description "IAM role for Amazon Transcribe to access S3 buckets" \
        --tags "Key=Project,Value=${PROJECT}" "Key=Environment,Value=${ENV}" \
        --region $REGION
    echo "Created role: $ROLE_NAME"
fi

# Attach inline policy for S3 access
cat > /tmp/transcribe-s3-policy.json << EOF
{
    "Version": "2012-10-17",
    "Statement": [
        {
            "Effect": "Allow",
            "Action": [
                "s3:GetObject"
            ],
            "Resource": [
                "arn:aws:s3:::${PROJECT}-audio-uploads-*/*",
                "arn:aws:s3:::${PROJECT}-artifacts-*/*"
            ]
        },
        {
            "Effect": "Allow",
            "Action": [
                "s3:PutObject"
            ],
            "Resource": [
                "arn:aws:s3:::${PROJECT}-artifacts-*/*"
            ]
        }
    ]
}
EOF

aws iam put-role-policy \
    --role-name "$ROLE_NAME" \
    --policy-name "${ROLE_NAME}-s3-policy" \
    --policy-document file:///tmp/transcribe-s3-policy.json \
    --region $REGION
echo "Attached S3 policy to $ROLE_NAME"

# Save role ARN for agent code
echo "TRANSCRIBE_DATA_ACCESS_ROLE_ARN=arn:aws:iam::${ACCOUNT_ID}:role/${ROLE_NAME}" > infra/scripts/.transcribe_env
echo "Done: Created/verified Transcribe Data Access Role"
```

### 2.2 Make executable and run

```bash
chmod +x infra/scripts/02_create_s3_buckets.sh
./infra/scripts/02_create_s3_buckets.sh
```

**VERIFICATION after Step 2:**

```bash
# Verify .s3_env exists
cat infra/scripts/.s3_env
# VERIFY: Contains BUCKETS and S3_ACCOUNT_ID lines

# Verify bucket count
aws s3api list-buckets --query 'Buckets[*].Name' --output table | grep "demo-" | wc -l
# VERIFY: Returns 3

# Verify versioning on one bucket (replace <account-id> with actual value)
source infra/scripts/.s3_env
FIRST_BUCKET=$(echo $BUCKETS | awk '{print $1}')
aws s3api get-bucket-versioning --bucket "$FIRST_BUCKET" --query 'Status' --output text
# VERIFY: Returns "Enabled"
```

---

## CHECKPOINT 2

Run ALL of these. ALL must pass.

```bash
# CHECK 2A: Get account ID
source infra/scripts/.s3_env 2>/dev/null || ACCOUNT_ID=$(aws sts get-caller-identity --query Account --output text)

# CHECK 2B: Exactly 4 buckets exist (audio-uploads, artifacts, claimcheck, code)
BUCKET_CHECK=$(aws s3api list-buckets --query "length(Buckets[?starts_with(Name, \`agentcore-demo-test1-\`) && ends_with(Name, \`-${ACCOUNT_ID}\`)])" --output text)
echo "BUCKET_COUNT=$BUCKET_CHECK"
# PASS CRITERION: Returns "4"

# CHECK 2C: Audio-uploads bucket has versioning
FIRST_BUCKET="agentcore-demo-test1-audio-uploads-${ACCOUNT_ID}"
VERS_CHECK=$(aws s3api get-bucket-versioning --bucket "$FIRST_BUCKET" --query 'Status' --output text)
echo "VERSIONING=$VERS_CHECK"
# PASS CRITERION: Returns "Enabled"
```

**IF ANY CHECK FAILS**: STOP. Do not proceed.
**IF ALL PASS**: Proceed to Step 3.

---

## STEP 3: RDS PostgreSQL

### 3.1 Create `infra/scripts/03_create_rds.sh`

Write the following file EXACTLY:

```bash
#!/bin/bash
set -euo pipefail

REGION="${AWS_DEFAULT_REGION:-us-east-1}"
PROJECT="agentcore-demo-test1"
ENV="demo"
DB_INSTANCE_ID="${PROJECT}-db"
MASTER_USER="postgres"

echo "=== Creating RDS PostgreSQL Instance ==="

# Source VPC and SG IDs
source infra/scripts/.vpc_env

# --- 1. Check if RDS instance already exists ---
EXISTING_DB=$(aws rds describe-db-instances \
    --db-instance-identifier "$DB_INSTANCE_ID" \
    --query 'DBInstances[0].DBInstanceStatus' --output text \
    --region $REGION 2>/dev/null || echo "not-found")

if [ "$EXISTING_DB" != "not-found" ] && [ "$EXISTING_DB" != "None" ] && [ "$EXISTING_DB" != "null" ]; then
    echo "RDS instance already exists, status: $EXISTING_DB"
else
    # --- 2. Create master password and store in Secrets Manager ---
    MASTER_PASSWORD=$(openssl rand -base64 32 | tr -dc 'a-zA-Z0-9' | head -c 24)
    
    # Check if secret already exists
    SECRET_EXISTS=$(aws secretsmanager describe-secret \
        --secret-id "${PROJECT}/db-password" \
        --region $REGION 2>/dev/null && echo "yes" || echo "no")
    
    if [ "$SECRET_EXISTS" == "no" ]; then
        aws secretsmanager create-secret \
            --name "${PROJECT}/db-password" \
            --description "Master password for ${PROJECT} RDS" \
            --secret-string "$MASTER_PASSWORD" \
            --tags "Key=Project,Value=${PROJECT}" "Key=Environment,Value=${ENV}" \
            --region $REGION
        echo "Created secret: ${PROJECT}/db-password"
    else
        # Retrieve existing password
        MASTER_PASSWORD=$(aws secretsmanager get-secret-value \
            --secret-id "${PROJECT}/db-password" \
            --query SecretString --output text --region $REGION)
        echo "Secret already exists, retrieved password"
    fi
    
    # --- 3. Create DB subnet group (need at least 2 subnets) ---
    # Get all subnets in the VPC
    SUBNET_IDS=$(aws ec2 describe-subnets \
        --filters "Name=vpc-id,Values=${VPC_ID}" \
        --query 'Subnets[*].SubnetId' --output text --region $REGION)
    
    if [ -z "$SUBNET_IDS" ]; then
        echo "No subnets found in VPC. Creating subnets..."
        # Get availability zones
        AZS=$(aws ec2 describe-availability-zones \
            --query 'AvailabilityZones[?State==`available`].ZoneName' \
            --output text --region $REGION | awk '{print $1, $2}')
        
        AZ1=$(echo $AZS | awk '{print $1}')
        AZ2=$(echo $AZS | awk '{print $2}')
        
        SUBNET1=$(aws ec2 create-subnet \
            --vpc-id $VPC_ID \
            --cidr-block 10.0.1.0/24 \
            --availability-zone $AZ1 \
            --tag-specifications "ResourceType=subnet,Tags=[{Key=Name,Value=${PROJECT}-subnet-1},{Key=Project,Value=${PROJECT}}]" \
            --query 'Subnet.SubnetId' --output text --region $REGION)
        
        SUBNET2=$(aws ec2 create-subnet \
            --vpc-id $VPC_ID \
            --cidr-block 10.0.2.0/24 \
            --availability-zone $AZ2 \
            --tag-specifications "ResourceType=subnet,Tags=[{Key=Name,Value=${PROJECT}-subnet-2},{Key=Project,Value=${PROJECT}}]" \
            --query 'Subnet.SubnetId' --output text --region $REGION)
        
        # Create internet gateway and route table for public access
        IGW=$(aws ec2 create-internet-gateway \
            --tag-specifications "ResourceType=internet-gateway,Tags=[{Key=Name,Value=${PROJECT}-igw},{Key=Project,Value=${PROJECT}}]" \
            --query 'InternetGateway.InternetGatewayId' --output text --region $REGION)
        aws ec2 attach-internet-gateway --internet-gateway-id $IGW --vpc-id $VPC_ID --region $REGION
        
        RT=$(aws ec2 create-route-table \
            --vpc-id $VPC_ID \
            --tag-specifications "ResourceType=route-table,Tags=[{Key=Name,Value=${PROJECT}-rt},{Key=Project,Value=${PROJECT}}]" \
            --query 'RouteTable.RouteTableId' --output text --region $REGION)
        aws ec2 create-route --route-table-id $RT --destination-cidr-block 0.0.0.0/0 --gateway-id $IGW --region $REGION
        aws ec2 associate-route-table --route-table-id $RT --subnet-id $SUBNET1 --region $REGION
        aws ec2 associate-route-table --route-table-id $RT --subnet-id $SUBNET2 --region $REGION
        
        SUBNET_IDS="$SUBNET1 $SUBNET2"
    else
        # Subnets already exist — extract first two IDs into SUBNET1, SUBNET2 vars
        SUBNET1=$(echo $SUBNET_IDS | awk '{print $1}')
        SUBNET2=$(echo $SUBNET_IDS | awk '{print $2}')
        if [ -z "$SUBNET2" ]; then
            echo "ERROR: only one subnet found; RDS requires 2 AZs. Delete the single subnet or add a second one in a different AZ."
            exit 1
        fi
        echo "Reusing existing subnets: SUBNET1=$SUBNET1, SUBNET2=$SUBNET2"
    fi
    
    # Format subnet IDs for DB subnet group
    SUBNET_ARRAY=$(echo $SUBNET_IDS | tr ' ' ',' | sed 's/\([^,]*\)/"\1"/g')
    
    # Check/create DB subnet group
    DB_SUBNET_GROUP="${PROJECT}-db-subnet-group"
    if ! aws rds describe-db-subnet-groups \
        --db-subnet-group-name "$DB_SUBNET_GROUP" \
        --region $REGION >/dev/null 2>&1; then
        # Build subnet list properly
        SUBNET_LIST=$(echo $SUBNET_IDS | tr ' ' '\n' | head -n 3)
        SUBNET_ARN_JSON=$(echo "$SUBNET_LIST" | jq -R . | jq -s .)
        
        aws rds create-db-subnet-group \
            --db-subnet-group-name "$DB_SUBNET_GROUP" \
            --db-subnet-group-description "Subnet group for ${PROJECT}" \
            --subnet-ids $(echo $SUBNET_IDS | tr '\n' ' ' | head -n 2) \
            --tags "Key=Project,Value=${PROJECT}" "Key=Environment,Value=${ENV}" \
            --region $REGION
        echo "Created DB subnet group: $DB_SUBNET_GROUP"
    else
        echo "DB subnet group already exists: $DB_SUBNET_GROUP"
    fi
    
    # --- 4. Create RDS instance ---
    echo "Creating RDS instance (this will take 5-10 minutes)..."
    aws rds create-db-instance \
        --db-instance-identifier "$DB_INSTANCE_ID" \
        --db-instance-class db.t4g.micro \
        --engine postgres \
        --engine-version "15" \
        --allocated-storage 20 \
        --storage-type gp3 \
        --db-name "postgres" \
        --master-username "$MASTER_USER" \
        --master-user-password "$MASTER_PASSWORD" \
        --vpc-security-group-ids "$RDS_SG_ID" \
        --db-subnet-group-name "$DB_SUBNET_GROUP" \
        --no-multi-az \
        --no-publicly-accessible \
        --storage-encrypted \
        --backup-retention-period 7 \
        --tags "Key=Project,Value=${PROJECT}" "Key=Environment,Value=${ENV}" \
        --region $REGION
    
    echo "RDS instance creation initiated: $DB_INSTANCE_ID"
fi

# --- 5. Wait for RDS to be available ---
echo "Waiting for RDS to become available..."
MAX_RETRIES=120  # 120 * 30s = 60 minutes max
RETRY=0
while true; do
    STATUS=$(aws rds describe-db-instances \
        --db-instance-identifier "$DB_INSTANCE_ID" \
        --query 'DBInstances[0].DBInstanceStatus' --output text \
        --region $REGION 2>/dev/null || echo "unknown")
    
    echo "  RDS status: $STATUS (attempt $((RETRY+1))/$MAX_RETRIES)"
    
    if [ "$STATUS" == "available" ]; then
        echo "RDS is available!"
        break
    fi
    
    RETRY=$((RETRY+1))
    if [ $RETRY -ge $MAX_RETRIES ]; then
        echo "ERROR: RDS did not become available within timeout"
        exit 1
    fi
    sleep 30
done

# Get RDS endpoint
RDS_ENDPOINT=$(aws rds describe-db-instances \
    --db-instance-identifier "$DB_INSTANCE_ID" \
    --query 'DBInstances[0].Endpoint.Address' --output text \
    --region $REGION)
echo "RDS Endpoint: $RDS_ENDPOINT"

# Get master password
MASTER_PASSWORD=$(aws secretsmanager get-secret-value \
    --secret-id "${PROJECT}/db-password" \
    --query SecretString --output text --region $REGION)

# --- 6. Create databases ---
echo "Creating databases..."

# Install psql if not present (Amazon Linux / Fedora family)
if ! command -v psql &>/dev/null; then
    if command -v dnf &>/dev/null; then
        sudo dnf install -y postgresql15 2>/dev/null || sudo dnf install -y postgresql
    elif command -v apt-get &>/dev/null; then
        sudo apt-get update && sudo apt-get install -y postgresql-client
    elif command -v apk &>/dev/null; then
        sudo apk add postgresql-client
    else
        echo "WARNING: Cannot install psql automatically. Databases must be created manually."
        # Save endpoint and exit — user can create DBs later
        echo "RDS_ENDPOINT=$RDS_ENDPOINT" > infra/scripts/.rds_env
        echo "DB_STATUS=endpoint_only" >> infra/scripts/.rds_env
        exit 0
    fi
fi

# Wait a moment for RDS to fully accept connections
sleep 10

# Create temporal database (for Temporal Server — schema NOT managed here)
PGPASSWORD="$MASTER_PASSWORD" psql \
    -h "$RDS_ENDPOINT" -U "$MASTER_USER" -d postgres \
    -tc "SELECT 1 FROM pg_database WHERE datname = 'temporal'" | grep -q 1 || \
PGPASSWORD="$MASTER_PASSWORD" psql \
    -h "$RDS_ENDPOINT" -U "$MASTER_USER" -d postgres \
    -c "CREATE DATABASE temporal;"
echo "Database 'temporal' created or already exists"

# Create temporal_visibility database (required by temporalio/auto-setup image per PCD §2.2)
PGPASSWORD="$MASTER_PASSWORD" psql \
    -h "$RDS_ENDPOINT" -U "$MASTER_USER" -d postgres \
    -tc "SELECT 1 FROM pg_database WHERE datname = 'temporal_visibility'" | grep -q 1 || \
PGPASSWORD="$MASTER_PASSWORD" psql \
    -h "$RDS_ENDPOINT" -U "$MASTER_USER" -d postgres \
    -c "CREATE DATABASE temporal_visibility;"
echo "Database 'temporal_visibility' created or already exists"

# Create application database
PGPASSWORD="$MASTER_PASSWORD" psql \
    -h "$RDS_ENDPOINT" -U "$MASTER_USER" -d postgres \
    -tc "SELECT 1 FROM pg_database WHERE datname = 'agentcore_demo_test1'" | grep -q 1 || \
PGPASSWORD="$MASTER_PASSWORD" psql \
    -h "$RDS_ENDPOINT" -U "$MASTER_USER" -d postgres \
    -c "CREATE DATABASE agentcore_demo_test1;"
echo "Database 'agentcore_demo_test1' created or already exists"

# --- 7. Create users (passwords captured and exported to .vpc_env per PCD §3 D12) ---
echo "Creating users..."

# temporal_user — full access to temporal + temporal_visibility DBs
TEMPORAL_PASSWORD=$(openssl rand -base64 24 | tr -dc 'a-zA-Z0-9' | head -c 20)
PGPASSWORD="$MASTER_PASSWORD" psql \
    -h "$RDS_ENDPOINT" -U "$MASTER_USER" -d postgres \
    -tc "SELECT 1 FROM pg_roles WHERE rolname = 'temporal_user'" | grep -q 1 || \
PGPASSWORD="$MASTER_PASSWORD" psql \
    -h "$RDS_ENDPOINT" -U "$MASTER_USER" -d postgres \
    -c "CREATE USER temporal_user WITH PASSWORD '${TEMPORAL_PASSWORD}';"

# Grant temporal_user privileges on BOTH the temporal and temporal_visibility databases
for DB in temporal temporal_visibility; do
    PGPASSWORD="$MASTER_PASSWORD" psql \
        -h "$RDS_ENDPOINT" -U "$MASTER_USER" -d "$DB" \
        -c "GRANT ALL PRIVILEGES ON ALL TABLES IN SCHEMA public TO temporal_user;" 2>/dev/null || true
    PGPASSWORD="$MASTER_PASSWORD" psql \
        -h "$RDS_ENDPOINT" -U "$MASTER_USER" -d "$DB" \
        -c "GRANT ALL PRIVILEGES ON ALL SEQUENCES IN SCHEMA public TO temporal_user;" 2>/dev/null || true
    PGPASSWORD="$MASTER_PASSWORD" psql \
        -h "$RDS_ENDPOINT" -U "$MASTER_USER" -d "$DB" \
        -c "GRANT ALL PRIVILEGES ON SCHEMA public TO temporal_user;" 2>/dev/null || true
    PGPASSWORD="$MASTER_PASSWORD" psql \
        -h "$RDS_ENDPOINT" -U "$MASTER_USER" -d "$DB" \
        -c "ALTER DEFAULT PRIVILEGES IN SCHEMA public GRANT ALL ON TABLES TO temporal_user;" 2>/dev/null || true
done
echo "User 'temporal_user' created/configured on temporal + temporal_visibility"

# app_user — CREATE, INSERT, SELECT, UPDATE, DELETE on agentcore_demo_test1
# ONLY INSERT on evidence_packs table (ALCOA+ enduring constraint)
APP_PASSWORD=$(openssl rand -base64 24 | tr -dc 'a-zA-Z0-9' | head -c 20)

PGPASSWORD="$MASTER_PASSWORD" psql \
    -h "$RDS_ENDPOINT" -U "$MASTER_USER" -d postgres \
    -tc "SELECT 1 FROM pg_roles WHERE rolname = 'app_user'" | grep -q 1 || \
PGPASSWORD="$MASTER_PASSWORD" psql \
    -h "$RDS_ENDPOINT" -U "$MASTER_USER" -d postgres \
    -c "CREATE USER app_user WITH PASSWORD '${APP_PASSWORD}';"

# Grant schema usage
PGPASSWORD="$MASTER_PASSWORD" psql \
    -h "$RDS_ENDPOINT" -U "$MASTER_USER" -d agentcore_demo_test1 \
    -c "GRANT USAGE ON SCHEMA public TO app_user;" 2>/dev/null || true

# Grant default table privileges
PGPASSWORD="$MASTER_PASSWORD" psql \
    -h "$RDS_ENDPOINT" -U "$MASTER_USER" -d agentcore_demo_test1 \
    -c "ALTER DEFAULT PRIVILEGES IN SCHEMA public GRANT CREATE, INSERT, SELECT, UPDATE, DELETE ON TABLES TO app_user;" 2>/dev/null || true

# Store ALL passwords in Secrets Manager (master + temporal_user + app_user)
SECRET_JSON="{\"master\":\"${MASTER_PASSWORD}\",\"temporal_user\":\"${TEMPORAL_PASSWORD}\",\"app_user\":\"${APP_PASSWORD}\"}"
aws secretsmanager put-secret-value \
    --secret-id "${PROJECT}/db-password" \
    --secret-string "$SECRET_JSON" \
    --region $REGION 2>/dev/null || \
aws secretsmanager create-secret \
    --name "${PROJECT}/db-password" \
    --secret-string "$SECRET_JSON" \
    --tags "Key=Project,Value=${PROJECT}" "Key=Environment,Value=${ENV}" \
    --region $REGION

echo "User 'app_user' created/configured"

# Append all RDS variables to .vpc_env per PCD §3 D12
# (.rds_env retained for backward compatibility; canonical source is .vpc_env)
{
    echo "RDS_ENDPOINT=$RDS_ENDPOINT"
    echo "RDS_HOST=$RDS_ENDPOINT"
    echo "RDS_DB_TEMPORAL=temporal"
    echo "RDS_DB_TEMPORAL_VISIBILITY=temporal_visibility"
    echo "RDS_DB_APP=agentcore_demo_test1"
    echo "RDS_USER_TEMPORAL=temporal_user"
    echo "RDS_USER_APP=app_user"
    echo "RDS_PASSWORD_TEMPORAL=$TEMPORAL_PASSWORD"
    echo "RDS_PASSWORD_APP=$APP_PASSWORD"
    echo "SUBNET1=$SUBNET1"
    echo "SUBNET2=$SUBNET2"
} >> infra/scripts/.vpc_env

echo "RDS_ENDPOINT=$RDS_ENDPOINT" > infra/scripts/.rds_env
echo "DB_STATUS=ready" >> infra/scripts/.rds_env
echo "Done: RDS=$RDS_ENDPOINT, 3 databases created, 2 users configured, credentials in .vpc_env"
```

### 3.2 Make executable and run

```bash
chmod +x infra/scripts/03_create_rds.sh
./infra/scripts/03_create_rds.sh
```

**NOTE**: This script takes 5-10 minutes due to RDS provisioning. Do NOT interrupt.

**VERIFICATION after Step 3:**

```bash
# Verify .rds_env exists
cat infra/scripts/.rds_env
# VERIFY: Contains RDS_ENDPOINT and DB_STATUS=ready (or endpoint_only if psql unavailable)

# Verify RDS status
aws rds describe-db-instances --db-instance-identifier agentcore-demo-test1-db --query 'DBInstances[0].DBInstanceStatus' --output text
# VERIFY: Returns "available"

# Verify secret exists
aws secretsmanager get-secret-value --secret-id agentcore-demo-test1/db-password --query SecretString --output text
# VERIFY: Returns a password string (or JSON with master and app_user)
```

---

## CHECKPOINT 3

Run ALL of these. ALL must pass.

```bash
# CHECK 3A: RDS is available
RDS_STATUS=$(aws rds describe-db-instances --db-instance-identifier agentcore-demo-test1-db --query 'DBInstances[0].DBInstanceStatus' --output text)
echo "RDS_STATUS=$RDS_STATUS"
# PASS CRITERION: Returns "available"

# CHECK 3B: Secret exists
aws secretsmanager get-secret-value --secret-id agentcore-demo-test1/db-password --query SecretString --output text > /dev/null
# PASS CRITERION: Command succeeds (exit code 0)

# CHECK 3C: RDS endpoint is reachable
source infra/scripts/.rds_env 2>/dev/null || true
RDS_EP=$(aws rds describe-db-instances --db-instance-identifier agentcore-demo-test1-db --query 'DBInstances[0].Endpoint.Address' --output text)
echo "RDS_ENDPOINT=$RDS_EP"
# PASS CRITERION: Returns a hostname (NOT "None" or "null")

# CHECK 3D: .rds_env file exists
ls -la infra/scripts/.rds_env
# PASS CRITERION: File exists
```

**IF CHECK 3A FAILS**: Wait 2 more minutes and re-run. RDS can take up to 10 minutes.
**IF ANY CHECK FAILS after waiting**: STOP. Report the exact error.
**IF ALL PASS**: Proceed to Step 4.

---

## STEP 4: IAM Roles

### 4.1 Create `infra/scripts/04_create_iam_roles.sh`

Write the following file EXACTLY:

```bash
#!/bin/bash
set -euo pipefail

REGION="${AWS_DEFAULT_REGION:-us-east-1}"
PROJECT="agentcore-demo-test1"
ENV="demo"

# Source S3 bucket names and account ID
source infra/scripts/.s3_env 2>/dev/null || true
if [ -z "${S3_ACCOUNT_ID:-}" ]; then
    S3_ACCOUNT_ID=$(aws sts get-caller-identity --query Account --output text)
fi

echo "=== Creating IAM Roles ==="

# --- 1. EC2 Instance Profile Role ---
EC2_ROLE_NAME="${PROJECT}-ec2-role"

if aws iam get-role --role-name "$EC2_ROLE_NAME" --region $REGION >/dev/null 2>&1; then
    echo "EC2 role already exists: $EC2_ROLE_NAME"
else
    # Trust policy for EC2
    cat > /tmp/ec2-trust-policy.json <<'EOF'
{
    "Version": "2012-10-17",
    "Statement": [{
        "Effect": "Allow",
        "Principal": {"Service": "ec2.amazonaws.com"},
        "Action": "sts:AssumeRole"
    }]
}
EOF

    aws iam create-role \
        --role-name "$EC2_ROLE_NAME" \
        --assume-role-policy-document file:///tmp/ec2-trust-policy.json \
        --tags "Key=Project,Value=${PROJECT}" "Key=Environment,Value=${ENV}" \
        --region $REGION
    echo "Created EC2 role: $EC2_ROLE_NAME"
fi

# EC2 permissions policy (scoped S3 buckets, Bedrock, Transcribe, CloudWatch)
cat > /tmp/ec2-permissions-policy.json <<EOF
{
    "Version": "2012-10-17",
    "Statement": [
        {
            "Effect": "Allow",
            "Action": [
                "s3:GetObject",
                "s3:PutObject",
                "s3:DeleteObject",
                "s3:ListBucket"
            ],
            "Resource": [
                "arn:aws:s3:::agentcore-demo-test1-audio-uploads-${S3_ACCOUNT_ID}",
                "arn:aws:s3:::agentcore-demo-test1-audio-uploads-${S3_ACCOUNT_ID}/*",
                "arn:aws:s3:::agentcore-demo-test1-artifacts-${S3_ACCOUNT_ID}",
                "arn:aws:s3:::agentcore-demo-test1-artifacts-${S3_ACCOUNT_ID}/*",
                "arn:aws:s3:::agentcore-demo-test1-claimcheck-${S3_ACCOUNT_ID}",
                "arn:aws:s3:::agentcore-demo-test1-claimcheck-${S3_ACCOUNT_ID}/*",
                "arn:aws:s3:::agentcore-demo-test1-code-${S3_ACCOUNT_ID}",
                "arn:aws:s3:::agentcore-demo-test1-code-${S3_ACCOUNT_ID}/*"
            ]
        },
        {
            "Effect": "Allow",
            "Action": [
                "bedrock:InvokeModel",
                "bedrock:InvokeModelWithResponseStream"
            ],
            "Resource": "*"
        },
        {
            "Effect": "Allow",
            "Action": ["cloudwatch:GetMetricData", "cloudwatch:GetMetricStatistics", "cloudwatch:ListMetrics"],
            "Resource": "*"
        },
        {
            "Effect": "Allow",
            "Action": [
                "transcribe:StartTranscriptionJob",
                "transcribe:GetTranscriptionJob",
                "transcribe:ListTranscriptionJobs",
                "transcribe:DeleteTranscriptionJob"
            ],
            "Resource": "*"
        },
        {
            "Effect": "Allow",
            "Action": [
                "iam:PassRole"
            ],
            "Resource": "arn:aws:iam::*:role/AmazonTranscribeDataAccessRole",
            "Condition": {
                "StringEquals": {
                    "iam:PassedToService": "transcribe.amazonaws.com"
                }
            }
        },
        {
            "Effect": "Allow",
            "Action": [
                "logs:CreateLogGroup",
                "logs:CreateLogStream",
                "logs:PutLogEvents",
                "logs:DescribeLogGroups",
                "logs:DescribeLogStreams"
            ],
            "Resource": [
                "arn:aws:logs:*:*:log-group:/agentcore-demo-test1/*",
                "arn:aws:logs:*:*:log-group:/aws/bedrock-agentcore/runtimes/*"
            ]
        },
        {
            "Effect": "Allow",
            "Action": [
                "cloudwatch:PutMetricData"
            ],
            "Resource": "*"
        },
        {
            "Effect": "Allow",
            "Action": [
                "bedrock-agentcore:InvokeAgentRuntime"
            ],
            "Resource": "*"
        }
    ]
}
EOF

aws iam put-role-policy \
    --role-name "$EC2_ROLE_NAME" \
    --policy-name "${PROJECT}-ec2-policy" \
    --policy-document file:///tmp/ec2-permissions-policy.json \
    --region $REGION
echo "Attached inline policy to EC2 role"

# Create instance profile and attach role
INSTANCE_PROFILE_NAME="${EC2_ROLE_NAME}"
if ! aws iam get-instance-profile --instance-profile-name "$INSTANCE_PROFILE_NAME" --region $REGION >/dev/null 2>&1; then
    aws iam create-instance-profile \
        --instance-profile-name "$INSTANCE_PROFILE_NAME" \
        --region $REGION
    aws iam add-role-to-instance-profile \
        --instance-profile-name "$INSTANCE_PROFILE_NAME" \
        --role-name "$EC2_ROLE_NAME" \
        --region $REGION
    echo "Created instance profile: $INSTANCE_PROFILE_NAME"
else
    echo "Instance profile already exists: $INSTANCE_PROFILE_NAME"
fi

# --- 2. AgentCore Execution Role ---
AGENTCORE_ROLE_NAME="${PROJECT}-agentcore-role"

if aws iam get-role --role-name "$AGENTCORE_ROLE_NAME" --region $REGION >/dev/null 2>&1; then
    echo "AgentCore role already exists: $AGENTCORE_ROLE_NAME"
else
    cat > /tmp/agentcore-trust-policy.json <<'EOF'
{
    "Version": "2012-10-17",
    "Statement": [{
        "Effect": "Allow",
        "Principal": {"Service": "bedrock-agentcore.amazonaws.com"},
        "Action": "sts:AssumeRole"
    }]
}
EOF

    aws iam create-role \
        --role-name "$AGENTCORE_ROLE_NAME" \
        --assume-role-policy-document file:///tmp/agentcore-trust-policy.json \
        --tags "Key=Project,Value=${PROJECT}" "Key=Environment,Value=${ENV}" \
        --region $REGION
    echo "Created AgentCore role: $AGENTCORE_ROLE_NAME"
fi

# AgentCore permissions policy
cat > /tmp/agentcore-permissions-policy.json <<EOF
{
    "Version": "2012-10-17",
    "Statement": [
        {
            "Effect": "Allow",
            "Action": [
                "bedrock:InvokeModel",
                "bedrock:InvokeModelWithResponseStream"
            ],
            "Resource": "*"
        },
        {
            "Effect": "Allow",
            "Action": [
                "s3:GetObject",
                "s3:PutObject",
                "s3:ListBucket"
            ],
            "Resource": [
                "arn:aws:s3:::agentcore-demo-test1-audio-uploads-${S3_ACCOUNT_ID}",
                "arn:aws:s3:::agentcore-demo-test1-audio-uploads-${S3_ACCOUNT_ID}/*",
                "arn:aws:s3:::agentcore-demo-test1-artifacts-${S3_ACCOUNT_ID}",
                "arn:aws:s3:::agentcore-demo-test1-artifacts-${S3_ACCOUNT_ID}/*",
                "arn:aws:s3:::agentcore-demo-test1-claimcheck-${S3_ACCOUNT_ID}",
                "arn:aws:s3:::agentcore-demo-test1-claimcheck-${S3_ACCOUNT_ID}/*",
                "arn:aws:s3:::agentcore-demo-test1-code-${S3_ACCOUNT_ID}",
                "arn:aws:s3:::agentcore-demo-test1-code-${S3_ACCOUNT_ID}/*"
            ]
        },
        {
            "Effect": "Allow",
            "Action": [
                "logs:CreateLogGroup",
                "logs:CreateLogStream",
                "logs:PutLogEvents"
            ],
            "Resource": [
                "arn:aws:logs:*:*:log-group:/agentcore-demo-test1/*",
                "arn:aws:logs:*:*:log-group:/aws/bedrock-agentcore/runtimes/*"
            ]
        },
        {
            "Effect": "Allow",
            "Action": [
                "transcribe:StartTranscriptionJob",
                "transcribe:GetTranscriptionJob",
                "transcribe:ListTranscriptionJobs"
            ],
            "Resource": "*"
        }
    ]
}
EOF

aws iam put-role-policy \
    --role-name "$AGENTCORE_ROLE_NAME" \
    --policy-name "${PROJECT}-agentcore-policy" \
    --policy-document file:///tmp/agentcore-permissions-policy.json \
    --region $REGION
echo "Attached inline policy to AgentCore role"

# Save role ARNs
echo "EC2_ROLE_NAME=${EC2_ROLE_NAME}" > infra/scripts/.iam_env
echo "AGENTCORE_ROLE_NAME=${AGENTCORE_ROLE_NAME}" >> infra/scripts/.iam_env
echo "Done: EC2 role=${EC2_ROLE_NAME}, AgentCore role=${AGENTCORE_ROLE_NAME}"
```

### 4.2 Make executable and run

```bash
chmod +x infra/scripts/04_create_iam_roles.sh
./infra/scripts/04_create_iam_roles.sh
```

**VERIFICATION after Step 4:**

```bash
# Verify .iam_env exists
cat infra/scripts/.iam_env
# VERIFY: Contains EC2_ROLE_NAME and AGENTCORE_ROLE_NAME

# Verify EC2 instance profile
aws iam get-instance-profile --instance-profile-name agentcore-demo-test1-ec2-role --query 'InstanceProfile.Roles[0].RoleName' --output text
# VERIFY: Returns "agentcore-demo-test1-ec2-role"

# Verify AgentCore role
aws iam get-role --role-name agentcore-demo-test1-agentcore-role --query 'Role.RoleName' --output text
# VERIFY: Returns "agentcore-demo-test1-agentcore-role"
```

---

## CHECKPOINT 4

Run ALL of these. ALL must pass.

```bash
# CHECK 4A: EC2 instance profile exists
EC2_PROFILE=$(aws iam get-instance-profile --instance-profile-name agentcore-demo-test1-ec2-role 2>&1)
echo "$EC2_PROFILE" | grep -q "InstanceProfile"
EC2_PROFILE_OK=$?
echo "EC2_PROFILE_OK=$EC2_PROFILE_OK"
# PASS CRITERION: Exit code 0 (grep found "InstanceProfile")

# CHECK 4B: AgentCore role exists
AGENTCORE_ROLE=$(aws iam get-role --role-name agentcore-demo-test1-agentcore-role 2>&1)
echo "$AGENTCORE_ROLE" | grep -q "Role"
AGENTCORE_ROLE_OK=$?
echo "AGENTCORE_ROLE_OK=$AGENTCORE_ROLE_OK"
# PASS CRITERION: Exit code 0

# CHECK 4C: .iam_env file exists
ls -la infra/scripts/.iam_env
# PASS CRITERION: File exists
```

**IF ANY CHECK FAILS**: STOP. Report the exact error.
**IF ALL PASS**: Proceed to Step 5.

---

## STEP 5: EC2 Instance Launch

### 5.1 Create `infra/scripts/05_launch_ec2.sh`

Write the following file EXACTLY:

```bash
#!/bin/bash
set -euo pipefail

REGION="${AWS_DEFAULT_REGION:-us-east-1}"
PROJECT="agentcore-demo-test1"
ENV="demo"
INSTANCE_TAG_NAME="${PROJECT}-server"

echo "=== Launching EC2 Instance ==="

# Source VPC and IAM info
source infra/scripts/.vpc_env 2>/dev/null || true
source infra/scripts/.iam_env 2>/dev/null || true
source infra/scripts/.rds_env 2>/dev/null || true

# --- 1. Check if instance already exists and is running ---
EXISTING_INSTANCE=$(aws ec2 describe-instances \
    --filters \
        "Name=tag:Name,Values=${INSTANCE_TAG_NAME}" \
        "Name=instance-state-name,Values=running,pending,stopped" \
    --query 'Reservations[0].Instances[0].[InstanceId,State.Name]' \
    --output text --region $REGION 2>/dev/null || echo "None None")

EXISTING_ID=$(echo "$EXISTING_INSTANCE" | awk '{print $1}')
EXISTING_STATE=$(echo "$EXISTING_INSTANCE" | awk '{print $2}')

if [ "$EXISTING_ID" != "None" ] && [ "$EXISTING_ID" != "None" ] && [ -n "$EXISTING_ID" ]; then
    echo "Instance already exists: $EXISTING_ID (state: $EXISTING_STATE)"
    INSTANCE_ID=$EXISTING_ID
    
    # If stopped, start it
    if [ "$EXISTING_STATE" == "stopped" ]; then
        aws ec2 start-instances --instance-ids "$INSTANCE_ID" --region $REGION
        echo "Starting stopped instance..."
    fi
else
    # --- 2. Get Amazon Linux 2023 AMI ---
    AMI_ID=$(aws ec2 describe-images \
        --owners amazon \
        --filters \
            "Name=name,Values=al2023-ami-*-x86_64" \
            "Name=virtualization-type,Values=hvm" \
            "Name=architecture,Values=x86_64" \
        --query 'sort_by(Images, &CreationDate)[-1].ImageId' \
        --output text --region $REGION)
    echo "Using AMI: $AMI_ID"
    
    # --- 3. Create user data script ---
    cat > /tmp/user-data.sh <<'USERDATA'
#!/bin/bash
set -euo pipefail
exec > >(tee /var/log/user-data.log | logger -t user-data) 2>&1

echo "=== Starting User Data Setup ==="

# a. Update packages
dnf update -y

# b. Install Docker, docker-compose-plugin, git
dnf install -y docker git

# Enable and start Docker
systemctl enable docker
systemctl start docker

# Install docker-compose-plugin
dnf install -y docker-compose-plugin 2>/dev/null || \
    (curl -L "https://github.com/docker/compose/releases/latest/download/docker-compose-linux-x86_64" -o /usr/local/bin/docker-compose && chmod +x /usr/local/bin/docker-compose)

# c. Install wkhtmltopdf
dnf install -y wkhtmltopdf 2>/dev/null || \
    (dnf install -y wget xorg-x11-fonts-75dpi xorg-x11-fonts-Type1 libXext libXrender openssl && \
     wget https://github.com/wkhtmltopdf/packaging/releases/download/0.12.6.1-3/wkhtmltopdf-0.12.6.1-3.amazonlinux2.x86_64.rpm -O /tmp/wkhtmltopdf.rpm && \
     rpm -ivh /tmp/wkhtmltopdf.rpm 2>/dev/null || dnf localinstall -y /tmp/wkhtmltopdf.rpm)

# d. Create docker group, add ec2-user
usermod -aG docker ec2-user || true

# e. Clone project repos (4 separate repos via SSH from GitHub)
# GitHub username (per PCD header). Override with GITHUB_USER env var if you fork.
GITHUB_USER="${GITHUB_USER:-ugocen}"
if [ -n "$GITHUB_USER" ]; then
    for REPO in agentcore-demo-test1-backend agentcore-demo-test1-frontend agentcore-demo-test1-infra agentcore-demo-test1-docs; do
        git clone "git@github.com:${GITHUB_USER}/${REPO}.git" "/opt/${REPO}" 2>/dev/null || echo "Repo ${REPO} already cloned or unavailable"
    done
fi

# f. Install PostgreSQL client (for database connectivity)
dnf install -y postgresql15 2>/dev/null || dnf install -y postgresql

# Mark setup as complete
echo "=== User Data Setup Complete ===" > /opt/setup-complete.flag
chown ec2-user:ec2-user /opt/setup-complete.flag
USERDATA

    # --- 4. Launch instance ---
    # SUBNET1 is sourced from .vpc_env (populated by 03_create_rds.sh).
    # Fail loudly if missing — do not silently launch into an unknown subnet.
    if [ -z "${SUBNET1:-}" ]; then
        echo "ERROR: SUBNET1 not found in .vpc_env. Run 03_create_rds.sh first."
        exit 1
    fi
    INSTANCE_ID=$(aws ec2 run-instances \
        --image-id "$AMI_ID" \
        --instance-type t3.large \
        --iam-instance-profile Name="${EC2_ROLE_NAME}" \
        --security-group-ids "${EC2_SG_ID}" \
        --subnet-id "${SUBNET1}" \
        --tag-specifications "ResourceType=instance,Tags=[{Key=Name,Value=${INSTANCE_TAG_NAME}},{Key=Project,Value=${PROJECT}},{Key=Environment,Value=${ENV}}]" \
        --block-device-mappings '[{"DeviceName":"/dev/xvda","Ebs":{"VolumeSize":20,"VolumeType":"gp3"}}]' \
        --user-data file:///tmp/user-data.sh \
        --query 'Instances[0].InstanceId' \
        --output text --region $REGION)
    
    echo "Launched instance: $INSTANCE_ID"
fi

# --- 5. Wait for instance to be running ---
echo "Waiting for instance to reach 'running' state..."
aws ec2 wait instance-running --instance-ids "$INSTANCE_ID" --region $REGION
echo "Instance is running: $INSTANCE_ID"

# --- 6. Wait for status checks to pass ---
echo "Waiting for status checks to pass..."
MAX_RETRIES=60
RETRY=0
while true; do
    STATUS=$(aws ec2 describe-instance-status \
        --instance-ids "$INSTANCE_ID" \
        --query 'InstanceStatuses[0].SystemStatus.Status' \
        --output text --region $REGION 2>/dev/null || echo "unknown")
    
    echo "  System status: $STATUS (attempt $((RETRY+1))/$MAX_RETRIES)"
    
    if [ "$STATUS" == "ok" ]; then
        echo "Instance status checks passed!"
        break
    fi
    
    RETRY=$((RETRY+1))
    if [ $RETRY -ge $MAX_RETRIES ]; then
        echo "WARNING: Status checks not passing yet, but continuing..."
        break
    fi
    sleep 15
done

# --- 7. Get public IP ---
PUBLIC_IP=$(aws ec2 describe-instances \
    --instance-ids "$INSTANCE_ID" \
    --query 'Reservations[0].Instances[0].PublicIpAddress' \
    --output text --region $REGION)
echo "Public IP: $PUBLIC_IP"

# Save instance info
echo "INSTANCE_ID=$INSTANCE_ID" > infra/scripts/.ec2_env
echo "PUBLIC_IP=$PUBLIC_IP" >> infra/scripts/.ec2_env
echo "Done: EC2=$INSTANCE_ID, IP=$PUBLIC_IP"
```

### 5.2 Make executable and run

```bash
chmod +x infra/scripts/05_launch_ec2.sh
./infra/scripts/05_launch_ec2.sh
```

**NOTE**: This script takes 3-5 minutes. Do NOT interrupt.

**VERIFICATION after Step 5:**

```bash
# Verify .ec2_env exists
cat infra/scripts/.ec2_env
# VERIFY: Contains INSTANCE_ID and PUBLIC_IP

# Verify instance state
source infra/scripts/.ec2_env
aws ec2 describe-instances --instance-ids "$INSTANCE_ID" --query 'Reservations[0].Instances[0].State.Name' --output text
# VERIFY: Returns "running"

# Verify public IP is set
aws ec2 describe-instances --instance-ids "$INSTANCE_ID" --query 'Reservations[0].Instances[0].PublicIpAddress' --output text
# VERIFY: Returns an IP address
```

---

## CHECKPOINT 5

Run ALL of these. ALL must pass.

```bash
# CHECK 5A: Instance is running
source infra/scripts/.ec2_env 2>/dev/null || true
if [ -z "${INSTANCE_ID:-}" ]; then
    INSTANCE_ID=$(aws ec2 describe-instances --filters "Name=tag:Name,Values=agentcore-demo-test1-server" --query 'Reservations[0].Instances[0].InstanceId' --output text)
fi
STATE=$(aws ec2 describe-instances --instance-ids "$INSTANCE_ID" --query 'Reservations[0].Instances[0].State.Name' --output text)
echo "INSTANCE_STATE=$STATE"
# PASS CRITERION: Returns "running"

# CHECK 5B: Public IP exists
PUBLIC_IP=$(aws ec2 describe-instances --instance-ids "$INSTANCE_ID" --query 'Reservations[0].Instances[0].PublicIpAddress' --output text)
echo "PUBLIC_IP=$PUBLIC_IP"
# PASS CRITERION: Returns an IP address (NOT "None" or "null")

# CHECK 5C: .ec2_env file exists
ls -la infra/scripts/.ec2_env
# PASS CRITERION: File exists with content
```

**IF CHECK 5A FAILS**: STOP.
**IF CHECK 5B FAILS**: Wait 2 minutes and retry. If still fails, STOP.
**IF ALL PASS**: Proceed to Step 6.

---

## STEP 6: CloudWatch Log Groups

### 6.1 Create `infra/scripts/06_create_cloudwatch_log_groups.sh`

Write the following file EXACTLY:

```bash
#!/bin/bash
set -euo pipefail

REGION="${AWS_DEFAULT_REGION:-us-east-1}"
PROJECT="agentcore-demo-test1"
ENV="demo"

echo "=== Creating CloudWatch Log Groups ==="

## CloudWatch log group canonical names per PCD §10.
##
## FOUR log groups total. Per-agent separation is NOT needed because:
##   1. Every log line carries `demo.agent_id` attribute (Logs Insights:
##      `filter demo.agent_id = "agent_1_transcriber"`).
##   2. AWS auto-creates a separate vended log group per agent at
##      /aws/bedrock-agentcore/runtimes/<agent_id>-<random>-<qualifier>
##      for container stdout + native runtime events — that already provides
##      per-agent isolation.
##   3. One shared log group means one retention policy, one IAM ARN, one
##      ADOT exporter — far less operational overhead.

LOG_GROUPS=(
    "/agentcore-demo-test1/agent-logs"
    "/agentcore-demo-test1/adot-collector"
    "/agentcore-demo-test1/fastapi"
    "/agentcore-demo-test1/temporal-worker"
)

for LOG_GROUP in "${LOG_GROUPS[@]}"; do
    echo "Processing log group: $LOG_GROUP"
    
    # Check if log group exists
    if aws logs describe-log-groups \
        --log-group-name-prefix "$LOG_GROUP" \
        --query "logGroups[?logGroupName=='${LOG_GROUP}']" \
        --output text --region $REGION | grep -q "$LOG_GROUP"; then
        echo "  Log group already exists: $LOG_GROUP"
    else
        aws logs create-log-group \
            --log-group-name "$LOG_GROUP" \
            --region $REGION
        echo "  Created log group: $LOG_GROUP"
    fi
    
    # Set retention to 7 days (idempotent)
    aws logs put-retention-policy \
        --log-group-name "$LOG_GROUP" \
        --retention-in-days 7 \
        --region $REGION
    echo "  Set retention to 7 days: $LOG_GROUP"
    
    # Tag the log group
    aws logs tag-log-group \
        --log-group-name "$LOG_GROUP" \
        --tags "Project=${PROJECT},Environment=${ENV}" \
        --region $REGION 2>/dev/null || echo "  Tags already set"
done

echo "Done: Created/verified 4 CloudWatch log groups (7-day retention)"
```

### 6.2 Make executable and run

```bash
chmod +x infra/scripts/06_create_cloudwatch_log_groups.sh
./infra/scripts/06_create_cloudwatch_log_groups.sh
```

**VERIFICATION after Step 6:**

```bash
# Verify all 4 canonical log groups exist (per PCD §10)
for LG in "/agentcore-demo-test1/agent-logs" "/agentcore-demo-test1/adot-collector" "/agentcore-demo-test1/fastapi" "/agentcore-demo-test1/temporal-worker"; do
    RESULT=$(aws logs describe-log-groups --log-group-name-prefix "$LG" --query "length(logGroups[?logGroupName=='${LG}'])" --output text)
    echo "Log group $LG: count=$RESULT"   # VERIFY: each returns "1"
    R=$(aws logs describe-log-groups --log-group-name-prefix "$LG" --query 'logGroups[0].retentionInDays' --output text)
    echo "  retention=$R days"             # VERIFY: each prints retention=7
done

# Confirm AgentCore vended log groups have NOT been auto-created yet
# (AWS provisions them on first invoke_agent_runtime — Phase 1)
aws logs describe-log-groups --log-group-name-prefix "/aws/bedrock-agentcore/runtimes" --query 'length(logGroups)' --output text
# VERIFY: Returns "0" before Phase 1.
```

---

## CHECKPOINT 6

Run ALL of these. ALL must pass.

```bash
# CHECK 6A: All 4 canonical log groups exist
LOG_GROUP_COUNT=$(aws logs describe-log-groups --query "length(logGroups[?starts_with(logGroupName, \`/agentcore-demo-test1/\`)])" --output text)
echo "LOG_GROUP_COUNT=$LOG_GROUP_COUNT"
# PASS CRITERION: Returns "4"

# CHECK 6B: Retention is 7 days on every group
NOT_SEVEN=$(aws logs describe-log-groups --query "length(logGroups[?starts_with(logGroupName, \`/agentcore-demo-test1/\`) && retentionInDays != \`7\`])" --output text)
echo "NOT_SEVEN=$NOT_SEVEN"
# PASS CRITERION: Returns "0"
```

**IF ANY CHECK FAILS**: STOP. Report the exact error.
**IF ALL PASS**: Proceed to Step 7.

---

## STEP 7: Install BOTH AgentCore packages

Two SEPARATE PyPI packages exist and you need both:

| Package | Version (May 2026) | Purpose | Provides |
|---|---|---|---|
| `bedrock-agentcore` | ~1.9.x | Runtime SDK that the agent code imports | `from bedrock_agentcore.runtime import BedrockAgentCoreApp` |
| `bedrock-agentcore-starter-toolkit` | ~0.3.x | Deployment CLI on the dev machine | `agentcore` command (`configure`, `deploy`, `invoke`, `status`, `destroy`, `stop-session`, `identity ...`, `memory ...`, `gateway ...`, `policy ...`) |

> **The `agentcore` CLI command is installed by `bedrock-agentcore-starter-toolkit`, NOT by `bedrock-agentcore`.** If you install only `bedrock-agentcore`, you will get the SDK but no `agentcore` command and `which agentcore` will return empty. This is the most common stumbling block during Phase 1.

> **There is NO `agentcore bootstrap` command.** The S3 source bucket (`bedrock-agentcore-codebuild-sources-<account>-<region>`), CodeBuild project, and IAM execution role are **auto-created on the first `agentcore deploy` call** (Phase 1). Phase 0 only installs the CLI.

> **Deployment mode reality.** The toolkit's default `--deployment-type direct_code_deploy` mode handles everything between Python source and a running MicroVM: zip → S3 → CodeBuild → container image → runtime. You don't write a Dockerfile. Alternative `--deployment-type container` mode auto-creates an ECR repo and uses an explicit Dockerfile. Demo uses the default.

### 7.1 Install BOTH packages on the Mac dev environment

```bash
# Ensure .venv is active
cd /Users/ugurgocen/projects/agentcore-demo-test1/agentcore-demo-test1-backend
source .venv/bin/activate

# SDK (for the agent code to import) + CLI (for deployment)
uv pip install 'bedrock-agentcore' 'bedrock-agentcore-starter-toolkit'

# Verify both
python -c "from bedrock_agentcore.runtime import BedrockAgentCoreApp; print('SDK OK')"
agentcore --version
# Expected: SDK OK; agentcore version 0.3.x
```

### 7.2 Verify the CLI surface

Full Mayıs 2026 toolkit (v0.3.x) command tree:

| Group | Commands | Purpose |
|---|---|---|
| project | `create`, `configure`, `dev` | Scaffolding, config, local hot-reload dev server |
| runtime | `deploy`, `invoke`, `status`, `destroy`, `stop-session` | Lifecycle (note: `deploy` replaced the old `launch`) |
| platform | `identity`, `gateway`, `memory`, `policy`, `obs`, `eval` | Auth, gateway, memory, Cedar policies, observability queries, built-in evals |
| MCP | `create_mcp_gateway`, `create_mcp_gateway_target` | MCP gateway provisioning |

Verify locally:
```bash
agentcore --help                # Lists all 16 commands
agentcore configure --help      # --entrypoint/-e, --protocol/-p (HTTP|MCP|A2A|AGUI), --deployment-type/-dt (container|direct_code_deploy), --non-interactive/-ni, --disable-memory/-dm, --disable-otel/-do, --vpc, --subnets, --security-groups, ...
agentcore deploy --help         # --env KEY=VAL, --auto-update-on-conflict/-auc, --local/-l, --local-build/-lb, --image-tag/-t, --force-rebuild-deps/-frd, --region/-r
agentcore status --help
agentcore destroy --help        # --delete-ecr-repo flag (relevant if container mode was used)
agentcore dev --help            # Local hot-reload Python dev server (faster than full deploy for iteration)
agentcore obs --help            # CloudWatch span/trace/log queries
```

> **Quick local iteration tip:** `agentcore dev` runs your agent locally with hot reload — much faster than re-deploying through CodeBuild while iterating on `main.py`. Use it for fast feedback before pushing a real deploy.

> **AWS recommends migrating to the new `@aws/agentcore` npm CLI** for new projects (printed as a warning on every command). The Python toolkit shown here still works as of May 2026; suppress the warning with `export AGENTCORE_SUPPRESS_RECOMMENDATION=1`.

### 7.3 Install on EC2 (Phase 1 deploys from EC2 for IMDS-based AWS auth)

```bash
ssh -i <key> ec2-user@<EC2_IP> 'bash -lc "
  python3 -m venv ~/.venv && source ~/.venv/bin/activate &&
  pip install bedrock-agentcore bedrock-agentcore-starter-toolkit &&
  python -c \"from bedrock_agentcore.runtime import BedrockAgentCoreApp; print(\\\"SDK OK\\\")\" &&
  agentcore --version
"'
```

> **Persistent Filesystem note.** Earlier drafts of this prompt called fictional `aws bedrock-agentcore create-filesystem` and `create-access-point` commands. Those subcommands do not exist in the AWS CLI. Filesystem creation lives under separate services (`aws s3files` for S3 Files, `aws efs` for EFS) and attachment uses the control-plane API `aws bedrock-agentcore-control update-agent-runtime --filesystem-configurations`. For the demo, filesystem mounting is **deferred to a follow-up step** (`PERSISTENT_FILESYSTEM_GUIDE.md`) and is **NOT required for Phase 1 or Phase 2** — agents use boto3 S3 for any IO they need.

### 7.3 Make script executable marker

```bash
# Create a marker file indicating bootstrap is complete
cat > infra/scripts/.agentcore_env << EOF
AGENTCORE_BOOTSTRAP=complete
AGENTCORE_REGION=us-east-1
BOOTSTRAP_DATE=$(date -u +%Y-%m-%dT%H:%M:%SZ)
EOF

echo "AgentCore bootstrap complete. Marker file created."
```

---

## CHECKPOINT 7

Run ALL of these. ALL must pass.

```bash
# CHECK 7A: AgentCore toolkit installed locally
agentcore --version > /dev/null 2>&1
echo "OK: AgentCore toolkit installed on Mac"

# CHECK 7B: AgentCore toolkit installed on EC2 (Phase 1 deploy runs from there)
ssh -i <key> ec2-user@<EC2_IP> "~/.venv/bin/agentcore --version" > /dev/null 2>&1
echo "OK: AgentCore toolkit installed on EC2"

# CHECK 7C: Project code bucket (for agent ZIPs) exists per PCD §3 D11
aws s3 ls "s3://agentcore-demo-test1-code-$(aws sts get-caller-identity --query Account --output text)" > /dev/null 2>&1
echo "OK: agentcore-demo-test1-code bucket exists"
```

> **No deploy-bucket check, no execution-role check.** Both are auto-created by `agentcore deploy` on first run (Phase 1). Checking them here would fail by design.
>
> **No ECR check.** This project uses S3 ZIP deployment exclusively (PCD §15.1).
>
> **No filesystem checks.** AgentCore Persistent Filesystem attachment is deferred to a follow-up step (see `PERSISTENT_FILESYSTEM_GUIDE.md`) and is not required for Phase 1 or Phase 2.

**IF ANY CHECK FAILS**: STOP. Report the exact error.
**IF ALL PASS**: Phase 0 infrastructure is COMPLETE. Proceed to GATE PASS CHECKLIST.

---

## GATE PASS CHECKLIST — Phase 0 Complete

Run EVERY check below. ALL must pass.

### Resource Verification Commands

Run each command and verify the output.

```bash
# === GATE 1: VPC ===
VPC_CIDR=$(aws ec2 describe-vpcs --filters "Name=tag:Name,Values=agentcore-demo-test1-vpc" --query 'Vpcs[0].CidrBlock' --output text)
echo "GATE 1 - VPC CIDR: $VPC_CIDR"
# PASS: Returns "10.0.0.0/16"

# === GATE 2: Security Groups ===
SG_COUNT=$(aws ec2 describe-security-groups --filters "Name=tag:Project,Values=agentcore-demo-test1" --query 'length(SecurityGroups)' --output text)
echo "GATE 2 - SG Count: $SG_COUNT"
# PASS: Returns "2" or more

# === GATE 3: S3 Buckets ===
source infra/scripts/.s3_env 2>/dev/null || S3_ACCOUNT_ID=$(aws sts get-caller-identity --query Account --output text)
BUCKET_COUNT=$(aws s3api list-buckets --query "length(Buckets[?contains(Name, \`demo-${S3_ACCOUNT_ID}\`) == \`true\`])" --output text)
echo "GATE 3 - S3 Bucket Count: $BUCKET_COUNT"
# PASS: Returns "3"

# === GATE 4: RDS ===
RDS_STATUS=$(aws rds describe-db-instances --db-instance-identifier agentcore-demo-test1-db --query 'DBInstances[0].DBInstanceStatus' --output text)
RDS_EP=$(aws rds describe-db-instances --db-instance-identifier agentcore-demo-test1-db --query 'DBInstances[0].Endpoint.Address' --output text)
echo "GATE 4 - RDS Status: $RDS_STATUS, Endpoint: $RDS_EP"
# PASS: Status is "available", endpoint is not None/null

# === GATE 5: IAM Roles ===
aws iam get-instance-profile --instance-profile-name agentcore-demo-test1-ec2-role --output text > /dev/null
echo "GATE 5A - EC2 Instance Profile: OK"
# PASS: Command succeeds

aws iam get-role --role-name agentcore-demo-test1-agentcore-role --output text > /dev/null
echo "GATE 5B - AgentCore Role: OK"
# PASS: Command succeeds

# === GATE 6: EC2 Instance ===
source infra/scripts/.ec2_env 2>/dev/null || true
if [ -z "${INSTANCE_ID:-}" ]; then
    INSTANCE_ID=$(aws ec2 describe-instances --filters "Name=tag:Name,Values=agentcore-demo-test1-server" --query 'Reservations[0].Instances[0].InstanceId' --output text)
    PUBLIC_IP=$(aws ec2 describe-instances --instance-ids "$INSTANCE_ID" --query 'Reservations[0].Instances[0].PublicIpAddress' --output text)
fi
EC2_STATE=$(aws ec2 describe-instances --instance-ids "$INSTANCE_ID" --query 'Reservations[0].Instances[0].State.Name' --output text)
echo "GATE 6 - EC2 State: $EC2_STATE, IP: $PUBLIC_IP"
# PASS: State is "running", IP is not None/null

# === GATE 7: CloudWatch Log Groups ===
LOG_COUNT=$(aws logs describe-log-groups --query "length(logGroups[?contains(logGroupName, \`agentcore-demo-test1\`) == \`true\`])" --output text)
echo "GATE 7 - Log Group Count: $LOG_COUNT"
# PASS: Returns "4"

# === GATE 8: AgentCore Bootstrap ===
source infra/scripts/.agentcore_env 2>/dev/null || AGENTCORE_BOOTSTRAP="unknown"
echo "GATE 8 - AgentCore Bootstrap: $AGENTCORE_BOOTSTRAP"
# PASS: "complete"

# === GATE 9: .vpc_env file ===
ls infra/scripts/.vpc_env && grep -q "VPC_ID" infra/scripts/.vpc_env
echo "GATE 9 - .vpc_env exists: OK"
# PASS: File exists and contains VPC_ID

# === GATE 10: .ec2_env file ===
ls infra/scripts/.ec2_env && grep -q "PUBLIC_IP" infra/scripts/.ec2_env
echo "GATE 10 - .ec2_env exists: OK"
# PASS: File exists and contains PUBLIC_IP
```

### Final Checklist

ALL of the following MUST be true:

- [ ] **GATE 1**: VPC exists with CIDR `10.0.0.0/16`
- [ ] **GATE 2**: 2 Security Groups exist (EC2 + RDS)
- [ ] **GATE 3**: 3 S3 buckets exist with versioning enabled
- [ ] **GATE 4**: RDS `agentcore-demo-test1-db` is `available`, has endpoint
- [ ] **GATE 5**: IAM roles exist (EC2 instance profile + AgentCore execution role)
- [ ] **GATE 6**: EC2 instance is `running`, has public IP
- [ ] **GATE 7**: 4 CloudWatch log groups exist
- [ ] **GATE 8**: AgentCore bootstrap complete (toolkit installed, S3 bucket exists, IAM role exists)
- [ ] **GATE 9**: `.vpc_env` file exists with `VPC_ID`
- [ ] **GATE 10**: `.ec2_env` file exists with `PUBLIC_IP`

---

## COMPLETION DECISION

**IF ALL 10 GATES PASS**: Phase 0 is **COMPLETE**.
- All AWS infrastructure is ready
- Proceed to Phase 1 using `Prompt_01_AgentCore_Deployment.md`

**IF ANY GATE FAILS**: Phase 0 is **INCOMPLETE**.
- STOP. Report which gate failed and the exact error.
- Do NOT proceed to Phase 1.
- Re-run the corresponding step script after fixing the issue.

---

## Appendix: File Inventory

After successful execution, these files exist:

| File | Purpose |
|------|---------|
| `infra/scripts/.vpc_env` | VPC_ID, EC2_SG_ID, RDS_SG_ID |
| `infra/scripts/.s3_env` | BUCKETS list, S3_ACCOUNT_ID |
| `infra/scripts/.rds_env` | RDS_ENDPOINT, DB_STATUS |
| `infra/scripts/.iam_env` | EC2_ROLE_NAME, AGENTCORE_ROLE_NAME |
| `infra/scripts/.ec2_env` | INSTANCE_ID, PUBLIC_IP |
| `infra/scripts/00_create_vpc.sh` | VPC creation script |
| `infra/scripts/01_create_security_groups.sh` | Security group creation |
| `infra/scripts/02_create_s3_buckets.sh` | S3 bucket creation |
| `infra/scripts/03_create_rds.sh` | RDS PostgreSQL creation |
| `infra/scripts/04_create_iam_roles.sh` | IAM role creation |
| `infra/scripts/05_launch_ec2.sh` | EC2 instance launch |
| `infra/scripts/06_create_cloudwatch_log_groups.sh` | CloudWatch log groups |
| `infra/scripts/.agentcore_env` | AgentCore bootstrap marker |

## Appendix: Cleanup Script (Emergency Use Only)

If you need to destroy everything and start over, create and run:

```bash
#!/bin/bash
# WARNING: This destroys ALL resources created by this prompt!
# Only use if you need to start completely fresh.
set -euo pipefail
REGION="${AWS_DEFAULT_REGION:-us-east-1}"
PROJECT="agentcore-demo-test1"

echo "=== EMERGENCY CLEANUP ==="

# 1. Terminate EC2
INSTANCE_ID=$(aws ec2 describe-instances --filters "Name=tag:Name,Values=${PROJECT}-server" --query 'Reservations[0].Instances[0].InstanceId' --output text --region $REGION 2>/dev/null || echo "None")
if [ "$INSTANCE_ID" != "None" ] && [ "$INSTANCE_ID" != "null" ]; then
    aws ec2 terminate-instances --instance-ids "$INSTANCE_ID" --region $REGION
    echo "Terminating EC2: $INSTANCE_ID"
    aws ec2 wait instance-terminated --instance-ids "$INSTANCE_ID" --region $REGION
fi

# 2. Delete RDS
aws rds delete-db-instance --db-instance-identifier "${PROJECT}-db" --skip-final-snapshot --region $REGION 2>/dev/null || echo "RDS not found or already deleting"

# 3. Delete secrets
aws secretsmanager delete-secret --secret-id "${PROJECT}/db-password" --force-delete-without-recovery --region $REGION 2>/dev/null || echo "Secret not found"

# 4. Delete S3 buckets (canonical names per PCD §3 D11)
ACCOUNT_ID=$(aws sts get-caller-identity --query Account --output text)
for BUCKET in "agentcore-demo-test1-audio-uploads-${ACCOUNT_ID}" "agentcore-demo-test1-artifacts-${ACCOUNT_ID}" "agentcore-demo-test1-claimcheck-${ACCOUNT_ID}" "agentcore-demo-test1-code-${ACCOUNT_ID}"; do
    # Delete all object versions first (versioning is enabled)
    aws s3api delete-objects --bucket "$BUCKET" --delete "$(aws s3api list-object-versions --bucket "$BUCKET" --query='{Objects: Versions[].{Key:Key,VersionId:VersionId}}' --output json 2>/dev/null)" --region $REGION 2>/dev/null || true
    aws s3 rm "s3://${BUCKET}" --recursive --region $REGION 2>/dev/null || true
    aws s3api delete-bucket --bucket "$BUCKET" --region $REGION 2>/dev/null || echo "Bucket not found: $BUCKET"
done

# 5. Delete CloudWatch log groups (canonical paths per PCD §10)
for LG in "/agentcore-demo-test1/agent-logs" "/agentcore-demo-test1/adot-collector" "/agentcore-demo-test1/fastapi" "/agentcore-demo-test1/temporal-worker"; do
    aws logs delete-log-group --log-group-name "$LG" --region $REGION 2>/dev/null || echo "Log group not found: $LG"
done

# 6. Detach and delete IAM roles + custom policies (NOT AWS-managed policies)
# The role was created with a custom least-privilege policy; only detach that one.
aws iam remove-role-from-instance-profile --instance-profile-name "${PROJECT}-ec2-role" --role-name "${PROJECT}-ec2-role" --region $REGION 2>/dev/null || true
aws iam delete-instance-profile --instance-profile-name "${PROJECT}-ec2-role" --region $REGION 2>/dev/null || true
aws iam delete-role-policy --role-name "${PROJECT}-ec2-role" --policy-name "${PROJECT}-ec2-policy" --region $REGION 2>/dev/null || true
aws iam delete-role --role-name "${PROJECT}-ec2-role" --region $REGION 2>/dev/null || true

aws iam delete-role-policy --role-name "${PROJECT}-agentcore-role" --policy-name "${PROJECT}-agentcore-policy" --region $REGION 2>/dev/null || true
aws iam delete-role --role-name "${PROJECT}-agentcore-role" --region $REGION 2>/dev/null || true

# 7. Delete security groups
VPC_ID=$(aws ec2 describe-vpcs --filters "Name=tag:Name,Values=${PROJECT}-vpc" --query 'Vpcs[0].VpcId' --output text --region $REGION 2>/dev/null || echo "None")
if [ "$VPC_ID" != "None" ] && [ "$VPC_ID" != "null" ]; then
    for SG in "${PROJECT}-ec2-sg" "${PROJECT}-rds-sg"; do
        SG_ID=$(aws ec2 describe-security-groups --filters "Name=tag:Name,Values=${SG}" "Name=vpc-id,Values=${VPC_ID}" --query 'SecurityGroups[0].GroupId' --output text --region $REGION 2>/dev/null || echo "None")
        if [ "$SG_ID" != "None" ] && [ "$SG_ID" != "null" ]; then
            aws ec2 delete-security-group --group-id "$SG_ID" --region $REGION 2>/dev/null || echo "Cannot delete SG: $SG (may have dependencies)"
        fi
    done
fi

echo "=== Cleanup complete ==="
```

---

*End of Prompt 00: Infrastructure Bootstrap*


---

## TESTING REQUIREMENTS — pytest Infrastructure Verification

Every gate checkpoint MUST have a corresponding pytest test. Create:

```
tests/
  infra/
    conftest.py              # boto3 clients, region=us-east-1, project fixtures
    test_gate_0_ec2.py       # test_ec2_running, test_ec2_has_public_ip,
                             # test_ec2_iam_role_attached, test_imdsv2_enforced
    test_gate_0_rds.py       # test_rds_available, test_rds_is_postgresql15,
                             # test_rds_not_public, test_rds_encrypted,
                             # test_three_databases_exist (temporal, temporal_visibility, agentcore_demo_test1)
    test_gate_0_s3.py        # test_four_buckets_exist (audio-uploads, artifacts, claimcheck, code),
                             # test_audio_uploads_bucket_versioning
    test_gate_0_iam.py       # test_ec2_role_exists, test_ec2_role_has_custom_policy,
                             # test_instance_profile_exists, test_no_aws_managed_full_access_attached
    test_gate_0_bedrock.py   # test_claude_sonnet_active
    test_gate_0_subnets.py   # test_two_subnets_in_different_azs (required by RDS subnet group)
    test_gate_0_vpc_env.py   # test_vpc_env_has_all_required_vars
```

> **AgentCore Persistent Filesystem testing is deferred to Phase 1** (`Prompt_01`), where the filesystem is exercised against deployed agents. The `boto3.client("s3files", ...)` shim is not a real service client and would error if used here.

Run: `pytest tests/infra/ -v`

Key fixtures (conftest.py):
```python
import pytest, boto3, os, pathlib

@pytest.fixture(scope="session")
def region(): return os.environ.get("AWS_REGION", "us-east-1")

@pytest.fixture(scope="session")
def ec2_client(region): return boto3.client("ec2", region_name=region)

@pytest.fixture(scope="session")
def rds_client(region): return boto3.client("rds", region_name=region)

@pytest.fixture(scope="session")
def s3_client(region): return boto3.client("s3", region_name=region)

@pytest.fixture(scope="session")
def iam_client(region): return boto3.client("iam", region_name=region)

@pytest.fixture(scope="session")
def bedrock_client(region): return boto3.client("bedrock", region_name=region)

@pytest.fixture(scope="session")
def project(): return "agentcore-demo-test1"

@pytest.fixture(scope="session")
def vpc_env_path():
    """Path to the .vpc_env file emitted by the infra scripts."""
    return pathlib.Path("infra/scripts/.vpc_env")
```
