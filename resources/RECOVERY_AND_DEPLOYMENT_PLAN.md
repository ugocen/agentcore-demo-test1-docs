# Recovery & Deployment Plan — AgentCore Demo Test 1

**Versiyon:** 1.0
**Tarih:** 2026-05-16
**Yazan:** Claude (Opus 4.7), durum tespiti + implementation plani
**Hedef Okuyucu:** Bir sonraki LLM (Claude / Gemini / GPT) veya gelistirici. Bu plani okuyup sifirdan calistirabilmelidir.

---

## 0. Bu Plan Ne Icin?

Demo proje (`agentcore-demo-test1`) tum kod tabani **kodlanmis ve hazir** durumda:
- Backend (FastAPI + Temporal + 3 agent + alembic + e2e testler) ✅
- Frontend (Next.js 14 + CopilotKit + AG-UI + tum komponentler) ✅
- Infra script'leri (00-06, AWS kaynaklarini provision ediyor) ✅
- AWS kaynaklari **bir kez provision edildi** (VPC, RDS, S3, EC2, IAM, CloudWatch, AgentCore agents)

**Ancak** projeyi AWS uzerinde **calistiramadik**. Uygulama hic kez ayaga kalkmadi. Sonra maliyet yaratmamak icin AWS servisleri durduruldu.

Bu plan, **AWS uzerinde demoyu canli tutmak** icin gerekli tum adimlari sirayla, kontrol noktalariyla aciklar.

---

## 1. Mevcut Durum (2026-05-16 itibariyle)

### 1.1 Kod Reposunun Durumu

| Repo | Yol | Durum |
|------|-----|-------|
| Backend | `/Users/ugurgocen/projects/agentcore-demo-test1/agentcore-demo-test1-backend/` | ✅ Hazir, son commit 2026-05-15 |
| Frontend | `/Users/ugurgocen/projects/agentcore-demo-test1/agentcore-demo-test1-frontend/` | ✅ Hazir, son commit 2026-05-15 |
| Infra | `/Users/ugurgocen/projects/agentcore-demo-test1/agentcore-demo-test1-infra/` | ✅ Hazir, son commit 2026-05-15 |
| Docs | `/Users/ugurgocen/projects/agentcore-demo-test1/agentcore-demo-test1-docs/` | ✅ Hazir |

**Not:** EC2 user-data script'i (`05_launch_ec2.sh`) repolari `git clone git@github.com:ugocen/...` ile aliyor. Eger lokal commit'ler GitHub'a push edilmediyse, EC2 bos repolarla baslar. **Mutlaka kontrol et.**

### 1.2 AWS Kaynaklari (Provisioned)

`/Users/ugurgocen/projects/agentcore-demo-test1/agentcore-demo-test1-infra/scripts/.*_env` dosyalarinda kayitli:

```
Account ID:           122524101917
Region:               us-east-1
VPC:                  vpc-081e9d2971d50f792
Subnets:              subnet-06d04fe30e2a9581a, subnet-09741eb87a0f43a6f
Security Groups:      EC2 sg-040c5e4c26e1c70a5, RDS sg-0a0f9ecb0de68fd62
IGW:                  igw-08405d61be6e7edf7

S3 Buckets:
  - agentcore-demo-test1-audio-uploads-122524101917
  - agentcore-demo-test1-artifacts-122524101917
  - agentcore-demo-test1-claimcheck-122524101917
  - agentcore-demo-test1-code-122524101917

RDS Endpoint:         agentcore-demo-test1-db.cqbouecye6zc.us-east-1.rds.amazonaws.com
RDS Databases:        temporal, temporal_visibility, agentcore_demo_test1
RDS Users:            temporal_user / LugnCQa8t1rqbsCdBSPg
                      app_user / 2s92RGFC1Na6UQOp9IPa

EC2 Instance:         i-0974a815b81d0c028 (t3.large, AL2023)
EC2 Public IP (eski): 34.200.228.155  [DIKKAT: instance restart edildiginde IP DEGISIR cunku Elastic IP atanmamis]

IAM Roles:            agentcore-demo-test1-ec2-role
                      agentcore-demo-test1-agentcore-role

AgentCore Runtimes:
  - agent_1_transcriber: arn:aws:bedrock-agentcore:us-east-1:122524101917:runtime/agent_1_transcriber-mjSXuCC153
  - agent_2_drafter:     arn:aws:bedrock-agentcore:us-east-1:122524101917:runtime/agent_2_drafter-XIh3stAQFo
  - agent_3_reviewer:    arn:aws:bedrock-agentcore:us-east-1:122524101917:runtime/agent_3_reviewer-BFKEb76k4q

AgentCore Memories:
  - agent_1_transcriber_mem-yQbtX9FEk6
  - agent_2_drafter_mem-yCt33i70tT
  - agent_3_reviewer_mem-kNI4Xg2NOw

CloudWatch Log Groups: /aws/agentcore, /aws/fastapi, /aws/temporal, /aws/temporal-worker
```

### 1.3 Tahmini AWS Durumu (Stopped Edildikten Sonra)

| Kaynak | Maliyet | Tahmini Mevcut Durum | Aksiyon |
|--------|---------|----------------------|---------|
| VPC, SG, IGW, Subnets | $0 (free) | aktif | dokunma |
| S3 Buckets | <$1/ay (icindeki veriye gore) | aktif | dokunma, veriler korunmali |
| IAM Roles | $0 (free) | aktif | dokunma |
| CloudWatch Log Groups | <$1/ay (7gun retention) | aktif | dokunma |
| RDS db.t4g.micro | ~$15/ay calistiginda | **muhtemelen stopped** (manuel stop edildiyse 7 gun sonra otomatik basliyor) | start edilecek |
| EC2 t3.large | ~$60/ay calistiginda | **muhtemelen stopped** | start edilecek |
| AgentCore Runtimes | invocation basina | **BILINMIYOR — DOGRULA** | yasiyor mu kontrol et, gerekirse `agentcore deploy` ile tekrar yarat |

### 1.4 Eksiklikler (Daha Once Calistiramadik Cunku...)

| # | Sorun | Kanit |
|---|-------|-------|
| G1 | Backend repo'da `.env` yok | `ls agentcore-demo-test1-backend/.env*` → bos |
| G2 | Frontend'de `.env.local` yok | sadece `.env.local.example` mevcut, `RUNTIME_ID_*`/`ACCOUNT_ID` placeholder dolu |
| G3 | EC2 deployment otomasyonu eksik | `user-data` sadece Docker + git clone yapiyor, `docker compose up` calismiyor; ECR/image build yok |
| G4 | AgentCore runtime'lari hayatta mi belirsiz | maliyet kapatma sirasinda `agentcore destroy` cagrilmis olabilir |
| G5 | RDS sifreleri `.rds_env` icinde plain-text (Secrets Manager'a aktarilmamis) | yine de demo icin yeterli, kullanilabilir |
| G6 | EC2'ye Elastic IP atanmamis — her restart'ta IP degisir | frontend ve env'lerde IP hard-coded olmamali |

### 1.5 Eksik Olmayan Seyler

- `docker-compose.yml` (backend reposunda, 6 servisli) tamamen yapilandirilmis: temporal-server, temporal-ui, fastapi, temporal-worker, nextjs-frontend, adot-collector
- `Dockerfile` her uc katmanda da var
- Alembic migration'i `001_initial.py` mevcut, `evidence_packs` + `chat_messages` tablolarini olusturuyor
- `tests/e2e/test_T01..T11.py` 11 sirasal smoke test dosyasi var
- ADOT config `agentcore-demo-test1-infra/adot/otel-collector-config.yaml` mevcut

---

## 2. Stratejik Karar: Hangi Yontemle Calistiriyoruz?

Iki secenek var. **OPSIYON A oneriliyor** cunku PCD'de hedef bu.

### OPSIYON A — AWS uzerinde calistir (HEDEF) ⭐

1. AWS servisleri yeniden ayaga kaldir
2. AgentCore runtime'lari dogrula veya tekrar deploy et
3. EC2'ye SSH bagla, `.env` olustur, `docker compose up`
4. Frontend de EC2 uzerinde docker-compose icinde calisir
5. Browser'dan `http://<EC2_PUBLIC_IP>:3000` adresinden eris

**Maliyet:** EC2 calisirken ~$60/ay (~$2/gun), RDS ~$15/ay, AgentCore invocation basina sent'ler.

### OPSIYON B — Lokal calistir (debug icin) ⚙️

1. RDS'i AWS'de tut, lokal Docker'dan eris (RDS publicly accessible olmali — degil)
2. Veya tum stack'i lokalde calistir (lokal postgres icin docker-compose.local.yml gerekir — yok)
3. Frontend lokal, backend lokal, agentleri AWS AgentCore'a invocate et

Opsiyon B daha **kompleks**, cunku:
- RDS'in `PubliclyAccessible=true` olmasi gerekir (su an degil)
- Lokal Postgres icin Temporal sema kurulumu gerekir (zor)
- Sadece debug icin uygundur

**Onerilen yaklasim: Opsiyon A, asagidaki adimlar.**

---

## 3. Detayli Implementation Plani (Opsiyon A)

### Faz R0: Hazirlik & On Kontroller (15 dk)

#### R0.1 AWS CLI Auth
```bash
aws sso login   # veya
aws configure   # access key + secret
aws sts get-caller-identity
# Beklenen: Account=122524101917, us-east-1
```

#### R0.2 SSH Key Kontrolu
```bash
ls -l ~/.ssh/agentcore-demo-test1*   # veya kullandigin key ne ise
# EC2 olustururken bir key-pair atanmis olmali. .ec2_env'de yok — `aws ec2 describe-instances --instance-ids i-0974a815b81d0c028 --query 'Reservations[0].Instances[0].KeyName'` ile ogren.
```

#### R0.3 GitHub Reposu Push Dogrulama
Onemli: EC2 user-data git clone ile cekiyor. Lokal commitler push edilmemisse EC2 bos repolarla baslar.
```bash
cd /Users/ugurgocen/projects/agentcore-demo-test1/agentcore-demo-test1-backend
git status
git log --oneline | head -5
git remote -v
git push origin main   # gerekirse

# Ayni seyi frontend, infra, docs icin de yap
```
Eger GitHub'a push edemiyorsan, **alternatif:** EC2'ye scp/rsync ile dosyalari elle gonder (R3.2 adiminda).

#### R0.4 Mevcut Maliyet Durumu
```bash
aws ce get-cost-and-usage --time-period Start=2026-05-01,End=2026-05-16 \
  --granularity DAILY --metrics UnblendedCost \
  --group-by Type=DIMENSION,Key=SERVICE | jq '.ResultsByTime[-1]'
# Beklenen: Bedrock + S3 birikiyor, EC2/RDS sifir (cunku stopped).
```

---

### Faz R1: AWS Kaynaklarini Yeniden Baslat (20 dk)

#### R1.1 RDS Baslat (3-5 dk)
```bash
aws rds describe-db-instances \
  --db-instance-identifier agentcore-demo-test1-db \
  --query 'DBInstances[0].DBInstanceStatus' --output text
# Beklenen: "stopped" — start et:

aws rds start-db-instance --db-instance-identifier agentcore-demo-test1-db
# Bekle "available" olana kadar:
aws rds wait db-instance-available --db-instance-identifier agentcore-demo-test1-db
```
**Gate R1.1:** Status `available`, endpoint yine `agentcore-demo-test1-db.cqbouecye6zc.us-east-1.rds.amazonaws.com`.

#### R1.2 EC2 Baslat (2-3 dk)
```bash
aws ec2 describe-instances --instance-ids i-0974a815b81d0c028 \
  --query 'Reservations[0].Instances[0].State.Name' --output text
# Beklenen: "stopped" — start et:

aws ec2 start-instances --instance-ids i-0974a815b81d0c028
aws ec2 wait instance-running --instance-ids i-0974a815b81d0c028

# Yeni public IP'yi al (Elastic IP atanmadigi icin IP DEGISMIS olabilir):
NEW_PUBLIC_IP=$(aws ec2 describe-instances --instance-ids i-0974a815b81d0c028 \
  --query 'Reservations[0].Instances[0].PublicIpAddress' --output text)
echo "Yeni IP: $NEW_PUBLIC_IP"

# .ec2_env dosyasini guncelle:
echo "INSTANCE_ID=i-0974a815b81d0c028" > /Users/ugurgocen/projects/agentcore-demo-test1/agentcore-demo-test1-infra/scripts/.ec2_env
echo "PUBLIC_IP=$NEW_PUBLIC_IP" >> /Users/ugurgocen/projects/agentcore-demo-test1/agentcore-demo-test1-infra/scripts/.ec2_env
```
**Gate R1.2:** Instance `running`, yeni IP `.ec2_env`'e yazildi.

> **OPSIYONEL (onerilir):** Elastic IP atayin ki bir daha IP degismez:
> ```bash
> ALLOC_ID=$(aws ec2 allocate-address --domain vpc --query AllocationId --output text)
> aws ec2 associate-address --instance-id i-0974a815b81d0c028 --allocation-id $ALLOC_ID
> ```

#### R1.3 AgentCore Runtime Durumu Kontrolu
```bash
aws bedrock-agentcore-control list-agent-runtimes --region us-east-1 \
  --query 'agentRuntimes[?starts_with(agentRuntimeName, `agent_`)].{name:agentRuntimeName,status:status,arn:agentRuntimeArn}'
```
3 sonuc bekleniyor — biri biri silinmisse R2'de yeniden deploy edilecek.

**Gate R1.3:** En az 3 runtime listede, `status` = `READY` veya `CREATING`.

---

### Faz R2: AgentCore Agent'larini Dogrula veya Yeniden Deploy (varsa atla — 0 dk, yoksa ~20 dk)

#### R2.1 Hizli Health Check (her agent icin)
```bash
cd /Users/ugurgocen/projects/agentcore-demo-test1/agentcore-demo-test1-backend
source .venv/bin/activate

# .venv yoksa olustur:
# uv venv .venv && source .venv/bin/activate && uv pip install -r requirements.txt boto3 bedrock-agentcore-starter-toolkit

aws bedrock-agentcore invoke-agent-runtime \
  --agent-runtime-arn arn:aws:bedrock-agentcore:us-east-1:122524101917:runtime/agent_1_transcriber-mjSXuCC153 \
  --runtime-session-id health-check-$(date +%s) \
  --qualifier DEFAULT \
  --payload '{"trace_id":"00-aaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaa-bbbbbbbbbbbbbbbb-01","workflow_template_id":"tmpl","workflow_id":"wf-hc","agent_run_id":"ar-hc","step_id":"s1","step_sequence":1,"agent_id":"agent_1_transcriber","agent_version":"1.0.0","requested_by":"system","execution_context":{},"task":{"type":"health_check"},"inputs":{},"execution_options":{"dry_run":true}}' \
  /tmp/agent1_response.json
cat /tmp/agent1_response.json
```
**Pass kriteri:** 8 block'lu JSON donuyor (status, resources, timing, financial, artifacts, quality, tool_calls, risk). Status'te `trace_id` echo edilmis.

> Diger 2 agent icin agent_runtime_arn'i degistirip tekrarla.

#### R2.2 Eger Runtime Silinmisse: Yeniden Deploy
```bash
cd /Users/ugurgocen/projects/agentcore-demo-test1/agentcore-demo-test1-backend/agents/agent_1_transcriber
source ../../.venv/bin/activate

# .bedrock_agentcore.yaml zaten var, agent_arn yenilenecek:
agentcore deploy --auto-update-on-conflict --env S3_BUCKET_ARTIFACTS=agentcore-demo-test1-artifacts-122524101917

# Tum agent'lar icin:
cd ../agent_2_drafter && agentcore deploy --auto-update-on-conflict --env S3_BUCKET_ARTIFACTS=agentcore-demo-test1-artifacts-122524101917
cd ../agent_3_reviewer && agentcore deploy --auto-update-on-conflict --env S3_BUCKET_ARTIFACTS=agentcore-demo-test1-artifacts-122524101917

# Yeni ARN'lari topla:
for d in agent_1_transcriber agent_2_drafter agent_3_reviewer; do
  echo -n "$d: "; grep "agent_arn:" ../$d/.bedrock_agentcore.yaml | head -1
done
```
**Cikti:** Yeni 3 ARN. Bunlari R3.2'de `.env` dosyasina yazacaksin.

**Gate R2:** 3 agent runtime `READY`, invoke-agent-runtime ile 8-block payload donuyor.

---

### Faz R3: EC2 Uzerinde Stack'i Hazirla ve Calistir (30 dk)

#### R3.1 EC2'ye SSH
```bash
ssh -i ~/.ssh/<key-name>.pem ec2-user@$NEW_PUBLIC_IP

# Icerideyken:
sudo cat /opt/setup-complete.flag   # var mi? user-data calismis mi?
docker --version
docker compose version  # plugin
ls /opt/   # repolari klonlamis mi?
```

#### R3.2 Repolari Hazirla (2 alternatif)

**A) git pull (eger reolar GitHub'da)**:
```bash
cd /opt/agentcore-demo-test1-backend && git pull
cd /opt/agentcore-demo-test1-frontend && git pull
cd /opt/agentcore-demo-test1-infra && git pull
cd /opt/agentcore-demo-test1-docs && git pull
```

**B) Lokal'den scp (GitHub'a push yapmadiysan, lokal makinende):**
```bash
# Lokal makinede:
cd /Users/ugurgocen/projects
rsync -avz --exclude='.venv' --exclude='node_modules' --exclude='.next' --exclude='__pycache__' --exclude='.git' \
  agentcore-demo-test1/ ec2-user@$NEW_PUBLIC_IP:/opt/agentcore-demo-test1/
# Sonra EC2'de:
ssh ec2-user@$NEW_PUBLIC_IP
sudo mv /opt/agentcore-demo-test1/agentcore-demo-test1-* /opt/
```

#### R3.3 Backend .env Olustur (EC2'de)
```bash
cd /opt/agentcore-demo-test1-backend

cat > .env << 'EOF'
# === RDS ===
RDS_HOST=agentcore-demo-test1-db.cqbouecye6zc.us-east-1.rds.amazonaws.com
RDS_PORT=5432
RDS_DB_APP=agentcore_demo_test1
RDS_USER_APP=app_user
RDS_PASSWORD_APP=2s92RGFC1Na6UQOp9IPa
RDS_DB_TEMPORAL=temporal
RDS_DB_TEMPORAL_VISIBILITY=temporal_visibility
RDS_USER_TEMPORAL=temporal_user
RDS_PASSWORD_TEMPORAL=LugnCQa8t1rqbsCdBSPg

# DB_* (docker-compose'da kullaniliyor)
DB_HOST=agentcore-demo-test1-db.cqbouecye6zc.us-east-1.rds.amazonaws.com
DB_PORT=5432
DB_NAME=agentcore_demo_test1
DB_USER=app_user
DB_PASSWORD=2s92RGFC1Na6UQOp9IPa

# === S3 ===
S3_BUCKET_AUDIO=agentcore-demo-test1-audio-uploads-122524101917
S3_BUCKET_ARTIFACTS=agentcore-demo-test1-artifacts-122524101917
S3_BUCKET_CLAIMCHECK=agentcore-demo-test1-claimcheck-122524101917
AUDIO_BUCKET=agentcore-demo-test1-audio-uploads-122524101917
ARTIFACTS_BUCKET=agentcore-demo-test1-artifacts-122524101917
CLAIMCHECK_BUCKET=agentcore-demo-test1-claimcheck-122524101917

# === AgentCore (R2'den gelen ARN'lari kullan; degistiyse guncelle) ===
AGENTCORE_ARN_TRANSCRIBER=arn:aws:bedrock-agentcore:us-east-1:122524101917:runtime/agent_1_transcriber-mjSXuCC153
AGENTCORE_ARN_DRAFTER=arn:aws:bedrock-agentcore:us-east-1:122524101917:runtime/agent_2_drafter-XIh3stAQFo
AGENTCORE_ARN_REVIEWER=arn:aws:bedrock-agentcore:us-east-1:122524101917:runtime/agent_3_reviewer-BFKEb76k4q

# === AWS ===
AWS_REGION=us-east-1

# === Temporal Server (docker-compose'da heredeki ad) ===
TEMPORAL_HOST=temporal-server
TEMPORAL_PORT=7233
TEMPORAL_TASK_QUEUE=brd-from-audio-tq

# === Diger ===
WORKFLOW_STREAMS_PATTERN=A
SELF_CORRECTION_CAP=3
HITL_TIMEOUT=900
WORKFLOW_TIMEOUT=3600
EOF
chmod 600 .env
```

#### R3.4 Frontend .env.local Olustur (EC2'de)
```bash
cd /opt/agentcore-demo-test1-frontend

# AgentCore runtime URL'leri: bedrock-agentcore endpoint formati
ACCOUNT_ID=122524101917
RUNTIME_1=agent_1_transcriber-mjSXuCC153
RUNTIME_2=agent_2_drafter-XIh3stAQFo
RUNTIME_3=agent_3_reviewer-BFKEb76k4q

cat > .env.local << EOF
NEXT_PUBLIC_BACKEND_URL=/api/v1
NEXT_PUBLIC_COPILOT_RUNTIME_URL=/api/copilotkit
AGENT_1_RUNTIME_URL=https://bedrock-agentcore.us-east-1.amazonaws.com/runtimes/${RUNTIME_1}/invocations?accountId=${ACCOUNT_ID}&qualifier=DEFAULT
AGENT_2_RUNTIME_URL=https://bedrock-agentcore.us-east-1.amazonaws.com/runtimes/${RUNTIME_2}/invocations?accountId=${ACCOUNT_ID}&qualifier=DEFAULT
AGENT_3_RUNTIME_URL=https://bedrock-agentcore.us-east-1.amazonaws.com/runtimes/${RUNTIME_3}/invocations?accountId=${ACCOUNT_ID}&qualifier=DEFAULT
NODE_ENV=production
EOF
chmod 600 .env.local
```

#### R3.5 ADOT Config'i Yerlestir
`docker-compose.yml`'de adot-collector servisi `../agentcore-demo-test1-infra/adot/otel-collector-config.yaml` mount ediyor. EC2'de bu yolun var oldugundan emin ol:
```bash
ls /opt/agentcore-demo-test1-infra/adot/otel-collector-config.yaml
# Yoksa scp ile yukle veya git pull
```

#### R3.6 Alembic Migration (RDS uzerinde tablolari olustur)
```bash
cd /opt/agentcore-demo-test1-backend
# Hizli yontem: gecici container'da
docker run --rm --env-file .env -v $(pwd):/app -w /app python:3.11-slim bash -c "
  apt-get update -qq && apt-get install -y -qq libpq-dev gcc
  pip install -q -r requirements.txt
  alembic upgrade head
"
# Veya psql ile baglanip kontrol et:
PGPASSWORD=2s92RGFC1Na6UQOp9IPa psql -h agentcore-demo-test1-db.cqbouecye6zc.us-east-1.rds.amazonaws.com \
  -U app_user -d agentcore_demo_test1 -c "\dt"
# Beklenen: alembic_version, chat_messages, evidence_packs
```

#### R3.7 Docker Compose Up (esas calistirma)
```bash
cd /opt/agentcore-demo-test1-backend
docker compose --env-file .env up -d --build

# Loglari izle:
docker compose logs -f
# 6 servis: temporal-server, temporal-ui, fastapi, temporal-worker, nextjs-frontend, adot-collector
```

**Cikabilecek sorunlar:**
- Temporal sema yok hatasi → `temporal-server` auto-setup imaji ilk acilista temporal/temporal_visibility DB'lerini kuruyor, RDS'e baglanmaya calisiyor. RDS SG'de EC2 SG'den 5432 izninin oldugundan emin ol.
- Frontend build hatasi → `NEXT_PUBLIC_BACKEND_URL` ve `NEXT_PUBLIC_COPILOT_RUNTIME_URL` `docker-compose.yml`'deki build-args'ta var (`/api/v1` ve `/api/copilotkit`). Sorun yok.

#### R3.8 Health Check
```bash
# EC2 lokali uzerinden:
curl -s http://localhost:8000/health | jq
curl -s http://localhost:3000 | head -20
curl -s http://localhost:8081  # Temporal UI
curl -s http://localhost:4317  # ADOT gRPC (port acik mi)

# Browser uzerinden (lokal makinenden):
# http://$NEW_PUBLIC_IP:3000      → Frontend
# http://$NEW_PUBLIC_IP:8000/docs → FastAPI Swagger
# http://$NEW_PUBLIC_IP:8081      → Temporal UI
```

**Gate R3:** 6 docker servis `running`, `/health` 200, Temporal UI'da default namespace gorunuyor, frontend acilirken hata yok.

---

### Faz R4: End-to-End Smoke Test (15 dk)

#### R4.1 EC2'de Test Suite'i Calistir
```bash
cd /opt/agentcore-demo-test1-backend
chmod +x scripts/e2e_test.sh
# uv yoksa kur:
which uv || (curl -LsSf https://astral.sh/uv/install.sh | sh)

bash scripts/e2e_test.sh
```
11 test sirasiyla calisir; cikti `tests/e2e/e2e-<epoch>.log`'a yazilir.

#### R4.2 Browser ile Manuel Test
1. `http://$NEW_PUBLIC_IP:3000` ac
2. Sol panelden bir workflow olustur (audio file yukle veya mic kayit)
3. CopilotKit sidebar agentlardan akan event'leri gostermeli
4. HITL clarification cikarsa cevapla, approval gate'inde onayla
5. Sonuc workspace canvas'inda gozukmeli

#### R4.3 CloudWatch Dogrulama
```bash
aws logs filter-log-events --region us-east-1 \
  --log-group-name /aws/agentcore \
  --filter-pattern '{ $.demo.workflow_id = "*" }' \
  --limit 10 | jq '.events[].message' | head -5
```
`demo.*` attribute'lari her event'te gorunuyor olmali (Phase 2'nin output'u).

**Gate R4:** 11/11 e2e test PASS, manuel demo akisi tamamlandi, CloudWatch'ta demo.* loglari mevcut.

---

### Faz R5: Maliyet Yonetimi ve Cleanup (opsiyonel)

Demo bittikten sonra tekrar durdurmak icin:
```bash
# EC2 stop (boyle yapilirsa Elastic IP atadiysan IP korunur):
aws ec2 stop-instances --instance-ids i-0974a815b81d0c028
# RDS stop (7 gun otomatik baslar):
aws rds stop-db-instance --db-instance-identifier agentcore-demo-test1-db
# AgentCore: invocation basina ucret, idle iken ucret YOK. Destroy edilmesi sart degil.
# Ama tamamen silmek istersen:
# cd agents/agent_1_transcriber && agentcore destroy
```

**Tum kaynaklari topu temizlik (geri donus YOK):**
- `agentcore-demo-test1-infra/scripts/99_teardown.sh` yazilmali (henuz yok). Sirayla:
  1. agentcore destroy x3
  2. EC2 terminate
  3. RDS delete (SkipFinalSnapshot=true)
  4. S3 buckets bos + delete
  5. IAM roles detach + delete
  6. SG/Subnet/IGW/VPC delete
  7. CloudWatch log groups delete

---

## 4. Sik Karsilasilan Sorunlar & Cozumler

| Belirti | Olasi Sebep | Cozum |
|---------|-------------|-------|
| `docker compose up` → fastapi `psycopg2.OperationalError` | RDS SG'de EC2 SG'den 5432 izni yok | `aws ec2 authorize-security-group-ingress --group-id sg-0a0f9ecb0de68fd62 --protocol tcp --port 5432 --source-group sg-040c5e4c26e1c70a5` |
| Temporal server `failed to connect` | RDS endpoint'i `temporal-server`'in environment'inde yanlis | docker-compose.yml'i kontrol et, `${RDS_HOST}` `.env`'den geliyor mu |
| Frontend `/api/v1/...` 502 | Next.js rewrites `fastapi:8000` icin proxy; FastAPI container ayakta mi | `docker compose ps fastapi`, loglari incele |
| Agent invoke `AccessDeniedException` | EC2 IAM rolu `bedrock-agentcore:InvokeAgentRuntime` izni yok | `ec2-permissions-policy.json` mevcut, role'e attach edildigini dogrula |
| Frontend AGENT_X_RUNTIME_URL `403` | bedrock-agentcore endpoint'i SigV4 imza ister; browser SigV4 yapamaz | bu URL'ler **backend'den** invoke ediliyor (CopilotKit route.ts), frontend direkt cagirmiyor. dogru tasarim. |
| AgentCore deploy `ResourceConflictException` | Ayni agent_id zaten var | `agentcore deploy --auto-update-on-conflict` |
| Tests T9 (CloudWatch) FAIL | ADOT collector calismiyor veya OTLP endpoint yanlis | `docker compose logs adot-collector`, IAM rolu cloudwatch:PutLogEvents izinli mi |
| EC2 user-data tamamlanmamis | `/var/log/user-data.log` kontrol et, hata varsa elle dogrula | `sudo bash -x /var/lib/cloud/instance/user-data.txt` |

---

## 5. Kabul Kriterleri (Definition of Done)

Bu proje "AWS'de canli" sayilmasi icin:

- [ ] `aws ec2 describe-instances ... i-0974a815b81d0c028 ...State.Name == running`
- [ ] `aws rds describe-db-instances ... agentcore-demo-test1-db ...DBInstanceStatus == available`
- [ ] `aws bedrock-agentcore-control list-agent-runtimes` 3 agent listede, status `READY`
- [ ] EC2'de `docker compose ps` 6 servisin hepsi `Up`
- [ ] `curl http://$EC2_IP:8000/health` 200
- [ ] `curl http://$EC2_IP:3000` 200 (Next.js sayfasi)
- [ ] `bash scripts/e2e_test.sh` 11/11 PASS
- [ ] Browser'dan workflow baslatip onaylama tamamlanabiliyor
- [ ] CloudWatch `/aws/agentcore` log grubunda `demo.*` attribute'lu event'ler var
- [ ] RDS'te `evidence_packs` tablosunda en az 1 satir (`outcome = APPROVED`)

---

## 6. Bir Sonraki LLM Icin Onemli Notlar

### 6.1 Ben Bu Plana Nereye Kadar Geldim?
**Hicbir Yere — sadece tespit + plan yazdim.** Implementasyon adimlarinin **hicbirini** uygulamadim. Sen bastan basla.

### 6.2 Hangi Dosyalar Var
- Backend kodu: `/Users/ugurgocen/projects/agentcore-demo-test1/agentcore-demo-test1-backend/` (eksiksiz)
- Frontend kodu: `/Users/ugurgocen/projects/agentcore-demo-test1/agentcore-demo-test1-frontend/` (eksiksiz)
- Infra script'leri: `/Users/ugurgocen/projects/agentcore-demo-test1/agentcore-demo-test1-infra/scripts/` (provision script'leri var, deploy yok)
- AWS state: `agentcore-demo-test1-infra/scripts/.{vpc,iam,s3,rds,ec2,agentcore}_env` (gercek ID'ler kayitli)
- AgentCore config: `agentcore-demo-test1-backend/agents/agent_{1,2,3}_*/.bedrock_agentcore.yaml` (gercek ARN'lar kayitli)

### 6.3 Ne Olduguna Suphem Var
- AgentCore runtime'lar yasiyor mu? **Test et: `aws bedrock-agentcore-control list-agent-runtimes`**
- GitHub'a push edildi mi? **Test et: lokal `git status` ve `git log @{u}..HEAD`**
- RDS gercekten "stopped" mi yoksa "terminated" mi? **Test et: `aws rds describe-db-instances`**
- AWS hesabinda billing limit asilmis olabilir mi? **Test et: AWS Billing console**

### 6.4 Eger Bir Sey Bozulmussa
- **AgentCore destroyed:** R2.2 ile yeniden deploy
- **RDS terminated:** Sifirdan provision et: `bash 03_create_rds.sh` (sema kaybi olur, alembic upgrade tekrar gerekir)
- **EC2 terminated:** `bash 05_launch_ec2.sh` (yeni instance, IP degisir, user-data tekrar calisir)
- **S3 buckets silinmis:** `bash 02_create_s3_buckets.sh` (icindeki veriler kayip)
- **VPC silinmis:** `bash 00_create_vpc.sh && bash 01_create_security_groups.sh && bash 04_create_iam_roles.sh` — ama bu zincirleme tum AWS ID'leri degistirir, `.env` dosyalarinin tamamini yeniden olusturmak gerekir.

### 6.5 Hangi Docs'larin Otoriter Oldugu
Onceligi olan dokumanlar (tutarsizlik halinde bunlara guven):
1. `Deliverable_0_PROJECT_CONTEXT.md` (PCD) — tum kararlar
2. `PHASE_PLAN.md` — faz tablosu (Phase 0-7)
3. `Deliverable_6_AWS_Operations_Guide.md` — AWS deploy detayi
4. `Prompt_01_AgentCore_Deployment.md` — agentcore configure/deploy komutlari
5. `Prompt_06_E2E_Integration.md` — test plani T1-T11

Bu dosya (RECOVERY_AND_DEPLOYMENT_PLAN.md), yukaridaki resmi planlardan **operatif farkliliklarin tespiti** + **mevcut state'ten geri donus** icin yazilmistir. Resmi planlar "tertemiz baslangic"a gore yazilmis; bu plan "yarim kalmis bir kuruluma" gore.

### 6.6 Iletisim Diliyle Ilgili Not
Kullanici Turkce konusuyor. Yanitlarini Turkce yaz (kod yorumlari + commit mesajlari Ingilizce kalir; bkz. DEMO_GELISTIRME_KILAVUZU.md `KURAL` satiri).

### 6.7 Riskler & Onlemler
- **EC2'ye Elastic IP atanmamis** → her restart'ta IP degisir, `.env.local`'lerini yeniden uretmek gerek. R1.2 sonunda Elastic IP atamayi onerdim.
- **RDS sifreleri plaintext** → demo icin yeterli ama production'a tasimadan once Secrets Manager'a aktar.
- **`.env`'ler `.gitignore`'da** → guvenli, ama backup almayi unutma; `.rds_env`/`.agentcore_env` dosyalari guncel oldugu icin oradan tekrar uretilebilir.
- **`docker compose --env-file .env up`** komutu Compose v2'de `--env-file`'i sadece variable substitution icin kullanir, container'a env vermez. docker-compose.yml zaten `${RDS_HOST}` gibi referanslarla `.env`'den okuyor; yine de container icine `--env-file` ile env vermek istersen `docker-compose.override.yml` ile servis bazinda `env_file:` ekle.

---

## 7. Tahmini Sure & Maliyet

| Faz | Sure | Maliyet (USD) |
|-----|------|---------------|
| R0 Hazirlik | 15 dk | $0 |
| R1 Servisleri baslat | 20 dk (cogu RDS bekleme) | $0 (saatlik tetiklenmedi) |
| R2 Agent dogrula | 5-30 dk | $0-1 (invoke ucretleri) |
| R3 EC2 stack | 30 dk | $0.10 (EC2 0.5 saat) |
| R4 E2E test | 15 dk | $0.50 (agent invocations + Bedrock LLM cagrisi) |
| **Toplam ilk demo** | **~1.5 saat** | **<$2** |
| Sonrasi (gunluk acik tutarsan) | — | **~$2.5/gun** (EC2+RDS) |

---

**SON.** Iyi sanslar bir sonraki LLM'e. Patladigi yerde bu dosyaya geri don.
