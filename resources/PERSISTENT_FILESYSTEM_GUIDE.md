# AgentCore Persistent Filesystems Guide

## S3 Files Integration for AgentCore demo test 1 Claim-Check Storage

**Date:** 2026-05-15 (corrected)
**Scope:** Replace boto3 S3 API with POSIX file operations via S3 Files mount
**AWS Service:** Amazon S3 Files (Bedrock AgentCore Persistent Filesystems — BYO S3)
**Reference:** https://docs.aws.amazon.com/bedrock-agentcore/latest/devguide/runtime-filesystem-configurations.html

---

## ⚠️ IMPORTANT API CORRECTION (2026-05-15)

Earlier versions of this guide showed `agentcore configure --filesystem-configurations` and `aws bedrock-agentcore create-filesystem` calls. **Both are incorrect.** The real API surface as of 2026-05 is:

| Operation | Correct command |
|---|---|
| Create S3 Files filesystem | `aws s3files create-file-system` (S3 Files is its own AWS service, NOT a `bedrock-agentcore` subcommand) |
| Create access point | `aws s3files create-access-point` |
| Create runtime + attach filesystem at creation | `aws bedrock-agentcore-control create-agent-runtime --filesystem-configurations '[...]'` |
| **Attach a filesystem to an EXISTING runtime** | `aws bedrock-agentcore-control update-agent-runtime --agent-runtime-id ... --filesystem-configurations '[...]'` |

Key namespace distinction:
- `bedrock-agentcore` (data plane) — invocation + memory only. **No filesystem CRUD.**
- `bedrock-agentcore-control` (control plane) — runtime creation/update + filesystem attachment.
- `s3files` — separate service for filesystem and access-point primitives.

The `agentcore configure` toolkit command has **no `--filesystem-configurations` flag**. The starter toolkit does not expose filesystem attachment — you must use the control-plane API after `agentcore deploy` finishes.

The sections below have been updated to reflect this. Examples that still call `aws bedrock-agentcore create-filesystem` or `agentcore configure --filesystem-configurations` are historical and marked accordingly.

---

## CRITICAL RULES (NON-NEGOTIABLE)

- **Deployment method: S3 ZIP ONLY.** There is NO ECR. There is NO `docker build` for agents. There is NO container push. Agent deployment is: `agentcore configure -e main.py --protocol HTTP` then `agentcore deploy`. CodeBuild handles the ARM64 build from the ZIP. The ONLY Dockerfile in this project is for `docker-compose` frontend/backend infrastructure (not for agent deployment).
- **Python environment isolation:** NEVER use global `pip install`. ALWAYS use `uv` with `.venv`:
  ```bash
  uv venv .venv
  source .venv/bin/activate  # Linux/Mac
  uv pip install -r requirements.txt
  ```
  - The ONLY exception is `RUN pip install` inside a Dockerfile.
- **Node.js environment isolation:** NEVER use `npm install -g`. ALWAYS install locally via `npm install`.
- **All code comments MUST be written in English only.**

---

## What Changed and Why

### The Old Way (Before)

Agents used `boto3.client("s3")` to read/write claim-check payloads:

```python
# OLD — inside agent microVM
s3 = boto3.client("s3")
s3.put_object(Bucket="claim-checks", Key="uuid.json", Body=data)
response = s3.get_object(Bucket="claim-checks", Key="uuid.json")
```

**Problems with this approach:**
- Every S3 call is an HTTPS network request (latency)
- Agent code depends on AWS SDK for storage
- Complex error handling (retries, throttling, partial failures)
- No atomic write semantics

### The New Way (Now)

Agents use ordinary POSIX file operations on a mounted S3 Files filesystem:

```python
# NEW — inside agent microVM
with open("/mnt/agentcore/claim-checks/uuid.json", "w") as f:
    json.dump(payload, f)

with open("/mnt/agentcore/claim-checks/uuid.json", "r") as f:
    payload = json.load(f)
```

**Benefits:**
- No network calls — files are local (mounted via NFSv4.2)
- Standard Python file I/O — no AWS SDK dependency for storage
- Atomic writes (close-to-open consistency)
- S3 Files handles sync to S3 automatically

### What Did NOT Change

| Component | Still Uses | Because |
|-----------|-----------|---------|
| Temporal Activities (EC2) | `boto3.client("s3")` | EC2 has no S3 Files mount |
| Evidence Pack Builder (EC2) | `boto3.client("s3")` | EC2 has no S3 Files mount |
| Claim-check reference dict | `_claim_check`, `bucket`, `key` | Both sides understand this format |
| ALCOA+ audit trail | S3 URI + hash + manifest | S3 Files is backed by the same S3 bucket |
| 256 KiB threshold | Still applies | Temporal event history size limit |

---

## Architecture Overview

```
┌─────────────────────────────────────────────────────────────────┐
│                        EC2 Server                                │
│                                                                  │
│  ┌────────────────────┐    ┌──────────────────────────────────┐  │
│  │ Temporal Activity  │    │ Evidence Pack Builder            │  │
│  │ (boto3 S3 API)     │    │ (boto3 S3 API)                   │  │
│  │                    │    │                                  │  │
│  │ s3.put_object()    │    │ s3.get_object()                  │  │
│  │ s3.get_object()    │    │                                  │  │
│  └────────┬───────────┘    └──────────┬───────────────────────┘  │
│           │                            │                          │
│           │  HTTPS (port 443)          │  HTTPS (port 443)       │
│           │                            │                          │
│           ▼                            ▼                          │
│     ┌──────────────────────────────────────────┐                  │
│     │    S3  (agentcore-demo-test1-claimcheck-{acct})     │                  │
│     │    claim-checks/uuid-123.json            │                  │
│     └────────────┬─────────────────────────────┘                  │
│                  │                                               │
│                  │  NFSv4.2 over TLS (port 2049)                 │
│                  │                                               │
└──────────────────┼───────────────────────────────────────────────┘
                   │
                   ▼
┌──────────────────────────────────────────────────────────────────┐
│                    AgentCore Runtime                               │
│                    (Firecracker MicroVM)                           │
│                                                                  │
│   S3 Files mount: /mnt/agentcore/claim-checks  ◄── S3 Files AP   │
│                                                                  │
│   ┌─────────────────────┐    ┌─────────────────────────────────┐ │
│   │ Agent 1 (Transcriber)│    │ Agent 2 (Drafter)               │ │
│   │                      │    │                                 │ │
│   │ open("...", "w")     │    │ open("...", "r")                │ │
   │                      │    │                                 │ │
│   └─────────────────────┘    └─────────────────────────────────┘ │
│                                                                  │
│   ┌──────────────────────────────────────────────────────────┐  │
│   │ Agent 3 (Reviewer — LangGraph)                            │  │
│   │                                                           │  │
│   │ with open("/mnt/agentcore/claim-checks/...", "r") as f:   │  │
│   │     payload = json.load(f)                                  │  │
│   └──────────────────────────────────────────────────────────┘  │
│                                                                  │
└──────────────────────────────────────────────────────────────────┘
```

---

## Prerequisites

Before configuring S3 Files, the following must already exist:

| Resource | Status | Created In |
|----------|--------|-----------|
| VPC with public + private subnets | Required | Step 2 (AWS Guide) |
| EC2 security group | Required | Step 3 (AWS Guide) |
| S3 claim-check bucket | Required | Step 4 (AWS Guide) |
| IAM role for EC2 (`agentcore-demo-test1-ec2-role`) | Required | Step 6 (AWS Guide) |
| EC2 instance running | Required | Step 7 (AWS Guide) |

---

## Step 1: Create the S3 Files File System

An S3 Files file system is the logical container that connects your S3 bucket to the NFS protocol.

```bash
#!/bin/bash
# 01_create_s3_files_system.sh

REGION="us-east-1"
PROJECT="agentcore-demo-test1"
ACCOUNT_ID=$(aws sts get-caller-identity --query Account --output text)

echo "=== Creating S3 Files File System ==="

# Create the S3 Files file system
FS_ARN=$(aws s3files create-file-system \
  --s3-bucket-arn "arn:aws:s3:::${PROJECT}-claimcheck-${ACCOUNT_ID}" \
  --region "${REGION}" \
  --tags Key=Project,Value="${PROJECT}" \
  --query 'fileSystemArn' --output text)

echo "S3 Files File System created: ${FS_ARN}"
echo "FS_ARN=${FS_ARN}" >> .vpc_env
```

**What this does:** Creates a logical NFS gateway between the S3 claim-check bucket and the VPC network.

**Wait for it to become available:**
```bash
aws s3files describe-file-system \
  --file-system-id "${FS_ARN}" \
  --query 'LifeCycleState'
# Wait until it returns "AVAILABLE" (may take 2-3 minutes)
```

---

## Step 2: Create an Access Point

An access point controls which VPC and subnets can mount the file system.

```bash
#!/bin/bash
# 02_create_access_point.sh

source .vpc_env

echo "=== Creating S3 Files Access Point ==="

# Create access point in the public subnet
AP_ARN=$(aws s3files create-access-point \
  --file-system-id "${FS_ARN}" \
  --subnet-id "${PUB_SUBNET}" \
  --vpc-id "${VPC_ID}" \
  --region "${REGION}" \
  --query 'AccessPointArn' --output text)

echo "Access Point created: ${AP_ARN}"
echo "AP_ARN=${AP_ARN}" >> .vpc_env
```

**What this does:** Creates an NFS mount target in your VPC's public subnet. The AgentCore Runtime will connect to this mount target.

**Verify mount target is available:**
```bash
aws s3files describe-mount-targets \
  --access-point-id "${AP_ARN}" \
  --query 'MountTargets[*].{SubnetId:SubnetId,LifeCycleState:LifeCycleState,IPAddress:IpAddress}'
```

You should see your subnet with `"LifeCycleState": "AVAILABLE"`.

---

## Step 3: Update the Security Group

S3 Files uses NFSv4.2 over TLS on port 2049. You must allow outbound traffic from your EC2 security group to the mount target.

```bash
#!/bin/bash
# 03_update_security_group.sh

source .vpc_env

echo "=== Updating Security Group for S3 Files (port 2049) ==="

# Get the mount target's security group
MT_SG=$(aws s3files describe-mount-target-security-groups \
  --mount-target-id "$(aws s3files describe-mount-targets \
    --access-point-id "${AP_ARN}" \
    --query 'MountTargets[0].MountTargetId' --output text)" \
  --query 'SecurityGroups[0]' --output text)

# Allow outbound port 2049 from EC2 SG to mount target SG
aws ec2 authorize-security-group-egress \
  --group-id "${EC2_SG}" \
  --protocol tcp \
  --port 2049 \
  --destination-group "${MT_SG}" \
  --description "S3 Files NFS access"

echo "Security group updated: EC2 SG can reach S3 Files on port 2049"
```

**Port reference:**

| Port | Protocol | Direction | Purpose |
|------|----------|-----------|---------|
| 2049 | TCP | EC2 SG → Mount Target SG | NFSv4.2 over TLS (S3 Files) |

---

## Step 4: Update the IAM Role

The EC2 IAM role needs additional permissions for S3 Files.

```bash
#!/bin/bash
# 04_update_iam_for_s3files.sh

source .vpc_env

echo "=== Adding S3 Files permissions to IAM Role ==="

# Get the file system resource ARN
FS_RESOURCE_ARN=$(aws s3files describe-file-system \
  --file-system-id "${FS_ARN}" \
  --query 'ResourceARN' --output text)

cat > /tmp/s3files-policy.json <<EOF
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Sid": "S3FilesAgentAccess",
      "Effect": "Allow",
      "Action": [
        "s3files:ClientMount",
        "s3files:ClientWrite",
        "s3files:GetAccessPoint"
      ],
      "Resource": "${FS_RESOURCE_ARN}",
      "Condition": {
        "StringEquals": {
          "s3files:AccessPointArn": "${AP_ARN}"
        }
      }
    }
  ]
}
EOF

# Add to the existing custom policy
POLICY_VERSION=$(aws iam create-policy-version \
  --policy-arn "${POLICY_ARN}" \
  --policy-document file:///tmp/s3files-policy.json \
  --set-as-default \
  --query 'PolicyVersion.VersionId' --output text)

echo "IAM policy updated with S3 Files permissions: ${POLICY_VERSION}"
```

**Required permissions explained:**

| Permission | What It Does |
|------------|-------------|
| `s3files:ClientMount` | Allows the EC2 instance to mount the S3 Files filesystem |
| `s3files:ClientWrite` | Allows write operations (create, modify, delete files) |
| `s3files:GetAccessPoint` | Allows discovering the mount target details |

---

## Step 5: Configure Agent Runtime with Filesystem Mount

Each agent needs a `filesystem-configurations.json` file that tells AgentCore Runtime to mount the S3 Files access point.

### 5.1 Create the Filesystem Configuration File

```json
{
  "filesystemConfigurations": [
    {
      "s3Files": {
        "fileSystemArn": "arn:aws:s3-files:us-east-1:123456789012:file-system/fs-abc123",
        "accessPointArn": "arn:aws:s3-files:us-east-1:123456789012:access-point/fsap-def456",
        "mountPath": "/mnt/agentcore/claim-checks",
        "readOnly": false
      }
    }
  ]
}
```

**Save this as:** `agents/agent_X_xxx/filesystem-configurations.json`

### 5.2 Deploy first (no filesystem), then attach via the control plane

```bash
# Step 1: deploy normally (no filesystem flag — that flag doesn't exist on the toolkit).
cd agents/agent_1_transcriber
agentcore configure -e main.py --protocol HTTP -r us-east-1
agentcore deploy
# Capture the resulting runtime ARN/ID.

# Step 2: attach the filesystem via the control plane.
RUNTIME_ID=$(aws bedrock-agentcore-control get-agent-runtime \
    --agent-runtime-name agent_1_transcriber \
    --query 'agentRuntimeId' --output text)

aws bedrock-agentcore-control update-agent-runtime \
    --agent-runtime-id "$RUNTIME_ID" \
    --filesystem-configurations "$(cat filesystem-configurations.json | jq '.filesystemConfigurations')"
```

**What happens:**
1. `agentcore configure` writes the toolkit's local config (no FS).
2. `agentcore deploy` builds + creates the runtime, no FS attached yet.
3. The control-plane `update-agent-runtime` call attaches the FS configuration and triggers a rolling refresh of the runtime.
4. Subsequent `invoke_agent_runtime` calls start MicroVMs with the filesystem mounted at the configured path (NFSv4.2 over TLS, port 2049).
5. Files written to the mount are synced to the backing S3 bucket transparently.

**Alternative (boto3, equivalent to the CLI):**

```python
import boto3, json
control = boto3.client("bedrock-agentcore-control", region_name="us-east-1")
runtime = control.get_agent_runtime(agentRuntimeName="agent_1_transcriber")
with open("filesystem-configurations.json") as f:
    fs_cfg = json.load(f)["filesystemConfigurations"]
control.update_agent_runtime(
    agentRuntimeId=runtime["agentRuntimeId"],
    filesystemConfigurations=fs_cfg,
)
```

### 5.3 Verify the Mount Inside the MicroVM

After deployment, the mount should be available. You can verify by:

```python
# Inside the agent's @app.entrypoint
def invoke(payload, context):
    import os
    mount_path = "/mnt/agentcore/claim-checks"

    # Check if mount exists
    if os.path.ismount(mount_path):
        print(f"S3 Files mounted at {mount_path}")
        # List files (synced from S3)
        files = os.listdir(mount_path)
        print(f"Files visible: {files}")
    else:
        print("WARNING: S3 Files mount not found!")
```

---

## Step 6: Update Agent Code to Use POSIX File Operations

### Before (boto3 S3 API)

```python
# agents/common/s3_claim_check.py (OLD)
import boto3
import json

def write_claim_check(payload: dict) -> dict:
    s3 = boto3.client("s3")
    key = f"claim-checks/{uuid.uuid4()}.json"
    s3.put_object(
        Bucket="agentcore-demo-test1-claimcheck-123456789",
        Key=key,
        Body=json.dumps(payload).encode(),
    )
    return {"_claim_check": True, "bucket": "...", "key": key}

def read_claim_check(ref: dict) -> dict:
    s3 = boto3.client("s3")
    response = s3.get_object(Bucket=ref["bucket"], Key=ref["key"])
    return json.loads(response["Body"].read())
```

### After (POSIX File Operations on S3 Files Mount)

```python
# agents/common/claim_check_io.py (NEW)
"""
Agent-side claim-check I/O via AgentCore Persistent Filesystems.
S3 Files is mounted at /mnt/agentcore/claim-checks.
Uses POSIX file operations — NOT boto3 S3 API.
"""
import json
import os
import uuid
from pathlib import Path

MOUNT = Path(os.environ.get(
    "CLAIM_CHECK_MOUNT",
    "/mnt/agentcore/claim-checks"
))


def write_claim_check(payload: dict) -> str:
    """Write payload to mounted S3 Files. Returns relative path."""
    file_name = f"claim-checks/{uuid.uuid4()}.json"
    file_path = MOUNT / file_name
    file_path.parent.mkdir(parents=True, exist_ok=True)

    with open(file_path, "w", encoding="utf-8") as f:
        json.dump(payload, f, ensure_ascii=False)

    return file_name


def read_claim_check(reference: dict) -> dict:
    """Read claim-check from mounted S3 Files."""
    key = reference.get("key", "")
    file_path = MOUNT / key if not key.startswith("/") else Path(key)

    with open(file_path, "r", encoding="utf-8") as f:
        return json.load(f)


def ensure_mount_available() -> None:
    """Verify S3 Files mount is accessible. Call at agent startup."""
    if not MOUNT.exists():
        raise RuntimeError(
            f"S3 Files mount not found at {MOUNT}. "
            "Re-run `aws bedrock-agentcore-control update-agent-runtime --filesystem-configurations '[...]'`."
        )
```

### Build Claim-Check Reference for Temporal

When an agent writes a claim-check, it must return a reference that Temporal (on EC2) can resolve via boto3:

```python
# agents/common/s3_files_reference.py
import os
from typing import Dict

S3_CLAIM_CHECK_BUCKET = os.environ.get(
    "S3_CLAIM_CHECK_BUCKET",
    "agentcore-demo-test1-claimcheck-123456789"
)


def build_claim_check_reference(
    file_path: str,
    original_size_bytes: int,
) -> Dict:
    """
    Build a claim-check reference dict that Temporal on EC2 can resolve.
    The EC2 side uses boto3; the agent side uses POSIX via S3 Files mount.
    Both sides read/write the same S3 bucket.
    """
    key = file_path if "/" in file_path else f"claim-checks/{file_path}"

    return {
        "_claim_check": True,
        "bucket": S3_CLAIM_CHECK_BUCKET,
        "key": key,
        "mount_path": "/mnt/agentcore/claim-checks",
        "original_size": original_size_bytes,
        "s3_uri": f"s3://{S3_CLAIM_CHECK_BUCKET}/{key}",
    }
```

---

## Complete Agent Integration Example

Here is how an agent uses S3 Files in practice:

```python
# agents/agent_2_drafter/agent.py
from bedrock_agentcore.runtime import BedrockAgentCoreApp
from common.claim_check_io import (
    ensure_mount_available,
    write_claim_check,
    read_claim_check,
)
from common.s3_files_reference import build_claim_check_reference
from common.payload_builder import build_complete_payload

app = BedrockAgentCoreApp()

@app.entrypoint
def invoke(payload: dict, context: dict) -> dict:
    session_state = context.get("session_state", {})

    # 1. Verify S3 Files mount is available (microVM only)
    ensure_mount_available()

    # 2. Read previous output (from Temporal via claim-check)
    previous = None
    if session_state.get("_claim_check"):
        previous_data = read_claim_check(session_state)
        previous = previous_data.get("previous_output")
    else:
        previous = session_state.get("previous_output")

    # 3. Do agent work...
    brd_markdown = generate_brd(previous)

    # 4. Build V5 payload
    v5_payload = build_complete_payload(
        status_code="COMPLETED",
        trace_id=session_state.get("trace_id"),
        workflow_id=session_state.get("workflow_id"),
        # ... other fields
    )

    # 5. If payload is large, write to S3 Files (POSIX)
    payload_size = len(json.dumps(v5_payload).encode())
    if payload_size > 256 * 1024:
        file_path = write_claim_check(v5_payload)
        claim_ref = build_claim_check_reference(
            file_path=file_path,
            original_size_bytes=payload_size,
        )
        return {
            "response": brd_markdown,
            "claim_reference": claim_ref,
        }

    # 6. Inline (small payload)
    return {
        "response": brd_markdown,
        "payload": v5_payload,
    }
```

---

## S3 Files Semantics Reference

| Feature | Behavior | Notes |
|---------|----------|-------|
| **Read/write** | Full POSIX support | `open`, `read`, `write`, `seek` |
| **Directories** | Supported | `mkdir`, `rmdir`, `listdir` |
| **File size** | Up to 48 TiB | Per-file limit |
| **Hard links** | Not supported | Use symlinks instead |
| **Atomic writes** | Close-to-open consistency | `close()` flushes to S3 |
| **Bidirectional sync** | S3 ↔ mount auto-sync | Changes in S3 appear in mount |
| **Custom metadata** | Not supported | Use filenames or content |
| **Encryption (transit)** | TLS | NFSv4.2 over TLS |
| **Encryption (rest)** | SSE-S3 | Bucket-level encryption |
| **Permissions** | 777 (full access) | Enforced at IAM level |

---

## Troubleshooting

| Problem | Cause | Solution |
|---------|-------|----------|
| Mount not found at `/mnt/agentcore/claim-checks` | Filesystem config not passed to `agentcore configure` | Re-run `aws bedrock-agentcore-control update-agent-runtime` with the FS config |
| Permission denied on write | IAM role missing `s3files:ClientWrite` | Add permission to `agentcore-demo-test1-ec2-policy` |
| Connection refused on port 2049 | Security group blocking outbound 2049 | Add egress rule from EC2 SG to mount target SG |
| Files not visible in S3 bucket | Sync delay | S3 Files uses close-to-open; close the file handle |
| `LifeCycleState: CREATING` for hours | VPC/subnet misconfiguration | Verify subnet AZ matches mount target AZ |
| Agent crashes on `ensure_mount_available()` | Deploy without filesystem config | Check `filesystem-configurations.json` was included |

---

## Quick Reference

| Task | Command |
|------|---------|
| Create S3 Files file system | `aws s3files create-file-system --s3-bucket-arn ...` |
| Create access point | `aws s3files create-access-point --file-system-id ... --subnet-id ...` |
| Check mount targets | `aws s3files describe-mount-targets --access-point-id ...` |
| Check file system status | `aws s3files describe-file-system --file-system-id ...` |
| Verify mount in microVM | `os.path.ismount("/mnt/agentcore/claim-checks")` |
| Attach FS to deployed agent | `aws bedrock-agentcore-control update-agent-runtime --agent-runtime-id <id> --filesystem-configurations file://fs.json` |
| View S3 Files docs | https://docs.aws.amazon.com/bedrock-agentcore/latest/devguide/runtime-filesystem-configurations.html |

---

*End of Persistent Filesystems Guide*
