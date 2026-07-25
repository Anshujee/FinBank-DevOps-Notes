# FinBank Banking Platform — Complete DevOps Project Guide

> **Project:** MNC-Level Banking Application on AWS  
> **Stack:** EKS + Terraform + Jenkins + ArgoCD + Helm + Docker + Prometheus  
> **Engineer:** Anshu (DevOps Jee) | GitHub: IndiaFinBank  
> **Region:** ap-south-1 (Mumbai) | AWS Account: <YOUR_AWS_ACCOUNT_ID>  
> **Version:** v1.0.0 | Production Readiness: ~85%

---

## Table of Contents

1. [What Is This Project?](#1-what-is-this-project)
2. [Complete Architecture — The Big Picture](#2-complete-architecture)
3. [Tech Stack — Every Tool and Why](#3-tech-stack)
4. [Phase 0-1 — Project Setup and Git Workflow](#4-phase-0-1-project-setup)
5. [Phase 2-3 — Application Development](#5-phase-2-3-application-development)
6. [Phase 3 — Docker Containerization](#6-phase-3-docker)
7. [Phase 4 — Jenkins CI/CD Pipeline](#7-phase-4-jenkins-cicd)
8. [Phase 5A — Terraform Infrastructure as Code](#8-phase-5a-terraform)
9. [Phase 5B — Kubernetes, Helm, and ArgoCD](#9-phase-5b-kubernetes-helm-argocd)
10. [Phase 5C — ALB Ingress and Networking](#10-phase-5c-alb-networking)
11. [Phase 6A-B — Multi-Environment Setup](#11-phase-6ab-multi-environment)
12. [Phase 6C — Monitoring with Prometheus and Grafana](#12-phase-6c-monitoring)
13. [Phase 6D — Production Hardening](#13-phase-6d-production-hardening)
14. [Security — Secrets Management with ESO and IRSA](#14-security-secrets-management)
15. [Complete End-to-End Request Flow](#15-end-to-end-request-flow)
16. [Quick Reference — Commands and Credentials](#16-quick-reference)
17. [Interview Q&A — 15 Key Questions](#17-interview-qa)

---

## 1. What Is This Project?

FinBank is a **production-grade banking application** built entirely on AWS. It is designed as a portfolio project to demonstrate real-world DevOps skills that match what MNC companies expect.

**What the application does:**
- Users can register, login, and manage bank accounts
- Transfer money (NEFT/RTGS/IMPS/UPI style)
- Apply for loans
- View transaction history
- Analytics dashboard showing registration trends, login activity, and transfer volumes

**Why it matters for interviews:**
Every DevOps concept you have learned — Docker, Kubernetes, Terraform, Jenkins, ArgoCD, Helm, Prometheus — is implemented in this project in a way that mirrors real MNC production environments. When an interviewer asks "have you used this in a real project?", the answer is yes.

---

## 2. Complete Architecture

This is how every tool in the project connects to every other tool. Read this until you can explain it without looking.

```
Developer (Mac M1/M2)
        |
        | git push to develop branch
        |
        v
    GitHub (3 repos)
    ├── IndiaFinBank/FinBank-backend    (Spring Boot Java)
    ├── IndiaFinBank/FinBank-frontend   (React + Vite)
    └── IndiaFinBank/FinBank-analytics  (FastAPI Python)
        |
        | triggers webhook
        |
        v
    Jenkins (localhost:9090 — running in Docker)
    9 stages: Checkout → Build → Test → SonarQube →
    Docker Build (local scan) → Trivy Scan →
    Docker Build+Push to ECR → Update Helm Values → Done
        |                           |
        | pushes image              | commits tag update [skip ci]
        v                           v
    AWS ECR                     GitHub Infra Repo
    (Docker images)             (Helm values updated)
                                    |
                                    | ArgoCD watches this repo
                                    v
                                ArgoCD (GitOps)
                                9 Applications:
                                3 services × 3 environments
                                    |
                                    | deploys to
                                    v
                            AWS EKS Cluster (finbank-dev)
                            ┌────────────────────────────┐
                            │  finbank-dev (namespace)   │
                            │  ├── backend pod (1)       │
                            │  ├── frontend pod (1)      │
                            │  └── analytics pod (1)     │
                            │                            │
                            │  finbank-stage (namespace) │
                            │  ├── backend pod (1)       │
                            │  ├── frontend pod (1)      │
                            │  └── analytics pod (1)     │
                            │                            │
                            │  finbank-prod (namespace)  │
                            │  ├── backend pods (2-5)    │
                            │  ├── frontend pods (2-4)   │
                            │  └── analytics pod (1)     │
                            └────────────────────────────┘
                                    |           |
                             connects to        |
                            ┌───────────┐   connects to
                            │RDS MySQL  │   ┌─────────────────┐
                            │3 databases│   │ElastiCache Redis │
                            │dev/stage/ │   │sessions + cache  │
                            │prod       │   └─────────────────┘
                            └───────────┘
                                    ↑
                            ALB (Application Load Balancer)
                            Path-based routing:
                            /api/v1/*      → backend service
                            /api/analytics/* → analytics service
                            /*             → frontend service
                                    ↑
                            Internet (User's Browser)

Secrets: AWS Secrets Manager → External Secrets Operator → K8s Secrets → Pods
Monitoring: Prometheus (scrapes) → Grafana (displays) + AlertManager (alerts)
```

**Layers explained simply:**

- **Layer 1 (You):** Write code, push to GitHub. Everything else is automated.
- **Layer 2 (Jenkins):** Builds, tests, scans for security vulnerabilities, creates Docker images, pushes to ECR.
- **Layer 3 (GitHub + ECR):** GitHub stores code AND Helm values (ArgoCD watches). ECR stores Docker images (EKS pulls from here).
- **Layer 4 (ArgoCD + EKS):** ArgoCD sees the new image tag in GitHub, renders Helm templates, deploys to EKS.
- **Layer 5 (Pods):** Spring Boot backend, Nginx/React frontend, FastAPI analytics — all running as containers.
- **Layer 6 (Data):** RDS MySQL stores permanent data, Redis stores sessions and cache, ALB handles incoming traffic.
- **Layer 7 (Monitoring):** Prometheus collects metrics, Grafana shows dashboards, HPA scales pods, PDB protects availability.

---

## 3. Tech Stack

| Category | Tool | What It Does in FinBank |
|---|---|---|
| **Cloud** | AWS (ap-south-1) | Everything runs here |
| **Container Orchestration** | AWS EKS (K8s 1.31) | Runs all application pods |
| **Infrastructure as Code** | Terraform | Creates VPC, EKS, RDS, ECR, Redis with one command |
| **CI/CD** | Jenkins | 9-stage build pipeline per service |
| **GitOps** | ArgoCD | Deploys from Git, auto-syncs, self-heals |
| **Packaging** | Helm 3 | One template, three environments |
| **Registry** | AWS ECR | Stores Docker images privately |
| **Database** | AWS RDS MySQL 8.0 | 3 databases (dev/staging/prod) |
| **Cache** | AWS ElastiCache Redis | Sessions, OTP tokens, rate limiting |
| **Secrets** | AWS Secrets Manager + ESO | Zero secrets in Git |
| **Load Balancer** | AWS ALB + Ingress Controller | Path-based routing to services |
| **Monitoring** | Prometheus + Grafana | Metrics and dashboards |
| **Code Quality** | SonarQube | Code smell and vulnerability analysis |
| **Security Scanning** | Trivy | Blocks CRITICAL/HIGH CVEs before ECR push |
| **Backend** | Spring Boot 3.x (Java 21) | 18 REST APIs, JWT auth |
| **Frontend** | React + Vite + Nginx | Banking web UI |
| **Analytics** | FastAPI (Python 3.12) | 5 analytics endpoints |

---

## 4. Phase 0-1: Project Setup

### What was built
- GitHub organization: **IndiaFinBank** with 3 application repos + 1 infra repo
- Jira project with Epics and Sprint tickets (ticket format: FBP-XXX)
- GitFlow branching strategy
- Developer workstation setup

### GitFlow — The Branch Strategy

GitFlow is a way to organise work so multiple developers can work without overwriting each other.

```
main          ←── production code, always stable, tagged with version numbers
  |
develop       ←── integration branch, all completed features merge here
  |
feature/FBP-14-spring-boot-setup   ←── one branch per Jira ticket
feature/FBP-35-docker-setup
feature/FBP-88-terraform-infra
```

**The workflow for every piece of work:**
1. Pick a Jira ticket (e.g., FBP-14)
2. Create a branch: `git checkout -b feature/FBP-14-spring-boot-setup`
3. Write code on that branch
4. Push to GitHub: `git push origin feature/FBP-14-spring-boot-setup`
5. Open a Pull Request to merge into `develop`
6. Code review happens on the PR
7. Merge → Jira ticket moves to Done

**Why this matters in interviews:**
If you say "I just push to main", interviewers immediately know you haven't worked in a team environment. GitFlow shows professional team development practices.

### Commit Convention

```
feat(FBP-14): add Spring Boot skeleton with JWT auth
fix(FBP-134): upgrade Spring Boot 3.5.14 to fix CVE
CI: Update image tag to 1.0.0-44 [skip ci]
```

The `[skip ci]` at the end is important. When Jenkins updates Helm values and pushes to GitHub, without `[skip ci]` it would trigger another pipeline run — infinite loop. This tag tells Jenkins to ignore that commit.

### Release Process

When ready to release v1.0.0:
```bash
git checkout -b release/v1.0.0    # from develop
# test and fix bugs here
git checkout main && git merge release/v1.0.0
git tag -a v1.0.0 -m "Release v1.0.0: first production release"
git push origin v1.0.0
git checkout develop && git merge main  # sync back
git branch -d release/v1.0.0
```

#### Why a Separate Release Branch? — The Real Company Way

Think of it like this. Your `develop` branch is a kitchen where chefs (developers) are constantly cooking new dishes. You cannot serve food directly from a live kitchen to customers — it's messy, things change every minute. So before service, you take a fixed set of dishes to the dining room, present them, and if something is wrong you fix only that plate. The `release` branch is that dining room — a frozen snapshot from `develop` that you polish before handing to customers (production).

**Step-by-step explanation of the real company flow:**

**Step 1 — Feature Freeze (decide what goes in the release)**

In a real company, the Product Manager and Engineering Lead sit together and decide: "v1.0.0 will contain these 12 features. Everything else waits for v1.1.0." This is called a **feature freeze**. Once decided, developers stop adding new features to `develop` for this release cycle.

```
develop branch at the time of feature freeze:
FBP-14 ✅ Spring Boot setup
FBP-35 ✅ Docker multi-arch build
FBP-88 ✅ Terraform EKS setup
FBP-102 ✅ Jenkins pipeline
FBP-120 🚧 New loan calculator (NOT READY — excluded from v1.0.0)
```

**Step 2 — Cut the release branch from develop**

```bash
git checkout develop               # make sure you are on develop
git pull origin develop            # get latest code
git checkout -b release/v1.0.0    # create release branch from here
git push origin release/v1.0.0    # push to GitHub so team can see it
```

At this point `release/v1.0.0` is an exact copy of `develop` at this moment. Developers continue pushing new features to `develop` — those changes do NOT affect the release branch. The release branch is now **isolated**.

**Step 3 — QA testing happens on the release branch**

The QA (Quality Assurance) team deploys `release/v1.0.0` to the **staging environment** and runs full test cycles:
- Functional testing (does login work? does transfer work?)
- Regression testing (did anything that worked before now break?)
- Performance testing (how many users can it handle?)
- Security testing (penetration testing, vulnerability scan)

In FinBank, this means:
```bash
# Jenkins pipeline runs against release/v1.0.0 branch
# Deploys to finbank-stage namespace
# QA team tests all 18 APIs manually + automated tests
```

**Step 4 — Only bug fixes go into the release branch**

During testing, QA finds bugs. Developers fix ONLY those bugs directly on `release/v1.0.0`. No new features allowed here.

```bash
# Developer fixes a bug found during QA
git checkout release/v1.0.0
git checkout -b fix/login-timeout-bug     # small fix branch
# fix the bug
git commit -m "fix(FBP-145): increase JWT timeout to prevent premature logout"
# merge fix back to release branch via Pull Request
git checkout release/v1.0.0
git merge fix/login-timeout-bug
git push origin release/v1.0.0
```

**Step 5 — Release approval (Change Advisory Board in large companies)**

In MNCs like TCS, Infosys, or a bank, there is a **CAB meeting (Change Advisory Board)** before every production release. Senior engineers, operations team, security team, and management review:
- What is changing?
- What is the rollback plan if it fails?
- What is the deployment window? (usually nights or weekends — low traffic)
- Who is on-call during the deployment?

Only after CAB approval does the deployment to production proceed.

**Step 6 — Merge to main and tag**

Once QA signs off and CAB approves:

```bash
git checkout main
git pull origin main                              # ensure main is up to date
git merge --no-ff release/v1.0.0                 # --no-ff keeps history clean
git tag -a v1.0.0 -m "Release v1.0.0: first production release"
git push origin main
git push origin v1.0.0                           # push the tag separately
```

The `--no-ff` flag (no fast-forward) creates a **merge commit** even if it could skip one. This keeps a visible record in git history that "at this point, v1.0.0 was released". Without it, git rewrites history and you lose the release boundary.

The **tag** `v1.0.0` is a permanent bookmark on the commit. Even 2 years later, you can run `git checkout v1.0.0` and get the exact code that was in production on release day. Tags are used for:
- Rollback (checkout old tag and redeploy)
- Audit trail (what code was running on a specific date)
- Docker image naming (image tag `v1.0.0` maps to git tag `v1.0.0`)

**Step 7 — Sync bug fixes back to develop**

The bugs fixed in `release/v1.0.0` must go back into `develop`, otherwise those bugs will reappear in the next release:

```bash
git checkout develop
git merge main                    # main now has release + all bug fixes
git push origin develop
```

**Step 8 — Clean up and declare the release done**

```bash
git branch -d release/v1.0.0           # delete local release branch
git push origin --delete release/v1.0.0  # delete from GitHub too
```

The release branch is temporary — it served its purpose. Keeping it around causes confusion about where to merge new work.

#### The Full Picture — All Branches Together

```
main        ────●─────────────────────────────────●── (tagged v1.0.0)
                 \                               /
release/v1.0.0    ●───●(bug fix)───●(bug fix)──●   (QA tested, then merged to main)
                 /
develop     ────●────●────●────●────●────●────●──── (active development continues)
                 \   /    \   /    \   /
feature       feat1  feat2  feat3  feat4  (merged via Pull Requests)
```

#### Semantic Versioning — What v1.0.0 Means

```
v  1  .  0  .  0
   |     |     |
   |     |     └── PATCH — bug fix, no new features (v1.0.1, v1.0.2)
   |     └──────── MINOR — new features, backward compatible (v1.1.0, v1.2.0)
   └────────────── MAJOR — breaking changes, APIs changed (v2.0.0)
```

Real examples from FinBank:
- `v1.0.1` — fixed the BCrypt $2b$ bug (patch, no new feature)
- `v1.1.0` — added loan calculator feature (minor, new feature added)
- `v2.0.0` — rewrote authentication to use OAuth2 (major, breaking API change)

#### How ArgoCD Fits Into This

In FinBank, once `main` is tagged `v1.0.0`, the Jenkins pipeline triggers on `main` and builds a Docker image tagged `v1.0.0`. It commits that tag to `values-prod.yaml`. ArgoCD detects the change and deploys `v1.0.0` to `finbank-prod` namespace automatically. The entire deployment is hands-off after the git tag is pushed.

```
git tag v1.0.0 + git push
    → Jenkins pipeline on main branch
    → Docker image: finbank-backend:v1.0.0 pushed to ECR
    → values-prod.yaml updated: tag: v1.0.0
    → ArgoCD detects change → deploys to finbank-prod
    → Production is live with v1.0.0
```

---

## 5. Phase 2-3: Application Development

### The 3-Tier Architecture

```
Tier 1 — Frontend (React)
   What the customer sees. Runs in the user's browser, NOT in Kubernetes.
   Makes HTTP calls to backend.

Tier 2 — Backend (Spring Boot Java)
   The brains. Validates requests, enforces business rules,
   talks to database and Redis, returns responses.
   JWT authentication happens here.

Tier 3 — Database (MySQL + Redis)
   MySQL: permanent data (users, accounts, transactions, loans)
   Redis: temporary data (JWT blacklist, OTP with TTL, session cache)
```

**Why frontend can't talk to database directly:**
If React called MySQL directly, anyone could open browser DevTools and run SQL queries against your bank. The backend is the security layer — every request is validated, authenticated, and authorised before touching data.

### The 3 Microservices

**FinBank-backend (Spring Boot Java 21)**
- 18 REST API endpoints
- JWT authentication with Spring Security
- BCrypt password hashing (uses `$2a$` prefix — important for Spring compatibility)
- Connects to MySQL (for data) and Redis (for session blacklist and caching)
- API context path: `/api/v1/` — all endpoints start with this

**FinBank-frontend (React 18 + Vite + Nginx)**
- Single Page Application (SPA)
- Pages: Login, Register, Dashboard, Transactions, Transfer, Loans
- After initial page load, JavaScript runs in the user's browser
- Makes API calls to `/api/v1/*` which the ALB routes to the backend

**FinBank-analytics (FastAPI Python 3.12)**
- 5 analytics endpoints:
  - `/api/analytics/registrations` — registration trends
  - `/api/analytics/logins` — login activity
  - `/api/analytics/transfers` — transfer volumes
  - `/api/analytics/active-accounts` — active users
  - `/api/analytics/health` — health check
- Reads from the same MySQL database (read-only)

### JWT Authentication Flow

```
1. User posts username + password to /api/v1/auth/login
2. Spring Security checks password against BCrypt hash in MySQL
3. If correct, Spring generates JWT token:
   - Contains: user ID, role, expiry (24 hours)
   - Signed with JWT_SECRET (from AWS Secrets Manager)
4. Token returned to browser
5. Browser stores token, sends as header on every subsequent request:
   Authorization: Bearer eyJhbGciOiJIUzI1NiJ9...
6. Spring Security's JwtAuthFilter intercepts every request:
   - Extracts token from header
   - Validates signature
   - Checks not expired
   - Checks not in Redis blacklist (for logged-out tokens)
7. If valid → request proceeds. If invalid → 401 Unauthorized
```

**Why Redis for token blacklist:**
JWT tokens can't be "cancelled" once issued because the server doesn't store them. When a user logs out, we add their token to Redis with a TTL equal to the remaining token lifetime. Even if someone steals the token, it's blocked immediately on logout.

---

## 6. Phase 3: Docker

### Multi-Stage Build — Why Image Size Matters

**Without multi-stage (bad approach):**
```dockerfile
FROM maven:3.9-eclipse-temurin-21
COPY . .
RUN mvn clean package
# Image size: ~800MB
# Contains: Maven, JDK, all source code, test files, build tools
# Problem: 700MB of stuff you don't need to run the app
```

**With multi-stage (what we do):**
```dockerfile
# Stage 1 — Builder (discarded after build)
FROM maven:3.9.6-eclipse-temurin-21-alpine AS builder
WORKDIR /app
COPY pom.xml .
RUN mvn dependency:go-offline -B    # Cache dependencies separately
COPY src ./src
RUN mvn clean package -DskipTests -B
# This stage: ~800MB. Never goes to ECR.

# Stage 2 — Runtime (this is what goes to production)
FROM eclipse-temurin:21-jre-alpine AS runtime
RUN addgroup -S finbank && adduser -S finbank -G finbank  # non-root user
WORKDIR /app
COPY --from=builder /app/target/*.jar app.jar  # Copy ONLY the JAR
RUN chown -R finbank:finbank /app
USER finbank
EXPOSE 8080
ENV JAVA_OPTS="-Xms256m -Xmx512m -XX:+UseContainerSupport"
ENTRYPOINT ["sh", "-c", "java $JAVA_OPTS -jar app.jar"]
# Final image: ~200MB (75% smaller)
# Contains: JRE + app JAR only. Nothing else.
```

**Key decisions explained:**
- `dependency:go-offline` before copying source: if `pom.xml` doesn't change, Maven cache layer is reused → saves 2-3 minutes per build
- `non-root user (finbank)`: security best practice — if someone exploits the app, they don't get root access to the container
- `UseContainerSupport`: tells JVM to respect container memory limits. Without this, JVM sees full node RAM and allocates too much → OOMKilled

### Multi-Architecture Builds — The Apple Silicon Problem

Mac M1/M2 uses **ARM64** (Apple Silicon). AWS EKS EC2 nodes use **AMD64** (Intel x86_64). These are two different CPU instruction sets — like two different languages.

**Without multi-arch:**
- Mac builds ARM64 image
- EKS tries to run ARM64 on AMD64 node
- Pod crashes with: `exec format error` — the CPU literally cannot execute the binary

**With Docker Buildx (what we do):**
```bash
# Create a special builder that can build for multiple architectures
docker buildx create --name finbank-multiarch-builder --driver docker-container --use

# Bootstrap it
docker buildx inspect finbank-multiarch-builder --bootstrap

# Build for BOTH architectures and push
docker buildx build \
  --platform linux/amd64,linux/arm64 \
  --tag <YOUR_AWS_ACCOUNT_ID>.dkr.ecr.ap-south-1.amazonaws.com/finbank-backend:1.0.0-44 \
  --push \
  .

# Cleanup
docker buildx rm finbank-multiarch-builder || true
```

The result in ECR is a **manifest list** — a wrapper that contains two images. When EKS pulls the image, AWS automatically serves the AMD64 version. When your Mac pulls it, it serves the ARM64 version. One tag, two architectures.

### Local Development with Docker Compose

For developers to run the full stack locally without AWS:
```yaml
# Infra_FinBank/local/docker-compose.yml
services:
  mysql:
    image: mysql:8.0
    environment:
      MYSQL_ROOT_PASSWORD: rootpassword
      MYSQL_DATABASE: finsecure_db
    volumes:
      - ./init-scripts:/docker-entrypoint-initdb.d
  redis:
    image: redis:7-alpine
  backend:
    build: ../../FinBank-backend
    depends_on: [mysql, redis]
    environment:
      SPRING_DATASOURCE_URL: jdbc:mysql://mysql:3306/finsecure_db
  frontend:
    build: ../../FinBank-frontend
    ports:
      - "3000:80"
```

**Important:** Jenkins runs INSIDE a Docker container on your Mac. Start it with `--dns 8.8.8.8` so it can resolve external hostnames (Docker Hub, GitHub):
```bash
docker run -d \
  --name jenkins \
  --dns 8.8.8.8 \
  --dns 8.8.4.4 \
  -p 9090:8080 \
  -v jenkins_home:/var/jenkins_home \
  -v /var/run/docker.sock:/var/run/docker.sock \
  jenkins/jenkins:lts
```

---

## 7. Phase 4: Jenkins CI/CD Pipeline

### Pipeline Overview

Each of the 3 services has its own Jenkins pipeline. The backend has 9 stages. Every code push to `develop` triggers the pipeline automatically.

```
Code push to develop
        ↓
Stage 1: Checkout — git clone the repo
        ↓
Stage 2: Install Tools — download AWS CLI + Trivy
         (architecture-aware: detects ARM64 vs AMD64)
        ↓
Stage 3: Build — mvn clean package -DskipTests
         JAR file created in target/
        ↓
Stage 4: Unit Tests — mvn test
         If any test fails → pipeline STOPS
        ↓
Stage 5: SonarQube — code quality analysis
         (wrapped in catchError so it doesn't block if no server)
        ↓
Stage 6: Docker Build (local) — build amd64 image for scanning
         docker buildx build --platform linux/amd64 --load
        ↓
Stage 7: Trivy Security Scan — scan for CVEs
         --exit-code 1 --severity CRITICAL,HIGH
         If CRITICAL or HIGH found → pipeline FAILS, image NOT pushed
        ↓
Stage 8: Docker Build + Push to ECR — multi-arch build
         linux/amd64,linux/arm64
         Tag: 1.0.0-{BUILD_NUMBER}
        ↓
Stage 9: Update Helm Chart in GitHub
         git clone Infra_FinBank
         sed updates image tag in ALL 3 values files
         git commit -m "CI: Update image tag to 1.0.0-44 [skip ci]"
         git push
         ArgoCD detects change → deploys
```

**Total time from push to running pod: ~5-7 minutes**

### Jenkins Credentials — Exact Strings Matter

Jenkins stores credentials with exact IDs. One character difference breaks everything.

```groovy
// In Jenkinsfile
withCredentials([
    string(credentialsId: 'aws-access-key-id', variable: 'AWS_ACCESS_KEY_ID'),
    string(credentialsId: 'aws-secret-access-key', variable: 'AWS_SECRET_ACCESS_KEY'),
    usernamePassword(
        credentialsId: 'github credentials',   // note: SPACE not HYPHEN
        usernameVariable: 'GIT_USER',
        passwordVariable: 'GIT_TOKEN'
    )
])
```

**The credential ID `'github credentials'` (with a space) must exactly match what is stored in Jenkins → Manage Jenkins → Credentials.**

### Trivy Security Gate

Trivy is an open-source container security scanner. It checks your Docker image against a database of known CVEs (Common Vulnerabilities and Exposures).

```bash
trivy image \
  --exit-code 1 \
  --severity CRITICAL,HIGH \
  --no-progress \
  finbank-backend:1.0.0-44

# If CVEs found:
# 2026-07-19 CRITICAL CVE-2026-40973 spring-webmvc...
# Pipeline FAILS. Image is NOT pushed to ECR. Production is protected.
```

**Real incident:** Build #43 failed because Trivy found two HIGH CVEs:
- `CVE-2026-40973` — Spring Boot temp directory vulnerability
- `CVE-2026-42583` — Netty resource exhaustion

**Fix:** Upgraded Spring Boot `3.5.13 → 3.5.14` and added `netty.version=4.1.133.Final` override in `pom.xml`. Lesson: CVEs appear continuously. Trivy in CI/CD catches them before they reach production.

### Stage 9 — The GitOps Bridge

Stage 9 is the most important stage. It is what connects Jenkins (CI) to ArgoCD (CD).

```bash
# Jenkins clones the infra repo
git clone https://github.com/IndiaFinBank/Infra_FinBank.git

# Updates image tag in ALL 3 environments in one commit
sed -i 's|  tag:.*|  tag: 1.0.0-44|' helm/finbank-backend/values.yaml
sed -i 's|  tag:.*|  tag: 1.0.0-44|' helm/finbank-backend/values-staging.yaml
sed -i 's|  tag:.*|  tag: 1.0.0-44|' helm/finbank-backend/values-prod.yaml

git add helm/finbank-backend/values.yaml \
        helm/finbank-backend/values-staging.yaml \
        helm/finbank-backend/values-prod.yaml

git commit -m "CI: Update finbank-backend image tag to 1.0.0-44 [skip ci]"
git push
```

ArgoCD watches the repo, sees the new tag, and automatically deploys. The `[skip ci]` prevents an infinite loop where the push triggers another pipeline.

---

## 8. Phase 5A: Terraform Infrastructure as Code

### What is IaC and Why Does It Matter?

Imagine you want to set up a big office building with electricity, plumbing, rooms, and security systems. You could hire workers to do it all by hand — but if the building burns down, they have to remember everything and do it again manually, differently each time.

**Terraform is like an architectural blueprint.** You write exactly what you want in code files, and Terraform reads those files and automatically builds everything in AWS. If you destroy it and run again — identical result, every time.

**Without IaC (manual clicks in AWS Console):**
- Nobody knows what was configured
- Can't reproduce it — every environment is different
- No audit trail of who changed what
- Takes hours to rebuild after a disaster

**With Terraform:**
- Every resource is defined in code files (version controlled in Git)
- `terraform apply` creates everything in 15 minutes
- After `terraform destroy`, run `apply` again — identical environment
- Peer review infrastructure changes via Pull Requests
- Full audit trail in Git history

---

### Terraform Module Structure

Think of it like a recipe book:

```
aws/terraform/
├── versions.tf     ← "what tools do we need?" (Terraform + AWS plugin versions)
├── variables.tf    ← "what are our settings?" (regions, sizes, passwords)
├── main.tf         ← "the main recipe" (calls all 6 modules)
├── outputs.tf      ← "what do we get back?" (URLs, endpoints, IDs)
├── databases.tf    ← "special step after cooking" (auto-creates MySQL databases)
├── environments/
│   └── dev/
│       └── terraform.tfvars   ← "actual values for dev" (like filling in blanks)
└── modules/
    ├── vpc/           # Module 1: The network
    ├── eks/           # Module 2: Kubernetes cluster
    ├── rds/           # Module 3: MySQL database
    ├── ecr/           # Module 4: Docker image storage
    ├── elasticache/   # Module 5: Redis cache
    └── secretsmanager/ # Module 6: Secrets vault
```

**How modules wire together — each module's output feeds the next module's input:**

```hcl
# main.tf — modules share outputs as inputs
module "vpc" {
  source = "./modules/vpc"
  vpc_cidr = "10.0.0.0/16"
}

module "eks" {
  source             = "./modules/eks"
  vpc_id             = module.vpc.vpc_id          # VPC output feeds EKS input
  private_subnet_ids = module.vpc.private_subnet_ids
}

module "rds" {
  source             = "./modules/rds"
  vpc_id             = module.vpc.vpc_id
  private_subnet_ids = module.vpc.private_subnet_ids
}

module "secretsmanager" {
  db_host           = module.rds.db_host               # from RDS
  redis_host        = module.elasticache.redis_endpoint # from ElastiCache
  oidc_provider_arn = module.eks.oidc_provider_arn     # from EKS
}
```

It is like a chain — VPC is created first, then EKS and RDS use VPC's outputs, then Secrets Manager uses outputs from RDS, ElastiCache, and EKS together.

---

### File: `versions.tf` — The Foundation

```hcl
terraform {
  required_version = ">= 1.0"
  required_providers {
    aws  = { source = "hashicorp/aws", version = "~> 5.0" }
    tls  = { source = "hashicorp/tls", version = "~> 4.0" }
    null = { source = "hashicorp/null", version = "~> 3.0" }
  }

  backend "s3" {
    bucket         = "finbank-terraform-state-860616633001"
    key            = "finbank/dev/terraform.tfstate"
    region         = "ap-south-1"
    dynamodb_table = "finbank-terraform-locks"
    encrypt        = true
  }
}

provider "aws" {
  region = var.aws_region
  default_tags {
    tags = { Project = "FinBank", Environment = var.environment, ManagedBy = "Terraform" }
  }
}
```

**What each part means:**

- **`required_version`** — minimum Terraform version. Like saying "this recipe needs at least oven model 2020."
- **`required_providers`** — plugins Terraform needs:
  - `aws` — the main plugin that creates EC2, S3, EKS, RDS etc.
  - `tls` — needed to read the EKS OIDC certificate thumbprint (a security fingerprint)
  - `null` — enables `null_resource` which runs shell commands after infrastructure is created (used in `databases.tf`)
- **`backend "s3"`** — where Terraform saves its memory (called **state**):
  - The S3 bucket stores `terraform.tfstate` — this maps your Terraform code to real AWS resources. Without it, Terraform doesn't know what it already built.
  - The DynamoDB table is a lock — prevents two people (or two Jenkins jobs) from running `terraform apply` at the same time, which would cause chaos.
  - **NEVER delete these.** If the state file is lost, Terraform thinks nothing exists and will try to create duplicates.
- **`provider "aws"`** — tells Terraform which AWS region to use and adds automatic tags to every single resource created. Every EC2, security group, subnet, etc. automatically gets tagged `Project=FinBank`, `ManagedBy=Terraform`. This makes cost tracking and auditing easy.

---

### File: `variables.tf` — The Settings

Variables are like blank fields in a form. You define what each field is, and fill in the values later via `terraform.tfvars` or environment variables.

```hcl
variable "db_password" {
  type      = string
  sensitive = true    # Never print this in logs or terminal output
}
```

Key variables defined:

| Variable | Purpose | Default Value |
|---|---|---|
| `aws_region` | Which AWS region | `ap-south-1` (Mumbai) |
| `environment` | dev / staging / prod | No default — must be provided |
| `project_name` | Prefix for all resource names | `finbank` |
| `vpc_cidr` | The network address range | `10.0.0.0/16` |
| `availability_zones` | Which data centers to use | `ap-south-1a`, `ap-south-1b` |
| `public_subnet_cidrs` | IP ranges for public subnets | `10.0.1.0/24`, `10.0.2.0/24` |
| `private_subnet_cidrs` | IP ranges for private subnets | `10.0.3.0/24`, `10.0.4.0/24` |
| `eks_node_instance_type` | Size of Kubernetes worker VMs | `t3.medium` |
| `db_instance_class` | Size of MySQL database VM | `db.t3.micro` |
| `db_password` | MySQL password — **sensitive** | Must be set via env variable |
| `jwt_secret` | Secret for signing login tokens — **sensitive** | Must be set via env variable |
| `redis_node_type` | Size of Redis cache VM | `cache.t3.micro` |

**Why `sensitive = true`?** For `db_password` and `jwt_secret`, Terraform will never print the value in terminal output or logs. The values come from environment variables — never from files committed to Git:
```bash
export TF_VAR_db_password='yourpassword'
export TF_VAR_jwt_secret='yoursecret'
```

---

### File: `environments/dev/terraform.tfvars` — The Actual Values

This is where you fill in the blanks defined in `variables.tf`. Think of `variables.tf` as the form template and `terraform.tfvars` as the filled-in form.

```hcl
environment  = "dev"
project_name = "finbank"
aws_region   = "ap-south-1"

vpc_cidr           = "10.0.0.0/16"
availability_zones = ["ap-south-1a", "ap-south-1b"]

eks_node_instance_type = "t3.medium"
eks_node_min_size      = 1
eks_node_max_size      = 2
eks_node_desired_size  = 1

db_instance_class = "db.t3.micro"
db_name           = "finsecure_db"
db_username       = "finsecure_user"
# db_password NOT here — comes from TF_VAR_db_password environment variable
```

For a staging environment, you would have a different `.tfvars` file with bigger instance sizes and `environment = "staging"`. This is how the same Terraform code deploys to different environments without changing any module code.

---

### File: `main.tf` — The Orchestrator

This is the brain. It calls all 6 modules and wires their outputs together as inputs for dependent modules.

```hcl
module "vpc"         { source = "./modules/vpc"; ... }
module "eks"         { source = "./modules/eks";  vpc_id = module.vpc.vpc_id; ... }
module "rds"         { source = "./modules/rds";  vpc_id = module.vpc.vpc_id; ... }
module "ecr"         { source = "./modules/ecr";  ... }
module "elasticache" { source = "./modules/elasticache"; vpc_id = module.vpc.vpc_id; ... }
module "secretsmanager" {
  db_host           = module.rds.db_host
  redis_host        = module.elasticache.redis_endpoint
  oidc_provider_arn = module.eks.oidc_provider_arn
  oidc_issuer_url   = module.eks.oidc_issuer_url
}
```

Terraform is smart enough to figure out the order automatically based on these dependencies — VPC first, then EKS and RDS in parallel, then Secrets Manager last.

---

### File: `outputs.tf` — The Results

After `terraform apply` finishes, outputs print useful values used by Jenkins and other tools:

```hcl
output "vpc_id"            { value = module.vpc.vpc_id }
output "eks_cluster_name"  { value = module.eks.cluster_name }
output "rds_endpoint"      { value = module.rds.db_endpoint;           sensitive = true }
output "ecr_backend_url"   { value = module.ecr.backend_repository_url }
output "redis_endpoint"    { value = module.elasticache.redis_endpoint; sensitive = true }
output "eso_irsa_role_arn" { value = module.secretsmanager.eso_irsa_role_arn }
```

`sensitive = true` on `rds_endpoint` and `redis_endpoint` means they will not appear in Jenkins logs. You retrieve them manually with `terraform output redis_endpoint`. Jenkins uses these values to fill in Helm chart `values.yaml` files for each environment.

---

### VPC Architecture

```
VPC: 10.0.0.0/16  (65,536 private IP addresses)
├── Public Subnets (ALB lives here — internet-facing)
│   ├── 10.0.1.0/24 — ap-south-1a
│   └── 10.0.2.0/24 — ap-south-1b
│   Tags: kubernetes.io/role/elb=1  (ALB controller discovers these subnets)
│
├── Private Subnets (EKS nodes, RDS, Redis — no direct internet access)
│   ├── 10.0.3.0/24 — ap-south-1a
│   └── 10.0.4.0/24 — ap-south-1b
│   Tags: kubernetes.io/role/internal-elb=1
│
├── Internet Gateway → attached to VPC, routes public subnet traffic to internet (two-way)
├── NAT Gateway → in public subnet, allows private subnet resources to reach internet (outbound only)
│   (so EKS nodes can pull Docker images from ECR, install OS packages, etc.)
│   (but nobody from the internet can reach private resources — ever)
└── Route Tables
    ├── Public RT:  0.0.0.0/0 → Internet Gateway
    └── Private RT: 0.0.0.0/0 → NAT Gateway
```

**Why private subnets for EKS nodes and RDS:**
RDS and EKS worker nodes are not directly reachable from the internet. You cannot SSH into the nodes. You cannot connect to MySQL directly from your laptop. They only communicate with other resources inside the VPC. This is a fundamental security principle for banking applications.

---

### Module 1: VPC — The Network (`modules/vpc/main.tf`)

**What it creates:** The entire private network that all FinBank resources live in.

Think of VPC as a building. Everything inside is isolated from the outside world. Subnets are like different floors. The Internet Gateway is the front door. The NAT Gateway is a one-way door from inside to outside.

**`aws_vpc.main`**
```hcl
resource "aws_vpc" "main" {
  cidr_block           = var.vpc_cidr    # 10.0.0.0/16
  enable_dns_hostnames = true             # Required for EKS — nodes must resolve DNS names
  enable_dns_support   = true
}
```
The VPC address range `10.0.0.0/16` means 65,536 possible private IP addresses. `enable_dns_hostnames = true` is required because EKS nodes need to resolve each other's DNS names to communicate.

**`aws_subnet.public` — created twice using `count`**
```hcl
resource "aws_subnet" "public" {
  count             = length(var.public_subnet_cidrs)  # count = 2, creates one subnet per AZ
  cidr_block        = var.public_subnet_cidrs[count.index]
  availability_zone = var.availability_zones[count.index]
  map_public_ip_on_launch = true   # Resources here automatically get a public IP

  tags = {
    "kubernetes.io/role/elb" = "1"   # CRITICAL — EKS ALB controller finds these subnets
  }
}
```
- `count = 2` creates one subnet in each Availability Zone (`ap-south-1a` and `ap-south-1b`)
- `map_public_ip_on_launch = true` means anything launched here gets a public IP automatically
- The `kubernetes.io/role/elb = 1` tag is **critical** — the AWS Load Balancer Controller reads this tag to discover which subnets to place internet-facing ALBs in
- **Public subnets hold:** The Application Load Balancer and the NAT Gateway

**`aws_subnet.private` — created twice using `count`**
```hcl
resource "aws_subnet" "private" {
  count             = length(var.private_subnet_cidrs)  # count = 2
  # No map_public_ip_on_launch — resources here get NO public IPs

  tags = {
    "kubernetes.io/role/internal-elb" = "1"
  }
}
```
- No public IPs assigned to anything here
- **Private subnets hold:** EKS worker nodes, RDS MySQL, ElastiCache Redis

**`aws_internet_gateway.main`**
```hcl
resource "aws_internet_gateway" "main" {
  vpc_id = aws_vpc.main.id   # Attached to the whole VPC, not individual subnets
}
```
The front door between the VPC and the internet. Public subnets use it for two-way internet access.

**`aws_eip.nat` + `aws_nat_gateway.main`**
```hcl
resource "aws_eip" "nat" {
  domain = "vpc"   # A static public IP address reserved for this NAT
}

resource "aws_nat_gateway" "main" {
  allocation_id = aws_eip.nat.id
  subnet_id     = aws_subnet.public[0].id   # NAT MUST live in a PUBLIC subnet
}
```
The NAT Gateway allows private subnet resources to reach the internet outbound (to pull Docker images, updates etc.) but the internet cannot reach them back. The Elastic IP gives the NAT Gateway a fixed, stable public IP address.

**Route Tables — the GPS routing rules**
```hcl
# Public: all external traffic goes through Internet Gateway
resource "aws_route_table" "public" {
  route { cidr_block = "0.0.0.0/0"; gateway_id = aws_internet_gateway.main.id }
}

# Private: all external traffic goes through NAT Gateway
resource "aws_route_table" "private" {
  route { cidr_block = "0.0.0.0/0"; nat_gateway_id = aws_nat_gateway.main.id }
}

# Associations — link each subnet to its route table (required, else subnets have no routing)
resource "aws_route_table_association" "public" {
  count          = length(aws_subnet.public)
  subnet_id      = aws_subnet.public[count.index].id
  route_table_id = aws_route_table.public.id
}

resource "aws_route_table_association" "private" {
  count          = length(aws_subnet.private)
  subnet_id      = aws_subnet.private[count.index].id
  route_table_id = aws_route_table.private.id
}
```
`0.0.0.0/0` means "all traffic going anywhere outside the VPC." Without route table associations, subnets have no idea where to send traffic — they would be completely isolated.

**VPC Outputs:** `vpc_id`, `public_subnet_ids`, `private_subnet_ids`, `nat_gateway_ip`

---

### Module 2: EKS — The Kubernetes Cluster (`modules/eks/main.tf`)

**What it creates:** The entire Kubernetes environment — the control plane (manager) and the worker nodes (EC2 VMs that run your pods).

#### Part A: IAM Roles — Permission Passes

Before creating anything, we set up permissions. Think of IAM roles as ID badges — only the right person/service can pick up the badge and use its permissions.

**Cluster IAM Role:**
```hcl
resource "aws_iam_role" "cluster" {
  name = "finbank-dev-eks-cluster-role"
  assume_role_policy = {
    Principal = { Service = "eks.amazonaws.com" }  # Only EKS service can use this role
    Action    = "sts:AssumeRole"
  }
}
resource "aws_iam_role_policy_attachment" "cluster_policy" {
  policy_arn = "arn:aws:iam::aws:policy/AmazonEKSClusterPolicy"
  role       = aws_iam_role.cluster.name
}
```
The EKS service needs permission to create network interfaces (ENIs) and security groups in your VPC. `AmazonEKSClusterPolicy` is the minimum permission set AWS requires.

**Node Group IAM Role:**
```hcl
resource "aws_iam_role" "node_group" {
  assume_role_policy = {
    Principal = { Service = "ec2.amazonaws.com" }  # EC2 instances (worker nodes) assume this
  }
}
# 3 required policies for worker nodes:
resource "aws_iam_role_policy_attachment" "node_policy" {
  policy_arn = "arn:aws:iam::aws:policy/AmazonEKSWorkerNodePolicy"
  # Allows nodes to register and connect to the EKS cluster
}
resource "aws_iam_role_policy_attachment" "cni_policy" {
  policy_arn = "arn:aws:iam::aws:policy/AmazonEKS_CNI_Policy"
  # CNI = Container Network Interface
  # Allows each pod to get its own real IP address from the VPC subnet
}
resource "aws_iam_role_policy_attachment" "ecr_policy" {
  policy_arn = "arn:aws:iam::aws:policy/AmazonEC2ContainerRegistryReadOnly"
  # Allows nodes to pull Docker images from ECR — read-only, never push
}
```

#### Part B: EKS Cluster — The Control Plane

```hcl
resource "aws_eks_cluster" "main" {
  name     = "finbank-dev"    # cluster name (project_name + environment)
  version  = "1.31"           # Kubernetes version
  role_arn = aws_iam_role.cluster.arn

  vpc_config {
    subnet_ids              = concat(var.public_subnet_ids, var.private_subnet_ids)
    # EKS needs ALL subnets — control plane ENIs in private, ALBs in public

    endpoint_private_access = true   # Nodes talk to API server through private network
    endpoint_public_access  = true   # Your Mac's kubectl can reach the API server
  }

  enabled_cluster_log_types = ["api", "audit", "authenticator", "controllerManager", "scheduler"]
  # Sends Kubernetes control plane logs to CloudWatch:
  # api = all API requests, audit = who changed what, authenticator = login events
}
```

- `concat(public + private)` — EKS needs to know about all subnets to place control plane components and load balancers in the right ones
- `endpoint_public_access = true` — needed so you can use `kubectl` from your laptop. In strict production you would set this to `false` and use a VPN

#### Part C: OIDC Provider — The Magic for IRSA

```hcl
data "tls_certificate" "cluster" {
  url = aws_eks_cluster.main.identity[0].oidc[0].issuer
  # Fetches the SHA1 fingerprint of EKS's identity certificate
}

resource "aws_iam_openid_connect_provider" "cluster" {
  client_id_list  = ["sts.amazonaws.com"]
  thumbprint_list = [data.tls_certificate.cluster.certificates[0].sha1_fingerprint]
  url             = aws_eks_cluster.main.identity[0].oidc[0].issuer
}
```

**What is OIDC and IRSA?**

The old problem: if a pod (application) needs to talk to AWS (e.g. read a secret from Secrets Manager), you would need AWS credentials. The bad old way was to hardcode AWS keys in environment variables — terrible security.

**IRSA (IAM Roles for Service Accounts)** solves this completely. The OIDC Provider is the trust bridge between Kubernetes and AWS IAM:

1. EKS generates an OIDC issuer URL — like a digital identity card issuer
2. AWS IAM trusts that issuer (this resource registers that trust)
3. A Kubernetes ServiceAccount gets a token from that issuer
4. When a pod starts, Kubernetes injects that token into the pod
5. The pod exchanges the token for temporary AWS credentials — without any hardcoded keys

The `tls_certificate` data source fetches the certificate fingerprint so AWS knows the token is genuinely from your EKS cluster.

**IMPORTANT — OIDC ID changes every rebuild:**
Every time you destroy and recreate EKS, a new OIDC provider URL is generated. Any IAM trust policies that reference the old URL will stop working. After every `terraform apply`, run:
```bash
aws eks describe-cluster --name finbank-dev --region ap-south-1 \
  --query "cluster.identity.oidc.issuer" --output text
```

#### Part D: Node Group — The Worker VMs

```hcl
resource "aws_eks_node_group" "main" {
  cluster_name   = aws_eks_cluster.main.name
  subnet_ids     = var.private_subnet_ids   # Worker nodes in PRIVATE subnets only
  instance_types = ["t3.medium"]            # 2 vCPU, 4GB RAM per node

  scaling_config {
    min_size     = 1
    max_size     = 2
    desired_size = 1    # dev: start with 1 node
  }

  update_config {
    max_unavailable = 1   # During Kubernetes node upgrades, only 1 node offline at a time
    # Other nodes keep serving traffic — zero-downtime node updates
  }

  disk_size = 20   # 20GB per node — stores OS, Docker image layers, pod logs
}
```

- Worker nodes run in **private subnets** — not reachable from the internet at all
- The `scaling_config` creates an Auto Scaling Group. If many pods are pending (waiting for capacity), it scales up to `max_size`. If nodes are idle, it scales down to `min_size`. Cost-efficient.
- `max_unavailable = 1` means during a node version upgrade only 1 node goes offline at a time — the others continue serving traffic

**EKS Outputs:** `cluster_name`, `cluster_endpoint`, `oidc_provider_arn`, `oidc_issuer_url`, `node_role_arn`

---

### Module 3: RDS — MySQL Database (`modules/rds/main.tf`)

**What it creates:** A managed MySQL 8.0 database server that Spring Boot connects to.

#### Security Group — The Firewall
```hcl
resource "aws_security_group" "rds" {
  ingress {
    from_port   = 3306           # MySQL port
    to_port     = 3306
    protocol    = "tcp"
    cidr_blocks = [var.vpc_cidr] # Allow ONLY from inside VPC (10.0.0.0/16)
    # Spring Boot pods inside VPC → can connect
    # Your laptop (outside VPC)  → cannot connect directly
    # The internet               → cannot connect
  }
}
```

#### DB Subnet Group
```hcl
resource "aws_db_subnet_group" "main" {
  subnet_ids = var.private_subnet_ids   # Private subnets in ap-south-1a + ap-south-1b
}
```
Tells AWS which subnets to place the RDS instance in. Must span 2 Availability Zones even if `multi_az = false` — this enables failover if you ever enable it later.

#### Parameter Group — Custom MySQL Config
```hcl
resource "aws_db_parameter_group" "main" {
  family = "mysql8.0"   # Like MySQL's my.cnf file, but managed by AWS

  parameter { name = "character_set_server"; value = "utf8mb4" }
  # utf8mb4 = full Unicode including emoji — important for banking notes/comments

  parameter { name = "max_connections"; value = "200" }
  # Spring Boot HikariCP pool uses ~10 connections per pod
  # 200 supports many pods running simultaneously

  parameter { name = "slow_query_log"; value = "1" }
  # Log all slow queries — essential for performance debugging

  parameter { name = "long_query_time"; value = "2" }
  # Any query taking more than 2 seconds is logged as slow
}
```

#### RDS Instance — The MySQL Server
```hcl
resource "aws_db_instance" "main" {
  identifier     = "finbank-dev-mysql"
  engine         = "mysql"
  engine_version = "8.0"
  instance_class = var.db_instance_class    # db.t3.micro for dev (free tier)

  allocated_storage     = 20    # Start with 20GB
  max_allocated_storage = 100   # Auto-expands up to 100GB when needed — pay only for what you use
  storage_type          = "gp2" # General Purpose SSD
  storage_encrypted     = true  # Data encrypted at rest — required for banking (RBI compliance)

  db_name  = var.db_name        # finsecure_db — the initial database
  username = var.db_username    # finsecure_user
  password = var.db_password    # from sensitive variable — never appears in logs

  publicly_accessible    = false   # No public endpoint created — only reachable from VPC
  db_subnet_group_name   = aws_db_subnet_group.main.name
  vpc_security_group_ids = [aws_security_group.rds.id]

  backup_retention_period = 7    # Keep 7 days of automated daily backups
  backup_window           = "03:00-04:00"        # Runs at 3AM — low traffic time
  maintenance_window      = "Mon:04:00-Mon:05:00" # OS patches on Monday 4AM

  multi_az            = var.multi_az             # dev: false  | prod: true (standby in different AZ)
  deletion_protection = var.deletion_protection  # dev: false  | prod: true (prevent accidents)
  skip_final_snapshot = var.skip_final_snapshot  # dev: true   | prod: false (take backup before destroy)

  enabled_cloudwatch_logs_exports = ["error", "slowquery"]
  # Sends MySQL error log and slow query log to CloudWatch for monitoring
}
```

**Critical:** Terraform creates the MySQL **server** with one initial database (`finsecure_db`). It does NOT create `finbank_staging_db` or `finbank_prod_db`. Those are created automatically by `databases.tf` — see that section below.

**RDS Outputs:** `db_endpoint` (full host:port for Spring Boot JDBC URL), `db_host` (hostname only), `db_port` (3306)

---

### Module 4: ECR — Docker Image Registry (`modules/ecr/main.tf`)

**What it creates:** Three private Docker registries — one each for backend, frontend, and analytics.

Think of ECR like a private DockerHub that lives inside AWS, close to your EKS cluster. Instead of pulling images from the public internet, EKS nodes pull from ECR in the same AWS region — faster, more secure, no rate limits.

#### Three Repositories (same pattern for each)
```hcl
resource "aws_ecr_repository" "backend" {
  name                 = "finbank-backend"
  image_tag_mutability = "MUTABLE"
  # MUTABLE = same tag can be overwritten (e.g. latest, 1.0.0-44)
  # IMMUTABLE = tag is permanent (safer for production, but less flexible)

  image_scanning_configuration {
    scan_on_push = true
    # Every image pushed by Jenkins is automatically scanned for CVE vulnerabilities
    # Results visible in ECR console — free basic scanning included
  }

  encryption_configuration {
    encryption_type = "AES256"
    # Images encrypted at rest using AWS managed keys
  }
}
# Same resource created for "finbank-frontend" and "finbank-analytics"
```

#### Lifecycle Policies — Automatic Cleanup
```hcl
resource "aws_ecr_lifecycle_policy" "backend" {
  policy = {
    rules = [
      {
        description = "Keep last 10 tagged images"
        selection   = { tagStatus = "tagged"; countNumber = 10 }
        action      = { type = "expire" }
        # Older than the 10 most recent versions → auto-deleted
        # Saves storage costs — you rarely need to roll back more than 10 versions
      },
      {
        description = "Delete untagged images after 1 day"
        selection   = { tagStatus = "untagged"; countType = "sinceImagePushed"; countNumber = 1 }
        action      = { type = "expire" }
        # Untagged = intermediate Docker build layers pushed accidentally
        # Deleted after 1 day to prevent storage accumulation
      }
    ]
  }
}
# Same lifecycle policy applied to frontend and analytics repositories
```

**ECR Outputs:** `backend_repository_url`, `frontend_repository_url`, `analytics_repository_url`
Jenkins pushes to these URLs. EKS nodes pull from these URLs.

---

### Module 5: ElastiCache — Redis Cache (`modules/elasticache/main.tf`)

**What it creates:** A managed Redis cluster for session storage, JWT token blacklisting, and API response caching.

Redis is an in-memory database — extremely fast (microseconds) compared to MySQL (milliseconds). Spring Boot uses Redis for:
- **Session management** — tracking who is currently logged in
- **JWT blacklisting** — when a user logs out, their token is stored in Redis so it cannot be reused even if still valid
- **API response caching** — frequently requested data is cached to reduce database load

#### Security Group
```hcl
resource "aws_security_group" "redis" {
  ingress {
    from_port   = 6379           # Default Redis port
    to_port     = 6379
    protocol    = "tcp"
    cidr_blocks = [var.vpc_cidr] # Only from inside VPC (10.0.0.0/16)
    # Spring Boot pods inside VPC → can connect
    # Internet → can NEVER reach Redis
  }
}
```

#### Subnet Group
```hcl
resource "aws_elasticache_subnet_group" "main" {
  subnet_ids = var.private_subnet_ids   # Private subnets — same as RDS
}
```
Redis lives in private subnets — completely isolated from the internet.

#### Redis Cluster
```hcl
resource "aws_elasticache_cluster" "main" {
  cluster_id     = "finbank-dev-redis"
  engine         = "redis"
  engine_version = "7.0"        # Latest stable Redis — supports all Spring Data Redis features

  node_type       = var.redis_node_type       # cache.t3.micro = 0.5GB RAM (free tier eligible)
  num_cache_nodes = var.redis_num_cache_nodes # 1 node for dev, 2+ for prod (replication + failover)
  port            = 6379

  subnet_group_name    = aws_elasticache_subnet_group.main.name
  security_group_ids   = [aws_security_group.redis.id]
  parameter_group_name = "default.redis7"   # AWS default settings for Redis 7

  apply_immediately = true
  # dev: apply config changes immediately
  # prod: set to false — only apply during maintenance window (Sunday 2AM) to avoid disruption
}
```

**ElastiCache Outputs:** `redis_endpoint` (hostname for Spring Boot), `redis_port` (always 6379)
These values flow into Secrets Manager and into `values.yaml` as `SPRING_DATA_REDIS_HOST`.

---

### Module 6: Secrets Manager — The Secure Vault (`modules/secretsmanager/main.tf`)

**What it creates:** Encrypted storage for all application secrets, plus the IAM plumbing that allows the External Secrets Operator (ESO) to read them automatically and inject them into pods as environment variables.

**The problem this solves:** Pods need DB passwords and JWT secrets. But you cannot hardcode them in code or Kubernetes YAML (visible to anyone with Git access). Secrets Manager is AWS's encrypted vault. ESO reads from the vault and creates Kubernetes Secrets automatically — your Spring Boot pods receive credentials as environment variables without ever touching the vault directly.

#### Three Secrets Created Using `for_each`
```hcl
resource "aws_secretsmanager_secret" "app_secrets" {
  for_each = { "dev" = "finsecure_db", "staging" = "finbank_staging_db", "prod" = "finbank_prod_db" }
  # for_each is like a loop — creates one secret per map entry
  # Result: finbank/dev/app-secrets, finbank/staging/app-secrets, finbank/prod/app-secrets

  name                    = "${var.project_name}/${each.key}/app-secrets"
  recovery_window_in_days = 0   # dev: delete immediately (no 30-day recovery window)
}
```

#### Secret Values — Filled with Real Connection Details
```hcl
resource "aws_secretsmanager_secret_version" "app_secrets" {
  for_each  = var.env_db_name_config
  secret_id = aws_secretsmanager_secret.app_secrets[each.key].id

  secret_string = jsonencode({
    db_host     = var.db_host     # Real RDS hostname from RDS module output
    db_name     = each.value      # dev=finsecure_db, staging=finbank_staging_db, prod=finbank_prod_db
    db_username = var.db_username
    db_password = var.db_password
    jwt_secret  = var.jwt_secret
    redis_host  = var.redis_host  # Real Redis hostname from ElastiCache module output
    redis_port  = var.redis_port
  })
  # Every terraform apply after RDS/Redis are created → secrets automatically contain correct values
}
```

#### IAM Policy — Controls Who Can Read the Secrets
```hcl
resource "aws_iam_policy" "secrets_reader" {
  policy = {
    Action   = ["secretsmanager:GetSecretValue", "secretsmanager:DescribeSecret"]
    Resource = "arn:aws:secretsmanager:ap-south-1:860616633001:secret:finbank/*/app-secrets-*"
    # * is a wildcard — allows reading dev, staging, and prod secrets
    # Only things that assume the IRSA role (below) can use this policy
  }
}
```

#### IRSA Role — The Trust Bridge for ESO
```hcl
locals {
  oidc_issuer = replace(var.oidc_issuer_url, "https://", "")
  # Strip https:// — AWS trust policy format requires URL without the protocol prefix
}

resource "aws_iam_role" "eso_irsa" {
  name = "finbank-eso-irsa-role"

  assume_role_policy = {
    Principal = { Federated = var.oidc_provider_arn }  # The EKS OIDC provider
    Action    = "sts:AssumeRoleWithWebIdentity"
    Condition = {
      "${oidc_issuer}:aud" = "sts.amazonaws.com"
      "${oidc_issuer}:sub" = "system:serviceaccount:external-secrets:external-secrets"
      # ONLY this exact Kubernetes ServiceAccount in the external-secrets namespace can assume this role
      # Not just any pod — specifically the ESO ServiceAccount
    }
  }
}

resource "aws_iam_role_policy_attachment" "eso_secrets_access" {
  policy_arn = aws_iam_policy.secrets_reader.arn
  role       = aws_iam_role.eso_irsa.name
  # Attach the read-secrets permission to the IRSA role
}
```

**How the full secrets flow works end to end:**
1. ESO is installed in the `external-secrets` namespace with a ServiceAccount named `external-secrets`
2. That ServiceAccount is annotated with the IRSA role ARN
3. When the ESO pod starts, Kubernetes injects a web identity token into it
4. ESO uses the token to call AWS STS (Security Token Service)
5. AWS checks: "Is this token from our OIDC provider? Is it the correct ServiceAccount?" → Yes
6. AWS gives ESO temporary credentials to read secrets
7. ESO reads the secret from Secrets Manager and creates a Kubernetes Secret object
8. Spring Boot pod reads the Kubernetes Secret as environment variables — never directly touches Secrets Manager

**Secrets Manager Outputs:** `secrets_arns` (paths used by ESO's SecretStore), `eso_irsa_role_arn` (annotated onto ESO's ServiceAccount)

---

### File: `databases.tf` — Auto-Creating MySQL Schemas

**The problem:** Terraform creates the MySQL server (RDS) with one initial database (`finsecure_db`). But the app needs 3 databases: `finsecure_db`, `finbank_staging_db`, and `finbank_prod_db`. There is no AWS resource in Terraform to create MySQL databases *inside* a server. And RDS is in a private subnet — your laptop cannot connect to it directly.

**The solution:** A `null_resource` with a `local-exec` provisioner runs a shell script on the Jenkins machine (which has `kubectl` and `aws` CLI). The script launches a temporary MySQL pod inside EKS (which is inside the VPC and can reach private RDS), creates all three databases, then destroys itself.

```hcl
resource "null_resource" "create_databases" {

  triggers = {
    rds_host = module.rds.db_host
    # Re-runs whenever the RDS hostname changes — i.e., after every destroy+apply cycle
    # null_resource normally runs only once; triggers force it to re-run when values change
  }

  depends_on = [module.rds, module.eks]
  # Wait for RDS AND EKS to both exist before running this script

  provisioner "local-exec" {
    command = <<-EOF
      set -e  # Stop script immediately if any command fails

      echo "Step 1: Waiting 90s for RDS to accept connections..."
      sleep 90
      # RDS reports 'available' to AWS before MySQL is actually ready to accept connections
      # 90 seconds is a safe buffer to avoid "Connection refused" errors

      echo "Step 2: Configure kubectl to talk to EKS cluster..."
      aws eks update-kubeconfig --region ${var.aws_region} --name ${var.project_name}-${var.environment}

      echo "Step 3: Clean up any leftover db-init pod from a previous failed run..."
      kubectl delete pod db-init --namespace=default --ignore-not-found=true

      echo "Step 4: Launch a temporary MySQL client pod inside EKS..."
      kubectl run db-init --image=mysql:8.0 --restart=Never --namespace=default -- sleep 300
      # This pod is inside EKS → inside VPC → can reach private RDS subnet
      # sleep 300 keeps it alive for 5 minutes while we run commands

      echo "Step 5: Wait until the pod is fully running..."
      kubectl wait pod/db-init --for=condition=Ready --timeout=120s --namespace=default

      echo "Step 6: Create finsecure_db (dev environment database)..."
      kubectl exec db-init --namespace=default -- mysql \
        -h ${module.rds.db_host} -u ${var.db_username} -p${var.db_password} \
        -e "CREATE DATABASE IF NOT EXISTS finsecure_db CHARACTER SET utf8mb4 COLLATE utf8mb4_unicode_ci;"

      echo "Step 7: Create finbank_staging_db (staging environment database)..."
      kubectl exec db-init --namespace=default -- mysql \
        -h ${module.rds.db_host} -u ${var.db_username} -p${var.db_password} \
        -e "CREATE DATABASE IF NOT EXISTS finbank_staging_db CHARACTER SET utf8mb4 COLLATE utf8mb4_unicode_ci;"

      echo "Step 8: Create finbank_prod_db (production environment database)..."
      kubectl exec db-init --namespace=default -- mysql \
        -h ${module.rds.db_host} -u ${var.db_username} -p${var.db_password} \
        -e "CREATE DATABASE IF NOT EXISTS finbank_prod_db CHARACTER SET utf8mb4 COLLATE utf8mb4_unicode_ci;"

      echo "Step 9: Verify all databases exist..."
      kubectl exec db-init --namespace=default -- mysql \
        -h ${module.rds.db_host} -u ${var.db_username} -p${var.db_password} \
        -e "SHOW DATABASES;"

      echo "Step 10: Delete the temporary pod — clean, no leftovers..."
      kubectl delete pod db-init --namespace=default --ignore-not-found=true
    EOF
  }
}
```

**Why `CREATE DATABASE IF NOT EXISTS`?** Safe to re-run — if the database already exists it simply does nothing. No errors, no data loss.

This launches a temporary MySQL client pod **inside EKS** (which can reach the private RDS subnet), creates all required databases, then self-destructs. Zero manual steps after every rebuild.

---

### Terraform State — Never Lose This

```
S3 Bucket: finbank-terraform-state-<YOUR_AWS_ACCOUNT_ID>
  └── finbank/dev/terraform.tfstate    ← maps your code to real AWS resources

DynamoDB Table: finbank-terraform-locks
  └── prevents two people running terraform apply simultaneously
```

**NEVER destroy these bootstrap resources.** If the state file is lost, Terraform loses track of all existing AWS resources and would try to create duplicates on the next apply, causing name conflicts and failures.

---

### How Everything Connects — The Full Picture

```
terraform apply
      │
      ├─► VPC Module (runs first — everything depends on it)
      │     Creates: VPC, 2 public subnets, 2 private subnets,
      │              Internet Gateway, NAT Gateway, Route Tables
      │     Outputs: vpc_id, public_subnet_ids, private_subnet_ids
      │
      ├─► EKS Module (uses vpc_id + subnet_ids from VPC)
      │     Creates: Cluster IAM role, Node IAM role, EKS cluster,
      │              OIDC provider, Node group (EC2 workers)
      │     Outputs: cluster_name, oidc_provider_arn, oidc_issuer_url
      │
      ├─► RDS Module (uses vpc_id + private_subnet_ids from VPC)
      │     Creates: Security group (port 3306 from VPC only),
      │              Subnet group, Parameter group, MySQL 8.0 instance
      │     Outputs: db_host, db_endpoint, db_port
      │
      ├─► ECR Module (no VPC dependency — global registry)
      │     Creates: 3 repositories (backend, frontend, analytics)
      │              + lifecycle policies for automatic image cleanup
      │     Outputs: backend/frontend/analytics repository URLs
      │
      ├─► ElastiCache Module (uses vpc_id + private_subnet_ids from VPC)
      │     Creates: Security group (port 6379 from VPC only),
      │              Subnet group, Redis 7.0 cluster
      │     Outputs: redis_endpoint, redis_port
      │
      ├─► Secrets Manager Module
      │     (uses db_host from RDS, redis_endpoint from ElastiCache,
      │      oidc_provider_arn + oidc_issuer_url from EKS)
      │     Creates: 3 secrets (dev/staging/prod) with real connection details,
      │              IAM policy (read secrets), IRSA role (for ESO)
      │     Outputs: secrets_arns, eso_irsa_role_arn
      │
      └─► databases.tf null_resource (depends on RDS + EKS both being ready)
            Runs shell script via local-exec:
            → kubectl → temporary mysql:8.0 pod inside EKS
            → Creates finsecure_db, finbank_staging_db, finbank_prod_db
            → Deletes the temporary pod
            Zero manual steps after every rebuild
```

---

### Passing Variables — Always Use TF_VAR_

Never mix `-var` flag and `TF_VAR_` environment variables for the same variable:

```bash
# WRONG — causes "Variable specified twice" error
export TF_VAR_db_password='<YOUR_DB_PASSWORD>'
terraform apply -var='db_password=<YOUR_DB_PASSWORD>'   # ERROR

# ALSO WRONG in zsh — curly braces cause glob expansion
terraform apply -var='{db_password=<YOUR_DB_PASSWORD>}'   # zsh mangles this

# CORRECT — always use environment variables only
export TF_VAR_db_password='<YOUR_DB_PASSWORD>'
export TF_VAR_jwt_secret='<YOUR_JWT_SECRET>'
terraform apply -var-file=environments/dev/terraform.tfvars -auto-approve
```

---

### Key Concepts Quick Reference

| Concept | What It Means in Simple English |
|---|---|
| **IaC** | Infrastructure as Code — build AWS resources from code files instead of clicking buttons |
| **Module** | A reusable group of related resources (like a function in programming) |
| **Variable** | A blank field — defined in `variables.tf`, filled in `terraform.tfvars` or env vars |
| **Output** | A value a module exposes so other modules or scripts can use it |
| **State file** | Terraform's memory — maps code to real AWS resources. Stored in S3. Never lose this. |
| **VPC** | Your private network on AWS — isolated from the internet |
| **Public subnet** | Resources here get public IPs — ALB (load balancer) lives here |
| **Private subnet** | Resources here have no public IPs — EKS nodes, RDS, Redis live here |
| **Internet Gateway** | The VPC's front door to/from the internet (two-way) |
| **NAT Gateway** | One-way door — private resources can call out, nothing can call in |
| **Security Group** | Firewall rules — which ports and IPs are allowed to connect |
| **IAM Role** | A permission pass for AWS services and EC2 instances |
| **OIDC + IRSA** | Lets Kubernetes pods assume IAM roles without any hardcoded AWS credentials |
| **null_resource** | A Terraform trick to run arbitrary shell scripts as part of `terraform apply` |
| **for_each** | A loop in Terraform — creates multiple resources from a map (e.g. 3 secrets at once) |
| **sensitive** | Marks a variable/output as secret — never appears in terminal output or CI/CD logs |

---

## 9. Phase 5B: Kubernetes, Helm, and ArgoCD

### Key Kubernetes Objects

**Namespace** — virtual partition inside the cluster. Like separate floors in a building.
```
finbank-dev    ← development environment
finbank-stage  ← staging/QA environment
finbank-prod   ← production environment
argocd         ← GitOps controller
monitoring     ← Prometheus + Grafana
external-secrets ← External Secrets Operator
```

**Deployment** — the desired state declaration. "I want 2 replicas of finbank-backend using image 1.0.0-44 with 512Mi memory." If a pod crashes, the Deployment controller creates a replacement automatically.

**Pod** — smallest deployable unit. Runs one container. Ephemeral — can be killed, restarted, or moved to another node at any time. Never hardcode pod IPs.

**Service** — stable DNS name for a group of pods. Pods die and get new IPs, but the Service IP never changes.
```
Internal DNS: finbank-backend.finbank-prod.svc.cluster.local
This always resolves to a healthy backend pod, regardless of restarts.
```

**ConfigMap** — non-sensitive configuration (database host URL, Redis host, Spring profile, log level).

**Ingress** — routing rules for the ALB. "Route /api/v1/* to the backend Service."

### Helm — The Package Manager

Without Helm, you need a separate set of YAML files for each environment (21 files for 3 services × 3 environments × ~7 templates each = 63 YAML files, all maintained separately).

With Helm, you write **templates once** and pass different **values files** per environment:

```
helm/
├── finbank-backend/
│   ├── Chart.yaml
│   ├── values.yaml          ← dev defaults
│   ├── values-staging.yaml  ← staging overrides
│   ├── values-prod.yaml     ← production overrides
│   └── templates/
│       ├── deployment.yaml    ← uses {{ .Values.replicaCount }}
│       ├── service.yaml
│       ├── configmap.yaml
│       ├── external-secret.yaml  ← ESO secret sync
│       ├── hpa.yaml              ← enabled only in prod
│       ├── pdb.yaml              ← enabled only in prod
│       └── resourcequota.yaml    ← enabled only in prod
```

**Environment differences via values files:**

| Setting | Dev (values.yaml) | Staging (values-staging.yaml) | Prod (values-prod.yaml) |
|---|---|---|---|
| `replicaCount` | 1 | 1 | 2 |
| `resources.requests.memory` | 256Mi | 256Mi | 512Mi |
| `resources.requests.cpu` | 250m | 250m | 500m |
| `database.name` | finsecure_db | finbank_staging_db | finbank_prod_db |
| `redis.database` | 0 | 1 | 2 |
| `hpa.enabled` | false | false | true |
| `pdb.enabled` | false | false | true |

### ArgoCD — GitOps in Action

**What GitOps means:**
Git is the single source of truth. Whatever is in the Git repo IS what should be running in Kubernetes. ArgoCD constantly watches Git and makes sure the cluster matches.

**9 ArgoCD applications (3 services × 3 environments):**

| App Name | Helm Path | Namespace | Values File |
|---|---|---|---|
| finbank-backend-dev | helm/finbank-backend | finbank-dev | values.yaml |
| finbank-backend-staging | helm/finbank-backend | finbank-stage | values-staging.yaml |
| finbank-backend-prod | helm/finbank-backend | finbank-prod | values-prod.yaml |
| finbank-frontend-dev | helm/finbank-frontend | finbank-dev | values.yaml |
| finbank-frontend-staging | helm/finbank-frontend | finbank-stage | values-staging.yaml |
| finbank-frontend-prod | helm/finbank-frontend | finbank-prod | values-prod.yaml |
| finbank-analytics-dev | helm/finbank-analytics | finbank-dev | values.yaml |
| finbank-analytics-staging | helm/finbank-analytics | finbank-stage | values-staging.yaml |
| finbank-analytics-prod | helm/finbank-analytics | finbank-prod | values-prod.yaml |

**Key ArgoCD settings:**
```yaml
syncPolicy:
  automated:
    prune: true      # Delete resources that are removed from Git
    selfHeal: true   # If someone manually changes the cluster, revert it
```

`selfHeal: true` is powerful — if someone runs `kubectl edit deployment finbank-backend` and changes the replica count, ArgoCD will revert it back to what's in Git within minutes. Git always wins.

**IMPORTANT — ArgoCD repo add must be a single line in zsh:**
```bash
# WRONG — zsh drops the credentials (auth flags silently ignored)
argocd repo add https://github.com/IndiaFinBank/Infra_FinBank.git \
  --username IndiaFinBank \
  --password ghp_xxxx

# CORRECT — run as one single line
argocd repo add https://github.com/IndiaFinBank/Infra_FinBank.git --username IndiaFinBank --password ghp_xxxx --insecure
```

### Debugging CrashLoopBackOff — The 3-Step Process

When a pod is in CrashLoopBackOff, follow this exact process:

```bash
# Step 1 — Look at events (what Kubernetes thinks happened)
kubectl describe pod <pod-name> -n finbank-dev
# Look at the Events section at the bottom

# Step 2 — Look at the last crash logs
kubectl logs <pod-name> -n finbank-dev --previous
# --previous shows logs from the crashed container, not the current one

# Step 3 — Check environment variables (wrong config is the most common cause)
kubectl describe pod <pod-name> -n finbank-dev | grep -A 30 'Environment:'
```

**Common causes seen in FinBank:**

| What you see in logs | Root cause | Fix |
|---|---|---|
| `exec format error` | ARM64 image on AMD64 node | Multi-arch buildx build |
| `Connection refused: localhost:3306` | `database.host: localhost` in values.yaml | Use real RDS endpoint |
| `Unknown database 'finbank_staging_db'` | DB schema not created | Run database creation script |
| `Redis connection refused` | Wrong env var name (`SPRING_REDIS_HOST`) | Use `SPRING_DATA_REDIS_HOST` |
| App uses default config | Wrong Spring profile | Change `profile: dev` to `profile: docker` |

---

## 10. Phase 5C: ALB Networking

### Why ALB Instead of Port Forward?

```
kubectl port-forward:
- Only works from your terminal
- Dies when terminal closes
- One connection at a time
- Not reachable from outside
- Only for debugging

ALB (Application Load Balancer):
- Public DNS name accessible from internet
- Routes to healthy pods automatically
- Handles thousands of concurrent connections
- Path-based routing to multiple services
- TLS termination
- Health checks built in
```

### Path-Based Routing

One ALB per environment routes all traffic to the correct service:

```
Request: GET http://finbank-alb-dev-xxx.ap-south-1.elb.amazonaws.com/api/v1/accounts
         ↓
ALB checks routing rules:
  Rule 1 (priority 1): /api/analytics/* → finbank-analytics Service → analytics pod
  Rule 2 (priority 2): /api/v1/*        → finbank-backend Service   → Spring Boot pod
  Rule 3 (priority 3): /*               → finbank-frontend Service  → Nginx pod

→ /api/v1/accounts matches Rule 2
→ forwarded to finbank-backend Service port 80
→ Service forwards to backend pod port 8080
→ Spring Boot returns account data
```

### ALB Controller Setup Requirements

```bash
# 1. Download official IAM policy
curl -O https://raw.githubusercontent.com/kubernetes-sigs/aws-load-balancer-controller/v2.7.2/docs/install/iam_policy.json

# 2. Create the policy in AWS
aws iam create-policy --policy-name AWSLoadBalancerControllerIAMPolicy --policy-document file://iam_policy.json

# 3. CRITICAL — add missing permission (not in official policy!)
aws iam put-role-policy \
  --role-name AWSLoadBalancerControllerRole \
  --policy-name ALBControllerExtraPermissions \
  --policy-document '{"Version":"2012-10-17","Statement":[{"Effect":"Allow","Action":["elasticloadbalancing:DescribeListenerAttributes","elasticloadbalancing:ModifyListenerAttributes"],"Resource":"*"}]}'

# 4. Install ALB Controller via Helm (MUST specify vpcId and region explicitly)
helm install aws-load-balancer-controller eks/aws-load-balancer-controller \
  -n kube-system \
  --set clusterName=finbank-dev \
  --set serviceAccount.create=false \
  --set serviceAccount.name=aws-load-balancer-controller \
  --set vpcId=<YOUR_VPC_ID> \       # ← REQUIRED: prevents IMDS hop limit issue
  --set region=ap-south-1           # ← REQUIRED
```

**Subnet IDs change after every rebuild!**
AWS assigns completely new IDs to subnets after destroy + apply. Always check after rebuilding:
```bash
aws ec2 describe-subnets \
  --filters "Name=vpc-id,Values=<VPC_ID>" \
  --query 'Subnets[*].{ID:SubnetId,AZ:AvailabilityZone,CIDR:CidrBlock}' \
  --output table
```
Then update your Ingress YAML with the new subnet IDs before applying.

### Ingress YAML Structure

```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: finbank-ingress
  namespace: finbank-dev
  annotations:
    kubernetes.io/ingress.class: alb
    alb.ingress.kubernetes.io/scheme: internet-facing
    alb.ingress.kubernetes.io/target-type: ip
    alb.ingress.kubernetes.io/subnets: subnet-xxx,subnet-yyy  # ← public subnet IDs
spec:
  rules:
    - http:
        paths:
          - path: /api/analytics
            pathType: Prefix
            backend:
              service:
                name: finbank-analytics
                port:
                  number: 80
          - path: /api/v1
            pathType: Prefix
            backend:
              service:
                name: finbank-backend
                port:
                  number: 80
          - path: /
            pathType: Prefix
            backend:
              service:
                name: finbank-frontend
                port:
                  number: 80
```

**Spring Boot context path is `/api/v1/` — use `/api/v1` in Ingress, not `/api`!**
If you use `/api`, the Ingress routes to backend but Spring Boot returns 404 because no endpoints exist under `/api` — they all start with `/api/v1`.

---

## 11. Phase 6A-B: Multi-Environment Setup

### How 3 Environments Share One Cluster

```
EKS Cluster: finbank-dev (single cluster, 3-4 nodes)
├── finbank-dev namespace      ← Dev environment
│   ├── Pods: 1 replica each (backend, frontend, analytics)
│   ├── Database: finsecure_db
│   ├── Redis: database 0
│   └── Secret: finbank/dev/app-secrets
│
├── finbank-stage namespace    ← Staging environment
│   ├── Pods: 1 replica each
│   ├── Database: finbank_staging_db
│   ├── Redis: database 1
│   └── Secret: finbank/staging/app-secrets
│
└── finbank-prod namespace     ← Production environment
    ├── Pods: 2-5 backend, 2-4 frontend, 1 analytics
    ├── Database: finbank_prod_db
    ├── Redis: database 2
    ├── Secret: finbank/prod/app-secrets
    ├── HPA: auto-scales on CPU 70%
    ├── PDB: always keeps 1 pod alive
    └── ResourceQuota: 4 CPU, 4Gi memory, 20 pods max
```

### Node Capacity Planning

**t3.medium node: ~17 pod limit** (due to AWS VPC CNI IP address allocation per ENI)

| Configuration | Nodes Needed |
|---|---|
| 3 environments (9 app pods + system pods) | 3 nodes |
| 3 environments + monitoring stack (9 more pods) | 4 nodes |

If pods are stuck in Pending with error "failed to assign an IP address to container" — the node has run out of IP addresses. Scale up the node group:
```bash
aws eks update-nodegroup-config \
  --cluster-name finbank-dev \
  --nodegroup-name finbank-dev-nodes \
  --scaling-config minSize=1,maxSize=5,desiredSize=4 \
  --region ap-south-1
```

### Spring Boot Profile — Use "docker" for All Containerised Environments

Spring Boot supports multiple configuration profiles. FinBank has:
```
src/main/resources/
├── application.properties        ← base config
├── application-dev.properties    ← local development (H2 in-memory DB, localhost)
├── application-docker.properties ← containerised (real MySQL, Redis endpoints)
└── application-test.properties   ← unit tests
```

**WRONG:** Using `profile: dev` in values-staging.yaml — Spring Boot loads the **local** config with localhost endpoints!

**CORRECT:** Use `profile: docker` in ALL environment values files (dev, staging, prod). The actual endpoint differences (which RDS database, which Redis DB number) are passed as separate environment variables by Helm.

---

## 12. Phase 6C: Monitoring

### kube-prometheus-stack Components

```bash
helm install kube-prometheus-stack prometheus-community/kube-prometheus-stack \
  -n monitoring --create-namespace \
  --set grafana.adminPassword='<YOUR_GRAFANA_PASSWORD>'
```

| Component | What It Does | Access |
|---|---|---|
| **Prometheus** | Scrapes metrics from all pods and nodes every 30 seconds | Port-forward 9090 |
| **Grafana** | Visualises metrics — 30+ pre-built K8s dashboards | Port-forward 3000 |
| **AlertManager** | Routes alerts via email, Slack, PagerDuty | Port-forward 9093 |
| **Node Exporter** | Exports CPU, memory, disk, network metrics per node | DaemonSet on every node |
| **kube-state-metrics** | Exports K8s object status (pod state, deployment replicas) | Cluster-level |

**metrics-server is separate from Prometheus** — install it separately:
```bash
kubectl apply -f https://github.com/kubernetes-sigs/metrics-server/releases/latest/download/components.yaml
```

| | metrics-server | Prometheus |
|---|---|---|
| **Purpose** | Real-time resource usage for HPA and `kubectl top` | Historical data for dashboards and alerting |
| **Data retention** | ~30 seconds (current only) | Days/weeks/months |
| **Consumer** | HPA, VPA, `kubectl top` | Grafana, AlertManager |

**Real data discovered from monitoring:**
- Backend pods use ~311Mi memory even at idle (JVM overhead)
- Frontend Nginx pods use only ~3Mi memory
- This directly informed production resource requests (backend: 512Mi, frontend: 64Mi)

---

## 13. Phase 6D: Production Hardening

### HPA — Horizontal Pod Autoscaler

HPA watches CPU usage and scales pods up or down automatically:

```yaml
apiVersion: autoscaling/v2
kind: HorizontalPodAutoscaler
metadata:
  name: finbank-backend-hpa
  namespace: finbank-prod
spec:
  scaleTargetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: finbank-backend      # ← MUST match Deployment name exactly
  minReplicas: 2
  maxReplicas: 5
  metrics:
    - type: Resource
      resource:
        name: cpu
        target:
          type: Utilization
          averageUtilization: 70  # Scale up when average CPU > 70%
```

```
Normal load:    2 pods (CPU 40%)
Traffic spike:  CPU rises to 75%
HPA adds pods:  3 pods → CPU drops to ~50%
Load drops:     After cooldown, scales back to 2
```

**HPA requires metrics-server to be installed.** Without it, HPA shows `<unknown>` for CPU and never scales.

### PDB — Pod Disruption Budget

PDB ensures minimum pods survive during voluntary disruptions (node drain, cluster upgrade):

```yaml
apiVersion: policy/v1
kind: PodDisruptionBudget
metadata:
  name: finbank-backend-pdb
  namespace: finbank-prod
spec:
  minAvailable: 1    # At least 1 pod must remain running
  selector:
    matchLabels:
      app: finbank-backend    # ← must use 'app' label, NOT 'app.kubernetes.io/name'
```

**Why PDB matters:** Without PDB, when upgrading a node, Kubernetes could evict ALL backend pods simultaneously → complete service outage. With `minAvailable: 1`, Kubernetes must wait for a new pod to start before evicting the next one.

### ResourceQuota — Namespace Resource Limits

```yaml
apiVersion: v1
kind: ResourceQuota
metadata:
  name: finbank-prod-quota
  namespace: finbank-prod
spec:
  hard:
    requests.cpu: "4"
    requests.memory: 4Gi
    pods: "20"
    services: "10"
```

Prevents production from consuming all cluster resources and starving other namespaces.

---

## 14. Security: Secrets Management

### The Problem

**Current approach in FinBank v1.0 (security gap):**
```
Application secrets in Helm values files on GitHub
→ Any developer with repo access can see DB passwords, JWT keys
→ GitHub commit history contains all past passwords forever
```

**Implemented solution (AWS Secrets Manager + ESO):**
```
AWS Secrets Manager
  → stores secrets encrypted (KMS), access-controlled (IAM), audit-logged
External Secrets Operator (runs in cluster)
  → watches ExternalSecret CRD objects
  → authenticates to AWS using IRSA (no hardcoded credentials)
  → syncs secrets from AWS SM into Kubernetes Secrets automatically
Kubernetes Secrets
  → mounted as environment variables in pods
  → never stored in Git, never in Docker images
```

### IRSA — How Pods Get AWS Permissions Without Hardcoded Keys

IRSA = IAM Roles for Service Accounts

```
EKS generates OIDC (OpenID Connect) provider
      ↓
IAM Role has trust policy: "Allow K8s ServiceAccount X to assume this role"
      ↓
ESO runs with that ServiceAccount
      ↓
AWS STS provides temporary credentials (expire every hour, auto-renewed)
      ↓
ESO uses those credentials to read from Secrets Manager
      ↓
Creates/updates K8s Secrets in each namespace
      ↓
Pods reference those K8s Secrets via secretKeyRef
```

**Zero hardcoded credentials anywhere. Zero secrets in Git. Zero secrets in Docker images.**

### ESO Setup Sequence

```bash
# Install ESO with IRSA annotation
helm install external-secrets external-secrets/external-secrets \
  -n external-secrets --create-namespace \
  --set serviceAccount.annotations."eks\.amazonaws\.com/role-arn"=arn:aws:iam::<YOUR_AWS_ACCOUNT_ID>:role/finbank-eso-irsa-role

# Create ClusterSecretStore (use v1, NOT v1beta1!)
kubectl apply -f k8s/eso/cluster-secret-store.yaml

# ExternalSecret syncs from AWS SM to K8s Secret
# (defined in Helm chart templates/external-secret.yaml)
```

### Secret Levels — Industry Standards

| Level | Approach | Security |
|---|---|---|
| 0 (what we avoid) | Secrets in Git | Worst — anyone with repo access sees everything |
| 1 (minimum) | Kubernetes Secrets | Base64 encoded, RBAC controlled — NOT encrypted |
| 2 (better) | Sealed Secrets | Encrypted in Git, ArgoCD-friendly |
| 3 (what FinBank uses) | AWS Secrets Manager + ESO | Encrypted, audited, rotatable — enterprise standard |
| 4 (maximum) | HashiCorp Vault | Dynamic short-lived secrets — Goldman Sachs level |

---

## 15. End-to-End Request Flow

This is the complete journey of one API request through the entire FinBank system:

```
User's browser (Mumbai, India)
  |
  | GET http://finbank-dev-alb.ap-south-1.elb.amazonaws.com/api/v1/accounts
  |
  v
AWS ALB (public subnet)
  DNS resolves ALB hostname → public IP
  ALB receives request, checks path: /api/v1/accounts → Rule 2 (backend)
  |
  v
ALB Target Group → forwards to EKS node port
  |
  v
Kubernetes Service (finbank-backend, ClusterIP)
  kube-proxy routes to a healthy pod
  |
  v
Backend Pod (Spring Boot 8080)
  JwtAuthFilter: extracts Bearer token from Authorization header
  Validates signature, checks expiry, checks Redis blacklist
  If valid: sets authentication context
  |
  v
AccountController.getAccounts()
  Checks authorization: does this user own these accounts?
  |
  v
AccountService → AccountRepository → JPA/Hibernate → MySQL (RDS)
  SELECT * FROM accounts WHERE user_id = ?
  RDS returns data (private subnet, VPC only)
  |
  v
Spring Boot builds JSON response
  Applies any Redis caching for frequently accessed data
  |
  v
Response travels back: Pod → Service → ALB → User's browser
  
Total time: ~50-200ms typical
```

---

## 16. Quick Reference

### Rebuild Sequence (run in this exact order)

```bash
# Step 1: Infrastructure
cd Infra_FinBank/aws/terraform
TF_VAR_db_password='<YOUR_DB_PASSWORD>' \
TF_VAR_jwt_secret='<YOUR_JWT_SECRET>' \
terraform apply -var-file=environments/dev/terraform.tfvars -auto-approve

# Step 2: Configure kubectl
aws eks update-kubeconfig --name finbank-dev --region ap-south-1

# Step 3: Get new OIDC ID (CHANGES EVERY REBUILD)
aws eks describe-cluster --name finbank-dev --region ap-south-1 \
  --query "cluster.identity.oidc.issuer" --output text

# Step 4: Scale nodes to 3 (or 4 for monitoring)
aws eks update-nodegroup-config --cluster-name finbank-dev \
  --nodegroup-name finbank-dev-nodes \
  --scaling-config minSize=1,maxSize=5,desiredSize=3 --region ap-south-1

# Step 5: Create namespaces
kubectl create namespace finbank-dev
kubectl create namespace finbank-stage
kubectl create namespace finbank-prod
kubectl create namespace argocd

# Step 6: Install ESO
helm repo add external-secrets https://charts.external-secrets.io
helm install external-secrets external-secrets/external-secrets \
  -n external-secrets --create-namespace \
  --set serviceAccount.annotations."eks\.amazonaws\.com/role-arn"=<ESO_IRSA_ROLE_ARN>

# Step 7: Apply ClusterSecretStore (apiVersion: external-secrets.io/v1 NOT v1beta1!)
kubectl apply -f k8s/eso/cluster-secret-store.yaml

# Step 8: Install ArgoCD
kubectl apply -n argocd -f https://raw.githubusercontent.com/argoproj/argo-cd/stable/manifests/install.yaml --server-side

# Step 9: Run Jenkins pipelines (builds images, pushes to ECR, creates ArgoCD apps)
# trigger: finbank-backend-pipeline, finbank-frontend-pipeline, finbank-analytics-pipeline

# Step 10: Install metrics-server
kubectl apply -f https://github.com/kubernetes-sigs/metrics-server/releases/latest/download/components.yaml

# Step 11: Setup ALB (get new subnet IDs first!)
aws ec2 describe-subnets --filters "Name=vpc-id,Values=<VPC_ID>" --output table
# Update Ingress YAMLs with new subnet IDs, then:
kubectl apply -f helm/finbank-ingress-dev.yaml
kubectl apply -f helm/finbank-ingress-staging.yaml
kubectl apply -f helm/finbank-ingress-prod.yaml

# Step 12: Install Monitoring (needs 4th node)
kubectl create namespace monitoring
helm install kube-prometheus-stack prometheus-community/kube-prometheus-stack \
  -n monitoring --set grafana.adminPassword='<YOUR_GRAFANA_PASSWORD>'
```

### Teardown Sequence (STRICT ORDER — skipping steps causes deadlocks)

```bash
# Step 1: Delete ingresses FIRST (removes ALBs from AWS)
kubectl delete ingress --all -n finbank-dev
kubectl delete ingress --all -n finbank-stage
kubectl delete ingress --all -n finbank-prod

# Step 2: Verify ALBs are gone from AWS
aws elbv2 describe-load-balancers --region ap-south-1 --no-cli-pager
# Wait until: {"LoadBalancers": []}

# Step 3: Delete ArgoCD apps
argocd app delete finbank-backend-dev finbank-backend-staging finbank-backend-prod \
  finbank-frontend-dev finbank-frontend-staging finbank-frontend-prod \
  finbank-analytics-dev finbank-analytics-staging finbank-analytics-prod --cascade

# Step 4: Delete ECR images (multi-arch requires 3-step deletion)
aws ecr delete-repository --repository-name finbank-backend --region ap-south-1 --force
aws ecr delete-repository --repository-name finbank-frontend --region ap-south-1 --force
aws ecr delete-repository --repository-name finbank-analytics --region ap-south-1 --force

# Step 5: Delete ALB IAM role
aws iam detach-role-policy --role-name AWSLoadBalancerControllerRole --policy-arn <ARN>
aws iam delete-role-policy --role-name AWSLoadBalancerControllerRole --policy-name ALBControllerExtraPermissions
aws iam delete-role --role-name AWSLoadBalancerControllerRole

# Step 6: Terraform destroy
TF_VAR_db_password='<YOUR_DB_PASSWORD>' \
TF_VAR_jwt_secret='<YOUR_JWT_SECRET>' \
terraform destroy -var-file=environments/dev/terraform.tfvars -auto-approve

# Step 7: Verify clean
aws eks list-clusters --region ap-south-1
aws rds describe-db-instances --region ap-south-1
```

### Key Numbers to Know by Heart

| Fact | Value |
|---|---|
| Pipeline duration (push to pod) | ~5-7 minutes |
| EKS cluster creation time | ~10-12 minutes |
| ALB warmup time after Ingress applied | 2-3 minutes |
| Backend pod memory at idle | ~311Mi (JVM overhead) |
| Frontend Nginx pod memory at idle | ~3Mi |
| t3.medium max pods | ~17 (ENI IP limit) |
| Backend REST APIs | 18 endpoints |
| ArgoCD applications | 9 (3 services × 3 environments) |
| Grafana pre-built dashboards | 30+ |
| Total challenges encountered and fixed | 22 fixed, 1 deferred, 1 non-blocking |
| Production readiness score | ~85% |

### Essential Commands

```bash
# Check pod status
kubectl get pods -n finbank-dev
kubectl get pods -n finbank-stage
kubectl get pods -n finbank-prod

# Check pod logs
kubectl logs -n finbank-dev deploy/finbank-backend --tail=30
kubectl logs -n finbank-dev deploy/finbank-backend --previous  # last crash

# Debug a pod
kubectl describe pod <pod-name> -n finbank-dev

# Check environment variables in pod
kubectl describe pod <pod-name> -n finbank-dev | grep -A 30 'Environment:'

# Check resource usage
kubectl top pods -n finbank-prod
kubectl top nodes

# Access services locally (debugging)
kubectl port-forward svc/finbank-backend 8081:80 -n finbank-dev &
kubectl port-forward svc/kube-prometheus-stack-grafana 3000:80 -n monitoring &

# ArgoCD sync status
kubectl get applications -n argocd

# Fix stuck Terraform state lock
terraform force-unlock -force <LOCK_ID>

# Get new OIDC ID after EKS rebuild
aws eks describe-cluster --name finbank-dev --region ap-south-1 \
  --query "cluster.identity.oidc.issuer" --output text

# Check ALBs
aws elbv2 describe-load-balancers --region ap-south-1 --no-cli-pager

# ECR login for Docker
aws ecr get-login-password --region ap-south-1 | \
  docker login --username AWS \
  --password-stdin <YOUR_AWS_ACCOUNT_ID>.dkr.ecr.ap-south-1.amazonaws.com
```

---

## 17. Interview Q&A

### Q1. "Walk me through your project architecture."

> "FinBank is a production-grade banking platform on AWS. It has three microservices — a Spring Boot backend with 18 REST APIs, a React frontend, and a FastAPI analytics service. All three run in Kubernetes on AWS EKS. When a developer pushes code, Jenkins automatically builds, tests, scans for security vulnerabilities, and pushes the Docker image to ECR. Jenkins then commits the new image tag to our Helm values file in Git. ArgoCD watches that Git repository and automatically deploys the new version to the cluster — this is called GitOps. Secrets are never in Git — they're stored in AWS Secrets Manager and synced into Kubernetes using External Secrets Operator with IRSA. Prometheus and Grafana monitor everything. We have three environments — dev, staging, and production — all in the same cluster but isolated by Kubernetes namespaces."

### Q2. "What CI/CD pipeline do you use?"

> "Jenkins with a 9-stage pipeline. Checkout the code, build the Maven JAR, run unit tests, run SonarQube analysis, build a Docker image locally for scanning, run Trivy to block any CRITICAL or HIGH CVEs, then build and push the multi-architecture image to ECR, and finally update the Helm values file in Git. The last stage is the GitOps bridge — it commits the new image tag with [skip ci] to prevent an infinite loop, and ArgoCD detects the change and deploys. Total time from code push to running pod is 5-7 minutes. The Trivy scan caught two real HIGH CVEs in our build #43 — it upgraded Spring Boot and fixed a Netty vulnerability before they ever reached ECR."

### Q3. "What is GitOps and how does ArgoCD work?"

> "GitOps means Git is the single source of truth for what should be running in Kubernetes. ArgoCD watches our Infra_FinBank Git repository. When Jenkins updates the image tag in the Helm values file and pushes to Git, ArgoCD detects the change within 3 minutes, renders the Helm templates with the new values, computes what's different from what's currently running, and applies the diff. With selfHeal enabled, if anyone manually changes anything in the cluster — like running kubectl edit — ArgoCD reverts it back to match Git. We have 9 ArgoCD applications — 3 services times 3 environments."

### Q4. "How do you manage secrets?"

> "We use AWS Secrets Manager with External Secrets Operator and IRSA — no secrets in Git, no secrets in Docker images. The flow is: secrets are stored in AWS Secrets Manager, encrypted with KMS. The External Secrets Operator pod uses IRSA — IAM Roles for Service Accounts — to authenticate to AWS without any hardcoded credentials. It watches ExternalSecret CRD objects in each namespace, fetches the corresponding secrets from AWS Secrets Manager, and creates Kubernetes Secrets automatically. Pods reference those Kubernetes Secrets via secretKeyRef. The entire chain has no static credentials anywhere. One important lesson: the OIDC ID that enables IRSA changes after every EKS cluster rebuild, so updating the IAM trust policy is now the first step in our rebuild checklist."

### Q5. "How do you handle different configurations across environments?"

> "Helm with environment-specific values files. We have one Helm chart per service, with three values files — values.yaml for dev, values-staging.yaml for staging, values-prod.yaml for production. The templates use variables like .Values.replicaCount and .Values.database.name. Production gets 2 replicas, HPA, PDB, and ResourceQuota. Dev and staging get 1 replica and no autoscaling. Jenkins Stage 9 updates the image tag in all three values files in a single Git commit, so all environments always use the same image version."

### Q6. "How does external traffic reach your application?"

> "We use the AWS Load Balancer Controller with Kubernetes Ingress resources. When we apply an Ingress YAML with class 'alb', the controller automatically creates an Application Load Balancer in our public VPC subnets, configures target groups pointing to EKS nodes, and sets up path-based routing. Root path goes to frontend, /api/v1 goes to backend, /api/analytics goes to the analytics service. One ALB per environment — three total. A key lesson: the official ALB Controller IAM policy was missing the DescribeListenerAttributes permission, so we added it as an inline policy. ALBs were being created but target group registration was silently failing."

### Q7. "Your pod is in CrashLoopBackOff. How do you debug it?"

> "Three steps. First, kubectl describe pod to check the Events section — tells you if it's an image pull failure, OOM kill, or startup crash. Second, kubectl logs with the --previous flag to see logs from the crashed container. Third, check environment variables with kubectl describe pod grep Environment. In FinBank I've hit CrashLoopBackOff multiple times. Once it was exec format error — ARM64 image on AMD64 node — fixed with multi-arch Buildx build. Another time it was Connection refused to localhost:3306 — Helm values had database host set to localhost instead of the actual RDS endpoint. A third time it was the wrong Spring Boot profile — using 'dev' instead of 'docker' which loaded localhost config."

### Q8. "How does your monitoring work?"

> "kube-prometheus-stack with Prometheus, Grafana, and AlertManager. Prometheus scrapes metrics from all pods and nodes every 30 seconds. Grafana provides 30+ pre-built Kubernetes dashboards. We also installed metrics-server separately — these are complementary, not replacements. Prometheus stores historical data for dashboards; metrics-server provides current CPU/memory snapshots that HPA uses for scaling decisions. Through monitoring, we discovered our Spring Boot backend uses ~311Mi memory even at idle due to JVM overhead, while Nginx frontend uses only ~3Mi. This data directly informed our production resource requests and limits."

### Q9. "Tell me about a production challenge you faced."

> "After a routine infrastructure rebuild, all backend pods across all three environments were in CrashLoopBackOff. I checked pod logs — saw 'connection refused' to database. Checked ExternalSecrets status — it showed SecretSyncError. Checked ClusterSecretStore — it was healthy. Then I checked IRSA and found the IAM trust policy was referencing the old OIDC Provider ID from the previous EKS cluster. Every EKS rebuild generates a new OIDC ID, and I had forgotten to update the trust policy. Fixed it with aws eks describe-cluster to get the new ID, updated the trust policy, restarted ESO, ExternalSecrets reconciled, all pods recovered. Total debugging time: 15 minutes. This is now step 3 in our rebuild checklist."

### Q10. "Why Terraform? Why not click in the AWS Console?"

> "Three reasons. First, reproducibility — when we destroyed our infrastructure to save costs, we could rebuild everything in 15 minutes with one terraform apply command. Console clicks take hours and are error-prone. Second, auditability — every infrastructure change is a Git commit. I can see who changed what and when, review it in a PR before applying, and roll back instantly by reverting. Third, consistency — dev, staging, and prod environments are defined by the same Terraform code with different variable values. No accidental configuration differences between environments."

### Q11. "How do you handle auto-scaling?"

> "Three mechanisms in production. First, HPA — Horizontal Pod Autoscaler — scales pods based on CPU. Backend scales from 2 to 5 replicas when CPU exceeds 70%. Frontend scales from 2 to 4. Second, PDB — Pod Disruption Budget — ensures at least 1 pod stays running during voluntary disruptions like node upgrades. Without PDB, a node drain could kill all backend pods simultaneously. Third, ResourceQuota — caps the production namespace at 4 CPU cores, 4Gi memory, and 20 pods maximum. This prevents a runaway deployment from consuming all cluster resources and starving other services."

---

*This project was built from zero to production-grade over 7 phases. Every concept here — Docker, Kubernetes, Terraform, Jenkins, ArgoCD, Helm, Prometheus — is implemented with real code that runs on real AWS infrastructure. Read the troubleshooting guide for the 24+ real challenges encountered and fixed during this journey.*
