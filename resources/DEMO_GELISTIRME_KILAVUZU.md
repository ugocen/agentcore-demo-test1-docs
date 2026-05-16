# AgentCore demo test 1 — Gelistirme Kilavuzu (Day 0)

**Versiyon:** V5 (Final)  
**Tarih:** 2026-05-12  
**On kosul:** Hicbir sey kurulu degil. Sadece bir AWS hesabi ve bu kilavuz.

---

## 0. Bu Kilavuz Ne Anlatiyor?

Bu proje 3 yapay zeka ajaninin (agent) calistigi bir demo uygulamadir:
1. **Agent 1** ses kaydini metne cevirir (Transcriber)
2. **Agent 2** bu metni BRD (Business Requirements Document) formatinda yazar (Drafter)  
3. **Agent 3** yazilan BRD'yi inceler, hata varsa duzeltir (Reviewer)

Tum bu surec sizin izleyip yonlendirebileceginiz bir web arayuzu uzerinden calisir.

**Teknolojiler:** AWS Bedrock AgentCore (agent host), Temporal (is akisi), FastAPI (API), Next.js (web arayuz), OpenTelemetry (monitoring).

### Proje Yapisi (4 Ayri Repo + Docs)

Bu proje tek repo yerine **4 ayrı Git reposu** seklinde orgnize edilmistir:

| Repo | Icerik | Faz |
|------|--------|-----|
| `agentcore-demo-test1-backend` | Python: FastAPI + Temporal + Agent kodlari | FAZ 1-4 |
| `agentcore-demo-test1-frontend` | TypeScript: Next.js + CopilotKit web arayuzu | FAZ 5 |
| `agentcore-demo-test1-infra` | Terraform + Docker Compose + CI/CD tanimlari | FAZ 0 |
| `agentcore-demo-test1-docs` | Tum dokumanlar (bu dosya dahil) | Tum fazlar |

**Docs reposu** icerisinde `resources/` klasorunde bulunan dosyalar:
- `Prompt_00` ile `Prompt_06`: Her faz icin AI kodlama talimatlari
- `Deliverable_0` ile `Deliverable_9`: Referans dokumanlari
- `ARCHITECTURE_DATAFLOW_GUIDE.md`, `GIT_WORKFLOW.md`, `REPO_STRATEGY.md`: Kilavuzlar
- `DEMO_GELISTIRME_KILAVUZU.md`: Bu dosya

**Kod repolarinda** `resources/` klasoru `.gitignore`'da vardir — bu dosyalar dogrudan kod reposuna kopyalanmaz, docs reposundan referans alinir.

### Ortam Kurali (Environment Isolation)

Bu projede tum paket kurulumlari izole ortamlarda yapilir. Global kurulum YASAKTIR:

| Dil | Arac | Izole Ortam | Komut |
|-----|------|-------------|-------|
| **Python** | `uv` | `.venv` | `uv venv .venv && source .venv/bin/activate && uv pip install -r requirements.txt` |
| **Node.js** | `pnpm` (REPO_STRATEGY §5.3 uyarınca tek tercih) | `node_modules` | `pnpm install` (global `-g` YASAK) |
| **Docker** | `docker` | Container | `RUN pip install` Dockerfile icinde IZINLI |

**KURAL:** Kod yorumlarinda SADECE Ingilizce kullanilir. Baska dil yok.

**KURAL:** Her gate checkpoint icin bir pytest test fonksiyonu yazilir. Testler `tests/` klasorunde tutulur. `pytest tests/ -v` ile calistirilir.

### Calisma Modlari (Cok Onemli)

Bu projede IKI farkli calisma modu vardir. Hangi fazda hangi modu kullanacaginizi asagidaki tablo belirler:

| Mod | Kullanilan Fazlar | Ne Yaparsiniz | Ne Yapar Antigravity |
|-----|-------------------|---------------|----------------------|
| **INTERAKTIF (Siz Yapin)** | **FAZ 0** (AWS Altyapi) | **Terminalinizde** komutlari calistirin, sonucu kontrol edin | Size adim adim komut verir, siz "yaptim" deyince kontrol eder |
| **OTOMATIK (Antigravity Yapsin)** | **FAZ 1-6** (Agent, Backend, Frontend, Test) | Oturup izlersiniz, gerektiginde onay verirsiniz | Kodu yazar, dosyalari olusturur, testleri calistirir |

#### Neden Bu Ayrim?

**AWS surecleri (FAZ 0):** AWS hesabinizi kendiniz ogrenmek istiyorsunuz. Bu nedenle:
- Antigravity size bir komut verir (ornegin `aws ec2 create-vpc ...`)
- Siz o komutu **kendi terminalinizde** calistirirsiniz
- Sonucu gordugunuzde "yaptim" veya sonucu yapistirirsiniz
- Antigravity ciktiyi kontrol eder, dogruysa sonraki adima gecer
- Bu sayede AWS CLI, IAM, VPC, S3, RDS gibi servisleri ogrenirsiniz

**Gelistirme surecleri (FAZ 1-6):** Kod yazimi otomatiktir:
- Antigravity prompt'taki talimatlari okuyup kodu direk yazar
- Dosyalari olusturur, testleri calistirir
- Siz sadece onay verir veya duzeltme istersiniz

### FAZ 0 (AWS) — Interaktif Mod Nasil Calisir?

```
Antigravity: "Asagidaki komutu terminalinizde calistirin:
  aws ec2 create-vpc --cidr-block 10.0.0.0/16 ..."

Siz: (Terminalinize gidersiniz, komutu copy-paste yapar, calistirirsiniz)
  $ aws ec2 create-vpc --cidr-block 10.0.0.0/16 ...
  { "Vpc": { "VpcId": "vpc-12345", ... }}

Siz: (Sonucu kopyalayıp Antigravity'e yapistirirsiniz)
  "yaptim, sonuc: vpc-12345"

Antigravity: (Sonucu kontrol eder)
  "Dogru, VPC olusturuldu. Siradaki komut: ..."
```

### FAZ 1-6 — Otomatik Mod Nasil Calisir?

```
Siz: Prompt_01_AgentCore_Deployment.md'yi Antigravity'e verirsiniz

Antigravity: (Prompt'u okur, kodu yazar, dosyalari olusturur)
  "3 agent dosyasi olusturuldu. S3 ZIP deploy ediliyor..."
  "Agent 1 ACTIVE, Agent 2 ACTIVE, Agent 3 ACTIVE."
  "Gate 1 gecti. Commit yapiliyor..."

Siz: (Sadece izlersiniz, gerektiginde onay verirsiniz)
```

---

## 1. Onceden Hazir Olmasi Gerekenler

Asagidakilerin HAZIR oldugunu varsayiyoruz. Eger biri eksikse, o adimi tamamlamadan devam ETMEYIN.

| # | Sey | Nasil Kontrol Edilir? |
|---|-----|----------------------|
| 1 | **AWS Hesabi** acik ve billing alarmi $50 kurulu | AWS Console > Billing > Budgets |
| 2 | **Antigravity** (AI kodlama araci) bilgisayarinizda kurulu ve calisiyor | Terminal'de `antigravity --version` calisiyor olmali |
| 3 | **AWS CLI** kurulu (`aws --version` calisiyor) | Terminal'de `aws --version` |
| 4 | **Git** kurulu | Terminal'de `git --version` |
| 5 | **Bu kilavuzun oldugu zip dosyasi** bilgisayarinizda acik | `ls` ile icerigi gorebiliyorsunuz |

**Henuz kurulu degilse:**
- AWS CLI: https://docs.aws.amazon.com/cli/latest/userguide/getting-started-install.html
- Antigravity: size saglayan ekip tarafindan verilen kurulum talimatlari

---

## 1a. Git Workflow — Coklu Repo (Her Fazda Uygulanir)

Bu proje 4 ayrı Git reposundan olusur. Her repo kendi `main` ve `dev` branch'ine sahiptir. Kod HICBIR ZAMAN dogrudan `main`'e push edilmez — her zaman `dev` uzerinden PR (Pull Request) ile birlesir.

### Repo Yapisi (4 Ayri Repo)

```
# KOD REPOLARI
agentcore-demo-test1-backend/     # Python: FastAPI + Temporal + Agent kodlari
agentcore-demo-test1-frontend/    # TypeScript: Next.js + CopilotKit
agentcore-demo-test1-infra/       # Terraform + Docker Compose + CI/CD

# DOKUMAN REPOSU (Bu klasordeki dosyalar)
agentcore-demo-test1-docs/
└── resources/                    # Prompt'lar, Deliverable'lar, Kilavuzlar
    ├── Prompt_00_Infra_Bootstrap.md
    ├── Prompt_01_AgentCore_Deployment.md
    ├── ... (tum Prompt ve Deliverable dosyalari)
    └── DEMO_GELISTIRME_KILAVUZU.md   # Bu dosya
```

**Backend + Infra reposu** FAZ 0-4'te kullanilir. **Frontend reposu** FAZ 5'te kullanilir. **Docs reposu** tum fazlarda referans olarak kullanilir.

### Git Reposu Kurulumu (Day 0)

```bash
# === 1. Backend Repo ===
mkdir agentcore-demo-test1-backend && cd agentcore-demo-test1-backend
git init
git remote add origin git@github.com:${GITHUB_USER:-ugocen}/agentcore-demo-test1-backend.git

# .gitignore
cat > .gitignore << 'EOF'
__pycache__/
*.py[cod]
.venv/
venv/
*.egg-info/
.pytest_cache/
.coverage/
.env.local
*.zip
.DS_Store
resources/
EOF

# Ilk commit (bos proje yapisi)
git add .
git commit -m "chore: initial backend structure

- FastAPI + Temporal scaffold
- Agent directories (transcriber, drafter, reviewer)
- uv + .venv configuration

Refs: Gate-00"

# Remote'a push et
git branch -M main
git push -u origin main

# Dev branch olustur
git checkout -b dev
git push -u origin dev

# === 2. Frontend Repo ===
cd ..
mkdir agentcore-demo-test1-frontend && cd agentcore-demo-test1-frontend
git init
git remote add origin git@github.com:${GITHUB_USER:-ugocen}/agentcore-demo-test1-frontend.git

cat > .gitignore << 'EOF'
node_modules/
.pnpm-store/
.next/
*.log
.env.local
.DS_Store
resources/
EOF

git add .
git commit -m "chore: initial frontend structure

- Next.js 14+ App Router scaffold
- CopilotKit integration directories
- Canvas + Chat component structure

Refs: Gate-00"

git branch -M main
git push -u origin main
git checkout -b dev
git push -u origin dev

# === 3. Infra Repo (Opsiyonel — terraform kullanilacaksa) ===
cd ..
mkdir agentcore-demo-test1-infra && cd agentcore-demo-test1-infra
git init
git remote add origin git@github.com:${GITHUB_USER:-ugocen}/agentcore-demo-test1-infra.git
git add .
git commit -m "chore: initial infra structure"
git branch -M main && git push -u origin main
git checkout -b dev && git push -u origin dev
```

### Branch Kurali (COK ONEMLI)

| Branch | Ne Icin | Push Dogrudan mi? |
|--------|---------|-------------------|
| `main` | Production kodu | **HAYIR — sadece PR ile** |
| `dev` | Butunleme (integration) | **HAYIR — sadece PR ile** |
| `feature/*` | Yeni ozellik gelistirme | Evet (kendi branch'inize) |
| `hotfix/*` | Acil duzeltme | Evet (kendi branch'inize) |

**KURAL:** `main` ve `dev` branch'lerine HICBIR ZAMAN dogrudan `git push` yapilmaz. Her zaman bir Pull Request (PR) acilir, reviewer onayi alinir, CI testleri gecer, oyle birlesir.

### Gunluk Calisma Akisi

```bash
# === SABAHI: Dev branch'i guncelle ===
cd agentcore-demo-test1-backend
git checkout dev
git pull origin dev

# === GELISTIRME: Feature branch'inde calis ===
git checkout -b feature/faz-4-hitl-timeout

# ... kod yaz, test et ...

# === KOMIT: Sik sik komit et ===
git add src/temporal/workflow/brd_workflow.py
git commit -m "feat: add 15-minute HITL timeout timer

- asyncio.Timer for clarification timeout
- Escalate to proceed_with_best_effort on expiry
- Configurable via CLARIFICATION_TIMEOUT_SECONDS

Refs: Gate-04"

git push -u origin feature/faz-4-hitl-timeout

# === PR: Dev branch'e birlestir ===
# GitHub/GitLab uzerinden PR ac:
#  Base: dev  <--  Compare: feature/faz-4-hitl-timeout
#  1 reviewer ata, CI'nin gecmesini bekle

# PR onaylandiktan sonra:
git checkout dev
git pull origin dev        # Feature branch'iniz dev'e birlesmis olacak

# Feature branch'i temizle
git branch -d feature/faz-4-hitl-timeout
git push origin --delete feature/faz-4-hitl-timeout
```

### Gate Tamamlandiginda Commit

Her gate gectiginde `dev` branch'ine ozellikle isaretlenmis komit atilir:

```bash
# Tum degisiklikleri stage et
git add -A

# Gate marker'li komit
git commit -m "feat: Gate-03 backend foundation complete

- FastAPI app with workflow endpoints
- Temporal client + worker scaffold
- S3 claim-check handler
- OpenTelemetry instrumentation
- All unit tests passing (pytest tests/unit -v)

Gate: 03/06
Refs: DEMO-PLAN"

# Dev branch'e push (PR ile)
git push origin HEAD

# Gate tag'i olustur (istege bagli ama onerilir)
git tag -a gate-03 -m "Gate 03 complete: Backend foundation"
git push origin gate-03
```

### Commit Mesaji Sablonu

```bash
<type>: <ozet>

- Yapilan degisiklik 1
- Yapilan degisiklik 2

Refs: <issue-veya-gate-id>
```

| Type | Kullanim |
|------|----------|
| `feat` | Yeni ozellik |
| `fix` | Bug duzeltmesi |
| `test` | Test ekleme/guncelleme |
| `refactor` | Kod yapisi degisikligi |
| `docs` | Dokumantasyon |
| `chore` | Bakim, guncelleme |
| `agent` | Agent kodu degisikligi |

### Requirements Dosyalari Guncel Tutma

Her faz sonunda backend repo'sunda requirements dosyalari senkronize edilir:

```bash
cd agentcore-demo-test1-backend

# uv.lock dosyasi otomatik guncellenir (uv pip install ile)
uv pip freeze > requirements.txt

git add requirements.txt uv.lock pyproject.toml
git commit -m "chore: sync requirements after Gate-0N

- uv.lock regenerated
- requirements.txt exported from .venv

Refs: Gate-0N"
```

### Frontend icin Benzer Surec

```bash
cd agentcore-demo-test1-frontend
git checkout dev
git pull origin dev

git checkout -b feature/faz-5-canvas-panel

# ... gelistirme ...

git add .
git commit -m "feat: add BRD preview canvas panel

- Markdown rendering with syntax highlighting
- Approve/Reject action buttons
- Zustand state integration

Refs: Gate-05"

git push -u origin feature/faz-5-canvas-panel
# PR ac -> dev branch'e birlestir
```

---

## 1b. AgentCore Bootstrap (Sadece Ilk Kullanimda)

AgentCore ilk kez kullanilacaginda bir kerelik bootstrap yapilir:

```bash
# .venv aktif oldugundan emin ol
source /Users/ugurgocen/projects/agentcore-demo-test1/agentcore-demo-test1-backend/.venv/bin/activate

# AgentCore toolkit yuklu mu kontrol et
agentcore --version
# Cikti yoksa: uv pip install bedrock-agentcore-starter-toolkit

# AWS account'ta AgentCore bootstrap (bir kere)
agentcore bootstrap --region us-east-1
# Bu islem 3-5 dk surer. S3 bucket, IAM rolleri, CodeBuild projesi olusturur.

# Basarili mi kontrol et
aws s3 ls | grep agentcore-deploy
# Cikti: agentcore-deploy-{account-id} bucket'i gormelisin
```

**Bu adim SADECE BIR KERE calistirilir.** Sonraki deploy'lar icin tekrarlanmaz.

### Antigravity Nedir ve Nasil Kullanilir?

Antigravity, bu projede kullandiginiz AI kodlama asistanidir. Size terminal uzerinden erisim saglar. Iki sekilde kullanacaksiniz:

```
# Yontem 1: Bir prompt dosyasini Antigravity'e vermek
cat Prompt_00_Infra_Bootstrap.md | antigravity run

# Yontem 2: Antigravity'i etkilesimli modda acip paste etmek
antigravity
# (Acilan ekrana prompt dosyasinin icerigini kopyalayip yapistirin)
```

Antigravity size kod uretecek. Siz:
1. Uretilen kodu okuyun
2. Mantikli gorunuyorsa onaylayin (`y` veya `approve`)
3. Bir seyler yanlissa duzeltmesini isteyin

---

## 2. Genel Kural: IMDS ile Kimlik Dogrulama

Bu projede **HICBIR YERDE** AWS erisim anahtari (access key/secret key) yazilmaz. EC2 sunucumuz AWS'in IMDS (Instance Metadata Service) sistemini kullanir. Bu, EC2'nun IAM Role'unden otomatik olarak gecici kimlik bilgileri almasi demektir.

**Yani:** Kodunuzda `.env` dosyanizda `AWS_ACCESS_KEY_ID` veya `AWS_SECRET_ACCESS_KEY` gorecekseniz, bu bir hatadir. IMDS kullanilmalidir.

---

## 3. 7 Faz ve Gecis Sirasi

```
FAZ 0: AWS Altyapisi Kurulumu
   |
   |-- Gate 0: EC2 running, SSH calisiyor, Docker kurulu,
   |           RDS available, S3 bucket'lar var, IAM role atanmis
   v
FAZ 1: 3 Agent'i AgentCore Uzerinden Deploy Etme
   |
   |-- Gate 1: 3 agent'in hepsi `/ping` donuyor, `invoke` calisiyor
   v
FAZ 2: Agent'larda V5 Payload ve OpenTelemetry
   |
   |-- Gate 2: 8-block payload dogru, OTel trace CloudWatch'ta gorunuyor
   v
FAZ 3: FastAPI Backend (API sunucusu)
   |
   |-- Gate 3: `/health` 200 donuyor, DB migration basarili, 5 container ayakta
   v
FAZ 4: Temporal Workflow (is akisi motoru)
   |
   |-- Gate 4: Workflow basliyor, HITL signal calisiyor, A2A iletisim var
   v
FAZ 5: Frontend (web arayuzu)
   |
   |-- Gate 5: 8 CopilotKit ozelligi calisiyor, dual stream geliyor
   v
FAZ 6: Son Test ve Demo
   |
   |-- Gate 6: 11/11 test basarili
```

**KURAL:** Bir fazin gate'i gecmeden SONRAKI FAZA GECMEYIN. Hicbir zaman atlamayin.

---

## 4. FAZ 0: AWS Altyapisi Kurulumu

### Bu Fazda Ne Yapilir?

AWS uzerinde projenin calisacagi tum altyapi olusturulur: sanal sunucu (EC2), veritabani (RDS), dosya depolama (S3), guvenlik kurallari (IAM Role).

### Ne Kullanilir?

- **Antigravity'e verilecek:** `Prompt_00_Infra_Bootstrap.md` + `Deliverable_6_AWS_Operations_Guide.md`
- **Referans:** `PERSISTENT_FILESYSTEM_GUIDE.md` (S3 Files icin)
- **Cikti:** EC2, RDS, S3, CloudWatch, IAM Role, S3 Files Access Point

### Adim Adim Nasil Yapilir?

**Adim 0.1:** Terminal'de proje klasorune gidin.
```bash
cd agentcore-demo-test1
ls
# Gormeniz gerekenler: Prompt_00, Deliverable_6, PERSISTENT_FILESYSTEM_GUIDE, infra/
```

**Adim 0.2:** `Deliverable_6_AWS_Operations_Guide.md` dosyasini acin ve basligi okuyun. Bu dosya tum AWS kurulumunu adim adim anlatir. Siz bu dosyayi Antigravity'e vereceksiniz.

**Adim 0.3:** Antigravity'i baslatin ve sunu yapmasini isteyin:
```
"Deliverable_6_AWS_Operations_Guide.md'deki tum adimlari sirasiyla uygula:
- Step 1: Admin user olustur
- Step 2: VPC olustur
- Step 3: Security Groups
- Step 4: S3 Bucket'lar (3 adet)
- Step 4b: S3 Files File System + Access Point olustur
- Step 5: RDS PostgreSQL
- Step 6: IAM Role (least-privilege custom policy)
- Step 7: EC2 t3.large baslat + Docker kur
- Step 8: CloudWatch Log Groups
- Step 9: Bedrock model access ac (Console'dan)
- Step 10: AgentCore CLI kur"
```

Antigravity her adimi size gosterecek. Onayladikca devam edecek.

### Kontrol Noktalari (Checkpoint) — Bu Fazda

Her 2-3 adimda Antigravity'den asagidaki komutlari calistirmasini isteyin:

| Checkpoint | Komut | Ne Gormelisiniz? |
|-----------|-------|-----------------|
| VPC sonrasi | `cat .vpc_env` | `VPC_ID`, `PUB_SUBNET`, `PRIV_SUBNET` degerleri |
| S3 sonrasi | `aws s3 ls` | 3 bucket listeleniyor (`agentcore-demo-test1-*`) |
| IAM sonrasi | `aws iam get-role --role-name agentcore-demo-test1-ec2-role` | Role var, custom policy attachli |
| EC2 sonrasi | `aws ec2 describe-instances --instance-id ...` | `"State": {"Name": "running"}` |
| Docker sonrasi | `ssh ec2-user@IP "docker --version"` | Docker versiyonu donuyor |
| RDS sonrasi | `aws rds describe-db-instances` | `"DBInstanceStatus": "available"` |
| S3 Files sonrasi | `aws s3files describe-file-system --file-system-id ...` | `"LifeCycleState": "AVAILABLE"` |

### GATE 0 Gecis Kriterleri

Asagidakilerin TUMU dogru olmalidir. Antigravity'e bu kontrolleri calistirtin:

```bash
# Kontrol 1: EC2 calisiyor mu?
aws ec2 describe-instances --instance-id YOUR_INSTANCE_ID \
  --query 'Reservations[0].Instances[0].State.Name'
# Beklenen: "running"

# Kontrol 2: SSH calisiyor mu?
ssh -i agentcore-demo-test1-key.pem ec2-user@YOUR_PUBLIC_IP "echo OK"
# Beklenen: "OK"

# Kontrol 3: Docker calisiyor mu?
ssh ec2-user@YOUR_PUBLIC_IP "docker ps"
# Beklenen: Bos liste (Docker daemon calisiyor)

# Kontrol 4: RDS erisilebilir mi?
ssh ec2-user@YOUR_PUBLIC_IP \
  "psql postgresql://postgres:PASS@DB_ENDPOINT:5432/postgres -c 'SELECT 1;'"
# Beklenen: "1"

# Kontrol 5: S3 bucket'lar var mi?
aws s3 ls | grep agentcore-demo-test1
# Beklenen: 3 bucket listeleniyor

# Kontrol 6: IAM Role EC2'ye atanmis mi?
aws ec2 describe-instances --instance-id YOUR_INSTANCE_ID \
  --query 'Reservations[0].Instances[0].IamInstanceProfile.Arn'
# Beklenen: "...instance-profile/agentcore-demo-test1-ec2-profile"

# Kontrol 7: IMDS calisiyor mu?
ssh ec2-user@YOUR_PUBLIC_IP \
  "curl -s http://169.254.169.254/latest/meta-data/iam/info"
# Beklenen: {"Code": "Success", ...}

# Kontrol 8: Bedrock model access acik mi?
aws bedrock list-foundation-models \
  --query 'modelSummaries[?modelId==`anthropic.claude-sonnet-4-6`].modelLifecycle.status'
# Beklenen: ["ACTIVE"]

# Kontrol 9: S3 Files calisiyor mu?
aws s3files describe-file-system --file-system-id YOUR_FS_ID \
  --query 'LifeCycleState'
# Beklenen: "AVAILABLE"
```

**9 kontrolun 9'u da gectiyse -> FAZ 1'e gecebilirsiniz.**

---

## 5. FAZ 1: 3 Agent'i AgentCore Uzerinden Deploy Etme

### Bu Fazda Ne Yapilir?

Uc yapay zeka ajaninin kodu yazilir ve AWS AgentCore Runtime uzerinden deploy edilir. Deploy yontemi: S3 zip (container yerine).

### Ne Kullanilir?

- **Antigravity'e verilecek:** `Prompt_01_AgentCore_Deployment.md` + `Deliverable_2_Reference_Strands_Agent_Code.md` (ornek kodlar icin)
- **Cikti:** 3 agent kodu (main.py) + S3 zip deploy

### Adim Adim

**Adim 1.1:** Antigravity'e sunu soyleyin:
```
"Prompt_01_AgentCore_Deployment.md'deki talimatlari uygula:
1. agents/agent_1_transcriber/main.py yaz (Strands, BedrockAgentCoreApp)
2. agents/agent_2_drafter/main.py yaz (Strands, HITL destekli)
3. agents/agent_3_reviewer/main.py yaz (LangGraph StateGraph, 4 node)
4. Her agent'in requirements.txt ve pyproject.toml'unu yaz
5. agents/common/ altinda payload_builder.py, validate_payload.py,
   otel_setup.py, claim_check_io.py, s3_files_reference.py yaz"
```

**Adim 1.2:** Her agent icin S3 Files mount yapilandirmasini kontrol edin. `filesystem-configurations.json` dosyasi olusmus olmali:
```json
{
  "filesystemConfigurations": [{
    "s3Files": {
      "fileSystemArn": "arn:aws:s3-files:us-east-1:...:file-system/fs-...",
      "accessPointArn": "arn:aws:s3-files:us-east-1:...:access-point/fsap-...",
      "mountPath": "/mnt/agentcore/claim-checks",
      "readOnly": false
    }
  }]
}
```

**Adim 1.3:** Deploy edin:
```bash
# Her agent icin (Antigravity bunu yapacak)
cd agents/agent_1_transcriber
agentcore configure -e main.py --protocol AGUI \
  --filesystem-configurations file://filesystem-configurations.json
agentcore deploy
```

### Kontrol Noktalari

| Checkpoint | Komut | Ne Gormelisiniz? |
|-----------|-------|-----------------|
| Kod yazildi | `ls agents/*/` | Her agent'ta main.py, requirements.txt, pyproject.toml |
| Common moduller | `ls agents/common/` | 5 Python dosyasi |
| Deploy Agent 1 | `agentcore deploy` ciktisi | "Successfully deployed" |
| Deploy Agent 2 | `agentcore deploy` ciktisi | "Successfully deployed" |
| Deploy Agent 3 | `agentcore deploy` ciktisi | "Successfully deployed" |

### GATE 1 Gecis Kriterleri

```bash
# Kontrol 1: Agent 1 /ping donuyor mu?
aws bedrock-agentcore-runtime invoke-agent-runtime \
  --agent-runtime-arn AGENT1_ARN --session-id test-1 \
  --qualifier DEFAULT --input-text "ping"
# Beklenen: Yanit geliyor (hata degil)

# Kontrol 2: Agent 2 /ping donuyor mu?
# (Yukaridaki komutu Agent 2 ARN ile tekrarla)

# Kontrol 3: Agent 3 /ping donuyor mu?
# (Yukaridaki komutu Agent 3 ARN ile tekrarla)

# Kontrol 4: 3 agent kodu dosyada mi?
ls agents/agent_*/main.py
# Beklenen: 3 dosya listeleniyor

# Kontrol 5: Filesystem config var mi?
ls agents/agent_1_transcriber/filesystem-configurations.json
# Beklenen: Dosya var
```

**5 kontrolun 5'i de gectiyse -> FAZ 2'ye gecebilirsiniz.**

---

## 6. FAZ 2: Agent'larda V5 Payload ve OpenTelemetry

### Bu Fazda Ne Yapilir?

Agent'larin 8-block V5 payload uretmesi ve OpenTelemetry trace'lerinin CloudWatch'a gitmesi saglanir.

### Ne Kullanilir?

- **Antigravity'e verilecek:** `Prompt_02_Agent_Telemetry.md`
- **Referans:** `Deliverable_7_CloudWatch_Telemetry_Guide.md`
- **Cikti:** Agent'lar payload uretebiliyor, trace'ler CloudWatch'ta

### Adim Adim

**Adim 2.1:** Antigravity'e sunu soyleyin:
```
"Prompt_02_Agent_Telemetry.md'deki talimatlari uygula:
1. agents/common/payload_builder.py'deki build_complete_payload()
   fonksiyonunun V5 spec'e uygun oldugunu dogrula
2. agents/common/validate_payload.py'deki validate_payload()
   fonksiyonunun 8 block kontrol ettigini dogrula
3. agents/common/otel_setup.py'deki ADOT Collector endpoint'inin
   dogru oldugunu kontrol et (port 4317)
4. Her agent'in main.py'sinde tracer'in kullanildigini dogrula"
```

### Kontrol Noktalari

| Checkpoint | Komut | Ne Gormelisiniz? |
|-----------|-------|-----------------|
| Payload yapisi | `cat agents/common/payload_builder.py` | 8 fonksiyon: status, resources, timing, financial, artifacts, quality, tool_calls, risk |
| Validate | `cat agents/common/validate_payload.py` | `validate_payload()` fonksiyonu 8 block kontrol ediyor |
| OTel setup | `cat agents/common/otel_setup.py` | `OTLPSpanExporter(endpoint="adot-collector:4317")` |

### GATE 2 Gecis Kriterleri

```bash
# Kontrol 1: Payload 8 block iceriyor mu?
python3 -c "
import sys
sys.path.insert(0, 'agents/common')
from payload_builder import build_complete_payload
p = build_complete_payload(status_code='COMPLETED', trace_id='test', workflow_id='test', agent_run_id='test', step_id='test')
assert 'status' in p, 'status eksik'
assert 'resources' in p, 'resources eksik'
assert 'timing' in p, 'timing eksik'
assert 'financial' in p, 'financial eksik'
assert 'artifacts' in p, 'artifacts eksik'
assert 'quality' in p, 'quality eksik'
assert 'tool_calls' in p, 'tool_calls eksik'
assert 'risk' in p, 'risk eksik'
print('8 block OK')
"

# Kontrol 2: tier = "native" mi?
python3 -c "
import sys
sys.path.insert(0, 'agents/common')
from payload_builder import build_complete_payload
p = build_complete_payload('COMPLETED', 't', 'w', 'a', 's')
assert p['status']['tier'] == 'native', f\"tier yanlis: {p['status']['tier']}\"
assert p['status']['step_type'] == 'agent_action'
print('tier ve step_type OK')
"

# Kontrol 3: validate_payload calisiyor mu?
python3 -c "
import sys
sys.path.insert(0, 'agents/common')
from payload_builder import build_complete_payload
from validate_payload import validate_payload
p = build_complete_payload('COMPLETED', 't', 'w', 'a', 's')
result = validate_payload(p)
assert result['valid'] == True, f\"validasyon basarisiz: {result}\"
print('validate_payload OK')
"

# Kontrol 4: OTel tracer ayarlari dogru mu?
grep -r "endpoint.*4317" agents/common/otel_setup.py
# Beklenen: "4317" portu geciyor
```

**4 kontrolun 4'u de gectiyse -> FAZ 3'e gecebilirsiniz.**

---

## 7. FAZ 3: FastAPI Backend

### Bu Fazda Ne Yapilir?

FastAPI tabanli API sunucusu ve docker-compose ile 5 container'li sistem ayaga kaldirilir.

### Ne Kullanilir?

- **Antigravity'e verilecek:** `Prompt_03_Backend_Foundation.md`
- **Cikti:** `docker-compose.yml`, FastAPI uygulamasi, 5 calisan container

### Adim Adim

**Adim 3.1:** Antigravity'e sunu soyleyin:
```
"Prompt_03_Backend_Foundation.md'deki talimatlari uygula:
1. docker-compose.yml yaz (5 servis: temporal-server, temporal-ui,
   fastapi, temporal-worker, nextjs-frontend)
2. backend/app/main.py yaz (FastAPI entrypoint + health endpoint)
3. backend/app/otel_setup.py yaz (OTel FastAPI instrumentation)
4. backend/app/auth/mock_user.py yaz (MOCK_USER sabiti)
5. backend/app/storage/rds_models.py yaz (SQLAlchemy modelleri)
6. backend/app/config/settings.py yaz (Pydantic Settings)
7. backend/app/temporal/workflows.py, activities.py, worker.py yaz
8. backend/Dockerfile yaz (FastAPI + Worker ortak image)"
```

**Adim 3.2:** Container'lari baslatin:
```bash
# Antigravity bunu yapacak
docker-compose up -d
```

### Kontrol Noktalari

| Checkpoint | Komut | Ne Gormelisiniz? |
|-----------|-------|-----------------|
| docker-compose.yml | `cat docker-compose.yml` | 5 servis tanimli |
| Container'lar ayakta | `docker-compose ps` | 5 container `Up` durumunda |

### GATE 3 Gecis Kriterleri

```bash
# Kontrol 1: 5 container calisiyor mu?
docker-compose ps
# Beklenen: temporal-server, temporal-ui, fastapi, worker, nextjs — hepsi Up

# Kontrol 2: /health 200 donuyor mu?
curl http://localhost:8000/health
# Beklenen: {"status":"ok"}

# Kontrol 3: FastAPI swagger erisilebilir mi?
curl http://localhost:8000/docs
# Beklenen: HTML icerik (Swagger UI)

# Kontrol 4: DB migration uygulanmis mi?
docker-compose exec fastapi alembic current
# Beklenen: Migration versiyonu gosteriyor (hata degil)

# Kontrol 5: Temporal Web UI erisilebilir mi?
curl http://localhost:8081
# Beklenen: HTML icerik (Temporal UI)
```

**5 kontrolun 5'i de gectiyse -> FAZ 4'e gecebilirsiniz.**

---

## 8. FAZ 4: Temporal Workflow + A2A

### Bu Fazda Ne Yapilir?

Uc agent'in sirasiyla calistigi bir is akisi (workflow) tanimlanir. Agent'lar birbirleriyle Temporal signal'leri uzerinden konusur (A2A). HITL (Human-in-the-Loop) clarification sistemi calisir.

### Ne Kullanilir?

- **Antigravity'e verilecek:** `Prompt_04_Temporal_Workflow.md`
- **Referans:** `Deliverable_4_Temporal_Operations_Guide.md`
- **Cikti:** Temporal workflow, HITL signal loop, Evidence Pack

### Adim Adim

**Adim 4.1:** Antigravity'e sunu soyleyin:
```
"Prompt_04_Temporal_Workflow.md'deki talimatlari uygula:
1. backend/app/temporal/workflows.py'deki BRDWorkflow'u yaz
   (3 agent sirasi, HITL loop, max 5 round, max 2 self-correction)
2. backend/app/temporal/activities.py'deki 3 activity'i yaz
   (invoke_agent_core_activity, claim-check ile)
3. backend/app/temporal/worker.py'deki worker'i yaz
4. HITL signal handler'lari yaz (clarification_response, final_decision)"
```

### Kontrol Noktalari

| Checkpoint | Komut | Ne Gormelisiniz? |
|-----------|-------|-----------------|
| Workflow kodu | `cat backend/app/temporal/workflows.py` | `@workflow.defn` decorator, 3 activity cagrisi |
| Activity kodu | `cat backend/app/temporal/activities.py` | `@activity.defn`, claim-check fonksiyonlari |

### GATE 4 Gecis Kriterleri

```bash
# Kontrol 1: Workflow baslatabiliyor musunuz?
curl -X POST http://localhost:8000/api/workflows \
  -H "Content-Type: application/json" \
  -d '{"title":"Test BRD","audio_s3_key":"test.mp3"}'
# Beklenen: {"workflow_id":"wf-..."}

# Kontrol 2: Workflow durumunu sorgulayabiliyor musunuz?
# (Yukaridaki workflow_id'yi kullanin)
curl http://localhost:8000/api/workflows/wf-XXXX/status
# Beklenen: {"status":"RUNNING"} veya benzeri

# Kontrol 3: HITL signal gonderebiliyor musunuz?
curl -X POST http://localhost:8000/api/workflows/wf-XXXX/signal \
  -H "Content-Type: application/json" \
  -d '{"signal_name":"clarification_response","data":{"response_text":"Test"}}'
# Beklenen: 200 OK

# Kontrol 4: Evidence Pack 4 durumda da uretiliyor mu?
# (Workflow'u farkli senaryolarla calistirip evidence_packs tablosunu kontrol edin)
docker-compose exec fastapi psql $DATABASE_URL \
  -c "SELECT status, COUNT(*) FROM evidence_packs GROUP BY status;"
# Beklenen: APPROVED, REJECTED, TERMINATED, FAILED kayitlari

# Kontrol 5: SSE stream calisiyor mu?
curl -N http://localhost:8000/api/stream?workflow_id=wf-XXXX
# Beklenen: SSE event'leri geliyor (text/event-stream)
```

**5 kontrolun 5'i de gectiyse -> FAZ 5'e gecebilirsiniz.**

---

## 9. FAZ 5: Frontend (Next.js + CopilotKit)

### Bu Fazda Ne Yapilir?

Web arayuzu gelistirilir: kullanici ses yukler, is akisini izler, HITL clarification cevaplar, BRD'yi onaylar/reddeder.

### Ne Kullanilir?

- **Antigravity'e verilecek:** `Prompt_05_Frontend.md`
- **Referans:** `Deliverable_5_CopilotKit_AGUI_Guide.md`
- **Cikti:** Next.js uygulamasi, CopilotKit entegrasyonu

### Adim Adim

**Adim 5.1:** Antigravity'e sunu soyleyin:
```
"Prompt_05_Frontend.md'deki talimatlari uygula:
1. Next.js 14 App Router projesi baslat (frontend/)
2. CopilotKit entegrasyonu kur (useAgent hook)
3. Landing page yaz (app/page.tsx)
4. Workspace page yaz (app/workspace/[wfId]/page.tsx)
5. 8 CopilotKit ozelligini implemente et (asilama listesi Prompt'ta)
6. AG-UI Stream hook'u yaz (lib/agentcore-agui-client.ts)
7. Workflow Stream hook'u yaz (lib/use-workflow-stream.ts)
8. ClarificationQuestionCard component'i yaz (HITL dual-response)"
```

### Kontrol Noktalari

| Checkpoint | Nasil Kontrol Edilir? | Ne Gormelisiniz? |
|-----------|----------------------|-----------------|
| CopilotKit render | Browser'da landing page | CopilotKit chat widget gorunuyor |
| AG-UI SSE | Browser > Network > SSE | `TEXT_MESSAGE_CONTENT` event'leri geliyor |
| Workflow SSE | Browser > Network > SSE | `workflow_started`, `agent_1_completed` event'leri geliyor |

### GATE 5 Gecis Kriterleri

```bash
# Kontrol 1: Frontend container calisiyor mu?
docker-compose ps | grep frontend
# Beklenen: Up

# Kontrol 2: Landing page erisilebilir mi?
curl http://localhost:3000
# Beklenen: HTML icerik

# Kontrol 3: 8 CopilotKit ozelligi calisiyor mu?
# (Browser'da test edin)
# Asilama listesi: Text Message, Tool Call, State Delta,
# Interrupt, Custom, Auth, Meta Action, Generative UI
# Beklenen: Hepsi render oluyor

# Kontrol 4: Dual stream calisiyor mu?
# Browser > Network sekmesinde:
# 1. AG-UI SSE (dogrudan AgentCore'dan) — port farkli
# 2. Workflow SSE (FastAPI uzerinden) — port 8000
# Beklenen: Iki bagimsiz SSE baglantisi gorunuyor

# Kontrol 5: HITL clarification calisiyor mu?
# Workflow baslatin, clarification sorusu geldiginde
# ClarificationQuestionCard component'i gorunmeli
# Cevap verdikten sonra workflow devam etmeli
# Beklenen: Kart render oluyor, cevap sonrasi workflow devam ediyor
```

**5 kontrolun 5'i de gectiyse -> FAZ 6'ya gecebilirsiniz.**

---

## 10. FAZ 6: Son Test ve Demo

### Bu Fazda Ne Yapilir?

Tum sistemin uctan uca testi yapilir. 11 senaryonun tumu basarili olmalidir.

### Ne Kullanilir?

- **Antigravity'e verilecek:** `Prompt_06_E2E_Integration.md`
- **Cikti:** 11/11 test basarili, demo kullanima hazir

### Adim Adim

**Adim 6.1:** Antigravity'e sunu soyleyin:
```
"Prompt_06_E2E_Integration.md'deki 11 testi sirasiyla calistir.
Her test icin: ne yapilacagini acikla, kontrol komutunu ver,
sonucu dogrula. Test 7, 8, 9 icin farkli senaryolar uygula
(APPROVED, REJECTED, TERMINATED)."
```

### 11 Test Listesi ve Kontrol Komutlari

| # | Test | Nasil Yapilir | Kontrol |
|---|------|--------------|---------|
| 1 | Tum container'lar healthy | `docker-compose ps` | 5 container Up |
| 2 | FastAPI health + Temporal UI | `curl localhost:8000/health` + browser'da `localhost:8081` | 200 + UI aciliyor |
| 3 | Workflow baslat + Agent 1 | Ses yukleyin, workflow baslatin | `agent_1_completed` event geliyor |
| 4 | HITL clarification cevapla | Clarification sorusu geldiginde cevap verin | `hitl_response_received` event geliyor |
| 5 | Draft olusum + STATE_DELTA | Agent 2 tamamladiginda | BRD markdown preview gorunuyor |
| 6 | Draft onayla + Agent 3 review | "Approve" butonuna tiklayin | `agent_3_completed` event geliyor |
| 7 | Final onay (APPROVED) | "Final Approve" tiklayin | Evidence Pack `APPROVED`, DB'de `status='APPROVED'` |
| 8 | Final red (REJECTED) | Yeni workflow, "Reject" tiklayin | Evidence Pack `REJECTED`, DB'de `status='REJECTED'` |
| 9 | Ortada restart (TERMINATED) | Workflow calisirken container restart | Evidence Pack `TERMINATED` |
| 10 | CloudWatch traces | AWS Console > CloudWatch > GenAI Dashboard | Trace'ler gorunuyor |
| 11 | Sayfa yenileme | F5'e basin | Chat history + canvas geri yukleniyor |

### GATE 6 Gecis Kriterleri

**11 testin 11'i de basarili olmalidir.**

Bir test basarisiz olursa:
1. Hangi fazla ilgili oldugunu belirleyin (ornegin test 4 = FAZ 4)
2. O fazin dokumantasyonuna donun
3. Sorunu cozun
4. Sadece o testi tekrar calistirin

---

## 11. Tamamlama Kontrol Listesi

Tum fazlar bittiginde sunlarin hepsi dogru olmalidir:

- [ ] FAZ 0: 9/9 kontrol gecti
- [ ] FAZ 1: 5/5 kontrol gecti
- [ ] FAZ 2: 4/4 kontrol gecti
- [ ] FAZ 3: 5/5 kontrol gecti
- [ ] FAZ 4: 5/5 kontrol gecti
- [ ] FAZ 5: 5/5 kontrol gecti
- [ ] FAZ 6: 11/11 test basarili

**Tum kutular tikliyse demo hazirdir.**

---

## 12. Ne Zaman Hangi Dokumana Bakilir?

| Sorun | Bakilacak Dosya |
|-------|----------------|
| AWS kurulumu hatasi | `Deliverable_6_AWS_Operations_Guide.md` |
| IAM policy eksik | `Deliverable_6_AWS_Operations_Guide.md` Step 6 |
| S3 Files mount hatasi | `PERSISTENT_FILESYSTEM_GUIDE.md` |
| Agent kodu hatasi | `Deliverable_2_Reference_Strands_Agent_Code.md` |
| Payload format hatasi | `Prompt_02_Agent_Telemetry.md` |
| Backend API hatasi | `Prompt_03_Backend_Foundation.md` |
| Workflow hatasi | `Prompt_04_Temporal_Workflow.md` + `Deliverable_4` |
| Frontend hatasi | `Prompt_05_Frontend.md` + `Deliverable_5` |
| CopilotKit hatasi | `Deliverable_5_CopilotKit_AGUI_Guide.md` |
| AG-UI event hatasi | `ARCHITECTURE_DATAFLOW_GUIDE.md` Section 2 |
| A2A sinyal hatasi | `ARCHITECTURE_DATAFLOW_GUIDE.md` Section 6 |
| Claim-check hatasi | `PERSISTENT_FILESYSTEM_GUIDE.md` Section 6 |
| Monitoring hatasi | `Deliverable_7_CloudWatch_Telemetry_Guide.md` |
| Genel mimari | `ARCHITECTURE_DATAFLOW_GUIDE.md` |
| Yeni developer eklemek | `DEVELOPER_ONBOARDING.md` |
| E2E test hatasi | `Prompt_06_E2E_Integration.md` |

---

## 13. Hizli Referans: Tum Onemli Komutlar

```bash
# Container durumu
docker-compose ps

# Loglar
docker-compose logs -f [servis-adi]

# Workflow baslat
curl -X POST http://localhost:8000/api/workflows \
  -H "Content-Type: application/json" \
  -d '{"title":"Test","audio_s3_key":"test.mp3"}'

# Workflow durumu
curl http://localhost:8000/api/workflows/[wf-id]/status

# Signal gonder
curl -X POST http://localhost:8000/api/workflows/[wf-id]/signal \
  -d '{"signal_name":"clarification_response","data":{"text":"Cevap"}}'

# DB sorgusu
docker-compose exec fastapi psql $DATABASE_URL -c "SELECT * FROM workflow_states;"

# CloudWatch logs
aws logs tail "/agentcore-demo-test1/fastapi" --follow

# S3 icerigi
aws s3 ls s3://agentcore-demo-test1-claimcheck-[account-id]/claim-checks/
```

---

*Kilavuz Sonu*
