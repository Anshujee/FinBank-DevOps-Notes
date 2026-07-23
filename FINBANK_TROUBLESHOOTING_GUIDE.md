# FinBank DevOps — Troubleshooting Guide & Interview War Stories

> **This file documents every real challenge encountered during the FinBank banking project.**
> Use it to prepare for interview questions like:
> - "What's the hardest problem you've debugged?"
> - "What went wrong in your project and how did you fix it?"
> - "What would you do differently next time?"

All 24 challenges are documented with: What Happened → Root Cause → Exact Fix → Interview Answer

---

## Table of Contents

**Category A — Docker & Containerisation Issues**
- [#1 — exec format error: all pods crash on EKS](#1-exec-format-error)
- [#2 — Cannot delete ECR images after multi-arch push](#2-ecr-multi-arch-deletion)
- [#17 — Jenkins cannot pull images from Docker Hub](#17-jenkins-docker-dns)
- [#20 — Maven stuck on Rosetta 2 emulation inside Jenkins](#20-rosetta-maven-error)

**Category B — Jenkins CI/CD Issues**
- [#3 — Jenkins credential ID mismatch breaks every pipeline](#3-jenkins-credential-mismatch)
- [#21 — Staging image tag never gets updated](#21-staging-image-tag)
- [#22 — IP address exhaustion: pods stuck Pending in staging](#22-ip-exhaustion)

**Category C — Kubernetes Pod Issues**
- [#5 — All pods in CrashLoopBackOff after first deploy](#5-localhost-in-values)
- [#6 — Spring Boot starts but immediately exits (empty database)](#6-empty-rds-schema)
- [#7 — Backend works in dev but broken in staging (wrong Spring profile)](#7-wrong-spring-profile)
- [#8 — Redis not connecting: Spring Boot 3.x changed env var names](#8-redis-env-var-names)
- [#19 — Login always fails: BCrypt hash prefix mismatch](#19-bcrypt-prefix)
- [#24 — Wrong Spring profile in staging (docker vs staging)](#24-wrong-staging-profile)

**Category D — Terraform Infrastructure Issues**
- [#9 — Terraform destroy fails with dependency error on ELB](#9-terraform-destroy-elb)
- [#10 — Orphaned ELB security group blocks VPC deletion](#10-orphaned-elb-sg)
- [#11 — Terraform apply hangs: state file locked](#11-terraform-state-lock)
- [#12 — "Variable specified twice" error in Terraform](#12-terraform-variable-conflict)
- [#20 — Zsh bracket expansion breaks terraform -var](#20-zsh-terraform)
- [#23 — Staging database missing after every rebuild](#23-staging-db-missing)

**Category E — AWS Networking Issues**
- [#13 — ALB created but health checks failing: missing IAM permission](#13-alb-missing-iam)
- [#14 — Backend ingress routing to wrong path](#14-wrong-ingress-path)
- [#14b — Subnet IDs change after every rebuild](#14b-subnet-ids-change)
- [#22b — IMDS hop limit: ALB Controller cannot detect cluster VPC](#22b-imds-hop-limit)

**Category F — GitOps & ArgoCD Issues**
- [#4 — ArgoCD repo add silently drops credentials in zsh](#4-argocd-repo-zsh)
- [#15 — All services broken after EKS rebuild: OIDC ID changed](#15-oidc-id-changes)
- [#18 — ArgoCD ApplicationSet goes into CrashLoop](#18-argocd-applicationset)

---

## Category A — Docker & Containerisation

---

### #1 — exec format error

**Severity:** Critical | **Category:** Docker Multi-Architecture | **Environments affected:** Dev, Staging, Prod

#### What Happened

After the first successful Jenkins pipeline run (images built and pushed to ECR), all pods immediately went into CrashLoopBackOff. I checked the pod logs with `kubectl logs <pod> --previous` and saw:

```
standard_init_linux.go:228: exec user process caused: exec format error
```

The pod kept restarting in an endless loop. Nothing else — just that single error.

#### Root Cause

My laptop is a Mac with Apple Silicon (M1/M2 chip), which uses **ARM64** architecture (64-bit ARM instruction set, same family as phones). AWS EKS worker nodes run on EC2 t3.medium instances, which use **AMD64** (Intel x86_64) architecture.

Think of it like two different languages. If you write a book in English and someone only reads French, they cannot understand it. Similarly, an ARM64 binary cannot be executed by an AMD64 CPU.

When Jenkins built the Docker image on my Mac (ARM64), it produced an ARM64 binary. ECR stored that ARM64 image. EKS nodes tried to run it on AMD64 hardware → immediate crash with "exec format error".

#### Exact Fix

Install Docker Buildx on Jenkins and build for **both architectures simultaneously**:

```bash
# In Jenkins Stage 8 (Docker Build + Push)

# Step 1: Create a special builder that can cross-compile
docker buildx create \
  --name finbank-multiarch-builder \
  --driver docker-container \
  --use

# Step 2: Bootstrap it (downloads the QEMU emulation layers)
docker buildx inspect finbank-multiarch-builder --bootstrap

# Step 3: Build for BOTH architectures and push to ECR in one command
docker buildx build \
  --platform linux/amd64,linux/arm64 \
  --tag <YOUR_AWS_ACCOUNT_ID>.dkr.ecr.ap-south-1.amazonaws.com/finbank-backend:1.0.0-${BUILD_NUMBER} \
  --push \
  .

# Step 4: Cleanup
docker buildx rm finbank-multiarch-builder || true
```

The result is a **manifest list** in ECR — a single image tag that contains two images inside it. When EKS pulls it on AMD64 nodes, AWS serves the AMD64 version. When your Mac pulls it, it gets ARM64.

Also add architecture detection in Stage 2 (Install Tools) because AWS CLI and Trivy also need architecture-specific downloads:

```bash
# Detect architecture
ARCH=$(uname -m)
if [ "$ARCH" = "arm64" ] || [ "$ARCH" = "aarch64" ]; then
  ARCH_SUFFIX="arm64"
else
  ARCH_SUFFIX="amd64"
fi

# Download Trivy for the correct architecture
curl -L "https://github.com/aquasecurity/trivy/releases/download/v0.50.0/trivy_0.50.0_Linux-${ARCH_SUFFIX}.tar.gz" -o trivy.tar.gz
```

#### Interview Answer

> "Our first deployment failed with 'exec format error' across all pods. I checked pod logs and saw the error came from Linux's execution layer — it meant the CPU couldn't understand the binary. The root cause was that I built Docker images on my Mac M1 (ARM64) and EKS runs AMD64 nodes. The CPU architectures are incompatible. I fixed it using Docker Buildx — a cross-compilation tool that builds for multiple architectures simultaneously. Now the pipeline runs `docker buildx build --platform linux/amd64,linux/arm64 --push` and ECR stores a manifest list containing both versions. EKS automatically pulls the AMD64 variant. This is now standard practice in any team that has Apple Silicon developers deploying to x86 cloud infrastructure."

---

### #2 — ECR Multi-Arch Deletion

**Severity:** Medium | **Category:** ECR Management

#### What Happened

When trying to clean up old ECR images to save storage costs, the normal deletion commands failed with errors. Some images deleted but left orphaned manifests. Trying to delete the repository itself failed with "repository is not empty" even after deleting all visible images.

#### Root Cause

Multi-architecture images in ECR have a 3-layer structure, not 1 layer like standard images:

```
Layer 1 — Manifest List (the tag, e.g., 1.0.0-44)
   ├── Layer 2a — Architecture Manifest for linux/amd64
   │       └── Layer 3a — Actual AMD64 image layers
   └── Layer 2b — Architecture Manifest for linux/arm64
           └── Layer 3b — Actual ARM64 image layers
```

If you delete Layer 1 (the tag) first, Layers 2a and 2b still exist as untagged digests. If you delete Layer 2s first, Layer 1 exists as a tag pointing to nothing. AWS ECR doesn't automatically clean up these orphaned manifests.

#### Exact Fix

Delete in the correct order: tagged manifest first, then architecture manifests, then the repository:

```bash
REPO="finbank-backend"
REGION="ap-south-1"
ACCOUNT="<YOUR_AWS_ACCOUNT_ID>"

# Step 1: Get the manifest digest for the tag (the top-level manifest list)
MANIFEST_DIGEST=$(aws ecr batch-get-image \
  --repository-name $REPO \
  --image-ids imageTag=1.0.0-44 \
  --query 'images[0].imageManifest' \
  --output text --region $REGION | python3 -c "import sys,json; print(json.load(sys.stdin)['manifests'][0]['digest'])" 2>/dev/null || echo "")

# Step 2: Delete the tagged image (manifest list)
aws ecr batch-delete-image \
  --repository-name $REPO \
  --image-ids imageTag=1.0.0-44 \
  --region $REGION

# Step 3: Delete all untagged images (leftover architecture manifests)
UNTAGGED_DIGESTS=$(aws ecr list-images \
  --repository-name $REPO \
  --filter tagStatus=UNTAGGED \
  --query 'imageIds[*]' \
  --region $REGION)

if [ "$UNTAGGED_DIGESTS" != "[]" ]; then
  aws ecr batch-delete-image \
    --repository-name $REPO \
    --image-ids "$UNTAGGED_DIGESTS" \
    --region $REGION
fi

# Step 4: Now delete the repository
aws ecr delete-repository \
  --repository-name $REPO \
  --region $REGION --force  # --force handles any remaining images
```

Simplest approach for full cleanup: just use `--force` flag on repository deletion:
```bash
aws ecr delete-repository --repository-name finbank-backend --region ap-south-1 --force
```

#### Interview Answer

> "Cleaning up multi-arch ECR images was more complex than I expected. Normal images are a single artifact. Multi-arch images are actually a 3-layer structure — a manifest list tag wrapping two architecture-specific manifests, each wrapping the actual image layers. Deleting just the tag leaves orphaned architecture manifests. I learned to delete in order: tagged manifest list first, then untagged digests, then the repository. For quick teardowns, `aws ecr delete-repository --force` handles all of this automatically."

---

### #17 — Jenkins Docker DNS

**Severity:** High | **Category:** Jenkins Networking

#### What Happened

Stage 2 of the Jenkins pipeline (Install Tools) failed with DNS resolution errors when trying to download AWS CLI and Trivy:

```
curl: (6) Could not resolve host: awscli.amazonaws.com
curl: (6) Could not resolve host: github.com
```

But when I ran the same curl command from my Mac terminal, it worked fine. The issue was specific to Jenkins.

#### Root Cause

Jenkins runs inside a Docker container on my Mac. By default, Docker containers use the internal Docker DNS resolver (127.0.0.11), not the host machine's DNS. When my Mac's VPN is active or there are Docker DNS configuration issues, the container cannot resolve public hostnames.

#### Exact Fix

Start the Jenkins container with explicit Google DNS servers:

```bash
docker run -d \
  --name jenkins \
  --dns 8.8.8.8 \
  --dns 8.8.4.4 \
  -p 9090:8080 \
  -v jenkins_home:/var/jenkins_home \
  -v /var/run/docker.sock:/var/run/docker.sock \
  -v $(which docker):/usr/bin/docker \
  jenkins/jenkins:lts
```

If Jenkins is already running, stop it, remove it, and recreate with the `--dns` flags.

#### Interview Answer

> "Jenkins would fail trying to download tools in Stage 2 with DNS resolution errors, but the same curl commands worked fine from my Mac. The issue was Jenkins running in a Docker container with Docker's internal DNS resolver. During VPN or network configuration changes, that resolver couldn't reach public hostnames. The fix was to start the Jenkins container with explicit `--dns 8.8.8.8` flags, which points the container directly to Google's DNS servers."

---

### #20 — Rosetta Maven Error

**Severity:** Medium | **Category:** Docker Architecture Emulation

#### What Happened

Inside the Jenkins container on Mac M1, the Maven build (Stage 3) produced a strange error about CPU architecture:

```
WARNING: The requested image's platform (linux/amd64) does not match the detected host platform (linux/arm64/v8)
...
Error: AVX2 instructions not found, Rosetta 2 translation failed
```

The build was slow and eventually crashed.

#### Root Cause

The Jenkins Docker image was pulled as an AMD64 image on an ARM64 Mac. macOS runs it through Rosetta 2 emulation (Apple's compatibility layer). Maven inside that emulated environment tried to use AVX2 CPU instructions (Advanced Vector Extensions — a performance feature of Intel CPUs) which Rosetta 2 does not emulate.

#### Exact Fix

Use the ARM64 version of the Jenkins image to avoid emulation entirely:

```bash
# Pull the native ARM64 Jenkins image
docker pull jenkins/jenkins:lts-jdk21

# Or explicitly pull ARM64 on Mac
docker pull --platform linux/arm64 jenkins/jenkins:lts-jdk21

docker run -d \
  --name jenkins \
  --platform linux/arm64 \   # ← run native ARM64
  --dns 8.8.8.8 \
  --dns 8.8.4.4 \
  -p 9090:8080 \
  -v jenkins_home:/var/jenkins_home \
  -v /var/run/docker.sock:/var/run/docker.sock \
  jenkins/jenkins:lts-jdk21
```

#### Interview Answer

> "Running Jenkins on Mac M1 in an AMD64 Docker image caused Rosetta 2 emulation failures — specifically around AVX2 instructions that Rosetta doesn't support. The fix was using the native ARM64 Jenkins image. This is a broader lesson about Mac M1 development: always pull native ARM64 images for your development tools, and use cross-compilation (Buildx) specifically for the production artifacts that need to run on AMD64 servers."

---

## Category B — Jenkins CI/CD

---

### #3 — Jenkins Credential Mismatch

**Severity:** Critical | **Category:** Jenkins Configuration

#### What Happened

Stage 9 of the pipeline (Update Helm Chart in GitHub) kept failing with authentication errors:

```
remote: Invalid username or password
fatal: Authentication failed for 'https://github.com/IndiaFinBank/Infra_FinBank.git'
ERROR: ERROR during step[withCredentials]
credential 'github-credentials' not found
```

The GitHub credentials were definitely configured in Jenkins. I verified them manually. But the pipeline kept reporting "not found".

#### Root Cause

The Jenkinsfile referenced credential ID `'github-credentials'` (with a hyphen). But when the credential was created in Jenkins UI, the ID was stored as `'github credentials'` (with a space).

```groovy
// In Jenkinsfile — what was written
credentialsId: 'github-credentials'   // hyphen

// In Jenkins → Credentials — what was actually stored
ID: 'github credentials'              // space ← Jenkins is case-sensitive and character-sensitive
```

Jenkins credential IDs are exact-match strings. One character difference means "not found".

#### Exact Fix

Two options — pick one and be consistent:

**Option A — Change Jenkins Jenkinsfile to match the credential ID in Jenkins:**
```groovy
withCredentials([
    usernamePassword(
        credentialsId: 'github credentials',  // ← SPACE to match what's in Jenkins
        usernameVariable: 'GIT_USER',
        passwordVariable: 'GIT_TOKEN'
    )
])
```

**Option B — Rename the Jenkins credential ID to match the Jenkinsfile:**
- Go to Jenkins → Manage Jenkins → Credentials → Global
- Click the credential → Update
- Change ID field from `github credentials` to `github-credentials`

**Best practice:** Standardize on hyphen-separated IDs (no spaces) for all credentials. The official format:
```
aws-access-key-id
aws-secret-access-key
sonarqube-token
github-credentials
```

#### Interview Answer

> "The Jenkins pipeline kept saying credential 'github-credentials' not found, even though I could see it in the Jenkins credentials list. After 20 minutes of checking everything else — GitHub PAT expiry, network access, webhook config — I finally compared the ID string character by character. The Jenkinsfile used a hyphen: 'github-credentials'. The actual stored credential used a space: 'github credentials'. Jenkins does exact string matching on credential IDs, so that single character difference meant it could never find it. The lesson is to establish a naming convention for credentials — I now always use lowercase hyphen-separated IDs — and copy IDs directly from the Jenkins UI, never type them by hand."

---

### #21 — Staging Image Tag Never Updated

**Severity:** High | **Category:** Jenkins CI/CD, GitOps

#### What Happened

After Jenkins pipeline ran successfully (green status, all stages passed), I noticed dev environment was running the new image (1.0.0-44) but staging was still on the old image (1.0.0-38). ArgoCD showed staging as "Synced and Healthy" — but with the old tag.

#### Root Cause

Stage 9 of the Jenkinsfile only updated the `values.yaml` file (dev defaults), not the environment-specific override files:

```bash
# WRONG — only updating dev values file
sed -i 's|  tag:.*|  tag: 1.0.0-44|' helm/finbank-backend/values.yaml
# values-staging.yaml NOT updated → staging ArgoCD app still sees old tag
```

The ArgoCD staging app uses `values-staging.yaml` as its values file. Since that file still had `tag: 1.0.0-38`, ArgoCD deployed 1.0.0-38 — and reported it as "Synced" because it matched Git exactly. Git had the wrong value.

#### Exact Fix

Update ALL three values files in Stage 9:

```bash
# CORRECT — update all 3 environment values files
NEW_TAG="1.0.0-${BUILD_NUMBER}"

sed -i "s|  tag:.*|  tag: ${NEW_TAG}|g" helm/finbank-backend/values.yaml
sed -i "s|  tag:.*|  tag: ${NEW_TAG}|g" helm/finbank-backend/values-staging.yaml
sed -i "s|  tag:.*|  tag: ${NEW_TAG}|g" helm/finbank-backend/values-prod.yaml

git add helm/finbank-backend/values.yaml \
        helm/finbank-backend/values-staging.yaml \
        helm/finbank-backend/values-prod.yaml

git commit -m "CI: Update finbank-backend image tag to ${NEW_TAG} [skip ci]"
git push
```

#### Interview Answer

> "After a successful pipeline run, I noticed staging was still running the old image while dev had the new one. ArgoCD showed staging as Synced and Healthy, which confused me — it meant the cluster matched Git, but Git had the wrong value. The pipeline's Stage 9 only updated values.yaml (the dev defaults) but not values-staging.yaml which the staging ArgoCD app actually uses. I updated Stage 9 to use sed on all three values files in the same commit. This also taught me that 'Synced' in ArgoCD means 'cluster matches Git' — not 'cluster has the latest version'. If Git itself is wrong, ArgoCD will dutifully deploy the wrong version."

---

### #22 — IP Exhaustion in Staging

**Severity:** Medium | **Category:** Kubernetes Networking, Resource Planning

#### What Happened

After adding staging environment pods, several staging pods stayed in Pending state indefinitely. Running `kubectl describe pod <pending-pod>` showed:

```
Warning  FailedScheduling  pod/finbank-backend-staging-xxx
  failed to assign an IP address to container:
  VPC subnet has insufficient available IP addresses
```

All dev pods were running fine. Only staging was affected.

#### Root Cause

AWS EKS uses the VPC CNI (Container Networking Interface) — each pod gets a real VPC IP address, not a virtual overlay IP. AWS allocates IP addresses from the subnet per ENI (Elastic Network Interface), and each EC2 instance type can only attach a limited number of ENIs with a limited number of IPs per ENI.

**t3.medium limits:** 3 ENIs × 6 IPs per ENI = 18 IPs per node. But some IPs are reserved for the node itself.

After accounting for dev pods, system pods (kube-system), and ArgoCD pods, the remaining IP capacity was not enough to schedule all staging pods when replicaCount in values-staging.yaml was 2 or more.

#### Exact Fix

**Immediate:** Reduce replicaCount to 1 in all staging values files:

```yaml
# values-staging.yaml
replicaCount: 1    # ← reduced from 2 to 1, saves 3 IPs (one per service)
```

**Long-term option A:** Scale up the node group to add more EC2 instances:
```bash
aws eks update-nodegroup-config \
  --cluster-name finbank-dev \
  --nodegroup-name finbank-dev-nodes \
  --scaling-config minSize=1,maxSize=5,desiredSize=4 \
  --region ap-south-1
```

**Long-term option B:** Use prefix assignment mode which gives each ENI 256 IPs instead of 6:
```bash
kubectl set env daemonset aws-node \
  -n kube-system \
  ENABLE_PREFIX_DELEGATION=true
```

#### Interview Answer

> "Staging pods were stuck in Pending with an error about insufficient VPC IP addresses. This is a common EKS gotcha — pods get real VPC IPs, not virtual overlay addresses. Each t3.medium can handle about 17 pods based on ENI count and IPs per ENI. With dev environment, system pods, ArgoCD, and monitoring all sharing the cluster, we ran out of IP capacity when staging pods with replicaCount 2 tried to schedule. The immediate fix was reducing staging to 1 replica per service. The permanent fix for production would be switching to prefix delegation mode in the AWS VPC CNI, which multiplies available IPs by 256 per ENI."

---

## Category C — Kubernetes Pod Issues

---

### #5 — Localhost in values.yaml

**Severity:** Critical | **Category:** Kubernetes Configuration

#### What Happened

After the first successful ArgoCD deployment, all three services (backend, frontend, analytics) went into CrashLoopBackOff. Backend logs showed:

```
com.mysql.cj.jdbc.exceptions.CommunicationsException:
  Communications link failure
  The last packet sent successfully to the server was 0 milliseconds ago.
  The driver has not received any packets from the server.
  url: jdbc:mysql://localhost:3306/finsecure_db
```

#### Root Cause

The Helm values.yaml had placeholder values from local development:

```yaml
# helm/finbank-backend/values.yaml — WRONG initial values
database:
  host: localhost    # ← only works on your laptop, not inside Kubernetes
  port: 3306
  name: finsecure_db

redis:
  host: localhost    # ← same problem
  port: 6379
```

Inside Kubernetes, each pod is isolated. `localhost` inside a pod refers to that pod itself, not your Mac, not RDS, not Redis. A pod trying to connect to `localhost:3306` is trying to connect to a MySQL server that doesn't exist inside that same pod.

#### Exact Fix

Replace all localhost values with actual AWS endpoint hostnames:

```yaml
# helm/finbank-backend/values.yaml — CORRECT
database:
  host: finbank-dev-mysql.c5abc123def.ap-south-1.rds.amazonaws.com
  port: 3306
  name: finsecure_db

redis:
  host: finbank-dev-redis.abc123.ng.0001.aps1.cache.amazonaws.com
  port: 6379
```

Get the actual endpoints:
```bash
# RDS endpoint
aws rds describe-db-instances \
  --db-instance-identifier finbank-dev-mysql \
  --query "DBInstances[0].Endpoint.Address" --output text --region ap-south-1

# ElastiCache Redis endpoint
aws elasticache describe-cache-clusters \
  --cache-cluster-id finbank-dev-redis \
  --show-cache-node-info \
  --query "CacheClusters[0].CacheNodes[0].Endpoint.Address" --output text --region ap-south-1
```

#### Interview Answer

> "After first deployment, all pods crashed with 'Communications link failure' trying to connect to localhost:3306. In Kubernetes, each pod is its own isolated network namespace — localhost means that pod, not any other service. The Helm values.yaml still had localhost from local development. I replaced them with the actual RDS and ElastiCache endpoint hostnames from AWS. This is a very common first-time Kubernetes mistake: localhost works on your laptop but means something completely different inside a container."

---

### #6 — Empty RDS Schema

**Severity:** Critical | **Category:** Database Management

#### What Happened

After fixing the localhost issue (#5), pods started successfully but the backend crashed during startup with:

```
java.sql.SQLSyntaxErrorException: Table 'finsecure_db.users' doesn't exist
org.springframework.beans.factory.BeanCreationException: Error creating bean 'accountRepository'
Application startup failure
```

The RDS instance was running and connectable, but had no tables.

#### Root Cause

Terraform created the **MySQL server** and the **empty database** (`finsecure_db`). But Spring Boot needs tables — `users`, `accounts`, `transactions`, `loans` — to exist before the app can start.

Without database schema initialisation, Spring Boot's JPA tries to find the `users` table on startup, can't find it, and crashes.

#### Exact Fix

Add `spring.jpa.hibernate.ddl-auto=update` to the Spring Boot configuration, or set it as an environment variable in Helm values:

```yaml
# In values.yaml under env section
env:
  SPRING_JPA_HIBERNATE_DDL_AUTO: "update"    # ← create tables if they don't exist
  SPRING_JPA_SHOW_SQL: "false"
  SPRING_JPA_DATABASE_PLATFORM: "org.hibernate.dialect.MySQL8Dialect"
```

With `ddl-auto: update`:
- If tables exist → Spring Boot uses them as-is
- If tables are missing → Spring Boot creates them based on Java entity classes
- It never drops existing data

**For production** — never use `update`, use `validate` or Flyway/Liquibase migrations:
```yaml
# Production: verify schema matches entity, fail if it doesn't
SPRING_JPA_HIBERNATE_DDL_AUTO: "validate"
```

#### Interview Answer

> "After getting past the localhost connection error, all backend pods failed during startup with 'Table users doesn't exist'. Terraform creates the RDS server and the empty database, but it doesn't create tables. Spring Boot's JPA doesn't know what tables to expect unless you tell it. Adding `spring.jpa.hibernate.ddl-auto=update` as an environment variable solved it — Spring Boot automatically created all tables based on the entity class definitions. For production, I would use `validate` mode and manage schema changes through Flyway migrations to prevent accidental schema changes during deployments."

---

### #7 — Wrong Spring Profile

**Severity:** High | **Category:** Spring Boot Configuration

#### What Happened

The backend ran fine in dev environment but crashed in staging. Staging pod logs showed:

```
HikariPool-1 - Starting...
HikariPool-1 - Exception during pool initialization.
com.mysql.cj.jdbc.exceptions.CommunicationsException:
  url: jdbc:mysql://localhost:3306/finsecure_db
```

The staging Helm values had the correct staging RDS endpoint, but the pod was still trying to connect to localhost!

#### Root Cause

The staging Helm values file set `profile: dev`:

```yaml
# values-staging.yaml — WRONG
env:
  SPRING_PROFILES_ACTIVE: "dev"    # ← loads application-dev.properties
```

`application-dev.properties` contains:
```properties
# application-dev.properties — for local laptop development
spring.datasource.url=jdbc:mysql://localhost:3306/finsecure_db
spring.data.redis.host=localhost
spring.datasource.username=root
```

The environment variables from the Helm ConfigMap were being overridden by the hardcoded `localhost` values in the `dev` Spring profile. Spring profile properties take priority over environment variables by default in Spring Boot 3.x.

#### Exact Fix

Use `docker` profile for ALL containerised environments (dev, staging, prod). The `docker` profile reads from environment variables, not hardcoded values:

```yaml
# values.yaml (dev)
env:
  SPRING_PROFILES_ACTIVE: "docker"

# values-staging.yaml
env:
  SPRING_PROFILES_ACTIVE: "docker"    # ← NOT "staging", NOT "dev"

# values-prod.yaml
env:
  SPRING_PROFILES_ACTIVE: "docker"
```

`application-docker.properties` reads from environment variables:
```properties
# application-docker.properties — for any containerised environment
spring.datasource.url=${DATABASE_URL}
spring.datasource.username=${DATABASE_USERNAME}
spring.datasource.password=${DATABASE_PASSWORD}
spring.data.redis.host=${REDIS_HOST}
```

The actual environment-specific values (which database, which Redis) are injected by Helm ConfigMap as separate env vars.

#### Interview Answer

> "Staging was crashing with connection to localhost even though the staging values.yaml had the correct RDS endpoint. The issue was the Spring profile. We had `SPRING_PROFILES_ACTIVE=dev` in staging, which loaded application-dev.properties — hardcoded with localhost values that overrode the environment variables from Kubernetes. The correct approach is to use a 'docker' profile for all containerised environments. The docker profile reads from environment variables, not hardcoded values. The actual environment differences — which database, which Redis — are injected as separate environment variables by the Helm chart."

---

### #8 — Redis Env Var Names

**Severity:** High | **Category:** Spring Boot 3.x Compatibility

#### What Happened

After fixing the Spring profile issue, the backend started but every operation that needed sessions (login, JWT storage, caching) was failing. Logs showed:

```
io.lettuce.core.RedisConnectionException:
  Unable to connect to localhost:6379
```

But `REDIS_HOST` was correctly set to the ElastiCache endpoint in the ConfigMap!

#### Root Cause

Spring Boot 3.x changed the property names for Redis configuration:

```
Spring Boot 2.x → spring.redis.host
Spring Boot 3.x → spring.data.redis.host    ← note: "data" in the middle
```

Correspondingly, the environment variable names changed:
```
Spring Boot 2.x environment variable → SPRING_REDIS_HOST
Spring Boot 3.x environment variable → SPRING_DATA_REDIS_HOST
```

The Helm ConfigMap was setting `SPRING_REDIS_HOST` (old format). Spring Boot 3.x ignored it and used the default value: `localhost:6379`.

#### Exact Fix

Update the ConfigMap template in `helm/finbank-backend/templates/configmap.yaml`:

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: finbank-backend-config
data:
  # Spring Boot 3.x Redis — MUST use SPRING_DATA_REDIS_HOST, not SPRING_REDIS_HOST
  SPRING_DATA_REDIS_HOST: {{ .Values.redis.host | quote }}
  SPRING_DATA_REDIS_PORT: {{ .Values.redis.port | quote }}
  SPRING_DATA_REDIS_DATABASE: {{ .Values.redis.database | quote }}

  # Database (unchanged between 2.x and 3.x)
  SPRING_DATASOURCE_URL: "jdbc:mysql://{{ .Values.database.host }}:{{ .Values.database.port }}/{{ .Values.database.name }}"
  SPRING_DATASOURCE_USERNAME: {{ .Values.database.username | quote }}
```

Check the full list of changed Spring Boot 3 env var prefixes:
- `SPRING_REDIS_*` → `SPRING_DATA_REDIS_*`
- `SPRING_ELASTICSEARCH_*` → `SPRING_ELASTICSEARCH_URIS` (consolidated)
- `SPRING_JPA_HIBERNATE_*` → unchanged

#### Interview Answer

> "After fixing the Spring profile, sessions were still failing with Redis connection refused to localhost. The ConfigMap had SPRING_REDIS_HOST set correctly, but the pod was ignoring it. Spring Boot 3.x renamed Redis configuration properties — from spring.redis.host to spring.data.redis.host, adding 'data' in the middle. The corresponding environment variable changed from SPRING_REDIS_HOST to SPRING_DATA_REDIS_HOST. Spring Boot 3 silently ignored the old key and used the default. This is a breaking change when migrating from Spring Boot 2 to 3 that's easy to miss because there's no error — the app starts successfully, but Redis falls back to localhost."

---

### #19 — BCrypt Prefix Mismatch

**Severity:** Medium | **Category:** Security, Spring Boot | **Status:** Deferred

#### What Happened

When testing the production environment, users with accounts created via API (JSON body with password) could log in fine. But users whose passwords were hashed by a Python script during database seeding couldn't log in — Spring Security returned 401 even with the correct password.

#### Root Cause

BCrypt has two hash format prefixes:
- `$2a$` — the universal standard, recognised by all implementations
- `$2b$` — a newer format from Python's `bcrypt` library, designed to fix a Windows-specific bug

When Python's `bcrypt` library (version 4.x+) hashes a password, it produces `$2b$10$...`. Spring Security's `BCryptPasswordEncoder.matches()` returns `false` for `$2b$` hashes even when the password is correct — it only expects `$2a$`.

```python
# Python inserts into database
import bcrypt
hashed = bcrypt.hashpw("password123".encode(), bcrypt.gensalt())
# Result: b'$2b$12$...'  ← $2b$ prefix

# Spring Security tries to verify
# BCryptPasswordEncoder.matches("password123", "$2b$12$...") → FALSE ← BUG
```

#### Current Status (Deferred)

Not fixed yet. The database seeding script currently uses Python to hash passwords. Options being evaluated:
1. Replace `$2b$` with `$2a$` in database after seeding: `UPDATE users SET password = REPLACE(password, '$2b$', '$2a$');`
2. Seed passwords using Spring's own BCryptPasswordEncoder via a setup endpoint
3. Use `$2y$` prefix which is also universally accepted
4. Upgrade Spring Security to a version that handles `$2b$`

#### Interview Answer

> "We have a deferred bug where Python-seeded accounts can't log in. Python's bcrypt library generates hashes with a $2b$ prefix, but Spring Security 6's BCryptPasswordEncoder only accepts $2a$. The prefix represents different BCrypt implementation versions — they're functionally equivalent for passwords, but Spring Security doesn't recognize $2b$. The simplest fix is a one-line SQL update to replace $2b$ with $2a$ after seeding. I deferred it because it only affects test seed data, not real user accounts which are created through the API using Spring's own hasher. This is a good example of why you should generate test data using the same hasher as your production application."

---

### #24 — Wrong Spring Profile in Staging

**Severity:** High | **Category:** Spring Boot Configuration (Repeat Pattern of #7)

#### What Happened

Staging environment pods were running but returning incorrect data — the staging API was reading from the dev database (`finsecure_db`) instead of `finbank_staging_db`. No error, no crash. Just wrong data.

#### Root Cause

After multiple environment rebuilds, the `values-staging.yaml` had drifted:

```yaml
# values-staging.yaml — had been accidentally reverted
env:
  SPRING_PROFILES_ACTIVE: "docker"    # ← correct
  SPRING_DATASOURCE_URL: "jdbc:mysql://{{ .Values.database.host }}/{{ .Values.database.name }}"
database:
  name: finsecure_db    # ← WRONG: should be finbank_staging_db
```

The Spring profile was correct (docker), but the database name variable in the values file pointed to the dev database.

#### Exact Fix

```yaml
# values-staging.yaml — CORRECT
database:
  host: finbank-dev-mysql.c5abc123def.ap-south-1.rds.amazonaws.com
  port: 3306
  name: finbank_staging_db   # ← staging database, NOT finsecure_db
  username: finsecure_user

redis:
  host: finbank-dev-redis.abc123.ng.0001.aps1.cache.amazonaws.com
  port: 6379
  database: 1                # ← Redis DB index 1 for staging (0=dev, 1=staging, 2=prod)
```

#### Interview Answer

> "Staging pods were running but silently reading from the dev database. No crash, no error — just wrong data. After some investigation, the values-staging.yaml had the correct Spring profile but the database.name variable pointed to finsecure_db (dev) instead of finbank_staging_db. This happened during a rebuild when values files were partially reset. The lesson is to add validation: immediately after deployment, verify the application is using the correct database by checking an endpoint or running a SQL query against both databases and comparing. We now add a post-deployment smoke test that confirms the correct database name from the environment."

---

## Category D — Terraform Infrastructure

---

### #9 — Terraform Destroy Blocked by ELB

**Severity:** High | **Category:** Terraform, AWS Networking

#### What Happened

Running `terraform destroy` to clean up the environment failed after partially destroying resources:

```
Error: error deleting Security Group (sg-0abc123): 
  DependencyViolation: resource sg-0abc123def has a dependent object
  There are active load balancers using this security group.
```

Terraform deleted the EKS nodes, some IAM roles, but got stuck on VPC-related resources because an ELB was still present.

#### Root Cause

When you apply an Ingress YAML in Kubernetes with `kubernetes.io/ingress.class: alb`, the AWS Load Balancer Controller creates a real Application Load Balancer in AWS on your behalf. This ALB:
- Exists in your VPC
- Has a security group attached to it
- Is **not tracked by Terraform state** (Kubernetes created it, not Terraform)

When Terraform tries to destroy the VPC, AWS refuses because the ALB security group is still attached to the VPC. Terraform doesn't know the ALB exists.

#### Exact Fix

**ALWAYS delete Kubernetes Ingress resources BEFORE running terraform destroy:**

```bash
# Step 1: Delete all Ingress objects — this tells ALB Controller to delete the ALBs
kubectl delete ingress --all -n finbank-dev
kubectl delete ingress --all -n finbank-stage
kubectl delete ingress --all -n finbank-prod

# Step 2: WAIT — verify ALBs are actually deleted from AWS (takes 1-2 minutes)
aws elbv2 describe-load-balancers \
  --region ap-south-1 \
  --no-cli-pager \
  --query 'LoadBalancers[*].{Name:LoadBalancerName,State:State.Code}'

# Wait until result is: []

# Step 3: ONLY THEN run terraform destroy
terraform destroy -var-file=environments/dev/terraform.tfvars -auto-approve
```

If you forget and terraform destroy is already stuck, manually delete the ALBs:
```bash
# Get ALB ARNs
aws elbv2 describe-load-balancers --region ap-south-1 --query 'LoadBalancers[*].LoadBalancerArn'

# Delete each ALB
aws elbv2 delete-load-balancer --load-balancer-arn <ARN>

# Also delete the ALB security group
aws ec2 delete-security-group --group-id sg-0abc123def
```

#### Interview Answer

> "Terraform destroy failed with DependencyViolation on VPC resources. The issue was that I had active Application Load Balancers in the VPC that were not managed by Terraform — they were created by the Kubernetes ALB Controller when I applied Ingress objects. Terraform had no knowledge of them. AWS refused to delete VPC subnets and security groups while the ALBs existed. The fix and now the first step in our teardown checklist: delete all Kubernetes Ingress resources first, wait for ALBs to disappear from AWS console, then run terraform destroy. This taught me that 'what Terraform manages' and 'what exists in AWS' can be different things when external controllers create resources."

---

### #10 — Orphaned ELB Security Group

**Severity:** Medium | **Category:** AWS Networking Cleanup

#### What Happened

Even after deleting Ingress objects and waiting for ALBs to disappear, `terraform destroy` still failed on VPC deletion with a security group dependency error.

#### Root Cause

The Classic ELB (if any existed) or the ALB leaves behind an orphaned security group even after the load balancer is deleted. AWS deletes the load balancer but the security group's deletion is eventually consistent — it might not be immediately deleted, or there's a separate ALB-managed security group that AWS doesn't auto-clean.

#### Exact Fix

Manually find and delete the orphaned security group before terraform destroy:

```bash
# Find security groups in the VPC that are NOT default
VPC_ID=$(terraform output -raw vpc_id)

aws ec2 describe-security-groups \
  --filters "Name=vpc-id,Values=$VPC_ID" \
  --query 'SecurityGroups[?GroupName!=`default`].{Name:GroupName,ID:GroupId}' \
  --output table

# For any unexpected SGs related to the ALB:
aws ec2 delete-security-group --group-id sg-0abc123def

# Now retry terraform destroy
terraform destroy -var-file=environments/dev/terraform.tfvars -auto-approve
```

#### Interview Answer

> "After deleting ALBs, terraform destroy still failed on a security group. AWS had left behind an orphaned security group from the ALB creation — even after the load balancer itself was gone, AWS didn't immediately clean up the associated security group. I found it with describe-security-groups filtered to our VPC, deleted it manually, then terraform destroy completed. This showed me that AWS cleanup is not always atomic — some resources are cleaned up asynchronously, so add a brief wait after ALB deletion before running terraform destroy."

---

### #11 — Terraform State Lock

**Severity:** High | **Category:** Terraform State Management

#### What Happened

After a terraform apply was interrupted (Ctrl+C or Jenkins timeout), the next terraform apply or terraform plan showed:

```
Error: Error acquiring the state lock

  Lock Info:
    ID:        abc123de-f456-7890-abcd-ef1234567890
    Path:      finbank/dev/terraform.tfstate
    Operation: OperationTypeApply
    Who:       anshu@Mac.local
    Version:   1.6.6
    Created:   2024-12-09 14:23:11.456789 +0000 UTC
    Info:

Terraform acquires a state lock to protect the state from being written by
multiple users at the same time. Please resolve the issue above and try again.
```

#### Root Cause

When Terraform runs, it acquires a lock in DynamoDB (`finbank-terraform-locks` table) to prevent concurrent runs. When the operation was interrupted, the lock was never released. DynamoDB still shows the lock as active even though no Terraform process is running.

#### Exact Fix

```bash
# Copy the Lock ID from the error message
# Then force-unlock (the -force flag skips the "are you sure?" prompt)
terraform force-unlock -force abc123de-f456-7890-abcd-ef1234567890

# Verify lock is released
terraform plan  # should succeed now
```

**Safety check before force-unlocking:** Make sure NO other Terraform process is actually running. If two people are running terraform simultaneously and you force-unlock one, you create a split-brain state. Only unlock when you're certain it's a stale lock from an interrupted process.

#### Interview Answer

> "After a Jenkins pipeline timeout during terraform apply, subsequent runs showed 'Error acquiring the state lock'. Terraform uses DynamoDB to implement distributed locking — when a run starts, it writes a lock record; when it finishes, it deletes it. An interrupted run leaves the lock behind. The fix is terraform force-unlock with the Lock ID from the error message. I always verify there's genuinely no other Terraform process running before force-unlocking, because force-unlocking a legitimate lock could allow two concurrent state modifications — which corrupts the state file."

---

### #12 — Terraform Variable Conflict

**Severity:** Medium | **Category:** Terraform Variable Handling

#### What Happened

Running terraform apply with both `-var` flags and `TF_VAR_` environment variables produced:

```
Error: Variable specified twice

  on command-line
  in environment variable

The variable "db_password" was provided twice. Please provide each
variable only once.
```

#### Root Cause

Terraform detects if you specify the same variable through multiple mechanisms and treats it as an error. You cannot mix `-var 'db_password=xxx'` and `TF_VAR_db_password=xxx` for the same variable.

#### Exact Fix

Pick ONE method and use it exclusively. Always use environment variables (they're more secure — not visible in process list or shell history):

```bash
# WRONG — mixing mechanisms
export TF_VAR_db_password='<YOUR_DB_PASSWORD>'
terraform apply -var='db_password=<YOUR_DB_PASSWORD>'  # ERROR: specified twice

# CORRECT — use only env vars
export TF_VAR_db_password='<YOUR_DB_PASSWORD>'
export TF_VAR_jwt_secret='<YOUR_JWT_SECRET>'
terraform apply -var-file=environments/dev/terraform.tfvars -auto-approve
```

The `-var-file` flag is fine to use alongside `TF_VAR_` because it reads from a file, not command line. Just don't put sensitive values in the `.tfvars` file.

#### Interview Answer

> "Terraform threw 'Variable specified twice' during apply. I had the same variable set as both a TF_VAR_ environment variable and a -var command line argument — a copy-paste mistake from switching between two methods. Terraform catches this as an error to prevent ambiguity about which value takes precedence. The fix is to pick one mechanism. I now always use TF_VAR_ environment variables for sensitive values because they're not visible in shell history or process lists, unlike -var flags."

---

### #20 — Zsh Bracket Expansion in Terraform

**Severity:** Medium | **Category:** Shell Compatibility

#### What Happened

On Mac with zsh as the default shell, running terraform with inline variables using curly braces caused strange errors:

```bash
terraform apply -var='{"db_password": "<YOUR_DB_PASSWORD>"}'
# zsh: no matches found: {"db_password": "<YOUR_DB_PASSWORD>"}
```

#### Root Cause

Zsh treats curly braces `{}` as glob expansion operators. When you type `{db_password: ...}`, zsh tries to expand it as a file glob and fails when no matching files are found.

Bash (Linux default) does not treat `{}` as globs in this context — it would work fine. Mac uses zsh by default since macOS Catalina.

#### Exact Fix

Don't use curly brace syntax for Terraform variables. Use `TF_VAR_` environment variables:

```bash
# WRONG — zsh mangles curly braces
terraform apply -var='{"db_password": "<YOUR_DB_PASSWORD>"}'

# ALSO WRONG — zsh-specific quoting issues
terraform plan -var '{db_password=<YOUR_DB_PASSWORD>}'

# CORRECT — plain string format, no curly braces
terraform apply -var='db_password=<YOUR_DB_PASSWORD>'

# BEST — use environment variables, no -var flag at all
export TF_VAR_db_password='<YOUR_DB_PASSWORD>'
terraform apply -var-file=environments/dev/terraform.tfvars -auto-approve
```

#### Interview Answer

> "Running Terraform on Mac with zsh produced 'no matches found' errors for variables that used curly braces. Zsh treats curly braces as glob patterns, trying to expand them as filesystem wildcards. Bash doesn't do this, so the same command works on Linux CI/CD but fails on Mac. The fix was to stop using curly brace syntax in -var flags — either use simple 'key=value' format or, better, switch entirely to TF_VAR_ environment variables which avoids the shell-specific parsing issue altogether."

---

### #23 — Staging Database Missing After Every Rebuild

**Severity:** High | **Category:** Database Management, Terraform

#### What Happened

After every `terraform destroy` + `terraform apply` cycle, the staging environment pods would crash because `finbank_staging_db` didn't exist in the rebuilt RDS instance. Had to manually connect to RDS and run `CREATE DATABASE` every time.

#### Root Cause

Terraform's RDS module creates the RDS server with one initial database (`finsecure_db` as specified in `var.db_name`). There's no Terraform resource type to create additional MySQL databases within an existing RDS instance. So after every rebuild, only `finsecure_db` existed.

#### Exact Fix

Added a `null_resource` to `databases.tf` that automatically creates the additional databases after Terraform provisions the infrastructure. It does this by running a temporary MySQL client pod inside EKS:

```hcl
# databases.tf
resource "null_resource" "create_databases" {
  triggers = {
    rds_host = module.rds.db_host  # re-runs whenever RDS endpoint changes
  }
  depends_on = [module.rds, module.eks]

  provisioner "local-exec" {
    command = <<-EOF
      sleep 90
      aws eks update-kubeconfig --region ${var.region} --name ${var.cluster_name}

      kubectl run db-init \
        --image=mysql:8.0 \
        --restart=Never \
        -n default -- sleep 120

      kubectl wait pod/db-init --for=condition=Ready \
        --timeout=120s -n default

      kubectl exec db-init -n default -- mysql \
        -h ${module.rds.db_host} \
        -u ${var.db_username} \
        -p${var.db_password} \
        -e "CREATE DATABASE IF NOT EXISTS finbank_staging_db CHARACTER SET utf8mb4 COLLATE utf8mb4_unicode_ci;"

      kubectl exec db-init -n default -- mysql \
        -h ${module.rds.db_host} \
        -u ${var.db_username} \
        -p${var.db_password} \
        -e "CREATE DATABASE IF NOT EXISTS finbank_prod_db CHARACTER SET utf8mb4 COLLATE utf8mb4_unicode_ci;"

      kubectl delete pod db-init -n default
    EOF
  }
}
```

This approach uses EKS pods as the MySQL client because the RDS instance is in private subnets — it's not reachable from your Mac directly. The pod runs inside the same VPC and can reach RDS.

#### Interview Answer

> "Every rebuild required manually creating the staging and prod databases — Terraform's RDS resource only creates the initial database. I automated this with a Terraform null_resource that launches a temporary MySQL client pod inside EKS, executes CREATE DATABASE IF NOT EXISTS statements for all three databases, then self-destructs. The IF NOT EXISTS means it's idempotent — safe to run multiple times. The pod runs inside EKS because RDS is in private subnets unreachable from outside the VPC. This eliminated a manual step that blocked every rebuild for 10 minutes."

---

## Category E — AWS Networking

---

### #13 — ALB Missing IAM Permission

**Severity:** High | **Category:** ALB Controller, IAM

#### What Happened

After installing the ALB Controller and applying Ingress YAMLs, the ALB was created in AWS (visible in EC2 console) but all targets showed as "unhealthy" in the target group. Pods were running. Services were correct. But the ALB couldn't route traffic to any pods.

ALB Controller logs showed:

```
{"level":"error","msg":"Error reconciling ingress",
 "error":"failed to reconcile listeners: failed to ModifyListener:
  AccessDeniedException: User arn:aws:iam::<YOUR_AWS_ACCOUNT_ID>:assumed-role/AWSLoadBalancerControllerRole
  is not authorized to perform: elasticloadbalancing:DescribeListenerAttributes
  on resource: arn:aws:elasticloadbalancing:ap-south-1:..."}
```

#### Root Cause

The official AWS Load Balancer Controller IAM policy JSON (downloaded from the GitHub repo) is missing `elasticloadbalancing:DescribeListenerAttributes` and `elasticloadbalancing:ModifyListenerAttributes` permissions. These were added to the ALB Controller in a recent version but the policy document wasn't updated.

Without these permissions, the controller cannot configure the ALB listener attributes (like access logging, idle timeout), causing the reconciliation to fail.

#### Exact Fix

Add an inline policy with the missing permissions to the ALB Controller IAM role:

```bash
# Add inline policy with missing permissions
aws iam put-role-policy \
  --role-name AWSLoadBalancerControllerRole \
  --policy-name ALBControllerExtraPermissions \
  --policy-document '{
    "Version": "2012-10-17",
    "Statement": [
      {
        "Effect": "Allow",
        "Action": [
          "elasticloadbalancing:DescribeListenerAttributes",
          "elasticloadbalancing:ModifyListenerAttributes"
        ],
        "Resource": "*"
      }
    ]
  }'

# Restart ALB Controller to pick up the new permissions
kubectl rollout restart deployment aws-load-balancer-controller -n kube-system

# Verify: delete and re-create the ingress to force reconciliation
kubectl delete ingress finbank-ingress -n finbank-dev
kubectl apply -f helm/finbank-ingress-dev.yaml
```

#### Interview Answer

> "ALB was being created but all targets were unhealthy. Checking ALB Controller logs revealed an AccessDeniedException — DescribeListenerAttributes was not allowed. The official ALB Controller IAM policy on GitHub was missing two permissions that a recent controller version started requiring. This is a common trap: official documentation and policy documents can lag behind code changes. I added an inline policy with the two missing actions and restarted the controller. The lesson: when AWS controllers report permission errors, always check the controller logs directly — the IAM error messages from AWS are specific about exactly which action is missing."

---

### #14 — Wrong Ingress Path

**Severity:** High | **Category:** Ingress Configuration, Spring Boot

#### What Happened

Frontend and analytics were accessible via the ALB. But every API call to the backend returned 404 Not Found, even though the backend pod was healthy and responding correctly when accessed directly via port-forward.

#### Root Cause

The Ingress was routing `/api` to the backend service, but Spring Boot's context path is `/api/v1`. All backend endpoints start with `/api/v1/something`. The path `/api` matched Ingress rule and got routed to backend, but Spring Boot returned 404 because it had no handler for `/api` — only for `/api/v1/...`.

```yaml
# WRONG
- path: /api           # ← routes to backend
  pathType: Prefix     #    but backend has no /api endpoints
                       #    Spring Boot: 404 for /api/accounts
                       #    Spring Boot: 200 for /api/v1/accounts

# CORRECT
- path: /api/v1        # ← routes /api/v1/* to backend
  pathType: Prefix     #    Spring Boot: 200 for /api/v1/accounts ✓
```

#### Exact Fix

Update the Ingress routing rule for backend from `/api` to `/api/v1`:

```yaml
spec:
  rules:
    - http:
        paths:
          - path: /api/analytics   # analytics service (priority 1)
            pathType: Prefix
            backend:
              service:
                name: finbank-analytics
                port:
                  number: 80
          - path: /api/v1          # backend service (priority 2)
            pathType: Prefix
            backend:
              service:
                name: finbank-backend
                port:
                  number: 80
          - path: /                # frontend (catch-all)
            pathType: Prefix
            backend:
              service:
                name: finbank-frontend
                port:
                  number: 80
```

**Rule ordering matters:** `/api/analytics` must come before `/api/v1` or both paths starting with `/api` would route to backend.

#### Interview Answer

> "Backend returned 404 via ALB but worked fine with port-forward. The Ingress had `/api` as the path prefix for backend routing. But Spring Boot's application.properties sets server.servlet.context-path=/api/v1 — every endpoint starts with /api/v1, nothing exists at /api. The request reached the correct pod but Spring Boot had no handler. Changing the Ingress path to /api/v1 fixed it. This highlighted the importance of knowing your application's actual URL structure before configuring Ingress rules."

---

### #14b — Subnet IDs Change After Rebuild

**Severity:** High | **Category:** AWS Infrastructure

#### What Happened

After a complete infrastructure rebuild (destroy + apply), the ALB Controller failed to create ALBs. Ingress objects showed error:

```
Failed build model due to: failed to resolve subnets: unable to find at least 2 qualifying subnets.
Subnets must reside in VPC vpc-0newid and must have allowed tagging.
Subnet subnet-0oldid not found in VPC vpc-0newid.
```

#### Root Cause

AWS assigns subnet IDs randomly. After `terraform destroy` deleted the VPC and subnets, `terraform apply` created new subnets with **completely new IDs**. But the Ingress YAML still had the old subnet IDs hardcoded:

```yaml
# Ingress annotation — hardcoded old IDs
alb.ingress.kubernetes.io/subnets: subnet-0abc123def,subnet-0ghi456jkl   # OLD IDs, no longer exist
```

#### Exact Fix

After every rebuild, fetch the new subnet IDs and update Ingress YAMLs:

```bash
# Get new public subnet IDs
VPC_ID=$(terraform output -raw vpc_id)
aws ec2 describe-subnets \
  --filters "Name=vpc-id,Values=$VPC_ID" "Name=tag:Name,Values=*public*" \
  --query 'Subnets[*].{ID:SubnetId,AZ:AvailabilityZone}' \
  --output table

# Update Ingress YAMLs with new IDs
sed -i "s|alb.ingress.kubernetes.io/subnets:.*|alb.ingress.kubernetes.io/subnets: subnet-NEWID1,subnet-NEWID2|" \
  helm/finbank-ingress-dev.yaml
```

**Better long-term solution:** Use subnet autodiscovery instead of hardcoding IDs. Tag your subnets with the right keys and let the ALB Controller find them automatically:

```yaml
# Terraform VPC module
resource "aws_subnet" "public" {
  tags = {
    "kubernetes.io/role/elb" = 1  # ← ALB controller autodiscovery tag
    "kubernetes.io/cluster/finbank-dev" = "shared"
  }
}
```

With these tags, you don't need the subnet annotation in Ingress at all — the controller finds subnets automatically.

#### Interview Answer

> "After a rebuild, ALB creation failed saying the subnet IDs in my Ingress annotations didn't exist. AWS assigns new random IDs to subnets every time they're recreated. I had old IDs hardcoded in the annotation. The quick fix is fetching new IDs after every rebuild and updating the YAML. The permanent fix is removing subnet annotations entirely and relying on autodiscovery — the ALB Controller scans the VPC for subnets tagged with kubernetes.io/role/elb=1, which Terraform always sets correctly. With autodiscovery, subnet IDs changing is irrelevant."

---

### #22b — IMDS Hop Limit

**Severity:** Medium | **Category:** ALB Controller, EC2 Metadata

#### What Happened

ALB Controller logs showed:

```
{"level":"error","msg":"failed to get VPC ID from ec2metadata",
 "error":"failed to get VPC ID: EC2MetadataError: failed to make EC2Metadata request"}
```

The controller was deployed successfully, but it couldn't detect which VPC the cluster was in.

#### Root Cause

The ALB Controller tries to determine the VPC ID by querying the EC2 Instance Metadata Service (IMDS) at `169.254.169.254`. But IMDS requests from pods have a hop count of 1 by default — and going through the pod's network namespace to the node adds an extra hop. Many IMDS requests from pods fail silently.

#### Exact Fix

Provide the VPC ID explicitly during Helm installation — don't let the controller autodetect it:

```bash
VPC_ID=$(terraform output -raw vpc_id)

helm upgrade --install aws-load-balancer-controller \
  eks/aws-load-balancer-controller \
  -n kube-system \
  --set clusterName=finbank-dev \
  --set serviceAccount.create=false \
  --set serviceAccount.name=aws-load-balancer-controller \
  --set vpcId=$VPC_ID \         # ← explicit VPC ID, no IMDS needed
  --set region=ap-south-1       # ← explicit region too
```

With `vpcId` and `region` specified, the controller never queries IMDS.

#### Interview Answer

> "The ALB Controller was failing to detect the VPC because EC2 instance metadata requests from pods can fail due to hop count limits. By default, IMDS allows 1 network hop, but container-to-node routing counts as an extra hop. The fix is to provide the VPC ID and region explicitly during ALB Controller Helm installation so it never needs to query IMDS. Always specify `--set vpcId=<id>` and `--set region=<region>` in the helm install command."

---

## Category F — GitOps & ArgoCD

---

### #4 — ArgoCD Repo Add Drops Credentials in Zsh

**Severity:** High | **Category:** ArgoCD, Zsh Shell

#### What Happened

After running `argocd repo add` with authentication flags, ArgoCD seemed to accept the command. But when ArgoCD tried to sync from the private GitHub repo, it showed:

```
rpc error: code = Unknown desc = authentication required
```

Running `argocd repo list` showed the repo was added but with status "failed" and no credentials.

#### Root Cause

In zsh, when a long command is split across multiple lines using `\` (backslash line continuation), some flags can be silently dropped if there's any whitespace issue after the backslash.

```bash
# WRONG — zsh may silently drop --username and --password flags
argocd repo add https://github.com/IndiaFinBank/Infra_FinBank.git \
  --username IndiaFinBank \
  --password ghp_xxxxxxxxxxxx
# Result: repo added but WITHOUT credentials
```

This is a subtle zsh quirk — bash handles this fine, but zsh's interactive parsing differs.

#### Exact Fix

Run the entire `argocd repo add` command as a single line with no backslash continuation:

```bash
# CORRECT — single line, no line breaks
argocd repo add https://github.com/IndiaFinBank/Infra_FinBank.git --username IndiaFinBank --password ghp_xxxxxxxxxxxx --insecure

# Verify credentials were saved
argocd repo list
# Should show: ConnectionStateSucceeded
```

#### Interview Answer

> "ArgoCD couldn't sync from GitHub — authentication failure even though I had run argocd repo add with credentials. The issue was subtle: I had split the command across multiple lines with backslash continuation in zsh. Some flags, specifically the auth flags, were silently dropped by zsh's line-continuation handling. Rerunning the exact same command as a single line worked immediately. This is one of those bugs that makes you question your sanity — the command looks correct, produces no error, but doesn't do what you expect. Lesson: for argocd CLI commands with authentication, always run as a single line."

---

### #15 — OIDC ID Changes After EKS Rebuild

**Severity:** Critical | **Category:** IRSA, AWS IAM, Security

#### What Happened

After a full infrastructure rebuild (terraform destroy + apply), all pods that needed secrets were crashing. External Secrets Operator logs showed:

```
SecretSyncError: failed to get secret from AWS Secrets Manager:
  AccessDeniedException: User arn:aws:sts::<YOUR_AWS_ACCOUNT_ID>:assumed-role/finbank-eso-irsa-role/xxx
  is not authorized to perform secretsmanager:GetSecretValue
```

But the IAM role and policy were correct — nothing had changed in the Terraform code.

#### Root Cause

IRSA works through OIDC federation:
1. EKS creates an OIDC provider with a unique ID (like `oidc.eks.ap-south-1.amazonaws.com/id/ABC123DEF456`)
2. Your IAM role has a trust policy saying: "Allow service accounts from THIS specific OIDC provider to assume this role"
3. When EKS is rebuilt, it generates a completely NEW OIDC provider with a NEW ID (like `ABC789GHI012`)
4. The IAM trust policy still references the OLD ID — which no longer exists
5. Every attempt by pods to assume the IAM role fails with AccessDeniedException

The OIDC provider ID is embedded in:
- The IAM role trust policy
- The ClusterSecretStore annotation

#### Exact Fix

After every EKS rebuild:

```bash
# Step 1: Get the NEW OIDC ID
OIDC_ID=$(aws eks describe-cluster \
  --name finbank-dev \
  --region ap-south-1 \
  --query "cluster.identity.oidc.issuer" \
  --output text | cut -d '/' -f 5)

echo "New OIDC ID: $OIDC_ID"

# Step 2: Update the IAM trust policy for ESO role
ACCOUNT_ID=<YOUR_AWS_ACCOUNT_ID>
cat > /tmp/trust-policy.json << EOF
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Principal": {
        "Federated": "arn:aws:iam::${ACCOUNT_ID}:oidc-provider/oidc.eks.ap-south-1.amazonaws.com/id/${OIDC_ID}"
      },
      "Action": "sts:AssumeRoleWithWebIdentity",
      "Condition": {
        "StringEquals": {
          "oidc.eks.ap-south-1.amazonaws.com/id/${OIDC_ID}:sub": "system:serviceaccount:external-secrets:external-secrets",
          "oidc.eks.ap-south-1.amazonaws.com/id/${OIDC_ID}:aud": "sts.amazonaws.com"
        }
      }
    }
  ]
}
EOF

aws iam update-assume-role-policy \
  --role-name finbank-eso-irsa-role \
  --policy-document file:///tmp/trust-policy.json

# Step 3: Restart ESO to pick up the new identity
kubectl rollout restart deployment external-secrets -n external-secrets

# Step 4: Verify
kubectl get externalsecrets -A
# Should show: Ready=True, Status=SecretSynced
```

This is now the **first item** on the post-rebuild checklist.

#### Interview Answer

> "After a rebuild, all secrets were failing with AccessDeniedException even though the IAM policy was correct. IRSA authentication relies on OIDC federation — the IAM role trust policy includes the specific OIDC provider ID for the EKS cluster. Every time you destroy and recreate EKS, it generates a completely new OIDC provider with a new ID. The old trust policy is now pointing to an ID that doesn't exist. The fix is updating the trust policy with the new OIDC ID after every rebuild. We scripted this using aws eks describe-cluster to get the new ID and aws iam update-assume-role-policy to apply it. It's now step 3 in our rebuild checklist."

---

### #18 — ArgoCD ApplicationSet CrashLoop

**Severity:** Low | **Status:** Non-blocking | **Category:** ArgoCD

#### What Happened

When trying to use ArgoCD ApplicationSet to manage all 9 apps (3 services × 3 environments) from a single manifest, the ApplicationSet controller went into CrashLoopBackOff:

```
Error: applicationsets.argoproj.io is forbidden:
  User "system:serviceaccount:argocd:argocd-applicationset-controller"
  cannot list resource "applicationsets" in API group "argoproj.io"
```

#### Root Cause

The version of ArgoCD installed via `kubectl apply` had a mismatch between the ApplicationSet controller's RBAC permissions and the API group version. The controller's ClusterRole didn't include permissions for the `applicationsets` resource under the `argoproj.io` API group.

#### Decision Made

**Not fixed — not used in FinBank.** We manage 9 ArgoCD apps individually via separate YAML files. This is acceptable for 9 apps. ApplicationSet would be useful at 30+ apps.

The 9 individual app YAMLs are:
```
argocd/
├── finbank-backend-app.yaml      (dev)
├── finbank-backend-staging.yaml  (staging)
├── finbank-backend-prod.yaml     (prod)
├── finbank-frontend-app.yaml
├── finbank-frontend-staging.yaml
├── finbank-frontend-prod.yaml
├── finbank-analytics-app.yaml
├── finbank-analytics-staging.yaml
└── finbank-analytics-prod.yaml
```

#### Interview Answer

> "I tried to use ArgoCD ApplicationSet to template all 9 applications, but the controller crashed due to missing RBAC permissions — version mismatch between the installed ArgoCD and the ApplicationSet CRD. I chose not to investigate further and instead manage 9 individual ArgoCD application YAMLs. At 9 applications, the overhead is manageable. ApplicationSet becomes valuable at 30+ applications where manual management creates drift. This was a pragmatic tradeoff: spend time debugging a non-critical tooling issue, or deliver working GitOps in the next 20 minutes."

---

## Summary Table — All 24 Challenges

| # | Category | Issue | Status | Key Lesson |
|---|---|---|---|---|
| 1 | Docker | exec format error (ARM64 vs AMD64) | Fixed | Always use Buildx for multi-arch when on Mac M1 |
| 2 | ECR | Multi-arch image deletion | Fixed | Delete tag → untagged → repo. Or use --force |
| 3 | Jenkins | Credential ID mismatch (space vs hyphen) | Fixed | Use exact IDs from Jenkins UI, standardize naming |
| 4 | ArgoCD | zsh drops auth flags on multiline command | Fixed | Run argocd repo add as single line |
| 5 | K8s | localhost in values.yaml → CrashLoop | Fixed | Kubernetes pods cannot reach localhost for external services |
| 6 | K8s | Empty RDS schema → startup failure | Fixed | DDL auto=update for dev, Flyway for prod |
| 7 | K8s | Wrong Spring profile (dev vs docker) | Fixed | Always use 'docker' profile in containers |
| 8 | K8s | Wrong Redis env var (Spring Boot 3 renamed them) | Fixed | Spring Boot 3 = SPRING_DATA_REDIS_HOST |
| 9 | Terraform | Destroy blocked by ALB (not in state) | Fixed | Delete Ingress FIRST before terraform destroy |
| 10 | Terraform | Orphaned ELB security group | Fixed | Manually delete orphaned SGs before VPC deletion |
| 11 | Terraform | State lock from interrupted apply | Fixed | terraform force-unlock -force LOCK_ID |
| 12 | Terraform | Variable specified twice | Fixed | Pick one: -var OR TF_VAR_, never both |
| 13 | ALB | Missing IAM permission (DescribeListenerAttributes) | Fixed | Add inline policy for missing permissions |
| 14 | ALB | Wrong Ingress path (/api vs /api/v1) | Fixed | Path must match Spring Boot context-path exactly |
| 14b | ALB | Subnet IDs change after rebuild | Fixed | Use autodiscovery tags instead of hardcoded IDs |
| 15 | IRSA | OIDC ID changes after EKS rebuild | Fixed | Update trust policy after every EKS rebuild |
| 16 | K8s | Wrong backend context path in Ingress | Fixed | Verify Spring context-path before writing Ingress |
| 17 | Docker | Jenkins container DNS resolution failure | Fixed | Start Jenkins with --dns 8.8.8.8 |
| 18 | ArgoCD | ApplicationSet controller RBAC crash | Non-blocking | Use individual app YAMLs instead |
| 19 | Security | BCrypt $2b$ vs $2a$ prefix mismatch | Deferred | Use Spring's hasher for all password generation |
| 20 | Shell | zsh curly brace glob expansion in terraform | Fixed | Use TF_VAR_ env vars instead of -var flags |
| 21 | Jenkins | Staging image tag never updated | Fixed | Update ALL values files in Stage 9 |
| 22 | K8s | IP exhaustion in staging (pods Pending) | Fixed | Set replicaCount=1 in staging; use prefix delegation |
| 22b | ALB | IMDS hop limit blocks VPC detection | Fixed | Set --set vpcId and --set region explicitly |
| 23 | Database | Staging DB missing after every rebuild | Fixed | null_resource in databases.tf auto-creates schemas |
| 24 | K8s | Wrong staging profile (dev database used) | Fixed | Verify database name in values-staging.yaml |

---

*These are real mistakes that happened during the FinBank project. Every one of them taught something that cannot be learned from tutorials. In interviews, describe 2-3 of these in detail — choose ones relevant to the role, explain the root cause clearly, and emphasise what you would do differently (and have done differently) as a result.*
