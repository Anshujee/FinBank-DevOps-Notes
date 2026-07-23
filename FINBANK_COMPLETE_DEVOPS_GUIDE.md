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

### Terraform Module Structure

```
Infra_FinBank/aws/terraform/
├── main.tf           # Orchestrator — calls all modules
├── variables.tf      # Input parameters
├── outputs.tf        # Exported values (subnet IDs, RDS endpoint, etc.)
├── versions.tf       # Provider versions (AWS, Kubernetes, null)
├── databases.tf      # null_resource to create MySQL schemas
├── environments/
│   └── dev/
│       └── terraform.tfvars   # Dev environment values
└── modules/
    ├── vpc/           # VPC, subnets, NAT Gateway, Internet Gateway, route tables
    ├── eks/           # EKS cluster, node group, IAM roles, OIDC provider
    ├── rds/           # MySQL instance, subnet group, security group, parameter group
    ├── elasticache/   # Redis cluster, subnet group, security group
    ├── ecr/           # 3 ECR repositories (backend, frontend, analytics)
    └── secretsmanager/ # 3 secrets (per env), IAM policy, IRSA role
```

**How modules wire together:**
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
```

### VPC Architecture

```
VPC: 10.0.0.0/16
├── Public Subnets (ALB lives here — internet-facing)
│   ├── 10.0.1.0/24 — ap-south-1a
│   └── 10.0.2.0/24 — ap-south-1b
│   Tags: kubernetes.io/role/elb=1  (ALB controller discovers these)
│
├── Private Subnets (EKS nodes, RDS, Redis — no direct internet)
│   ├── 10.0.3.0/24 — ap-south-1a
│   └── 10.0.4.0/24 — ap-south-1b
│   Tags: kubernetes.io/role/internal-elb=1
│
├── Internet Gateway → attached to VPC, routes public subnet traffic to internet
├── NAT Gateway → in public subnet, allows private subnet resources to reach internet
│   (to pull Docker images from ECR, install OS packages, etc.)
└── Route Tables
    ├── Public RT: 0.0.0.0/0 → Internet Gateway
    └── Private RT: 0.0.0.0/0 → NAT Gateway
```

**Why private subnets for EKS nodes and RDS:**
RDS and EKS worker nodes are not directly reachable from the internet. You can't SSH into the nodes. You can't connect to MySQL directly from your laptop. They only communicate with other resources inside the VPC.

### EKS Module — What It Creates

```hcl
# IAM Role for EKS Cluster (AWS manages control plane using this)
resource "aws_iam_role" "cluster" { ... }

# IAM Role for Worker Nodes (EC2s that run your pods)
# Permissions needed:
# - AmazonEKSWorkerNodePolicy (connect to cluster)
# - AmazonEKS_CNI_Policy (give pods VPC IP addresses)
# - AmazonEC2ContainerRegistryReadOnly (pull images from ECR)

# OIDC Provider — enables pods to assume IAM roles (IRSA)
resource "aws_iam_openid_connect_provider" "cluster" { ... }

# EKS Cluster
resource "aws_eks_cluster" "main" {
  name    = "finbank-dev"
  version = "1.31"
  # Control plane logging to CloudWatch:
  enabled_cluster_log_types = ["api", "audit", "authenticator", ...]
}

# Worker Nodes (EC2 instances)
resource "aws_eks_node_group" "main" {
  instance_types = ["t3.medium"]  # 2 vCPU, 4GB RAM
  scaling_config {
    min_size     = 1
    max_size     = 5
    desired_size = 3    # 3 for apps, 4 when monitoring added
  }
}
```

**IMPORTANT — OIDC ID Changes Every Rebuild:**
Every time you destroy and recreate EKS, it generates a NEW OIDC provider ID. Any IAM trust policies that reference the old OIDC ID will break. After every `terraform apply`, run:
```bash
aws eks describe-cluster --name finbank-dev --region ap-south-1 \
  --query "cluster.identity.oidc.issuer" --output text
```
Then update the IRSA trust policy with the new ID.

### RDS Module — MySQL Configuration

```hcl
resource "aws_db_instance" "main" {
  identifier     = "finbank-dev-mysql"
  engine         = "mysql"
  engine_version = "8.0"
  instance_class = "db.t3.micro"    # Free tier eligible

  db_name  = var.db_name             # finsecure_db (initial database)
  username = var.db_username          # finsecure_user
  password = var.db_password          # from TF_VAR_db_password env var

  publicly_accessible = false         # NOT reachable from internet
  multi_az            = false         # Single AZ for dev (set true for prod)

  backup_retention_period = 7         # 7 days of automated backups
  storage_encrypted       = true      # Data encrypted at rest
  deletion_protection     = false     # Can delete for dev

  enabled_cloudwatch_logs_exports = ["error", "slowquery"]
}
```

**Critical:** Terraform creates the MySQL SERVER with one initial database (`finsecure_db`). It does NOT create `finbank_staging_db` or `finbank_prod_db`. Those must be created separately — see `databases.tf` section below.

### databases.tf — Auto-Creating Schemas

After every terraform destroy + apply, the staging and prod databases are missing. We solved this permanently with a `null_resource`:

```hcl
resource "null_resource" "create_databases" {
  triggers = {
    rds_host = module.rds.db_host  # Re-runs when RDS changes
  }
  depends_on = [module.rds, module.eks]

  provisioner "local-exec" {
    command = <<-EOF
      sleep 90  # Wait for RDS to be ready to accept connections
      aws eks update-kubeconfig --region ap-south-1 --name finbank-dev
      kubectl run db-init --image=mysql:8.0 --restart=Never -n default -- sleep 120
      kubectl wait pod/db-init --for=condition=Ready --timeout=120s -n default
      kubectl exec db-init -n default -- mysql -h ${HOST} -u ${USER} -p${PASS} \
        -e 'CREATE DATABASE IF NOT EXISTS finbank_staging_db ...'
      kubectl exec db-init -n default -- mysql -h ${HOST} -u ${USER} -p${PASS} \
        -e 'CREATE DATABASE IF NOT EXISTS finbank_prod_db ...'
      kubectl delete pod db-init -n default
    EOF
  }
}
```

This launches a temporary MySQL client pod **inside EKS** (which can reach the private RDS subnet), creates all required databases, then self-destructs. Zero manual steps.

### Terraform State — Never Lose This

```
S3 Bucket: finbank-terraform-state-<YOUR_AWS_ACCOUNT_ID>
  └── finbank/dev/terraform.tfstate    ← maps your code to real AWS resources

DynamoDB Table: finbank-terraform-locks
  └── prevents two people running terraform simultaneously
```

**NEVER destroy these bootstrap resources.** If the state file is lost, Terraform loses track of all existing AWS resources and would try to create duplicates, causing conflicts.

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
