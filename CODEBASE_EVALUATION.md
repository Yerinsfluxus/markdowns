# Fluxx Backend — Codebase Evaluation Report

**Date:** April 6, 2026  
**Prepared by:** Yerins  
**For:** Management Review Meeting  
**Codebase version:** Latest `dev` branch (commit `ee34914`)

---

## 1. What Is This Report?

This is a technical evaluation of the Fluxx backend system — the engine that powers all of Fluxx's features (user accounts, wallets, payments, stock trading, loans, KYC verification, etc.). Think of it as a health check-up for the software.

We assess: **Does it work? Is it secure? Is it finished? What risks exist?**

---

## 2. System Overview (Plain Language)

Fluxx's backend is built as **8 separate services** (like departments in a company, each responsible for a specific job). They communicate through a shared message system and all connect to the same database.

| Service | What It Does | Status |
|---------|-------------|--------|
| **API Gateway** | Front door — routes all user requests to the right service | ✅ Working |
| **Auth Service** | Login, registration, password reset, OTP verification | ✅ Working (1 gap) |
| **Back Office** | Admin dashboard — settings, compliance, audit logs, GL accounts | ✅ Working |
| **FinOps (Wallet)** | Wallets, money transfers, virtual accounts, virtual cards, bill payments | ✅ Working (2 gaps) |
| **Marketplace** | Stock trading, loan applications, products, support tickets | ✅ Working (2 gaps) |
| **Notification** | Push notifications (mobile), email sending | ⚠️ Partially working |
| **Validation** | KYC (Know Your Customer), identity + document verification | ✅ Working (1 gap) |
| **Eureka Server** | Internal service-finder (helps services locate each other) | ✅ Working |

**Size:** ~54,000 lines of code across 344 files, with 251 API endpoints.

---

## 3. Feature Completeness — Does Everything Work?

### ✅ Fully Completed Features

| Feature | Endpoints | Notes |
|---------|-----------|-------|
| User registration & login | 59 | Email/password, token refresh, password reset |
| Admin user management | Included above | Invite, roles, permissions |
| FIP (Partner) onboarding | Included above | Registration, login, user management |
| Wallet management | 44 | Create, fund, withdraw, lock/unlock wallets |
| Money transfers | Included above | Internal transfers, bank withdrawals via Fincra |
| Virtual accounts | Included above | Create and manage via Fincra |
| Virtual cards | Included above | Issue, fund, unload via BridgeCard |
| Bill payments | Included above | Utility and service payments |
| Stock trading (buy/sell) | 18 | Search, quote, buy, sell, portfolio via Alpaca |
| Stock watchlists | Included above | Add/remove watched stocks |
| Portfolio management | Included above | Positions, history, trade activities |
| Loan applications | 83 (marketplace total) | Apply, match with offers, decision workflow |
| Loan decisions (maker-checker) | Included above | Submit, confirm, batch approve/deny |
| Product management | Included above | CRUD for marketplace products |
| Support tickets | Included above | Create, track, resolve |
| KYC verification | 10 | Document, identity, address verification via SmileID |
| Document upload | Included above | S3 storage for user documents |
| Email notifications | 3 | Templates, send via SMTP |
| Push notifications | Included above | Firebase Cloud Messaging (mobile) |
| Admin security settings | 50 | Transaction limits, KYC config, compliance |
| General Ledger (GL) accounts | Included above | Fund, withdraw, summary — recently updated |
| Audit logs | Included above | Track all admin actions |
| Maker-checker approvals | Included above | Two-person approval for sensitive operations |
| Swagger API docs | All services | Auto-generated, browsable documentation |
| Service health checks | All services | `/health` and `/status` endpoints |
| Rate limiting | Gateway | Prevents API abuse |

### ⚠️ Partially Complete Features (Gaps Found)

| Feature | What's Missing | Impact | Severity |
|---------|---------------|--------|----------|
| **SMS notifications** | SMS provider (Termii) is coded but **no environment configuration is set up** — SMS messages won't actually send | Users won't receive OTP codes by text message | 🟡 Medium |
| **AML screening on deposits** | Code comment says "TODO: trigger AML check when user deposits" — **not implemented** | Regulatory compliance gap — deposits aren't screened for money laundering | 🔴 High |
| **Virtual card transaction tracking** | After funding/unloading a virtual card, the system **doesn't save the provider's reference number** | Cannot reconcile card transactions with BridgeCard records | 🟡 Medium |
| **Stock trade → wallet linking** | When a stock is purchased, the wallet transaction ID is **not recorded** on the trade | Cannot trace which wallet debit funded which stock purchase | 🟡 Medium |
| **Get single transaction by ID** | Endpoint to look up one specific transaction **doesn't exist** (only list/search works) | Users/admins cannot deep-link to a specific transaction | 🟢 Low |

### ❌ Not Yet Started (In Feature Branches, Not Merged)

| Feature | Branch | Status |
|---------|--------|--------|
| **FX currency swap** | `feat/fx-swap` | Code exists in branch, not merged to main codebase |

---

## 4. Bugs & Code Issues Found

### 🔴 Critical Issues (Must Fix Before Go-Live)

#### Issue 1: Production Passwords Exposed in Code Repository

A file containing **real server credentials** was accidentally saved into the code history:
- The actual database server address (IP: `51.20.206.248`)
- Database username and password (`fluxx_app / fluxx_app`)
- Redis cache password

**Why this matters:** Anyone with access to the code repository (current and former employees, contractors) can see these credentials and potentially access the production database directly.

Additionally, live Alpaca stock trading API keys exist in a local file that could be accidentally committed.

**What needs to happen:**
- Change all exposed passwords immediately
- Remove the file from the code repository's history
- Set up a proper secrets management system

---

#### Issue 2: No Automated Testing

Out of 54,000+ lines of code, there are only **3 test files**. Effectively **0% test coverage**.

**Why this matters in plain language:**
- When developers add new features or fix bugs, there's **no safety net** to catch if they accidentally broke something else
- Core financial operations (sending money, buying stocks, processing loans) have **zero automated verification**
- For a fintech app handling real money, this is a significant operational risk

**What needs to happen:**
- Add automated tests for all money-handling features (wallets, transfers, stock trades)
- Make tests mandatory before any code can be deployed

---

#### Issue 3: Services Crash Instead of Recovering

When a supporting system (database, cache, message queue) has even a brief outage, the Fluxx services **crash completely** instead of waiting and retrying. There are 25 places in the code where this happens.

**Why this matters:** A brief 10-second database hiccup would take down all 7 Go services simultaneously, requiring manual restarts.

**What needs to happen:**
- Add retry logic so services wait and reconnect automatically
- Allow services to run in a reduced capacity if non-critical systems are down

---

#### Issue 4: Money Precision Bug

When sending money to a bank account through Fincra, the system converts the amount from a **precise decimal** to a **less precise floating-point number** (`float32`). 

**Why this matters:** A transfer of ₦10,000.15 could be processed as ₦10,000.1504 or ₦10,000.1496 due to floating-point arithmetic. Over thousands of transactions, these rounding errors add up and cause reconciliation problems.

**What needs to happen:**
- Use string-based or integer-based (cents) amounts when communicating with payment providers

---

### 🟡 High Priority Issues

| # | Issue | Impact |
|---|-------|--------|
| 5 | **Hardcoded admin password** — The default admin account is created with a fixed password written directly in the code (`Password3500@`) | Anyone reading the source code can log in as admin |
| 6 | **AML threshold ($10,000) hardcoded** — The anti-money-laundering reporting threshold is a fixed number in the code instead of a configurable setting | Cannot adjust without redeploying code; regulatory risk |
| 7 | **20+ silently ignored errors** — In several places, the code ignores error responses (e.g., when sending SMS, processing webhooks) | Failures happen silently with no alerts |
| 8 | **No staging/test environment** — Code deploys directly from developer branch to production server | No way to test changes in a safe environment first |

---

## 5. Deployment & Infrastructure

### Current Setup
- **Hosting:** AWS EC2 (single server)
- **Database:** PostgreSQL 15 on the same server
- **Containers:** Docker images stored in AWS ECR
- **CI/CD:** GitHub Actions — auto-builds and deploys when code is pushed to `dev` branch

### What's Good
- Each service has its own Docker container
- Automated build and deploy pipeline exists
- Docker images use security best practices (non-root users, minimal base images)
- Monitoring tools are set up (logging, tracing, metrics)

### What's Missing
- **No staging environment** — code goes straight to production
- **No automated tests in pipeline** — broken code can be deployed
- **No security scanning** — vulnerabilities in dependencies aren't checked
- **Single server** — if EC2 instance goes down, everything goes down
- **All services share one database** — no isolation between services

---

## 6. External Integrations (Third-Party Services)

| Provider | What For | Status | Risk |
|----------|---------|--------|------|
| **Alpaca** | Stock trading (US stocks) | ✅ Integrated | API keys exposed locally |
| **Fincra** | Bank transfers, virtual accounts | ✅ Integrated | Money precision issue (see Bug #4) |
| **BridgeCard** | Virtual debit cards | ✅ Integrated | Transaction tracking incomplete |
| **SmileID** | KYC / Identity verification | ✅ Integrated | No issues found |
| **Firebase (FCM)** | Mobile push notifications | ✅ Integrated | Requires credentials file |
| **Termii** | SMS messaging | ⚠️ Coded but not configured | SMS won't send without env setup |
| **AWS S3** | Document storage | ✅ Integrated | No issues found |
| **Wella Health** | Wellness products | ✅ Integrated | No issues found |

---

## 7. Team & Development Activity

### Contributors (by commit count)

| Developer | Commits | Focus Areas |
|-----------|---------|-------------|
| Onyetube Kosisochukwu (Collins) | 336 | Core services, back-office, FIP, GL accounts |
| Marvelous (Vester) | 245 | Auth, wallets, finops, integrations |
| Yerins | 47 | Stock trading, marketplace features |
| GitHub Copilot Bot | 7 | AI-assisted fixes |

### Development Pace

| Month | Commits | Trend |
|-------|---------|-------|
| Nov 2025 | 34 | — |
| Dec 2025 | 48 | ↑ |
| Jan 2026 | 53 | ↑ |
| Feb 2026 | 86 | ↑↑ |
| Mar 2026 | 112 | ↑↑↑ |
| Apr 2026 (so far) | 6 | — |

Development is **accelerating** — the team shipped 3x more in March than November. However, this speed comes with trade-offs: very few tests are being written, and the codebase is accumulating technical debt.

---

## 8. Repository Hygiene

| Metric | Current State | Ideal State | Assessment |
|--------|--------------|-------------|------------|
| Remote branches | 130 | < 20 | ❌ Needs cleanup |
| Stale branches (no activity > 90 days) | 20+ | 0 | ❌ Needs pruning |
| Branch naming convention | Inconsistent (typos, mixed formats) | Consistent (`feat/`, `fix/`, `chore/`) | ⚠️ Needs standardization |
| Committed secrets | 1 `.env` file with production credentials | 0 | ❌ Critical |
| `.gitignore` coverage | Most files covered | All sensitive files | ⚠️ Needs review |

---

## 9. Codebase Numbers at a Glance

| What | Count |
|------|-------|
| Total lines of code | ~54,000 |
| Code files | 344 |
| API endpoints | 251 |
| Database tables/models | 43 |
| External integrations | 8 |
| Test files | 3 out of 344 (**< 1%**) |
| Services | 8 |
| Active developers | 3 |
| Total commits | ~786 |
| CI/CD pipelines | 8 (one per service) |

---

## 10. Overall Scorecard

| Area | Score (out of 5) | What This Means |
|------|:-:|-----------------|
| **Feature completeness** | ⭐⭐⭐⭐ | Most planned features are built and functional. A few gaps remain in SMS, AML, and transaction tracking |
| **Code compiles & builds** | ⭐⭐⭐⭐⭐ | All 7 Go services and the Java service compile successfully with zero errors |
| **Architecture & design** | ⭐⭐⭐⭐ | Well-organized microservices with clear responsibilities. Good use of design patterns |
| **Security** | ⭐⭐ | Exposed credentials in repository history. No security scanning. Hardcoded passwords |
| **Testing & quality assurance** | ⭐ | Near-zero automated tests. No safety net for changes |
| **CI/CD & deployment** | ⭐⭐⭐ | Pipeline exists and works, but no tests, no staging, no security checks |
| **Monitoring & observability** | ⭐⭐⭐⭐ | Good logging, tracing, and metrics infrastructure in place |
| **Code documentation** | ⭐⭐⭐ | Swagger API docs are good. Internal code documentation is sparse |
| **Repository hygiene** | ⭐⭐ | 130 branches, inconsistent naming, stale branches |
| **Resilience & fault tolerance** | ⭐⭐ | Services crash on dependency failures instead of recovering |

---

## 11. Recommended Actions (What Should Happen Next)

### 🔴 Immediate — This Week (Before Any More Deployments)

| # | Action | Why |
|---|--------|-----|
| 1 | **Change all exposed passwords** (database, Redis, Alpaca API keys) | Credentials are visible in code history |
| 2 | **Remove the `.env` file from git** and scrub from history | Stop ongoing exposure |
| 3 | **Fix the money precision bug** in Fincra bank transfers | Prevents financial discrepancies |

### 🟡 Short Term — Next 2-4 Weeks

| # | Action | Why |
|---|--------|-----|
| 4 | **Add automated tests** for wallet, transfers, and stock trading | Prevent silent breakage of money-handling features |
| 5 | **Set up a staging environment** | Test changes before they hit real users |
| 6 | **Complete SMS notification setup** | Users need OTP by text message |
| 7 | **Implement AML screening on deposits** | Regulatory compliance requirement |
| 8 | **Replace crash-on-failure with retry logic** | Prevent full outages from brief hiccups |
| 9 | **Remove hardcoded admin password** from code | Basic security hygiene |

### 🟢 Medium Term — Next 1-3 Months

| # | Action | Why |
|---|--------|-----|
| 10 | **Add test execution and security scanning to CI/CD** | Prevent broken or vulnerable code from deploying |
| 11 | **Clean up 100+ stale branches** | Repository hygiene |
| 12 | **Adopt a database migration tool** | Currently no way to safely update database structure |
| 13 | **Complete virtual card and stock trade tracking** | Needed for financial reconciliation |
| 14 | **Set up proper secrets management** (AWS Secrets Manager or similar) | Never store passwords in code again |

---

## 12. Summary for Leadership

**The Fluxx backend is ~90% feature-complete**, with most core functionality (wallets, payments, stock trading, KYC, loans, admin tools) built and working. The code compiles cleanly, the architecture is well-designed, and the team's development speed is increasing.

**However, the platform is not yet production-ready** for three reasons:

1. **Security** — Real server passwords are exposed in the code repository. This must be fixed immediately.
2. **No testing** — With 54,000 lines of financial code and virtually zero automated tests, every code change risks breaking money-handling features without anyone knowing.
3. **No safety net for deployments** — Code goes directly to production with no staging environment, no tests, and no security scanning.

**Bottom line:** The team has built a lot of good functionality quickly. The next priority should shift from building new features to **hardening what exists** — security, testing, and deployment processes — before scaling further.

---

*This evaluation is based on static code analysis of the repository as of April 6, 2026. It does not include live environment testing or performance benchmarking.*
