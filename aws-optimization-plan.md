# Riverly AWS Optimization Plan
> Last audited: April 7, 2026

---

## 1. Account Verification — Two Separate Accounts Confirmed

| | Fatai (migrating from Flux) | Yerin (Riverly owner) |
|---|---|---|
| **AWS Account ID** | `912921195732` | `132597214585` |
| **IAM User** | `Fatai` | `YERINS` |
| **Region** | `eu-north-1` (Stockholm) | `eu-north-1` (Stockholm) |
| **RDS Endpoint** | `riverly.cxcqeu66ucvt.eu-north-1.rds.amazonaws.com` | `riverly-staging-db.cjsgayq2obtz.eu-north-1.rds.amazonaws.com` |
| **Redis** | Redis Cloud (external, free tier) | ElastiCache `riverly-redis.n1kcro...` |

They are **completely separate AWS accounts**. No shared resources, no cross-account billing.

---

## 2. Current Resource Inventory & Cost Estimate

### Fatai's Account (`912921195732`)

| Resource | Spec | Est. Monthly Cost |
|---|---|---|
| ECS Fargate | 0.25 vCPU, 512MB RAM, 24/7 | ~$9 |
| RDS PostgreSQL | `db.t4g.micro`, 20GB gp2, single-AZ | ~$12 |
| Redis | Redis Cloud free tier (external) | **$0** |
| ECR storage | 43 images (~6.5GB total) | ~$0.65 |
| Secrets Manager | 1 secret | ~$0.40 |
| CloudWatch Logs | ~38MB, no expiry | ~$1 (grows over time) |
| Elastic IP | 1 (attached to Fargate) | ~$0 while in use |
| S3 | `lira-documents-storage`, `docs.creovine.com` | minimal |
| **Estimated Total** | | **~$23–25/month** |

### Yerin's Account (`132597214585`)

| Resource | Spec | Est. Monthly Cost |
|---|---|---|
| ECS Fargate | 0.5 vCPU, 1024MB RAM, 24/7 | ~$18 |
| RDS PostgreSQL | `db.t3.micro`, 20GB gp2, single-AZ | ~$15 |
| ElastiCache Redis | `cache.t3.micro`, 1 node | ~$24 |
| ECR storage | 15 images (~2.2GB total) | ~$0.22 |
| CloudWatch Logs | ~25MB, no expiry | ~$1 (grows over time) |
| S3 | `riverly-staging-assets` | minimal |
| **Estimated Total** | | **~$58–62/month** |

> **Key insight:** Yerin's account costs ~2.5× more than Fatai's, almost entirely due to **ElastiCache** ($24/month) and a **larger Fargate task spec** (2× CPU/RAM). These are the two biggest wins.

---

## 3. Optimization Action Plan

### Priority 1 — High Impact, Low Risk

#### Yerin: Replace ElastiCache with Redis Cloud free tier
**Saves ~$24/month**

ElastiCache `cache.t3.micro` costs ~$24/month. Redis Cloud offers a **30MB free tier** which is more than enough for staging (sessions, OTP caching, etc.).

Steps:
1. Create a free Redis Cloud account at https://redis.com/try-free/
2. Create a free database in `eu-central-1` or `eu-west-1` (closest to Stockholm)
3. Update the env variable: `Redis__ConnectionString=<redis-cloud-host>:<port>,password=<password>`
4. Update the ECS task definition with the new connection string
5. Delete the ElastiCache cluster

#### Yerin: Right-size ECS Fargate task
**Saves ~$9/month**

Current spec (0.5 vCPU / 1024MB) is double what Fatai's staging runs on. Drop to 0.25 vCPU / 512MB — the API is a lightweight .NET staging app. Can always scale up if needed.

```bash
# Update task definition to 256 CPU / 512 memory
# This is done in the task definition JSON or via AWS Console
```

---

### Priority 2 — Medium Impact, Quick Wins

#### Both Accounts: Add ECR Lifecycle Policy
**Prevents unbounded storage growth**

Fatai's account already has 43 images. Without a policy, this grows with every deploy.

**Fatai's account:**
```bash
aws ecr put-lifecycle-policy --repository-name riverly-api --region eu-north-1 \
  --lifecycle-policy-text '{
    "rules": [
      {
        "rulePriority": 1,
        "description": "Keep last 10 tagged images",
        "selection": {
          "tagStatus": "tagged",
          "tagPrefixList": [],
          "countType": "imageCountMoreThan",
          "countNumber": 10
        },
        "action": {"type": "expire"}
      },
      {
        "rulePriority": 2,
        "description": "Remove untagged images after 1 day",
        "selection": {
          "tagStatus": "untagged",
          "countType": "sinceImagePushed",
          "countUnit": "days",
          "countNumber": 1
        },
        "action": {"type": "expire"}
      }
    ]
  }'
```

**Yerin's account:** Same command but with `AWS_ACCESS_KEY_ID=AKIAR5X3I5V4RCWKLS4O`

#### Both Accounts: Set CloudWatch Log Retention
**Prevents logs from accumulating indefinitely**

Neither account has a retention policy on their ECS log group — logs will grow forever.

**Fatai's account:**
```bash
aws logs put-retention-policy \
  --log-group-name /ecs/riverly-task \
  --retention-in-days 30 \
  --region eu-north-1
```

**Yerin's account:**
```bash
AWS_ACCESS_KEY_ID=AKIAR5X3I5V4RCWKLS4O \
AWS_SECRET_ACCESS_KEY=... \
aws logs put-retention-policy \
  --log-group-name /ecs/riverly-api \
  --retention-in-days 30 \
  --region eu-north-1
```

#### Both Accounts: Upgrade RDS from gp2 → gp3
**Saves ~20% on RDS storage cost, no downtime**

`gp2` costs $0.115/GB-month. `gp3` costs $0.092/GB-month for the same or better performance.

**Fatai's account:**
```bash
aws rds modify-db-instance \
  --db-instance-identifier riverly \
  --storage-type gp3 \
  --apply-immediately \
  --region eu-north-1
```

**Yerin's account:**
```bash
aws rds modify-db-instance \
  --db-instance-identifier riverly-staging-db \
  --storage-type gp3 \
  --apply-immediately \
  --region eu-north-1
```

---

### Priority 3 — Security (Must Fix — Not Just Cost)

#### Fatai's Account: Close Port 5432 to the Public Internet
**Security risk — RDS port 5432 is exposed to `0.0.0.0/0`**

Anyone on the internet can attempt to connect to the database. This should only allow traffic from within the VPC.

```bash
# Remove public access to Postgres
aws ec2 revoke-security-group-ingress \
  --group-id sg-0ea66a7c552936587 \
  --protocol tcp --port 5432 \
  --cidr 0.0.0.0/0 \
  --region eu-north-1
```

> Note: After removing this, ECS tasks can still reach RDS because they are in the same VPC/security group. Only external internet access gets blocked.

#### Yerin's Account: Reduce RDS Backup Retention from 7 to 1 Day (for staging)
**7-day backup retention on staging is unnecessary and adds storage cost**

7 days of automated snapshots for a 20GB staging DB = up to 140GB of snapshot storage billed. For staging, 1 day is sufficient.

```bash
aws rds modify-db-instance \
  --db-instance-identifier riverly-staging-db \
  --backup-retention-period 1 \
  --apply-immediately \
  --region eu-north-1
```

---

## 4. Projected Savings After All Changes

### Yerin's Account

| Change | Monthly Saving |
|---|---|
| Replace ElastiCache → Redis Cloud free | ~$24 |
| Right-size Fargate (0.5→0.25 vCPU, 1GB→512MB) | ~$9 |
| RDS gp2 → gp3 | ~$0.46 |
| Reduce RDS backup retention (7→1 day) | ~$2–5 |
| **Total savings** | **~$35–38/month** |

**Yerin's account: $62/month → ~$24/month**

### Fatai's Account

| Change | Monthly Saving |
|---|---|
| ECR lifecycle policy (clean up 33 old images) | ~$0.40 |
| RDS gp2 → gp3 | ~$0.46 |
| CloudWatch 30-day retention | prevents future growth |
| Fix port 5432 | security, not cost |
| **Total savings** | **~$1–2/month** (already well-optimised) |

**Fatai's account: already lean at ~$23–25/month — no major changes needed**

---

## 5. What NOT to Introduce as the Platform Grows

These are the things that caused high bills on Flux and similar setups:

| Service | Why It's Expensive | Alternative |
|---|---|---|
| **Application Load Balancer (ALB)** | $18/month minimum even with no traffic | Use Caddy + DuckDNS (already done) |
| **NAT Gateway** | $32/month + $0.045/GB data charges — #1 hidden AWS cost | Use public subnets for Fargate tasks |
| **Multi-AZ RDS** | Doubles DB cost | Only enable for production, not staging |
| **ElastiCache** | $18–35/month for smallest node | Redis Cloud free tier for staging |
| **CloudFront** | Can add up with data transfer | Only add when you have real traffic |
| **Multiple ECS tasks** | Each task = separate Fargate compute charge | Keep desired count = 1 for staging |
| **Unused Elastic IPs** | $4/month when not attached | Release unattached EIPs immediately |

---

## 6. When Fatai Completes Migration to Riverly AWS

When Fatai fully migrates to the Riverly AWS (Yerin's account `132597214585`), make sure:

1. **One ECR repository** — both share `riverly-api`, use tags to distinguish (e.g. `fatai-sha` vs `yerin-sha`)
2. **Separate task definitions** — register separate task def families if needed (`riverly-task-fatai`, `riverly-task-yerin`)
3. **Shared RDS** is fine for staging — use separate databases/schemas within the same RDS instance to save cost
4. **IAM** — Fatai already exists as a user in Yerin's account (`132597214585`). Assign him least-privilege permissions (ECS + ECR + RDS read, no billing/IAM access)
5. **Delete Fatai's separate AWS account** (`912921195732`) once fully migrated — or at minimum, delete all running resources (RDS, ECS service) to stop billing

---

## 7. Summary Checklist

### Immediate Actions (Do Now)
- [ ] Yerin: Delete ElastiCache, switch to Redis Cloud free tier
- [ ] Yerin: Reduce ECS task from 0.5 vCPU/1GB → 0.25 vCPU/512MB
- [ ] Yerin: Set CloudWatch log retention to 30 days (`/ecs/riverly-api`)
- [ ] Yerin: Change RDS `riverly-staging-db` from gp2 → gp3
- [ ] Yerin: Reduce RDS backup retention from 7 → 1 day
- [ ] Yerin: Add ECR lifecycle policy (keep 10 tagged, delete untagged after 1 day)
- [ ] Fatai: Fix security group — close port 5432 to `0.0.0.0/0`
- [ ] Fatai: Set CloudWatch log retention to 30 days (`/ecs/riverly-task`)
- [ ] Fatai: Change RDS `riverly` from gp2 → gp3
- [ ] Fatai: Add ECR lifecycle policy (keep 10 tagged, delete untagged after 1 day)

### Before Fatai's Migration Completes
- [ ] Decide on shared vs separate RDS databases
- [ ] Set up least-privilege IAM for Fatai in Yerin's account
- [ ] Plan shutdown of Fatai's separate AWS account (`912921195732`)

### Things to Never Add (for staging)
- [ ] No NAT Gateway
- [ ] No Application Load Balancer
- [ ] No Multi-AZ RDS
- [ ] No ElastiCache (use Redis Cloud free tier)
