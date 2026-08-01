# Axia Security — Deployment & Installation Guide

**Deploying the Axia platform into a dedicated AWS account with Terraform**

> **Audience:** DevOps / platform engineers standing up a self-hosted Axia
> deployment in a dedicated AWS account (customer-owned or Axia-managed per-client).
> **Scope:** end-to-end — AWS account bootstrap → Terraform infrastructure →
> application services running on the host → verification.
> **What you need:** the Axia deployment bundle (`axia-deploy-<version>.tar.gz`). You do
> **not** need access to Axia's source repositories — everything the commands below
> reference ships inside the bundle (see §2).

---

## 1. Product Overview

Axia is a security-event analysis platform. It ingests events from data sources
(Microsoft 365 / Outlook, Microsoft Purview DLP, Cursor and other webhook sources, API),
enriches and classifies them, detects sensitive-data exposure with an LLM, and surfaces
cases in a web dashboard. Integration-specific setup for **Microsoft Purview** and
**Cursor** is in §8.

### 1.1 Target architecture (per deployment / per AWS account)

```
                          AWS Account (region of your choice)
 ┌───────────────────────────────────────────────────────────────────────────┐
 │  VPC 10.0.0.0/16   (3 AZs · 3 public /20 · 3 private /20 · 1 NAT gateway)    │
 │                                                                             │
 │   ┌── private subnet ───────────────┐   ┌── private subnet ─────────────┐  │
 │   │  Platform EC2 (t3a.large)        │   │  Classifier / ML EC2          │  │
 │   │  Ubuntu 22.04 + Docker           │   │  (c7i-flex.4xlarge)           │  │
 │   │  docker-compose:                 │   │  docker-compose:              │  │
 │   │   kafka, redis,                  │◄─►│   llm-service (Bedrock),      │  │
 │   │   object-storage-service,        │   │   parser-service, rapidocr,   │  │
 │   │   ingestion pipeline,            │   │   data-classification         │  │
 │   │   domain services, app layer     │   │                               │  │
 │   └───────────┬─────────────────────┘   └───────────────┬───────────────┘  │
 │               │ 27017 (TLS)                              │                  │
 │        ┌──────▼───────────┐                    ┌─────────▼─────────┐        │
 │        │ DocumentDB        │                    │  AWS Bedrock       │        │
 │        │ (MongoDB-compat)  │                    │  (Claude models)   │        │
 │        └───────────────────┘                    └───────────────────┘        │
 │        ┌───────────────────┐                                                 │
 │        │ S3 content-store   │   Reachable only via SSM Session Manager        │
 │        │ + axia-internal    │   (no public SSH; EC2 in private subnets)       │
 │        └───────────────────┘                                                 │
 └───────────────────────────────────────────────────────────────────────────┘
        ▲ container images pulled (cross-account) from Axia ECR
        │  <AXIA_ECR_REGISTRY>/<service>:<tag>   (registry provided by Axia)
```

**Hosts.** The full product runs across **two EC2 instances**, each with its own
`docker-compose` file:

| Host | Compose source | Runs |
|---|---|---|
| **Platform EC2** | `compose/platform/docker-compose.yml` | Kafka, Redis, object-storage, the ingestion pipeline (connectors + enrichment workers + event-analyzer + webhook), domain services (case, chat, events, engagement), and the app layer (application-backend/frontend, legacy app-backend/dashboard) |
| **Classifier / ML EC2** | `compose/ml/docker-compose.yml` | `llm-service` (Bedrock gateway), `parser-service`, `rapidocr` (OCR), `data-classification` |

The two hosts communicate over private-IP URLs (`LLM_SERVICE_URL`,
`PARSER_SERVICE_URL`, and the ML box reaching Kafka via `KAFKA_ADVERTISED_HOST`).

### 1.2 Terraform and Application

Terraform **only builds the AWS infrastructure**; it never starts the application.
The split:

| Terraform (`terraform/`) | Application layer (docker-compose on the host) |
|---|---|
| VPC, subnets, NAT, security groups | Kafka, Redis, all Axia service containers |
| Platform EC2 + Classifier EC2 (Docker pre-installed via `user_data.sh`) | Started with `docker compose up -d` over SSM |
| DocumentDB cluster (the "MongoDB") | Config generated from Terraform outputs + `config.env` |
| S3 content-store bucket + access credentials | |
| IAM roles/policies (ECR pull, Bedrock invoke, SSM, CloudWatch, S3) | |
| CloudWatch log groups | |

---

## 2. Prerequisites

### 2.0 What Axia must provide (self-host dependencies)

Container images are stored in Axia's AWS ECR and are pulled cross-account. In order
to grant you access to these images, notify Axia and provide your AWS Account ID.

### 2.1 The deployment bundle

Axia ships a self-contained deployment bundle — you do not clone the source repos.
Axia distributes it as an AWS S3 presigned URL; Axia will share the link with you.
Download it and unpack it, then run every command below from inside it:

```bash
curl -o axia-deploy-<version>.tar.gz "<presigned-S3-URL-from-Axia>"
tar -xzf axia-deploy-<version>.tar.gz
cd axia-deploy-<version>/
```

**Bundle contents**

```
axia-deploy-<version>/
├── terraform/
│   ├── README.md        # terraform usage notes
│   ├── init/            # terraform-backend.yaml (bootstrap)
│   ├── vpc/             # network stack (+ backend.hcl.example, terraform.tfvars.example)
│   └── platform/        # EC2 + DocumentDB + S3 + IAM  (+ backend.hcl.example, terraform.tfvars.example)
├── compose/
│   ├── platform/docker-compose.yml   # platform host services
│   └── ml/docker-compose.yml         # ML/classifier host services
├── configs/
│   ├── platform/config.env.example   # platform client-config template
│   ├── ml/config.env.example         # ML client-config template
│   ├── fluent-bit-cloudwatch.conf    # CloudWatch log shipping config
│   └── extract_service.lua           # fluent-bit Lua filter
└── scripts/
    ├── generate-deployment-config.sh
    ├── deploy-v2.sh
    └── encode-mongodb-password.py
```

The bundle's `terraform/platform/` and `terraform/vpc/` stacks each ship a
`backend.hcl.example` and `terraform.tfvars.example` for you to fill in and copy to
`backend.hcl`/`terraform.tfvars` (see §4.1); the scripts are pre-fixed to bundle-relative paths.

### 2.2 Operator workstation tooling

| Tool | Purpose |
|---|---|
| AWS CLI **v2** | account bootstrap, Terraform auth, SSM sessions, reading outputs |
| Terraform **≥ 1.0** (AWS provider ≥ 6.10.0) | provisioning |
| `jq` | generating deployment config from TF outputs |
| `python3` | password URL-encoding in the config generator |
| Docker | (only for local testing) |
| `tar` | unpacking the Axia deployment bundle |
| **Session Manager plugin** for AWS CLI | connecting to the private-subnet EC2 hosts |

### 2.3 AWS prerequisites

- **A dedicated AWS account** for this deployment (each Axia environment = its own account).
- **Admin access** to that account for the one-time bootstrap (§3), then the scoped
  deployment role for day-to-day applies.
- **Bedrock model access enabled** in the account/region for the Claude models used by
  `llm-service` and `chat-service` (request access in the Bedrock console). Note the
  exact model IDs available via `aws bedrock list-foundation-models`.
- **Region:** pick **one region** for the whole environment and use it consistently.

### 2.4 Data-source / integration credentials (collect before deploy)

Only what you intend to enable:

- **Microsoft 365 / Outlook:** an Entra app registration — `MICROSOFT_TENANT_ID`,
  `MICROSOFT_CLIENT_ID`, `MICROSOFT_CLIENT_SECRET` (Graph permissions + admin
  consent), and `OUTLOOK_ORG_DOMAIN`.
- **Microsoft Purview DLP:** an Entra app with the Graph `SecurityAlert.Read.All`
  application permission (can reuse the Outlook app). Full steps in **§8.1**.
- **Cursor (user-prompt events):** captured via Cursor's local `beforeSubmitPrompt` hook
  deployed to developer endpoints, forwarding to the webhook receiver — you set a webhook
  source secret. Fleet-wide deployment needs Cursor Enterprise. See **§8.2**.
- **Webhook / API:** the inbound webhook and API connectors need no third-party
  credentials — configure their source/topic settings in `config.env`.
- **SMTP** (optional reviewer-alert email): host/port/user/password/from-address.

---

## 3. Phase 0 — AWS account bootstrap (one-time)

This creates the Terraform state backend that Terraform assumes.
Run it once per account, with admin credentials.

### 3.1 Set your context

```bash
cd axia-deploy-<version>/
export AWS_REGION=<your-region>
export ACCOUNT_ID=$(aws sts get-caller-identity --query Account --output text)
echo "Deploying to account ${ACCOUNT_ID} in ${AWS_REGION}"
```

### 3.2 Create the Terraform state backend (S3 + DynamoDB lock)

```bash
cd terraform/init
aws cloudformation create-stack \
  --stack-name terraform-backend \
  --template-body file://terraform-backend.yaml \
  --region ${AWS_REGION}

# Wait, then confirm
aws cloudformation describe-stacks --stack-name terraform-backend \
  --region ${AWS_REGION} --query 'Stacks[0].StackStatus'
```

This yields the state bucket by convention: **`terraform-state-<ACCOUNT_ID>-<REGION>`**
versioned with public access blocked.

---

## 4. Phase 1 — Terraform infrastructure

Apply the stacks **in this order**: `terraform/vpc/` (optional) → `terraform/platform/`.
Skip `terraform/vpc/` if you want to use your existing VPC and subnets instead (see §4.3).

Backend convention for every stack: bucket `terraform-state-<ACCOUNT_ID>-<REGION>`,
key `<REGION>/<stack>/terraform.tfstate`, region `<REGION>`.

### 4.1 Per-environment config files

Both `terraform/vpc` and `terraform/platform` ship a `terraform.tfvars.example` and a
`backend.hcl.example`, with `${ACCOUNT_ID}` / `${AWS_REGION}` placeholders. Expand them
into the real `terraform.tfvars` / `backend.hcl` files with `envsubst` (recommended), or
copy the `.example` files by hand and fill in the placeholders yourself. Terraform loads
a file named `terraform.tfvars` automatically, so `plan`/`apply` don't need a `-var-file`
flag once it exists:

```bash
cat terraform.tfvars.example | envsubst > terraform.tfvars
cat backend.hcl.example      | envsubst > backend.hcl
```

Run this from inside each stack's directory, right before applying it — see §4.2
(`terraform/vpc`) and §4.3 (`terraform/platform`).

### 4.2 Apply the VPC stack (optional)

Creates `10.0.0.0/16` with 3 public + 3 private `/20` subnets across 3 AZs and a NAT
gateway. These are defaults — change `vpc_cidr`, `public_subnet_cidrs`, and
`private_subnet_cidrs` in `terraform.tfvars` if you need different ranges. Its output
is consumed by the platform stack via remote state. Skip this section if you're using
an existing VPC (§4.3).

```bash
cd ../vpc     # axia-deploy/terraform/vpc
export AWS_REGION=<your-region>
export ACCOUNT_ID=$(aws sts get-caller-identity --query Account --output text)
cat terraform.tfvars.example | envsubst > terraform.tfvars
cat backend.hcl.example      | envsubst > backend.hcl

terraform init -backend-config=backend.hcl
terraform plan
terraform apply
```

### 4.3 Apply the platform stack

Reads the VPC remote state and creates: the **Platform EC2** (`t3a.large`, latest
Ubuntu 22.04 AMI, gp3 100 GB encrypted, first private subnet, Docker bootstrapped by
`user_data.sh`), the **Classifier EC2**, the **DocumentDB cluster** (master user
`docdbadmin`, password auto-managed in Secrets Manager, TLS + encryption + 7-day
backups + deletion protection), the **S3 content-store** bucket and access user, all
IAM roles/policies (ECR pull, Bedrock invoke, SSM, CloudWatch, S3, axia-internal), and
CloudWatch log groups.

```bash
cd ../platform     # axia-deploy/terraform/platform
export AWS_REGION=<your-region>
export ACCOUNT_ID=$(aws sts get-caller-identity --query Account --output text)
cat terraform.tfvars.example | envsubst > terraform.tfvars
cat backend.hcl.example      | envsubst > backend.hcl
```

In the generated `terraform.tfvars`, fill in:

- `roles_allowed_to_assume_platform_role` — SSO/admin role names in THIS account
  allowed to assume the platform EC2 role.

If you skipped `terraform/vpc/` and want to use your existing VPC and subnets instead:
- leave `vpc_state_key` empty (`""`)
- set `vpc_name` to the `Name` tag on the existing VPC
- set `private_subnet_ids` / `public_subnet_ids` to the existing subnet IDs

Other platform variables you may set (`variables.tf`): `project_name` (default
`platform` — drives resource names and the EC2 `Name` tag), `documentdb_instance_class`
(default `db.t3.medium`), `documentdb_instance_count` (default `1`),
`documentdb_engine_version` (default `8.0.0`), `classifier_instance_type` (default
`c7i-flex.4xlarge`), `classifier_volume_size` (default `200` GB), and — set
`enable_alerts = true` to also set `alerts_email` and `alerts_client_name`.

```bash
terraform init -backend-config=backend.hcl
terraform plan
terraform apply
```

**Capture the outputs** — you need them for the app layer:

```bash
terraform output ssm_command                          # connect to the platform host
terraform output documentdb_cluster_endpoint
terraform output documentdb_get_password_command
terraform output content_store_bucket_name
terraform output s3_content_access_get_credentials_command
terraform output classifier_ssm_command               # connect to the ML host
```

---

## 5. Phase 2 — Deploy the application layer (Platform host)

The application is delivered onto the EC2 host as a `docker-compose` stack: generate
config from Terraform outputs, copy it and the compose file to the host over SSM, and
run the deploy script. Covered below.

Six files must be present in `/var/lib/axia` on the host before you deploy:

| File | Where it comes from |
|---|---|
| `deployment-config.json` | Generated into `axia-deploy/configs/` by `scripts/generate-deployment-config.sh` (§5.1). |
| `config.env` | Copy `configs/platform/config.env.example` and fill in the per-client values (§5.2). |
| `docker-compose.yml` | Bundled at `compose/platform/docker-compose.yml`. |
| `deploy-v2.sh` | Bundled at `scripts/deploy-v2.sh`; run it from `/var/lib/axia` on the host (§5.3). |
| `fluent-bit-cloudwatch.conf` | Bundled at `configs/fluent-bit-cloudwatch.conf`. Fluent Bit config for the log-shipping sidecar: tails every container's Docker log, reassembles multi-line stack traces into single records, and ships each service's logs to its own CloudWatch stream. |
| `extract_service.lua` | Bundled at `configs/extract_service.lua`. Lua filter the config above calls to derive each container's service name from its `com.docker.compose.service` label, so CloudWatch streams are named per-service rather than by container ID. |

### 5.1 Generate the deployment config

From `axia-deploy/scripts` (after a successful `terraform apply` in `terraform/platform`),
run the generator. It reads `terraform output`, fetches the DocumentDB password from
Secrets Manager and the S3 credentials from SSM, URL-encodes the password, and writes
`configs/deployment-config.json`:

```bash
cd ../../scripts     # axia-deploy/scripts
./generate-deployment-config.sh
```

> The script resolves the platform stack relative to itself (`../terraform/platform`),
> so no path editing is required.

The resulting JSON contains: `account_id`, `aws_region`, `ecr_registry`, `ecr_region`,
`documentdb_endpoint`, the full TLS `mongodb_uri`, `content_store_bucket`,
`content_store_region`, `s3_access_key`, `s3_secret_key`, and `s3_url`.

Sample `deployment-config.json`:

```json
{
  "account_id": "<account-id>",
  "aws_region": "<region>",
  "ecr_registry": "603685643976.dkr.ecr.us-east-2.amazonaws.com",
  "ecr_region": "us-east-2",
  "documentdb_endpoint": "<documentdb-cluster-endpoint>",
  "mongodb_uri": "mongodb://docdbadmin:<url-encoded-pw>@<documentdb-cluster-endpoint>:27017/security_platform?directConnection=true&serverSelectionTimeoutMS=2000&tls=true&tlsAllowInvalidCertificates=true&retryWrites=false&authMechanism=SCRAM-SHA-1",
  "content_store_bucket": "<account-id>-<region>-content-store",
  "content_store_region": "<region>",
  "s3_access_key": "<s3-access-key>",
  "s3_secret_key": "<s3-secret-key>",
  "s3_url": "https://s3.<region>.amazonaws.com"
}
```

### 5.2 Create `config.env`

Copy `configs/platform/config.env.example` to `config.env` and fill in the per-client,
non-generated values (data-source credentials from §2.4). Leave a section commented if
you are not enabling that connector.

### 5.3 Copy files to the host and deploy

Connect via SSM (private subnet — no SSH):

```bash
cd ../terraform/platform     # axia-deploy/terraform/platform
$(terraform output -raw ssm_command)   # aws ssm start-session --target i-xxxx ...
```

Get the compose file and deploy assets onto the host, into `/var/lib/axia`. The six
files the deploy script requires in that directory are:

- `deployment-config.json` (from §5.1)
- `config.env` (from §5.2)
- `docker-compose.yml` — the bundled `compose/platform/docker-compose.yml`
- `deploy-v2.sh` — the bundled deploy script
- `fluent-bit-cloudwatch.conf` — bundled at `configs/fluent-bit-cloudwatch.conf`
- `extract_service.lua` — bundled at `configs/extract_service.lua`

Then run the deploy script on the host:

```bash
cd /var/lib/axia     # on the EC2 host
./deploy-v2.sh
docker compose ps
```

It:

1. installs `jq`/`python3` if missing;
2. downloads the DocumentDB CA bundle to `/var/lib/axia/certs/global-bundle.pem`
   (`https://truststore.pki.rds.amazonaws.com/global/global-bundle.pem`);
3. logs Docker in to ECR
   (`aws ecr get-login-password | docker login ... <ecr_registry>`);
4. generates `/var/lib/axia/.env` (maps config → service env keys, fans the Mongo URI
   out to the services, sets the Bedrock `LLM_MODEL`,
   `KAFKA_BOOTSTRAP_SERVERS=kafka:29092`, the S3/bucket vars) and **appends your entire
   `config.env`**;
5. `docker compose pull` then `docker compose up -d --pull always`.

The script is idempotent — safe to re-run for upgrades.

---

## 6. Phase 3 — Deploy the ML / Classifier host

Repeat the deploy on the **Classifier EC2** using the bundled
`compose/ml/docker-compose.yml`, which runs `llm-service`, `parser-service`,
`rapidocr`, and `data-classification`. This host reuses the same
`deployment-config.json` generated in §5.1 and the same `deploy-v2.sh` script — only
`config.env` and `docker-compose.yml` differ.

Six files must be present in `/var/lib/axia` on this host before you deploy:

| File | Where it comes from |
|---|---|
| `deployment-config.json` | The same file generated in §5.1 — copy it to this host too. |
| `config.env` | Copy `configs/ml/config.env.example` and fill in the per-client values (§6.1). |
| `docker-compose.yml` | Bundled at `compose/ml/docker-compose.yml`. |
| `deploy-v2.sh` | Bundled at `scripts/deploy-v2.sh`; run it from `/var/lib/axia` on the host (§6.2). |
| `fluent-bit-cloudwatch.conf` | Bundled at `configs/fluent-bit-cloudwatch.conf`. |
| `extract_service.lua` | Bundled at `configs/extract_service.lua`. |

### 6.1 Create `config.env`

Copy `configs/ml/config.env.example` to `config.env` and fill in the per-client values:

- `OBJECT_STORAGE_URL`, `SETTINGS_SERVICE_URL`, `REDIS_URL` — point these at the
  **platform host's private IP** (the ML box reaches platform services over
  private-IP URLs; `KAFKA_ADVERTISED_HOST` / `LLM_SERVICE_URL` / `PARSER_SERVICE_URL`
  are the platform-side equivalents, set in *its* `config.env`, §5.2).
- `LLM_MODEL`, `PROMPT_PATH` — set per the Bedrock model you were granted access to.
- `CLASSIFIER_PREFILTER_ENABLED` — toggles the `data-classification` pre-filter in
  front of `llm-service`; ships as `true` (the service itself defaults to `false` if
  unset).
- `COMPOSE_PROJECT_NAME` — keep this value the same on every redeploy from this host;
  changing it creates a duplicate stack instead of reconciling the existing one (the
  compose file defaults to `event-analyzer` if unset).
- `DB_NAME` — matches the services' own default (`security_platform`); no need to
  change it.
- `LOG_LEVEL` — ships as `DEBUG`, more verbose than the services' own default
  (`info`/`INFO`); lower it for production if you don't want debug-level noise.
- Leave `DATA_BUCKET` and `AXIA_BUCKET` blank or remove the lines — both are
  generated automatically from `deployment-config.json` by `deploy-v2.sh`; a
  non-empty value here overrides the generated one.

### 6.2 Copy files to the host and deploy

Connect via SSM (private subnet — no SSH):

```bash
# still in axia-deploy/terraform/platform (from §5.3)
$(terraform output -raw classifier_ssm_command)   # aws ssm start-session --target i-xxxx ...
```

Get the compose file and deploy assets onto the host, into `/var/lib/axia`. The six
files the deploy script requires in that directory are:

- `deployment-config.json` (the same file from §5.1)
- `config.env` (from §6.1)
- `docker-compose.yml` — the bundled `compose/ml/docker-compose.yml`
- `deploy-v2.sh` — the bundled deploy script
- `fluent-bit-cloudwatch.conf` — bundled at `configs/fluent-bit-cloudwatch.conf`
- `extract_service.lua` — bundled at `configs/extract_service.lua`

Then run the deploy script on the host:

```bash
cd /var/lib/axia     # on the EC2 host
./deploy-v2.sh
docker compose ps
```

Requirements specific to this host:

- Bedrock must be reachable from the instance role, with **`AWS_DEFAULT_REGION` set**
  (a missing region is the classic `NoRegionError` for `llm-service`).
- The ML box reaches the platform Kafka via `KAFKA_ADVERTISED_HOST` = the platform
  host's private IP; the platform services reach the ML box via `LLM_SERVICE_URL` and
  `PARSER_SERVICE_URL` = the ML host's private IP.

---

## 7. Phase 4 — Verification

On each host:

```bash
cd /var/lib/axia     # on each host
docker compose ps                                   # all Up / healthy
docker compose logs --tail 100 | grep -iE 'error|exception|failed'
```

Service health endpoints (reach them from the host, or via SSM port-forward):

**Platform host**

| Component | Port | Check |
|---|---|---|
| kafka | 9092 | TCP connect (broker accepting connections) |
| kafka-ui | 8081 | web UI, no healthcheck |
| redis | 6379 | `redis-cli ping` → `PONG` |
| object-storage-service | 8080 | `/healthz` |
| dashboard (v1 UI) | 5173 | HTTP 200 on `/` |
| application-frontend (v2 UI) | 5174 | HTTP 200 on `/` |
| application-backend (v2 API) | 8088 | `/healthz` |
| app-backend (legacy v1 API) | 8003 | `/health` |
| case-service | 8005 | `/healthz` |
| chat-service | 8008 | `/healthz` |
| events-service | 8001 | `/healthz` |
| engagement-service | 8010 | `/healthz` |
| webhook-connector | 8011 | `/healthz` |

The remaining platform containers (`event-analyzer`, the `connector-*` ingestion
services, and the `enrichment-worker-*` consumers) publish no port and have no
healthcheck — verify them via `docker compose ps` (State column) and their logs.

**Classifier / ML host**

| Component | Port | Check |
|---|---|---|
| llm-service | 8000 | `/health` |
| parser-service | 8090 | `/healthz` |
| rapidocr | internal only (8829) | `/health` — reachable only from `parser-service` on the compose network |
| data-classification | internal only (8000) | `/health` — reachable only from `llm-service` on the compose network |
| fluent-bit-cloudwatch | — | `pidof fluent-bit` (no HTTP check) |

To reach the UI or DocumentDB from your workstation, use SSM port-forwarding
(the platform stack prints ready-made commands):

```bash
# still in axia-deploy/terraform/platform (from §5.3)
terraform output documentdb_ssm_port_forward_command   # localhost:27017 → DocumentDB
```

---

## 8. Data-source integrations

Axia ingests external events through **connectors**. This section covers the two
integrations for this deployment: **Microsoft Purview DLP** (Axia polls Microsoft
Graph outbound) and **Cursor user-prompt events** (forwarded to Axia's webhook receiver
by a hook on each developer's machine). Both are additive — configure only what you use.
All values below go in `config.env` (§5.2).

### 8.1 Microsoft Purview DLP

**How it works.** The `connector-purview` service (the shared `connector` image with
`CONNECTOR_TYPE=purview`) authenticates to Microsoft with the OAuth2 client-credentials
flow and **polls the Microsoft Graph Security API** for DLP alerts. Each alert is
published to the `dlp.events` Kafka topic, where the existing `enrichment-worker-dlp`
consumes it. The connector reaches *out* to Microsoft — no inbound network exposure is
needed.

The Graph call it makes:

```
GET https://graph.microsoft.com/v1.0/security/alerts_v2?$filter=serviceSource eq 'dataLossPrevention'
```

(incremental polls add `and lastUpdateDateTime gt <timestamp>`; results are paged via
`@odata.nextLink`).

**Purview-side prerequisites.**
- DLP policies must be **configured and enabled** in the Microsoft Purview portal, with
  rules set to generate alerts — no policies means no alerts.
- Licensing: single-event alerts work on any DLP subscription; aggregated
  (threshold-based) alerts require E5 / the Purview compliance tier.
- Newly created/modified policies can take **up to ~3 hours** to start generating alerts.

**Azure Entra app registration (one-time, in the customer tenant).**
1. **Entra admin center → App registrations → New registration.** Name it (e.g.
   "Axia Purview DLP Connector"), single-tenant, no redirect URI. **Register.**
2. On **Overview**, copy **Application (client) ID** → `MICROSOFT_CLIENT_ID` and
   **Directory (tenant) ID** → `MICROSOFT_TENANT_ID`.
3. **Certificates & secrets → New client secret** → copy the **Value** (shown once) →
   `MICROSOFT_CLIENT_SECRET`.
4. **API permissions → Add a permission → Microsoft Graph → Application permissions** →
   add **`SecurityAlert.Read.All`**.
5. **Grant admin consent** for the tenant (required — the token won't carry the
   permission until consent is recorded).

> You can reuse the **same Entra app as the Outlook connector**: just add the
> `SecurityAlert.Read.All` permission to it and reuse the same tenant/client/secret.

**Configure the connector** (`config.env`):

```bash
MICROSOFT_TENANT_ID=<tenant-id>
MICROSOFT_CLIENT_ID=<client-id>
MICROSOFT_CLIENT_SECRET=<client-secret>
PURVIEW_START_DATE=            # optional backfill start (e.g. 2026-01-01); blank = from now
```

Stack-provided defaults you normally leave alone: `MICROSOFT_GRAPH_BASE_URL`
=`https://graph.microsoft.com/v1.0`, `MICROSOFT_AUTHORITY_BASE_URL`
=`https://login.microsoftonline.com`, `PURVIEW_PENDING_QUEUE_NAME`=`dlp.events`.

**Verify.** `docker compose logs -f connector-purview` shows a successful token fetch and
poll; DLP alerts land on the `dlp.events` topic (visible in kafka-ui, §9-ports) and flow
through `enrichment-worker-dlp` into the events service.

**Microsoft references:** [list `alerts_v2`](https://learn.microsoft.com/en-us/graph/api/security-list-alerts_v2?view=graph-rest-1.0) ·
[`serviceSource` enum](https://learn.microsoft.com/en-us/graph/api/resources/enums-security) ·
[client-credentials / `.default`](https://learn.microsoft.com/en-us/graph/auth-v2-service) ·
[DLP alerts prerequisites & licensing](https://learn.microsoft.com/en-us/purview/dlp-alerts-get-started).

### 8.2 Cursor (user-prompt events)

We ingest **only the user-prompt events** — the text developers submit to Cursor's AI —
for sensitive-data detection.

**How it works.** Unlike Purview, you don't point Cursor at a URL — Cursor has no
cloud-side webhook or API that emits user prompts (its only outbound webhook is Cloud-Agent
status; its Admin API, audit logs, and code-tracking exports all exclude prompt text).
Prompt capture is done with a **local Cursor hook** — `beforeSubmitPrompt` — that runs a
small script on each developer's machine right after the user hits send; the script receives
the prompt as JSON on stdin and forwards it to Axia's webhook receiver:

```
Developer's Cursor ──beforeSubmitPrompt hook──▶ forwarder script ──HTTPS POST──▶ webhook-connector :8011
   (prompt text)                                (adds X-Webhook-Secret)          POST /api/v1/webhooks/cursor
                                                                                 → enrichment.pending
                                                                                 → enrichment-worker-webhook
```

Because **your** script makes the HTTP call (not Cursor's cloud), it sets the static
`X-Webhook-Secret` header itself — so this path is **directly compatible** with the
receiver, and the HMAC shim that a Cursor-cloud webhook would need does **not** apply here.
"Enable only the user-prompt events" simply means: declare only the `beforeSubmitPrompt`
hook and no other hook types.

**Step 1 — register the source on the Axia side** (`config.env`):

```bash
# name:secret pairs, ';'-separated (this is the static X-Webhook-Secret value)
WEBHOOK_ENRICHMENT_SOURCES=cursor:<axia-webhook-secret>
WEBHOOK_ENRICHMENT_TOPIC=enrichment.pending
```

**Step 2 — expose the receiver to developer machines.** The receiver (`:8011`) sits on
the private-subnet platform host, so front it with an **HTTPS ingress** (ALB / reverse
proxy) reachable from where developers run Cursor (corporate network / VPN, or internet
for remote staff). Terminate TLS at the ingress; forward `POST /api/v1/webhooks/cursor`.

**Step 3 — deploy the Cursor hook to developer endpoints.** Add a `hooks.json` plus a
forwarder script. Scope options: `~/.cursor/hooks.json` (per user),
`<project>/.cursor/hooks.json` (per repo, committable), or **Enterprise-managed hooks**
pushed from the Cursor team dashboard / MDM for tamper-resistant fleet-wide rollout.

`hooks.json` — declare **only** the prompt hook:

```json
{
  "version": 1,
  "hooks": {
    "beforeSubmitPrompt": [{ "command": ".cursor/hooks/forward-prompt.sh" }]
  }
}
```

`forward-prompt.sh` — forward the event to Axia and never block the developer:

```bash
#!/usr/bin/env bash
payload="$(cat)"                                   # beforeSubmitPrompt event on stdin
gen_id="$(printf '%s' "$payload" | jq -r '.generation_id // empty')"
curl -s -X POST "https://<ingress-host>/api/v1/webhooks/cursor" \
  -H "Content-Type: application/json" \
  -H "X-Webhook-Secret: <axia-webhook-secret>" \
  -H "X-Idempotency-Key: ${gen_id}" \
  --data "$payload" >/dev/null
echo '{"continue": true}'    # allow the prompt; leave failClosed off so errors never block
```

**Payload & idempotency.** The `beforeSubmitPrompt` event carries `prompt` (the text),
`user_email`, `conversation_id`, `generation_id`, `attachments[]`, `workspace_roots[]`,
`hook_event_name`. There is **no timestamp** (stamp on receipt) and **no `id`/`event_id`**
field — pass `generation_id` as `X-Idempotency-Key` (as above) so the receiver dedupes per
prompt; `conversation_id` + `generation_id` is the stable composite key.

**Enablement, privacy & caveats.**
- **Central config & enforcement via MDM.** Deploy and enforce the hook fleet-wide with
  your **MDM** (and/or Cursor Enterprise managed hooks from the team dashboard) so it is
  applied consistently and developers **cannot remove or bypass it**. Ad-hoc per-repo or
  per-user `hooks.json` is fine for testing but is removable by developers.
- **Monitoring consent.** Capturing developers' prompt text is user-input monitoring —
  handle as a policy/consent matter and align with Cursor Privacy Mode policy.
- **Fail open.** Leave `failClosed` off (default) so a forwarder error never blocks a
  developer; enable it only if you intend a blocking DLP gate.

**Cursor references:** [Hooks (`beforeSubmitPrompt`)](https://cursor.com/docs/hooks) ·
[LLM Safety & Controls — hooks-based DLP](https://cursor.com/docs/enterprise/llm-safety-and-controls).

---

## 9. Configuration reference

### 9.1 Service & port map

| Port | Service | Host | Role |
|---|---|---|---|
| 5173 | dashboard | platform | Frontend v1 (nginx :80) |
| 5174 | application-frontend | platform | Frontend v2 (nginx) |
| 8088 | application-backend | platform | FastAPI backend v2 (internal :8080) |
| 8003 | app-backend | platform | Legacy NestJS backend v1 (internal :3000) |
| 8080 | object-storage-service | platform | S3-fronting object storage API |
| 8001 | events-service | platform | Events storage API |
| 8005 | case-service | platform | Case/analysis persistence |
| 8008 | chat-service | platform | Intel chat (Bedrock agent) |
| 8010 | engagement-service | platform | Outbound ticketing (Jira/ADO) |
| 8011 | webhook-connector | platform | Webhook ingest → Kafka |
| 8081 | kafka-ui | platform | Kafka web UI (internal :8080) |
| 9092 | kafka | platform | Broker (internal :29092, KRaft) |
| 6379 | redis | platform | Pub/sub + config-plane listener |
| 8000 | llm-service | ML | Bedrock LLM gateway / sensitive-data detection |
| 8090 | parser-service | ML | Content parser |
| 8829 | rapidocr | ML | OCR backend (internal) |
| 27017 | DocumentDB | (managed) | MongoDB-compatible datastore |

### 9.2 Container images

- **Public:** `confluentinc/cp-kafka:8.1.1`, `ghcr.io/kafbat/kafka-ui:v1.4.2`,
  `redis:7.4-alpine`.
- **Axia (ECR):** `object-storage-service`, `event-analyzer`, `enrichment-worker`,
  `connector`, `webhook`, `case-service`, `chat-service`, `events-service`,
  `engagement-service`, `application-backend`, `application-frontend`, `app-backend`,
  `dashboard`; and on the ML host `llm-service`, `parser-service`, `parser-rapidocr`,
  `data-classification`.
  Registry pattern: `<AXIA_ECR_REGISTRY>/<service>:<tag>` (the exact registry is
  provided by Axia). Tags are pinned per release (e.g. `v1.4.0` core / `v1.0.0`
  app-layer); un-pinned entries default to `latest`.

### 9.3 Key environment variables & secrets

Config uses a namespaced convention:
`SERVICES__ENV__<SERVICE>__<VAR>` (non-secret) and
`SERVICES__SECRETS__<SERVICE>__<VAR>` (secret).

| Category | Variables |
|---|---|
| AWS/registry | `ACCOUNT_ID`, `AWS_REGION`, `ECR_REGISTRY` |
| Database | `MONGODB_URI` / per-service `SERVICES__SECRETS__{APP_BACKEND_SERVICE,EVENTS_SERVICE,CHAT_SERVICE,INGESTION_SERVICE}__(MONGODB_URI\|DB_URI)` |
| Object storage | `SERVICES__SECRETS__OBJECT_STORAGE_SERVICE__AWS_ACCESS_KEY_ID` / `__AWS_SECRET_ACCESS_KEY`, `...__URL`, `...__REGION`, `DATA_BUCKET`, `AXIA_BUCKET` |
| Auth | `JWT_SECRET` (required for application-backend), `AUTH_COOKIE_SECURE` |
| LLM | `LLM_MODEL` (Bedrock model id), `PROMPT_PATH`, `AWS_DEFAULT_REGION` |
| Kafka | `KAFKA_BOOTSTRAP_SERVERS`, `KAFKA_SECURITY_PROTOCOL`, `KAFKA_ADVERTISED_HOST` |
| Cross-host | `PARSER_SERVICE_URL`, `LLM_SERVICE_URL` |
| Outlook/365 | `MICROSOFT_TENANT_ID`, `MICROSOFT_CLIENT_ID`, `MICROSOFT_CLIENT_SECRET`, `OUTLOOK_ORG_DOMAIN` |
| Purview DLP (§8.1) | `MICROSOFT_TENANT_ID`, `MICROSOFT_CLIENT_ID`, `MICROSOFT_CLIENT_SECRET`, `PURVIEW_START_DATE`; defaults `MICROSOFT_GRAPH_BASE_URL`, `PURVIEW_PENDING_QUEUE_NAME=dlp.events` |
| Webhook sources — Cursor (§8.2) | `WEBHOOK_ENRICHMENT_SOURCES` (`cursor:<secret>;…`), `WEBHOOK_ENRICHMENT_TOPIC`, `WEBHOOK_DLP_SOURCES`, `WEBHOOK_DLP_TOPIC`, `WEBHOOK_BATCH_SOURCES`, `WEBHOOK_BATCH_TOPIC` |
| SMTP (optional) | `SMTP_HOST/PORT/SECURE/USER/PASSWORD/FROM_ADDRESS/FROM_NAME` |

In production the DocumentDB password comes from **Secrets Manager** and the S3
credentials from **SSM Parameter Store** — both retrieved at deploy time by the config
generator, not stored in the repo.

---

## 10. Operations

- **Connect to a host:** `aws ssm start-session --target <instance-id> --region <region>`
  (or `terraform output ssm_command`). Hosts are in private subnets — there is no SSH.
- **DocumentDB shell from your laptop:** use `documentdb_ssm_port_forward_command` to
  map `localhost:27017` to the cluster, then connect with a TLS client using the RDS
  CA bundle (`authMechanism=SCRAM-SHA-1`).
- **Upgrade a service:** update the image tag in `.env` on the host, then re-run the
  deploy (`docker compose pull && up -d`).
- **Logs:** `docker compose logs -f <service>` on the host; container logs also ship to
  CloudWatch (`/platform/services`, `/ml`).
- **Off-hours shutdown:** the platform EC2 can be tagged `shut-down=true` to opt into an
  off-hours scheduler (optional cost control).

---

## 11. Known gotchas & caveats

1. **Two hosts required.** The platform compose file does not include `llm-service` or
   the parser — those live on the ML host. Both must run and be wired by private-IP
   URLs.
2. **DocumentDB, not plain MongoDB.** Connections require TLS + the RDS CA bundle
   mounted at `/app/certs/global-bundle.pem` and `authMechanism=SCRAM-SHA-1`. URL-encode
   the password (`scripts/encode-mongodb-password.py`).
3. **Bedrock region.** `llm-service` and `chat-service` need `AWS_DEFAULT_REGION` set,
   or they fail with `NoRegionError`. Verify the exact model id against
   `aws bedrock list-foundation-models` — model-id mismatches are a common failure.
4. **The image registry points at Axia's ECR** in the generated `deployment-config.json`
   by design — that is where images are pulled from cross-account, not your account.
   (The bundled config generator and `deploy-v2.sh` are already path-fixed and correctly
   named — no edits needed.)
5. **Region consistency.** Pin one region across the whole deployment — state bucket,
   VPC, DocumentDB, S3, and the deployment config — and set it before you apply.
6. **Cross-account ECR trust is Axia-side.** The host's IAM role grants pull actions,
   but Axia must add your account to each repository policy or pulls will be denied.
7. **AMI / user_data drift.** The EC2 resources set `ignore_changes = [ami, user_data]`
   so Terraform will not replace a running host on AMI updates. Use targeted applies for
   non-EC2 changes.
8. **Purview alerts can lag.** New/modified DLP policies take up to ~3 hours to start
   generating alerts, and app-only reads require `SecurityAlert.Read.All` **with admin
   consent** — a missing consent looks like "authenticated but zero alerts" (§8.1).
9. **Cursor has no cloud-side prompt events.** User prompts are captured only via the
    local `beforeSubmitPrompt` hook deployed to each developer endpoint (configure and
    enforce it via MDM for a tamper-resistant fleet-wide rollout). The hook's forwarder script sets
    `X-Webhook-Secret` itself, so no HMAC shim is needed for this path (§8.2). Capturing
    prompt text is user-input monitoring — treat as a consent/policy matter.

---

## 12. Bundle path reference

Everything the phases above reference, and where it lives inside
`axia-deploy-<version>/`:

| Topic | Path (in the bundle) |
|---|---|
| Bootstrap CloudFormation | `terraform/init/{terraform-backend.yaml,github-actions-role.yaml}` |
| Network stack | `terraform/vpc/` |
| Core infra stack (EC2 / DocumentDB / S3 / IAM) | `terraform/platform/` |
| Host bootstrap script | `terraform/platform/scripts/user_data.sh` |
| Example env config | `terraform/platform/{example.backend.hcl,example.tfvars}` |
| Deploy scripts | `scripts/{generate-deployment-config.sh,deploy-v2.sh,encode-mongodb-password.py}` |
| Client config template | `configs/platform/config.env.example` |
| Platform host compose | `compose/platform/docker-compose.yml` |
| ML host compose | `compose/ml/docker-compose.yml` |

Axia-internal references (not shipped): the `apps-ci`/`tenants` Terraform stacks, the
Terraform source-of-truth map, and the internal deployment-strategy / ports docs.

---

*Deployment guide for the Axia platform. The commands above operate the Axia
deployment bundle; no access to Axia's source repositories is required.*
