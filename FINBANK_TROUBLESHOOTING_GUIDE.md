# FinBank DevOps — Troubleshooting Guide & Interview War Stories

> **This file documents every real challenge encountered during the FinBank banking project on AWS.**
> Use it to prepare for interview questions like:
> - "What is the hardest problem you have debugged?"
> - "Tell me about a time something broke in production."
> - "What went wrong in your project and how did you fix it?"
>
> Every challenge follows the same structure:
> **What Happened → Exact Error → Root Cause → How to Diagnose → Exact Fix → Prevention → Interview Answer (STAR format)**
>
> STAR = **S**ituation → **T**ask → **A**ction → **R**esult

---

## Table of Contents

**Category A — Docker & Container Issues**
- [#1 — exec format error: all pods crash on EKS](#1-exec-format-error)
- [#2 — ECR multi-arch image deletion fails](#2-ecr-multi-arch-deletion)
- [#3 — Jenkins cannot resolve DNS to download tools](#3-jenkins-dns-failure)
- [#4 — Maven crashes inside Jenkins: Rosetta 2 AVX2 error on Mac M1](#4-rosetta-maven-error)
- [#5 — Trivy install fails: permission denied on /usr/local/bin](#5-trivy-permission-denied)

**Category B — Jenkins CI/CD**
- [#6 — Jenkins credential ID mismatch: authentication fails every pipeline run](#6-jenkins-credential-mismatch)
- [#7 — Staging image tag never updates after pipeline succeeds](#7-staging-image-tag)
- [#8 — zsh bracket expansion silently breaks terraform -var flag](#8-zsh-terraform-var)
- [#9 — New CVEs block build #43: Trivy finds critical vulnerabilities](#9-new-cves-block-build)

**Category C — Kubernetes & EKS**
- [#10 — All pods CrashLoopBackOff: localhost in values.yaml](#10-localhost-in-values)
- [#11 — Spring Boot crashes on startup: database tables do not exist](#11-empty-rds-schema)
- [#12 — Backend connects to localhost in staging: wrong Spring profile active](#12-wrong-spring-profile)
- [#13 — Redis connection fails: Spring Boot 3.x renamed all env var names](#13-redis-env-var-names)
- [#14 — Login always fails: BCrypt hash prefix mismatch](#14-bcrypt-prefix)
- [#15 — Staging pods load dev config: docker profile instead of staging](#15-docker-vs-staging-profile)
- [#16 — Pods stuck in Pending: VPC IP address exhaustion on EKS](#16-ip-exhaustion)
- [#17 — eksctl IRSA silently fails: pods cannot assume IAM role](#17-eksctl-irsa-silent-failure)
- [#18 — PDB never matches any pods: label selector mismatch](#18-pdb-label-mismatch)
- [#19 — HPA and PDB reference wrong deployment name in Helm template](#19-hpa-pdb-name-mismatch)

**Category D — Terraform Infrastructure**
- [#20 — Terraform destroy fails: ELB still holds security group references](#20-terraform-destroy-elb)
- [#21 — Orphaned ELB security group blocks VPC deletion](#21-orphaned-elb-sg)
- [#22 — Terraform state file locked: no commands can run](#22-terraform-state-lock)
- [#23 — "Variable specified twice" error breaks terraform plan](#23-variable-specified-twice)
- [#24 — Staging database missing after every Terraform rebuild](#24-staging-db-missing)
- [#25 — Double port in JDBC URL: Terraform db_endpoint includes :3306](#25-double-port-jdbc)

**Category E — AWS Networking**
- [#26 — ALB created but health checks fail: missing IAM policy](#26-alb-missing-iam)
- [#27 — Backend ingress routes to wrong path: rewrite annotation missing](#27-ingress-wrong-path)
- [#28 — Subnet IDs change after every EKS rebuild](#28-subnet-ids-change)
- [#29 — IMDS hop limit: ALB Ingress Controller cannot detect cluster VPC](#29-imds-hop-limit)

**Category F — GitOps, ArgoCD & Application**
- [#30 — ArgoCD repo credentials silently dropped in zsh](#30-argocd-zsh-credentials)
- [#31 — All services break after EKS rebuild: OIDC provider ID changed](#31-oidc-id-changed)
- [#32 — ArgoCD ApplicationSet crashes on install](#32-argocd-applicationset-crash)
- [#33 — Registration API returns Validation Failed: missing required fields](#33-validation-failed-registration)

---

## Category A — Docker & Container Issues

---

### #1 — exec format error

**Severity:** Critical | **Category:** Docker Multi-Architecture | **Environments affected:** Dev, Staging, Prod

#### What Happened

After the first successful Jenkins pipeline run — images built, pushed to ECR, ArgoCD synced — every single pod went into CrashLoopBackOff. No network issue, no config error. The pod log showed just one line:

```
standard_init_linux.go:228: exec user process caused: exec format error
```

The pods kept restarting. Nothing else in the logs.

#### Root Cause

My laptop is a Mac with Apple Silicon (M1 chip). It uses **ARM64** architecture — the same family of chip used in phones and tablets.

AWS EKS worker nodes run on EC2 t3.medium instances. Those use **AMD64** (Intel x86_64) architecture — the traditional "PC" chip.

These two chip types speak different instruction languages. A program compiled for ARM64 cannot run on AMD64, and vice versa.

When Jenkins ran the Docker build on my Mac, it produced an ARM64 image. That image was pushed to ECR. When EKS (AMD64) tried to run it: `exec format error` — the CPU literally could not understand the binary.

Think of it like a French-only speaker being handed a book written entirely in Japanese. Same "book", unreadable.

#### How to Diagnose

```bash
# Check what architecture your Docker image was built for
docker inspect <YOUR_AWS_ACCOUNT_ID>.dkr.ecr.ap-south-1.amazonaws.com/finbank-backend:1.0.0-44 \
  | grep -i architecture

# Check what architecture your EKS nodes use
kubectl get nodes -o jsonpath='{range .items[*]}{.metadata.name}{"\t"}{.status.nodeInfo.architecture}{"\n"}{end}'

# Check the pod error
kubectl describe pod finbank-backend-xxxxxx -n finbank-dev
kubectl logs finbank-backend-xxxxxx -n finbank-dev --previous
```

#### Exact Fix (Step by Step)

Use **Docker Buildx** — a cross-compilation tool built into Docker. It builds for multiple architectures at once and pushes a single image tag that works on all of them.

**Step 1: Update Stage 2 of Jenkinsfile — install Buildx and detect architecture**

```groovy
stage('Install Tools') {
    steps {
        sh '''
            # Detect what CPU this Jenkins agent runs on
            ARCH=$(uname -m)
            if [ "$ARCH" = "arm64" ] || [ "$ARCH" = "aarch64" ]; then
                ARCH_SUFFIX="arm64"
            else
                ARCH_SUFFIX="amd64"
            fi

            # Install Trivy for the correct architecture
            curl -L "https://github.com/aquasecurity/trivy/releases/download/v0.50.0/trivy_0.50.0_Linux-${ARCH_SUFFIX}.tar.gz" -o trivy.tar.gz
            tar -xzf trivy.tar.gz trivy
            chmod +x trivy

            # Install AWS CLI for the correct architecture
            curl "https://awscli.amazonaws.com/awscli-exe-linux-${ARCH_SUFFIX}.zip" -o "awscliv2.zip"
            unzip -q awscliv2.zip
            ./aws/install --update
        '''
    }
}
```

**Step 2: Update the Docker Build stage to use Buildx**

```groovy
stage('Docker Build & Push') {
    steps {
        sh '''
            # Create a Buildx builder that can cross-compile
            docker buildx create \
              --name finbank-multiarch-builder \
              --driver docker-container \
              --use

            # Start it up (downloads QEMU emulation layers)
            docker buildx inspect finbank-multiarch-builder --bootstrap

            # Build for BOTH architectures and push to ECR in one command
            # --platform means: build a version for AMD64 AND a version for ARM64
            # --push means: push directly to ECR (not to local Docker)
            docker buildx build \
              --platform linux/amd64,linux/arm64 \
              --tag <YOUR_AWS_ACCOUNT_ID>.dkr.ecr.ap-south-1.amazonaws.com/finbank-backend:1.0.0-${BUILD_NUMBER} \
              --push \
              .

            # Clean up the builder
            docker buildx rm finbank-multiarch-builder || true
        '''
    }
}
```

ECR now stores a **manifest list** — a single tag (`1.0.0-44`) that contains two images inside. When EKS pulls it on AMD64 nodes, it gets the AMD64 version. When your Mac pulls it locally, it gets ARM64. All automatic.

#### Prevention

- Always use `docker buildx build --platform linux/amd64,linux/arm64` in Jenkins pipelines when developers use Mac M1/M2.
- Add a pipeline check: if `docker inspect` shows architecture is `arm64`, fail the build with a clear error before pushing.

#### Interview Answer (STAR Format)

**Situation:** During the FinBank project, after my first successful Jenkins pipeline run that built and pushed Docker images to ECR, every pod went into CrashLoopBackOff on EKS immediately after ArgoCD synced.

**Task:** I needed to diagnose why all pods were crashing and fix the deployments across all three microservices (backend, frontend, analytics).

**Action:** I checked pod logs with `kubectl logs --previous` and saw `exec format error`. I recognised this error means the CPU cannot understand the binary's instruction set. I checked the node architecture with `kubectl get nodes` — AMD64. I checked the image I built on my Mac M1 — ARM64. The mismatch was clear. I updated the Jenkins pipeline to use Docker Buildx with `--platform linux/amd64,linux/arm64`, which builds both architecture versions and pushes a manifest list to ECR. EKS automatically picks the AMD64 version.

**Result:** All pods started successfully. The fix took about 30 minutes to implement. This is now standard practice in the pipeline — any developer on Mac M1 or M2 can push images that run correctly on AMD64 EC2 nodes.

---

### #2 — ECR Multi-Arch Deletion

**Severity:** Medium | **Category:** ECR Image Management

#### What Happened

When cleaning up old ECR images to save storage costs, the standard deletion command failed. Some images deleted but left orphaned entries. Trying to delete the ECR repository itself failed with "repository is not empty" — even after deleting all visible tags.

```
An error occurred (ImageNotFoundException) when calling the BatchDeleteImage operation
An error occurred (RepositoryNotEmptyException) when calling the DeleteRepository operation:
  The repository with name 'finbank-backend' in registry with id '<YOUR_AWS_ACCOUNT_ID>' cannot be deleted
  because it still contains images
```

#### Root Cause

Multi-arch images in ECR have a **3-layer structure**, not 1 layer like regular images:

```
Layer 1: Manifest List   → the tag you see (e.g., 1.0.0-44)
   ├── Layer 2a: AMD64 manifest   → invisible in the console, no tag
   │     └── Layer 3a: AMD64 image layers (the actual data)
   └── Layer 2b: ARM64 manifest   → invisible in the console, no tag
         └── Layer 3b: ARM64 image layers (the actual data)
```

When you delete Layer 1 (the visible tag), AWS removes just that entry. Layers 2a and 2b survive as **untagged digests** — invisible in the console but still counted as images by the repository. ECR refuses to delete a repository that has any images, tagged or untagged.

#### How to Diagnose

```bash
# List ALL images including untagged ones
aws ecr list-images \
  --repository-name finbank-backend \
  --region ap-south-1 \
  --output table

# Check untagged images specifically
aws ecr list-images \
  --repository-name finbank-backend \
  --filter tagStatus=UNTAGGED \
  --region ap-south-1
```

#### Exact Fix (Step by Step)

**Option A — Force delete the entire repository (fastest for cleanup):**

```bash
aws ecr delete-repository \
  --repository-name finbank-backend \
  --region ap-south-1 \
  --force    # --force deletes all images inside, tagged and untagged
```

**Option B — Delete images in the correct order (when you want to keep the repository):**

```bash
REPO="finbank-backend"
REGION="ap-south-1"

# Step 1: Delete tagged images (the manifest lists)
aws ecr batch-delete-image \
  --repository-name $REPO \
  --image-ids imageTag=1.0.0-44 \
  --region $REGION

# Step 2: Delete all untagged images (orphaned architecture manifests)
UNTAGGED=$(aws ecr list-images \
  --repository-name $REPO \
  --filter tagStatus=UNTAGGED \
  --query 'imageIds' \
  --region $REGION \
  --output json)

if [ "$UNTAGGED" != "[]" ]; then
  aws ecr batch-delete-image \
    --repository-name $REPO \
    --image-ids "$UNTAGGED" \
    --region $REGION
fi
```

**Option C — Enable ECR Lifecycle Policy to auto-clean old images:**

```json
{
  "rules": [
    {
      "rulePriority": 1,
      "description": "Keep only last 5 tagged images",
      "selection": {
        "tagStatus": "tagged",
        "tagPrefixList": ["1.0.0"],
        "countType": "imageCountMoreThan",
        "countNumber": 5
      },
      "action": { "type": "expire" }
    },
    {
      "rulePriority": 2,
      "description": "Delete untagged images after 1 day",
      "selection": {
        "tagStatus": "untagged",
        "countType": "sinceImagePushed",
        "countUnit": "days",
        "countNumber": 1
      },
      "action": { "type": "expire" }
    }
  ]
}
```

Apply with:
```bash
aws ecr put-lifecycle-policy \
  --repository-name finbank-backend \
  --lifecycle-policy-text file://lifecycle-policy.json \
  --region ap-south-1
```

#### Prevention

- Apply ECR Lifecycle Policies to all repositories during Terraform provisioning. This cleans up untagged manifests automatically after 1 day.
- Never try to delete a multi-arch ECR repository without `--force` flag.

#### Interview Answer (STAR Format)

**Situation:** While doing a cost cleanup on the FinBank project's ECR repositories, I tried to delete old images and got "repository is not empty" errors even after deleting all visible tagged images.

**Task:** I needed to fully clean up the ECR repository so I could delete and recreate it as part of a Terraform teardown.

**Action:** I listed all images including untagged ones and discovered dozens of orphaned architecture manifests — the AMD64 and ARM64 sub-manifests left behind when I deleted the parent manifest list tags. I learned that multi-arch images have a 3-layer structure and you must delete in the correct order: parent manifest list first, then the untagged architecture manifests. For the cleanup I used `--force` flag. Going forward I added ECR Lifecycle Policies in Terraform to auto-expire untagged images after 1 day.

**Result:** Repositories cleaned up successfully. The Lifecycle Policy is now part of the Terraform ECR module so it applies to all three service repositories automatically.

---

### #3 — Jenkins DNS Failure

**Severity:** High | **Category:** Jenkins Networking

#### What Happened

Stage 2 of the Jenkins pipeline (Install Tools) failed immediately with DNS errors while trying to download AWS CLI and Trivy:

```
curl: (6) Could not resolve host: awscli.amazonaws.com
curl: (6) Could not resolve host: github.com
Retrying... (1/3)
curl: (6) Could not resolve host: awscli.amazonaws.com
ERROR: Build failed in stage: Install Tools
```

The same curl commands worked perfectly from my Mac terminal. The issue was specific to Jenkins.

#### Root Cause

Jenkins runs inside a Docker container. Docker containers use Docker's internal DNS resolver (at IP `127.0.0.11`) by default — not your Mac's system DNS.

When the Mac's VPN is active, or when Docker's network configuration is inconsistent, the container's internal DNS resolver cannot reach public DNS servers. The container cannot resolve `github.com` or `awscli.amazonaws.com` even though your Mac can.

Think of it like being in a hotel room with no internet, while your phone (on mobile data) works fine. Two different network paths.

#### How to Diagnose

```bash
# Test DNS inside the Jenkins container
docker exec jenkins nslookup github.com
docker exec jenkins curl -v https://github.com 2>&1 | head -20

# Check what DNS servers the container is using
docker exec jenkins cat /etc/resolv.conf

# Check Docker's DNS configuration
docker inspect jenkins | grep -A5 '"Dns"'
```

#### Exact Fix (Step by Step)

**Step 1: Stop and remove the existing Jenkins container**

```bash
docker stop jenkins
docker rm jenkins
```

**Step 2: Recreate Jenkins with explicit Google DNS servers**

```bash
docker run -d \
  --name jenkins \
  --dns 8.8.8.8 \
  --dns 8.8.4.4 \
  -p 9090:8080 \
  -v jenkins_home:/var/jenkins_home \
  -v /var/run/docker.sock:/var/run/docker.sock \
  -v $(which docker):/usr/bin/docker \
  jenkins/jenkins:lts-jdk21
```

The `--dns 8.8.8.8` and `--dns 8.8.4.4` flags tell the container to always use Google's public DNS instead of Docker's internal resolver.

**Step 3: Verify DNS works inside container**

```bash
docker exec jenkins nslookup github.com
# Should return: Address: 140.82.x.x (GitHub's IP)
```

#### Prevention

Always include `--dns 8.8.8.8 --dns 8.8.4.4` in the `docker run` command for Jenkins. Document this in your runbook so rebuilds always use these flags.

#### Interview Answer (STAR Format)

**Situation:** During the FinBank project setup, Jenkins would fail on Stage 2 — Install Tools — with DNS resolution errors for AWS and GitHub domains, but the same curl commands worked from my Mac terminal.

**Task:** I needed Jenkins to be able to download Trivy and AWS CLI during every pipeline run.

**Action:** I checked DNS resolution inside the Jenkins container with `docker exec jenkins nslookup github.com` and got no response, while `nslookup github.com` on my Mac worked. I traced the issue to Docker's internal DNS resolver being unable to reach public servers. The fix was recreating the Jenkins container with `--dns 8.8.8.8 --dns 8.8.4.4` to bypass Docker's DNS and use Google's public resolvers directly.

**Result:** DNS resolution inside Jenkins worked immediately. All tool downloads in Stage 2 succeeded. I added these DNS flags to the team's Jenkins setup documentation so rebuilds don't hit the same issue.

---

### #4 — Rosetta Maven Error

**Severity:** Medium | **Category:** Docker Architecture Emulation

#### What Happened

Inside the Jenkins container on Mac M1, the Maven build in Stage 3 produced a warning and then crashed:

```
WARNING: The requested image's platform (linux/amd64) does not match the detected host platform (linux/arm64/v8)
...
Error: An AVX2 instruction was encountered but the CPU does not support it.
Rosetta 2 translation layer: unsupported instruction
BUILD FAILED
```

The build was also extremely slow before it crashed.

#### Root Cause

Jenkins was running as an AMD64 Docker image (`jenkins/jenkins:lts`) on an ARM64 Mac M1. macOS Rosetta 2 is Apple's compatibility layer — it translates AMD64 instructions to ARM64 in real-time.

Maven tried to use **AVX2** instructions (Advanced Vector Extensions 2 — a set of special high-performance CPU instructions available on Intel chips). Rosetta 2 does not emulate AVX2. The translation failed and the build crashed.

The extra slowness was from all other AMD64 instructions being emulated (translated) in real-time — expensive on the CPU.

#### How to Diagnose

```bash
# Check what platform the Jenkins container is running as
docker inspect jenkins | grep -i platform

# Check inside the container
docker exec jenkins uname -m
# If it shows x86_64, you're running AMD64 emulated on ARM64 Mac
# If it shows aarch64, you're running native ARM64
```

#### Exact Fix (Step by Step)

Use the native **ARM64 Jenkins image** to eliminate Rosetta 2 emulation entirely:

```bash
# Stop and remove the old Jenkins container
docker stop jenkins && docker rm jenkins

# Pull the native ARM64 Jenkins image
docker pull --platform linux/arm64 jenkins/jenkins:lts-jdk21

# Recreate Jenkins with native ARM64 platform
docker run -d \
  --name jenkins \
  --platform linux/arm64 \
  --dns 8.8.8.8 \
  --dns 8.8.4.4 \
  -p 9090:8080 \
  -v jenkins_home:/var/jenkins_home \
  -v /var/run/docker.sock:/var/run/docker.sock \
  -v $(which docker):/usr/bin/docker \
  jenkins/jenkins:lts-jdk21
```

The Maven build now runs natively on ARM64 — no translation layer, no AVX2 issue, 3-5x faster.

**Note:** This is separate from the Buildx fix in #1. Jenkins itself runs ARM64 natively, but the Docker images it *builds* are compiled for both AMD64 and ARM64 using Buildx.

#### Prevention

On any Mac M1/M2 setup, always pull and run Jenkins with `--platform linux/arm64`. Document this in the team setup guide.

#### Interview Answer (STAR Format)

**Situation:** On the FinBank project, Maven builds inside the Jenkins container were very slow and eventually crashed with an error about AVX2 CPU instructions not being available.

**Task:** I needed Maven to build the Spring Boot backend without crashes.

**Action:** I diagnosed that Jenkins was running an AMD64 Docker image through Rosetta 2 emulation. Rosetta doesn't support AVX2 instructions that Maven uses. I switched to the native ARM64 Jenkins image (`--platform linux/arm64`) which runs without any emulation layer. This is a different concern from the Buildx multi-arch fix — Jenkins itself runs ARM64 natively, while the artifacts it builds target both architectures.

**Result:** Maven builds became significantly faster (no emulation overhead) and the AVX2 crash stopped. This taught me to always check whether development tools are running natively or emulated on Mac M1.

---

### #5 — Trivy Permission Denied

**Severity:** High | **Category:** CI/CD Security Scanning

#### What Happened

Stage 2 of the Jenkins pipeline failed with a permission error when trying to install Trivy (the security scanner):

```
curl -L https://github.com/aquasecurity/trivy/releases/download/v0.50.0/trivy_0.50.0_Linux-arm64.tar.gz -o trivy.tar.gz
tar -xzf trivy.tar.gz -C /usr/local/bin trivy
install: cannot create regular file '/usr/local/bin/trivy': Permission denied
ERROR: Stage 'Install Tools' failed
```

#### Root Cause

The Jenkins container runs as the `jenkins` user — not the root user. This is a security best practice: never run application containers as root.

`/usr/local/bin` is a system directory owned by `root`. The `jenkins` user does not have write permission to it. Trying to extract Trivy directly into `/usr/local/bin` fails because `jenkins` cannot write there.

Think of it like trying to put files into a room that's locked — you have the files, but no key to the door.

#### How to Diagnose

```bash
# Check what user Jenkins runs as
docker exec jenkins whoami
# Output: jenkins

# Check permissions on /usr/local/bin
docker exec jenkins ls -la /usr/local/ | grep bin
# Output: drwxr-xr-x  root root  /usr/local/bin
# 'r-x' for others = read and execute only, no write

# Confirm you cannot write there
docker exec jenkins touch /usr/local/bin/test 2>&1
# Output: touch: cannot touch '/usr/local/bin/test': Permission denied
```

#### Exact Fix (Step by Step)

Install Trivy to the `jenkins` user's home directory (`$HOME/bin`) instead of the system directory:

```groovy
stage('Install Tools') {
    steps {
        sh '''
            # Detect architecture
            ARCH=$(uname -m)
            if [ "$ARCH" = "arm64" ] || [ "$ARCH" = "aarch64" ]; then
                ARCH_SUFFIX="arm64"
            else
                ARCH_SUFFIX="amd64"
            fi

            # Create a personal bin directory for the jenkins user
            mkdir -p $HOME/bin

            # Download and extract Trivy to the jenkins user's own directory
            curl -L "https://github.com/aquasecurity/trivy/releases/download/v0.50.0/trivy_0.50.0_Linux-${ARCH_SUFFIX}.tar.gz" -o trivy.tar.gz
            tar -xzf trivy.tar.gz -C $HOME/bin trivy
            chmod +x $HOME/bin/trivy

            # Add $HOME/bin to PATH so Trivy can be called by name
            export PATH="$HOME/bin:$PATH"

            # Verify it works
            trivy --version
        '''
    }
}
```

**Important:** The `export PATH` only lasts for that `sh` block. In later pipeline stages that call `trivy`, add the PATH export at the top of each `sh` block:

```groovy
stage('Security Scan') {
    steps {
        sh '''
            export PATH="$HOME/bin:$PATH"
            trivy image --exit-code 1 --severity CRITICAL,HIGH \
              <YOUR_AWS_ACCOUNT_ID>.dkr.ecr.ap-south-1.amazonaws.com/finbank-backend:1.0.0-${BUILD_NUMBER}
        '''
    }
}
```

#### Prevention

- Never install tools to `/usr/local/bin` in CI pipelines that run as non-root users.
- Always install to `$HOME/bin` or `$WORKSPACE` (the pipeline workspace directory).
- Add Trivy to the Jenkins Docker image in a custom Dockerfile so it does not need to be installed every run.

#### Interview Answer (STAR Format)

**Situation:** In the FinBank Jenkins pipeline, Stage 2 (Install Tools) kept failing with "Permission denied" when trying to install Trivy into `/usr/local/bin`.

**Task:** I needed Trivy to be available in all pipeline stages so the security scanning stage could run.

**Action:** I ran `docker exec jenkins whoami` and confirmed Jenkins runs as the non-root `jenkins` user. `/usr/local/bin` is a system directory writable only by root. The fix was redirecting the Trivy install to `$HOME/bin` — a directory the jenkins user owns — and adding `$HOME/bin` to PATH so Trivy is callable by name in subsequent stages.

**Result:** Stage 2 completed without permission errors and Trivy security scanning in Stage 7 worked correctly. I also noted this is a good security practice: running CI as non-root means a compromised build cannot modify system binaries.

---

## Category B — Jenkins CI/CD

---

### #6 — Jenkins Credential Mismatch

**Severity:** Critical | **Category:** Jenkins Configuration

#### What Happened

Stage 9 of the pipeline (Update Helm Chart in GitHub) failed every run with authentication errors:

```
remote: Invalid username or password.
fatal: Authentication failed for 'https://github.com/Anshujee/FinBank-DevOps-Notes.git'
ERROR: ERROR during step
  hudson.plugins.git.GitException: Command "git push" returned status code 128
  credentialId 'github-credentials' not found in the credential store
```

The credentials were definitely configured in Jenkins — I could see them in the Jenkins UI. But the pipeline kept saying "not found".

#### Root Cause

The Jenkinsfile referenced credential ID `'github-credentials'` (with a hyphen). The credential in Jenkins was created with ID `'github credentials'` (with a **space**).

```
Jenkinsfile used:    'github-credentials'   ← hyphen
Jenkins stored:      'github credentials'   ← space
```

Jenkins does **exact string matching** on credential IDs. One character difference (hyphen vs space) means it cannot find the credential. It throws "not found" instead of "wrong credential" because — from Jenkins' perspective — that specific ID genuinely does not exist.

#### How to Diagnose

```bash
# Go to Jenkins → Manage Jenkins → Credentials → Global → (any credential) → Update
# Look at the "ID" field EXACTLY as stored — character by character

# In your Jenkinsfile, search for credentialsId
grep -n "credentialsId" Jenkinsfile
```

Compare the two strings character by character. Look for: spaces vs hyphens, uppercase vs lowercase, underscores vs hyphens.

#### Exact Fix (Step by Step)

**Option A — Change the Jenkinsfile to match what Jenkins has stored:**

```groovy
withCredentials([
    usernamePassword(
        credentialsId: 'github credentials',  // ← add a space to match Jenkins
        usernameVariable: 'GIT_USER',
        passwordVariable: 'GIT_TOKEN'
    )
])
```

**Option B (recommended) — Rename the Jenkins credential to match the Jenkinsfile standard:**

1. Go to Jenkins → Manage Jenkins → Credentials → Global
2. Find the GitHub credential → click the dropdown → Update
3. In the **ID** field, change `github credentials` to `github-credentials`
4. Save

**Standard naming convention for all FinBank credentials:**
```
github-credentials          ← GitHub PAT
aws-access-key-id           ← AWS access key
aws-secret-access-key       ← AWS secret key
sonarqube-token             ← SonarQube analysis token
dockerhub-credentials       ← Docker Hub login (if needed)
```

Rule: all lowercase, words separated by hyphens, no spaces, no underscores.

#### Prevention

- Write all credential IDs in a shared document before setting them up in Jenkins.
- Copy credential IDs from Jenkins UI using copy-paste — never type them by hand.
- Use a consistent convention from day one: lowercase-hyphenated.

#### Interview Answer (STAR Format)

**Situation:** In the FinBank Jenkins pipeline, Stage 9 — which pushes updated Helm chart values to GitHub — failed every run with "credential not found" even though the credential was visible in the Jenkins UI.

**Task:** I needed Stage 9 to authenticate to GitHub and push the updated image tag so ArgoCD could pick it up.

**Action:** I spent about 20 minutes checking GitHub PAT expiry, network access, webhook configuration — everything looked fine. Eventually I compared the credential ID in the Jenkinsfile against the Jenkins UI string character by character. The Jenkinsfile had `github-credentials` (hyphen); Jenkins had stored it as `github credentials` (space). Jenkins uses exact string matching on IDs. I renamed the credential in Jenkins to use a hyphen. I also established a naming convention: all lowercase, hyphen-separated, no spaces.

**Result:** Stage 9 authenticated and pushed successfully. I documented the credential naming convention in the project README so future teammates don't hit the same issue.

---

### #7 — Staging Image Tag Never Updated

**Severity:** High | **Category:** Jenkins CI/CD, GitOps

#### What Happened

After a successful Jenkins pipeline run (all 9 stages green), the dev environment was running the new image tag `1.0.0-44`. Staging was still on `1.0.0-38`. ArgoCD showed staging as "Synced and Healthy" — which was confusing because it was running the old version.

#### Root Cause

Stage 9 of the Jenkinsfile updated only `values.yaml` (the dev defaults), not the environment-specific override files:

```bash
# What Stage 9 was doing — WRONG
sed -i 's|  tag:.*|  tag: 1.0.0-44|' helm/finbank-backend/values.yaml
# Only updates dev. Staging and prod Helm values not touched.
```

ArgoCD's staging application uses `values-staging.yaml` as its values file. Since that file still had `tag: 1.0.0-38`, ArgoCD deployed `1.0.0-38`. And reported it as "Synced" — because the cluster matched Git exactly. Git itself had the wrong value.

This is an important lesson: **"Synced" in ArgoCD means "cluster matches Git" — not "cluster has the latest build"**. If Git is wrong, ArgoCD will dutifully deploy the wrong version.

#### How to Diagnose

```bash
# Check what image tag is in each values file
grep "tag:" helm/finbank-backend/values.yaml
grep "tag:" helm/finbank-backend/values-staging.yaml
grep "tag:" helm/finbank-backend/values-prod.yaml

# Check what image is actually running in staging
kubectl get pods -n finbank-stage -o jsonpath='{range .items[*]}{.spec.containers[0].image}{"\n"}{end}'

# Check what ArgoCD thinks staging is synced to
argocd app get finbank-backend-staging
```

#### Exact Fix (Step by Step)

Update ALL three values files in Stage 9:

```groovy
stage('Update Helm Chart Image Tags') {
    steps {
        withCredentials([
            usernamePassword(
                credentialsId: 'github-credentials',
                usernameVariable: 'GIT_USER',
                passwordVariable: 'GIT_TOKEN'
            )
        ]) {
            sh '''
                NEW_TAG="1.0.0-${BUILD_NUMBER}"

                git config user.email "jenkins@finbank.local"
                git config user.name "Jenkins CI"

                # ← Update ALL THREE environment values files
                sed -i "s|  tag:.*|  tag: ${NEW_TAG}|g" helm/finbank-backend/values.yaml
                sed -i "s|  tag:.*|  tag: ${NEW_TAG}|g" helm/finbank-backend/values-staging.yaml
                sed -i "s|  tag:.*|  tag: ${NEW_TAG}|g" helm/finbank-backend/values-prod.yaml

                # Stage all three files
                git add helm/finbank-backend/values.yaml \
                        helm/finbank-backend/values-staging.yaml \
                        helm/finbank-backend/values-prod.yaml

                git commit -m "CI: Update finbank-backend image to ${NEW_TAG} [skip ci]"

                git push https://${GIT_USER}:${GIT_TOKEN}@github.com/Anshujee/FinBank-DevOps-Notes.git HEAD:main
            '''
        }
    }
}
```

#### Prevention

- After every pipeline run, verify all environments by running:
  ```bash
  grep "tag:" helm/finbank-backend/values*.yaml
  ```
  All should show the same tag.
- Add a post-stage validation step in Jenkins that checks all three files match before declaring success.

#### Interview Answer (STAR Format)

**Situation:** In the FinBank project, after successful Jenkins pipeline runs, staging was consistently running older image versions even though dev was updated. ArgoCD showed staging as "Synced and Healthy" which made it harder to spot the problem.

**Task:** I needed all environments to receive the new image tag after each pipeline run.

**Action:** I compared the tag values across `values.yaml`, `values-staging.yaml`, and `values-prod.yaml` in Git. Only `values.yaml` had the new tag. Stage 9 only ran `sed` on the base values file. I updated Stage 9 to run `sed` on all three files in the same commit and push. I also learned that ArgoCD's "Synced" status means the cluster matches Git — not that Git is correct. A stale Git value causes a stale deployment that ArgoCD calls "healthy."

**Result:** All three environments now update simultaneously after each pipeline run. The `[skip ci]` flag in the commit message also prevents an infinite loop where the Git push would trigger another pipeline run.

---

### #8 — zsh Bracket Expansion Breaks Terraform

**Severity:** Medium | **Category:** Shell Scripting, Terraform

#### What Happened

When trying to override a Terraform variable from the command line to test a staging configuration, the command crashed with a bizarre error:

```
zsh: no matches found: {ap-south-1a,ap-south-1b}
```

The command was:
```bash
terraform apply -var="availability_zones={ap-south-1a,ap-south-1b}"
```

Terraform never even started. Zsh intercepted the argument and threw an error.

#### Root Cause

Zsh (the default macOS shell since Catalina) has a feature called **brace expansion**. When zsh sees `{ap-south-1a,ap-south-1b}` in a command, it tries to expand it into multiple separate arguments: `ap-south-1a` and `ap-south-1b` — like two separate filenames.

In this context, zsh tries to find files or directories named `ap-south-1a` and `ap-south-1b` in the current directory. They don't exist, so zsh throws "no matches found" and never even runs the command.

Terraform never sees the argument at all. This is a shell-level error, not a Terraform error.

#### How to Diagnose

```bash
# Test if zsh is expanding your braces
echo {ap-south-1a,ap-south-1b}
# If it prints two separate words, zsh is expanding them
# Output: ap-south-1a ap-south-1b

# Check what shell you're using
echo $SHELL
# Output: /bin/zsh
```

#### Exact Fix (Step by Step)

**Option A — Escape the braces with backslashes:**

```bash
terraform apply -var="availability_zones=\{ap-south-1a,ap-south-1b\}"
```

**Option B (recommended) — Use single quotes around the entire value:**

```bash
terraform apply -var='availability_zones={ap-south-1a,ap-south-1b}'
```

Single quotes in bash/zsh prevent ALL shell interpretation of the content inside. The string is passed to Terraform exactly as written.

**Option C — Use a .tfvars file instead of -var flags:**

```hcl
# terraform.tfvars or staging.tfvars
availability_zones = ["ap-south-1a", "ap-south-1b"]
```

```bash
terraform apply -var-file="staging.tfvars"
```

This is the cleanest approach for multi-value variables and avoids shell escaping entirely.

#### Prevention

- For complex variable values (lists, maps, strings with special characters), always use `.tfvars` files rather than `-var` flags.
- If using `-var` flags with list/map values, always use single quotes.

#### Interview Answer (STAR Format)

**Situation:** While testing Terraform configuration changes for the FinBank staging environment, a `terraform apply` command with a list variable in `-var` flag failed with "zsh: no matches found" before Terraform even started.

**Task:** I needed to pass a list of availability zones as a Terraform variable override from the command line.

**Action:** I recognised the error was from zsh's brace expansion feature, not Terraform. Zsh was intercepting `{ap-south-1a,ap-south-1b}` and trying to glob-expand it into filenames. Two fixes: use single quotes around the value to prevent shell interpretation, or move the variable to a `.tfvars` file. I switched to a `staging.tfvars` file which is cleaner and avoids shell escaping issues entirely.

**Result:** Terraform accepted the variable correctly. I documented this in the team notes: always use `.tfvars` files for list and map variables, never `-var` flags for complex types on zsh.

---

### #9 — New CVEs Block Build #43

**Severity:** Critical | **Category:** Security Scanning, Dependency Management

#### What Happened

Build #43 in the Jenkins pipeline passed all stages through Stage 6 (Docker push) but failed in Stage 7 (Trivy Security Scan):

```
2026-05-14T09:23:11.442Z  INFO  Detected OS: debian 12.5
2026-05-14T09:23:11.443Z  INFO  Detecting Debian vulnerabilities...
2026-05-14T09:23:45.211Z  INFO  Number of language-specific files: 1

finbank-backend:1.0.0-43 (jar)
===============================
Total: 2 (CRITICAL: 2, HIGH: 0)

┌─────────────────────────────────────┬────────────────────┬──────────┬──────────────────────────────────────┐
│ Library                             │ Vulnerability ID   │ Severity │ Fixed Version                        │
├─────────────────────────────────────┼────────────────────┼──────────┼──────────────────────────────────────┤
│ spring-boot-3.5.13.jar              │ CVE-2026-40973     │ CRITICAL │ 3.5.14                               │
├─────────────────────────────────────┼────────────────────┼──────────┼──────────────────────────────────────┤
│ netty-codec-http-4.1.108.Final.jar  │ CVE-2026-42583     │ CRITICAL │ 4.1.133.Final                        │
└─────────────────────────────────────┴────────────────────┴──────────┴──────────────────────────────────────┘

FAIL: 2 CRITICAL vulnerabilities found. Blocking build.
ERROR: Build failed: exit code 1
```

The image was never deployed. Builds #44, #45, #46 also failed until dependencies were upgraded.

#### Root Cause

**CVE-2026-40973** — A vulnerability in Spring Boot 3.5.13 that allows malicious file writes through temporary directory path traversal. An attacker can write files outside the intended temp directory.

**CVE-2026-42583** — A vulnerability in Netty 4.1.108 (an HTTP networking library that Spring Boot uses internally) that allows memory exhaustion under specific HTTP/2 connection patterns.

The Trivy scanner in Stage 7 is configured with `--exit-code 1` for CRITICAL and HIGH severity, which means it blocks the build if any such vulnerabilities are found. This is the correct behavior — the pipeline deliberately refuses to deploy vulnerable images.

#### How to Diagnose

```bash
# Scan the image locally to see what Trivy finds
export PATH="$HOME/bin:$PATH"
trivy image \
  --severity CRITICAL,HIGH \
  <YOUR_AWS_ACCOUNT_ID>.dkr.ecr.ap-south-1.amazonaws.com/finbank-backend:1.0.0-43

# Check your current Spring Boot version in pom.xml
grep -A2 "spring-boot" backend/pom.xml | grep "version"

# Check what Netty version Spring Boot uses (transitive dependency)
cd backend && mvn dependency:tree | grep netty
```

#### Exact Fix (Step by Step)

**Step 1: Update Spring Boot version in `backend/pom.xml`**

```xml
<!-- backend/pom.xml — change this line -->
<parent>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-parent</artifactId>
    <version>3.5.14</version>  <!-- ← was 3.5.13, changed to 3.5.14 -->
    <relativePath/>
</parent>
```

**Step 2: Force the Netty version to fix CVE-2026-42583**

Spring Boot manages Netty as a transitive dependency (Netty is pulled in through Spring WebFlux or Spring Boot Web). To force a specific Netty version, add a property in `pom.xml`:

```xml
<!-- In backend/pom.xml, inside the <properties> section -->
<properties>
    <java.version>17</java.version>
    <netty.version>4.1.133.Final</netty.version>  <!-- ← force safe Netty version -->
</properties>
```

Spring Boot's dependency management respects this property and will use `4.1.133.Final` instead of the vulnerable `4.1.108.Final`.

**Step 3: Verify the fix locally before pushing**

```bash
cd backend
mvn dependency:tree | grep netty
# Should show: netty-codec-http:4.1.133.Final

mvn clean package -DskipTests
docker build -t finbank-backend:test .
trivy image --severity CRITICAL,HIGH finbank-backend:test
# Should show: Total: 0 (CRITICAL: 0, HIGH: 0)
```

**Step 4: Commit and push to trigger a new pipeline build**

```bash
git add backend/pom.xml
git commit -m "fix: upgrade Spring Boot 3.5.14 and force Netty 4.1.133 to fix CVE-2026-40973, CVE-2026-42583"
git push origin main
```

Build #47 (or whichever is next) should pass Trivy scan with zero CRITICAL/HIGH findings.

#### Prevention

- Subscribe to Spring Boot security advisories: `https://spring.io/security`
- Run `trivy image` locally before committing code that changes dependencies.
- Set up a Dependabot or Renovate bot to automatically raise PRs when known-vulnerable library versions are detected.
- Review Trivy output in Jenkins every build — even when it passes — to track MEDIUM severity items before they become CRITICAL.

#### Interview Answer (STAR Format)

**Situation:** On build #43 of the FinBank backend pipeline, the Trivy security scan stage found two CRITICAL CVEs — one in Spring Boot 3.5.13 and one in Netty 4.1.108. The pipeline was configured to block deployment for any CRITICAL vulnerability, so builds #43 through #46 all failed at Stage 7. The new version was not deployed to any environment.

**Task:** I needed to upgrade the affected libraries so the image passed Trivy's security gate and could be deployed.

**Action:** I checked the CVE details — CVE-2026-40973 was a temp directory path traversal in Spring Boot, fixed in 3.5.14. CVE-2026-42583 was a Netty memory exhaustion issue, fixed in 4.1.133.Final. I updated `pom.xml` to use Spring Boot 3.5.14 as the parent version. Netty is a transitive dependency (pulled in by Spring Boot), so I added `<netty.version>4.1.133.Final</netty.version>` to the properties section to force the safe version. I verified with `mvn dependency:tree | grep netty` and ran a local Trivy scan to confirm zero CRITICAL findings before pushing.

**Result:** Build #47 passed Trivy with 0 CRITICAL, 0 HIGH findings and deployed successfully. I also added a note in our sprint retrospective to subscribe to Spring Boot security advisories and check Trivy output weekly even on passing builds.

---

## Category C — Kubernetes & EKS

---

### #10 — localhost in values.yaml

**Severity:** Critical | **Category:** Kubernetes Configuration

#### What Happened

After the first successful ArgoCD sync, all three services (backend, frontend, analytics) went into CrashLoopBackOff. Backend pod logs showed:

```
com.mysql.cj.jdbc.exceptions.CommunicationsException: Communications link failure
The last packet sent successfully to the server was 0 milliseconds ago.
The driver has not received any packets from the server.
Caused by: java.net.ConnectException: Connection refused (Connection refused)
  url: jdbc:mysql://localhost:3306/finbank_dev_db
```

#### Root Cause

The Helm `values.yaml` still had placeholder values from local laptop development:

```yaml
database:
  host: localhost    # ← on laptop this means your own machine
redis:
  host: localhost    # ← same problem
```

Inside Kubernetes, **every pod is its own isolated network**. `localhost` inside a pod means that specific pod's own loopback — not your Mac, not RDS, not Redis. A backend pod trying to connect to `localhost:3306` is trying to find a MySQL server running inside itself. There is none.

#### How to Diagnose

```bash
# Check what endpoint the pod is using
kubectl exec -it deployment/finbank-backend -n finbank-dev -- env | grep -E "DB_HOST|REDIS|DATASOURCE"

# Get the actual RDS endpoint
aws rds describe-db-instances \
  --db-instance-identifier finbank-dev-mysql \
  --query "DBInstances[0].Endpoint.Address" \
  --output text --region ap-south-1

# Get the actual Redis endpoint
aws elasticache describe-cache-clusters \
  --cache-cluster-id finbank-dev-redis \
  --show-cache-node-info \
  --query "CacheClusters[0].CacheNodes[0].Endpoint.Address" \
  --output text --region ap-south-1
```

#### Exact Fix (Step by Step)

**Step 1: Get real endpoints from AWS**

```bash
# RDS
aws rds describe-db-instances \
  --db-instance-identifier finbank-dev-mysql \
  --query "DBInstances[0].Endpoint.Address" \
  --output text --region ap-south-1
# Output: finbank-dev-mysql.cbmkgsmi8kk5.ap-south-1.rds.amazonaws.com

# Redis
aws elasticache describe-cache-clusters \
  --cache-cluster-id finbank-dev-redis \
  --show-cache-node-info \
  --query "CacheClusters[0].CacheNodes[0].Endpoint.Address" \
  --output text --region ap-south-1
# Output: finbank-dev-redis.eafcpo.0001.aps1.cache.amazonaws.com
```

**Step 2: Update `helm/finbank-backend/values.yaml`**

```yaml
env:
  DB_HOST: "finbank-dev-mysql.cbmkgsmi8kk5.ap-south-1.rds.amazonaws.com"
  DB_PORT: "3306"
  DB_NAME: "finbank_dev_db"
  SPRING_DATA_REDIS_HOST: "finbank-dev-redis.eafcpo.0001.aps1.cache.amazonaws.com"
  SPRING_DATA_REDIS_PORT: "6379"
  SPRING_DATA_REDIS_DATABASE: "0"
```

**Step 3: Update `values-staging.yaml` for staging**

```yaml
env:
  DB_HOST: "finbank-dev-mysql.cbmkgsmi8kk5.ap-south-1.rds.amazonaws.com"
  DB_NAME: "finbank_staging_db"
  SPRING_DATA_REDIS_DATABASE: "1"    # staging uses Redis database 1
```

**Step 4: Commit and push — ArgoCD auto-syncs**

```bash
git add helm/
git commit -m "fix: replace localhost with real AWS RDS and Redis endpoints"
git push origin main
# ArgoCD syncs within 3 minutes, or force it:
argocd app sync finbank-backend-dev
```

#### Prevention

- Never commit `localhost` in any Helm values file. Add a CI check that fails if `grep -r "localhost" helm/` finds anything in values files.

#### Interview Answer (STAR Format)

**Situation:** First ArgoCD deployment of all three FinBank microservices to EKS — all pods went into CrashLoopBackOff immediately after sync.

**Task:** Diagnose and fix the crash so all three services could start and connect to their data stores.

**Action:** Pod logs showed "Connection refused" to `localhost:3306`. I checked the Helm `values.yaml` and found `host: localhost` for both database and Redis — left from local development. In Kubernetes, `localhost` is each pod's own loopback; there is no MySQL there. I fetched the actual RDS and ElastiCache endpoints from AWS CLI and updated all Helm values files. ArgoCD resynced and pods came up healthy.

**Result:** All three services started. This is a very common first-Kubernetes mistake. I now always validate no Helm values file contains `localhost` before any deployment.

---

### #11 — Empty RDS Schema

**Severity:** Critical | **Category:** Database Initialisation

#### What Happened

After fixing the localhost issue (#10), pods connected to RDS but crashed on Spring Boot startup:

```
java.sql.SQLSyntaxErrorException: Table 'finbank_dev_db.users' doesn't exist
org.springframework.beans.factory.BeanCreationException:
  Error creating bean with name 'userRepository': Invocation of init method failed
Application startup failure
```

RDS was running. Connection was successful. But the database had no tables.

#### Root Cause

Terraform created the MySQL server and the empty database (`finbank_dev_db`). It did not create tables. Spring Boot's JPA tries to find tables like `users`, `accounts`, `transactions` on startup. When they don't exist it fails immediately.

#### How to Diagnose

```bash
mysql -h finbank-dev-mysql.cbmkgsmi8kk5.ap-south-1.rds.amazonaws.com \
  -u finbank_admin -p<YOUR_DB_PASSWORD> finbank_dev_db \
  -e "SHOW TABLES;"
# Output: Empty set
```

#### Exact Fix (Step by Step)

Set Spring Boot's DDL auto mode to `update` in Helm values:

```yaml
# helm/finbank-backend/values.yaml
env:
  SPRING_JPA_HIBERNATE_DDL_AUTO: "update"
  # "update": if tables don't exist → create them
  #            if tables exist → leave data alone, alter only new columns
  SPRING_JPA_DATABASE_PLATFORM: "org.hibernate.dialect.MySQL8Dialect"
  SPRING_JPA_SHOW_SQL: "false"
```

For production environments — use `validate` + Flyway migrations instead:

```yaml
# helm/finbank-backend/values-prod.yaml
env:
  SPRING_JPA_HIBERNATE_DDL_AUTO: "validate"
  # "validate": verify DB schema matches entity classes; fail loudly if mismatch; NEVER modify DB
```

#### Prevention

- Dev: `ddl-auto: update`. Staging/prod: `ddl-auto: validate` + Flyway.
- Add a post-deploy check: run `SHOW TABLES` and assert expected tables exist.

#### Interview Answer (STAR Format)

**Situation:** After fixing localhost connectivity in FinBank, backend pods were now reaching RDS but crashing on startup with "Table users doesn't exist."

**Task:** Get the database schema in place so Spring Boot could start.

**Action:** I connected to RDS with `mysql` CLI and confirmed the database was empty — Terraform creates the server and database, not the schema. I set `spring.jpa.hibernate.ddl-auto=update` as an environment variable in the Helm values. Spring Boot read all `@Entity` classes and created the tables on first startup.

**Result:** All tables were created automatically. Backend started and API endpoints responded. I noted for prod values to use `validate` mode with Flyway migrations instead.

---

### #12 — Wrong Spring Profile in Staging

**Severity:** High | **Category:** Spring Boot Configuration

#### What Happened

Backend worked in dev but crashed in staging with the same `localhost:3306` connection error even though staging Helm values had the correct staging endpoint:

```
HikariPool-1 - Exception during pool initialization.
url: jdbc:mysql://localhost:3306/finbank_dev_db
```

The staging values had the correct `DB_HOST` set. But the pod still tried `localhost`.

#### Root Cause

The staging Helm values had `SPRING_PROFILES_ACTIVE: "dev"`. The `dev` Spring profile contains hardcoded localhost values in `application-dev.properties`. In Spring Boot 3.x, profile properties override environment variables. So the `dev` profile's `localhost` overrode the correct staging endpoint from the Helm env block.

```yaml
# values-staging.yaml — BUG
env:
  SPRING_PROFILES_ACTIVE: "dev"    # ← loads application-dev.properties with localhost
```

#### Exact Fix

Use `docker` profile for ALL containerised environments. The `docker` profile reads everything from environment variables — no hardcoded values:

```yaml
# values.yaml, values-staging.yaml, values-prod.yaml — ALL use:
env:
  SPRING_PROFILES_ACTIVE: "docker"
```

`application-docker.properties`:
```properties
spring.datasource.url=jdbc:mysql://${DB_HOST}:${DB_PORT}/${DB_NAME}
spring.datasource.username=${DB_USERNAME}
spring.datasource.password=${DB_PASSWORD}
spring.data.redis.host=${SPRING_DATA_REDIS_HOST}
spring.data.redis.port=${SPRING_DATA_REDIS_PORT}
```

The environment-specific values (DB_HOST, DB_NAME, REDIS_DATABASE) come from the respective Helm values files.

#### Interview Answer (STAR Format)

**Situation:** FinBank backend connecting to `localhost` in staging despite correct endpoint in staging Helm values.

**Task:** Fix staging so it connects to the staging RDS instance.

**Action:** Staging Helm values had `SPRING_PROFILES_ACTIVE: "dev"`. The `dev` Spring profile has hardcoded `localhost` in `application-dev.properties` which overrides env vars in Spring Boot 3.x. I changed all Helm values files to use the `docker` profile — designed to read all config from environment variables. The env vars themselves differ per environment through their respective values files.

**Result:** Staging connected to its correct RDS endpoint. One profile (`docker`) works for all containerised environments; env vars handle the differences.

---

### #13 — Redis Env Vars Renamed in Spring Boot 3.x

**Severity:** High | **Category:** Spring Boot 3.x Breaking Changes

#### What Happened

After upgrading the backend from Spring Boot 2.x to 3.x, Redis stopped connecting:

```
org.springframework.data.redis.RedisConnectionFailureException: Unable to connect to Redis
Caused by: Unable to connect to localhost:6379
```

`SPRING_REDIS_HOST` was set correctly in Helm values. But Spring Boot was still trying `localhost:6379`.

#### Root Cause

Spring Boot 3.x renamed all Redis configuration keys:

| Spring Boot 2.x | Spring Boot 3.x |
|---|---|
| `SPRING_REDIS_HOST` | `SPRING_DATA_REDIS_HOST` |
| `SPRING_REDIS_PORT` | `SPRING_DATA_REDIS_PORT` |
| `SPRING_REDIS_PASSWORD` | `SPRING_DATA_REDIS_PASSWORD` |
| `SPRING_REDIS_DATABASE` | `SPRING_DATA_REDIS_DATABASE` |

The old `SPRING_REDIS_*` names are silently ignored by Spring Boot 3.x. With old names set and new names not set, Spring Boot uses default values: `localhost:6379`.

#### Exact Fix

Update all Helm values files:

```yaml
# helm/finbank-backend/values.yaml — Spring Boot 3.x env var names
env:
  # Remove old names (SPRING_REDIS_*) and add new names:
  SPRING_DATA_REDIS_HOST: "finbank-dev-redis.eafcpo.0001.aps1.cache.amazonaws.com"
  SPRING_DATA_REDIS_PORT: "6379"
  SPRING_DATA_REDIS_DATABASE: "0"    # dev uses Redis DB 0

# values-staging.yaml
env:
  SPRING_DATA_REDIS_HOST: "finbank-dev-redis.eafcpo.0001.aps1.cache.amazonaws.com"
  SPRING_DATA_REDIS_PORT: "6379"
  SPRING_DATA_REDIS_DATABASE: "1"    # staging uses Redis DB 1
```

#### Interview Answer (STAR Format)

**Situation:** After upgrading FinBank backend to Spring Boot 3.x, Redis caching broke — "Unable to connect to localhost:6379" despite the correct ElastiCache endpoint being in Helm values.

**Task:** Restore Redis connectivity without changing the actual Redis infrastructure.

**Action:** I confirmed `SPRING_REDIS_HOST` was set but then checked the Spring Boot 3.x migration guide. All Redis env var names changed from `SPRING_REDIS_*` to `SPRING_DATA_REDIS_*`. The old names are ignored in 3.x. I updated all three Helm values files to use the new names.

**Result:** Redis connected immediately after redeployment. I documented this breaking change in the project notes with a before/after table.

---

### #14 — BCrypt Hash Prefix Mismatch

**Severity:** High | **Category:** Authentication / Security

#### What Happened

After seeding user accounts into the database, login always returned HTTP 401 — even with the correct password:

```
WARN  PasswordEncoder: Password mismatch for user: admin@finbank.com
Encoded:  $2a$10$vI8aWBnW3fID.ZQ4/zo1G...
Supplied: $2b$10$someotherhashstring...
```

#### Root Cause

BCrypt has two version prefixes: `$2a$` and `$2b$`. They use slightly different algorithms. Comparing a `$2a$` hash using a `$2b$` encoder always fails — even if the original password is correct. The seed data used `$2a$`, but the Spring Security encoder was configured for `$2b$`.

#### How to Diagnose

```bash
mysql -h finbank-dev-mysql.cbmkgsmi8kk5.ap-south-1.rds.amazonaws.com \
  -u finbank_admin -p<YOUR_DB_PASSWORD> finbank_dev_db \
  -e "SELECT email, LEFT(password_hash, 4) AS prefix FROM users LIMIT 5;"
# Check if prefix is $2a$ or $2b$
```

#### Exact Fix

Configure Spring Security's `BCryptPasswordEncoder` to match the stored hash version:

```java
@Bean
public PasswordEncoder passwordEncoder() {
    // Explicitly use $2a$ version to match seed data
    return new BCryptPasswordEncoder(BCryptPasswordEncoder.BCryptVersion.$2A);
}
```

Or use `DelegatingPasswordEncoder` which handles multiple formats:

```java
@Bean
public PasswordEncoder passwordEncoder() {
    return PasswordEncoderFactories.createDelegatingPasswordEncoder();
    // Accepts {bcrypt}$2a$..., {bcrypt}$2b$..., etc.
}
```

#### Interview Answer (STAR Format)

**Situation:** Login always returned 401 in FinBank after seeding user accounts, even with correct passwords.

**Task:** Diagnose why password validation was failing and restore authentication.

**Action:** Logs showed "password mismatch." I checked the stored hashes in the DB — they used `$2a$` BCrypt prefix. Spring Security encoder was comparing using `$2b$` logic. These BCrypt versions are incompatible for comparison. I configured the encoder to explicitly use `$2a$` version. Long-term I switched to `DelegatingPasswordEncoder` which handles multiple BCrypt versions gracefully.

**Result:** Login worked correctly. `DelegatingPasswordEncoder` protects against future hash migration issues.

---

### #15 — Staging Database Cross-Contamination

**Severity:** High | **Category:** Multi-Environment Configuration

#### What Happened

Staging backend was connecting to the dev RDS database. Accounts created in staging appeared in dev, and dev data showed up in staging queries.

```
# Staging pod log
[INFO] DataSource URL: jdbc:mysql://...rds.amazonaws.com:3306/finbank_dev_db
```

The staging values file had the correct staging RDS endpoint but the wrong database name.

#### Root Cause

The staging values file was copied from dev and the `DB_NAME` field was not updated:

```yaml
# values-staging.yaml — BUG
env:
  DB_HOST: "finbank-staging-mysql..."    # ← correct host
  DB_NAME: "finbank_dev_db"              # ← WRONG — should be finbank_staging_db
```

#### Exact Fix

```yaml
# helm/finbank-backend/values-staging.yaml — CORRECTED
env:
  DB_HOST: "finbank-dev-mysql.cbmkgsmi8kk5.ap-south-1.rds.amazonaws.com"
  DB_NAME: "finbank_staging_db"          # ← correct database name
  SPRING_DATA_REDIS_DATABASE: "1"        # staging uses Redis DB 1
```

Create the staging database if missing:
```sql
CREATE DATABASE IF NOT EXISTS finbank_staging_db CHARACTER SET utf8mb4 COLLATE utf8mb4_unicode_ci;
```

Environment reference table:

| Environment | DB Name | Redis DB |
|---|---|---|
| dev | finbank_dev_db | 0 |
| staging | finbank_staging_db | 1 |
| prod | finbank_prod_db | 2 |

#### Interview Answer (STAR Format)

**Situation:** FinBank staging was writing data that appeared in dev — both environments shared the same database.

**Task:** Separate staging and dev data stores so environments are fully independent.

**Action:** I checked the staging pod's env vars and found `DB_NAME: finbank_dev_db` — copied from dev and not updated. I corrected it to `finbank_staging_db`, created that database on the staging RDS, and added a reference table in the repo documenting which DB name and Redis database number each environment uses.

**Result:** Environments became fully isolated. The reference table helps future teammates avoid cross-contamination.

---

### #16 — VPC IP Address Exhaustion

**Severity:** Medium | **Category:** Kubernetes Networking, Capacity Planning

#### What Happened

After adding staging environment pods, several staging pods got stuck in `Pending`:

```
Warning  FailedScheduling  pod/finbank-backend-staging-7d9f4b-xxxxx
  failed to assign an IP address to container:
  VPC subnet 'subnet-0a1b2c3d4e5f' has insufficient available IP addresses.
  Available: 0 / Max: 14
```

#### Root Cause

AWS EKS VPC CNI gives each pod a real VPC IP address. t3.medium supports 3 ENIs × 6 IPs per ENI = 18 IPs, minus reserved node IPs ≈ 15 pod slots per node. With 2 nodes (30 slots) already consumed by kube-system (~8 pods), ArgoCD (~6), monitoring (~5), and dev pods (~9) = 28 used, there were only 2 free IPs — not enough for staging pods with replicaCount 2.

#### How to Diagnose

```bash
kubectl describe pod finbank-backend-staging-xxxxx -n finbank-stage | grep -A10 Events

# Check subnet available IPs
aws ec2 describe-subnets \
  --filters "Name=tag:Name,Values=finbank-private-*" \
  --query "Subnets[*].{Subnet:SubnetId,Available:AvailableIpAddressCount}" \
  --region ap-south-1
```

#### Exact Fix (Step by Step)

**Immediate — reduce staging replicas:**

```yaml
# helm/finbank-backend/values-staging.yaml
replicaCount: 1    # staging doesn't need HA
```

**Long-term — enable prefix delegation (256 IPs per ENI instead of 6):**

```bash
kubectl set env daemonset aws-node -n kube-system \
  ENABLE_PREFIX_DELEGATION=true WARM_PREFIX_TARGET=1
kubectl rollout restart daemonset/aws-node -n kube-system
```

#### Interview Answer (STAR Format)

**Situation:** After deploying staging to FinBank EKS cluster, staging pods were stuck in Pending — "insufficient available IP addresses."

**Task:** Get staging pods running without disrupting dev.

**Action:** EKS VPC CNI gives each pod a real VPC IP. t3.medium ≈ 15 pod slots per node. The cluster was nearly full with dev and system pods. Immediate fix: reduce staging replicaCount to 1. Long-term: enabled prefix delegation on the VPC CNI, multiplying IPs per ENI from 6 to 16.

**Result:** Staging pods scheduled immediately. Prefix delegation gave enough headroom for future growth.

---

### #17 — eksctl IRSA Silent Failure

**Severity:** Critical | **Category:** AWS IAM, IRSA, EKS

#### What Happened

After running `eksctl create iamserviceaccount` to set up IRSA for the External Secrets Operator, the command succeeded with no errors. But ESO pods got access denied:

```
err="AccessDeniedException: User: arn:aws:sts::<YOUR_AWS_ACCOUNT_ID>:assumed-role/eksctl-finbank-XXXX
is not authorized to perform: secretsmanager:GetSecretValue on resource: finbank/dev/database"
```

`eksctl` reported success. The IAM role was created. But the role had **zero policies attached**.

#### Root Cause

`eksctl create iamserviceaccount` appeared to succeed but the policy attachment silently failed. The role existed with the correct trust policy (OIDC) but had no permission policies — so pods could assume the role but had no permissions once inside it.

#### How to Diagnose

```bash
# Check what policies are actually attached to the IRSA role
ROLE_NAME=$(kubectl get serviceaccount external-secrets -n external-secrets \
  -o jsonpath='{.metadata.annotations.eks\.amazonaws\.com/role-arn}' | cut -d'/' -f2)

aws iam list-attached-role-policies --role-name $ROLE_NAME
# If "AttachedPolicies": [] → silent failure confirmed
```

#### Exact Fix (Step by Step)

Switch to Terraform-managed IRSA — more reliable, auditable, version-controlled:

```bash
# Step 1: Delete eksctl-created resources
eksctl delete iamserviceaccount \
  --name external-secrets --namespace external-secrets \
  --cluster finbank-dev --region ap-south-1

kubectl delete serviceaccount external-secrets -n external-secrets
```

```hcl
# Step 2: Add to Terraform (modules/eks/irsa.tf)
data "aws_iam_openid_connect_provider" "eks" {
  url = aws_eks_cluster.finbank.identity[0].oidc[0].issuer
}

resource "aws_iam_role" "external_secrets" {
  name = "finbank-dev-external-secrets-role"
  assume_role_policy = jsonencode({
    Version = "2012-10-17"
    Statement = [{
      Effect = "Allow"
      Principal = {
        Federated = data.aws_iam_openid_connect_provider.eks.arn
      }
      Action = "sts:AssumeRoleWithWebIdentity"
      Condition = {
        StringEquals = {
          "${replace(data.aws_iam_openid_connect_provider.eks.url, "https://", "")}:sub" =
            "system:serviceaccount:external-secrets:external-secrets"
          "${replace(data.aws_iam_openid_connect_provider.eks.url, "https://", "")}:aud" =
            "sts.amazonaws.com"
        }
      }
    }]
  })
}

resource "aws_iam_role_policy" "external_secrets" {
  name = "external-secrets-policy"
  role = aws_iam_role.external_secrets.id
  policy = jsonencode({
    Version = "2012-10-17"
    Statement = [{
      Effect = "Allow"
      Action = ["secretsmanager:GetSecretValue", "secretsmanager:DescribeSecret"]
      Resource = "arn:aws:secretsmanager:ap-south-1:<YOUR_AWS_ACCOUNT_ID>:secret:finbank/*"
    }]
  })
}

resource "kubernetes_service_account" "external_secrets" {
  metadata {
    name      = "external-secrets"
    namespace = "external-secrets"
    annotations = {
      "eks.amazonaws.com/role-arn" = aws_iam_role.external_secrets.arn
    }
  }
}
```

```bash
# Step 3: Apply and verify
terraform apply -target=module.eks.aws_iam_role.external_secrets
kubectl rollout restart deployment/external-secrets -n external-secrets

# Verify policy is attached
aws iam list-attached-role-policies --role-name finbank-dev-external-secrets-role
```

#### Prevention

- Always use Terraform for IRSA in Terraform-managed clusters. Never `eksctl create iamserviceaccount`.
- After any IRSA setup, immediately verify: `aws iam list-attached-role-policies --role-name <role>`. If empty → silent failure.

#### Interview Answer (STAR Format)

**Situation:** `eksctl create iamserviceaccount` appeared to succeed for the FinBank External Secrets Operator, but pods immediately got 403 AccessDeniedException from Secrets Manager.

**Task:** Give ESO the correct IAM permissions to read secrets without hardcoding credentials.

**Action:** I ran `aws iam list-attached-role-policies` on the eksctl-created role and found zero policies attached — a silent failure. I deleted the eksctl resources and moved IRSA to Terraform, which creates the role, attaches the policy, and creates the Kubernetes ServiceAccount annotation atomically. Terraform also gives clear plan and error output — no silent failures.

**Result:** ESO pods fetched secrets successfully. IRSA is now always Terraform-managed in this project.

---

### #18 — PDB Label Mismatch

**Severity:** Medium | **Category:** Kubernetes Reliability

#### What Happened

The PodDisruptionBudget for the backend was deployed but during a node drain, all backend pods were evicted simultaneously — causing a brief outage. The PDB existed but protected nothing.

```bash
kubectl describe pdb finbank-backend-pdb -n finbank-dev
# Current Healthy:  0     ← PDB sees ZERO matching pods
# Total Replicas:   0     ← This is the bug
# Disruptions Allowed: 2  ← allows ALL pods to be evicted
```

#### Root Cause

The PDB selector label and the Deployment pod labels did not match:

```yaml
# PDB selector (what PDB was looking for)
selector:
  matchLabels:
    app.kubernetes.io/name: finbank-backend   # ← long-form label

# Deployment pod template labels (what pods actually had)
labels:
  app: finbank-backend                         # ← short-form label
```

Zero pods matched the PDB. It allowed all disruptions.

#### How to Diagnose

```bash
# Check what labels pods actually have
kubectl get pods -n finbank-dev --show-labels | grep finbank-backend

# Check what the PDB selector uses
kubectl get pdb finbank-backend-pdb -n finbank-dev -o yaml | grep -A5 selector

# They must use the same label KEY
```

#### Exact Fix

Update the PDB to use the same label key the Deployment sets on pods:

```yaml
# helm/finbank-backend/templates/pdb.yaml
apiVersion: policy/v1
kind: PodDisruptionBudget
metadata:
  name: {{ .Release.Name }}-pdb
spec:
  minAvailable: 1
  selector:
    matchLabels:
      app: {{ .Release.Name }}    # ← must match Deployment pod labels exactly
```

Verify after deploy:
```bash
kubectl describe pdb finbank-backend-pdb -n finbank-dev
# Should show: Current Healthy: 2, Disruptions Allowed: 1
```

#### Interview Answer (STAR Format)

**Situation:** FinBank backend had a PDB configured for HA, but during a node drain all backend pods were evicted at once. The PDB showed "Disruptions Allowed: 2" — not protecting any pods.

**Task:** Fix the PDB so it correctly limits pod evictions.

**Action:** `kubectl describe pdb` showed "Total Replicas: 0" — the PDB matched zero pods. I compared the PDB's `matchLabels` (`app.kubernetes.io/name: finbank-backend`) against actual pod labels (`app: finbank-backend`). Different keys. I updated the PDB to use the `app` label key that the Deployment actually sets.

**Result:** After fix: "Current Healthy: 2, Disruptions Allowed: 1." The next node drain correctly left one pod running at all times.

---

### #19 — HPA References Wrong Deployment Name

**Severity:** High | **Category:** Kubernetes Autoscaling

#### What Happened

The HPA was deployed via Helm but showed `<unknown>` for metrics and never scaled:

```bash
kubectl get hpa -n finbank-dev
# NAME                 REFERENCE              TARGETS       MINPODS  MAXPODS  REPLICAS
# finbank-backend-hpa  Deployment/finbank     <unknown>/70%    2        5        0

kubectl describe hpa finbank-backend-hpa -n finbank-dev
# Warning  FailedGetScale  deployments.apps "finbank" not found
```

#### Root Cause

The HPA template used `{{ .Release.Name }}` for `scaleTargetRef.name`:

```yaml
# hpa.yaml — WRONG
spec:
  scaleTargetRef:
    name: {{ .Release.Name }}    # becomes "finbank" if chart installed with release name "finbank"
```

But the Deployment was named `finbank-backend` (hardcoded in `deployment.yaml`). The HPA looked for `Deployment/finbank` which doesn't exist.

#### Exact Fix

Hardcode the deployment name in the HPA to match the actual Deployment:

```yaml
# helm/finbank-backend/templates/hpa.yaml — CORRECT
spec:
  scaleTargetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: finbank-backend    # ← hardcoded to match deployment.yaml
  minReplicas: 2
  maxReplicas: 5
  metrics:
  - type: Resource
    resource:
      name: cpu
      target:
        type: Utilization
        averageUtilization: 70
```

Verify after fix:
```bash
kubectl get hpa -n finbank-dev
# TARGETS column should show: 45%/70% (a real percentage, not <unknown>)
```

#### Interview Answer (STAR Format)

**Situation:** FinBank backend HPA showed `<unknown>` for CPU targets and never scaled. Warning: "Deployment/finbank not found."

**Task:** Fix the HPA so it correctly monitors the backend Deployment and scales pods under load.

**Action:** `kubectl describe hpa` showed it trying to scale `Deployment/finbank` — the Helm release name. The actual Deployment was named `finbank-backend` (hardcoded in `deployment.yaml`). The HPA template used `{{ .Release.Name }}` which resolved to just "finbank." I hardcoded `finbank-backend` in the HPA template to match the Deployment.

**Result:** HPA started monitoring the correct Deployment. During a load test it correctly scaled from 2 to 4 pods and back down after load subsided.

---

## Category D — Terraform Infrastructure

---

### #20 — Terraform Destroy Fails: ELB Dependency

**Severity:** High | **Category:** Terraform State, AWS Infrastructure

#### What Happened

`terraform destroy` failed after deleting many resources but got stuck on VPC deletion:

```
Error: error deleting VPC (vpc-0a1b2c3d4e5f67890): DependencyViolation:
  The vpc 'vpc-0a1b2c3d4e5f67890' has dependencies and cannot be deleted.
```

#### Root Cause

The AWS ALB was created by the **Kubernetes ALB Ingress Controller** — not by Terraform. Terraform's state has no record of this ALB. When Terraform tried to delete the VPC, AWS rejected it because the ALB still existed inside that VPC.

#### How to Diagnose

```bash
VPC_ID="vpc-0a1b2c3d4e5f67890"

# Find ALBs in the VPC
aws elbv2 describe-load-balancers \
  --query "LoadBalancers[?VpcId=='$VPC_ID'].[LoadBalancerName,LoadBalancerArn]" \
  --region ap-south-1
```

#### Exact Fix (Step by Step)

**Step 1: Delete Kubernetes Ingress resources BEFORE terraform destroy**

```bash
kubectl delete ingress --all -n finbank-dev
kubectl delete ingress --all -n finbank-stage
kubectl delete ingress --all -n finbank-prod

# Wait for ALBs to be removed by the Ingress Controller
aws elbv2 describe-load-balancers --region ap-south-1
# Wait until no finbank ALBs appear
```

**Step 2: Now run terraform destroy**

```bash
terraform destroy -auto-approve
```

**Step 3: If ALBs remain, delete manually**

```bash
ALB_ARN=$(aws elbv2 describe-load-balancers \
  --query "LoadBalancers[?VpcId=='$VPC_ID'].LoadBalancerArn" \
  --output text --region ap-south-1)
aws elbv2 delete-load-balancer --load-balancer-arn $ALB_ARN --region ap-south-1
terraform destroy -auto-approve
```

#### Interview Answer (STAR Format)

**Situation:** `terraform destroy` on FinBank infrastructure failed at VPC deletion with "DependencyViolation."

**Task:** Identify the blocking resource and clean it up.

**Action:** Listed all resources in the VPC with AWS CLI and found an ALB that Terraform didn't know about — created by the Kubernetes ALB Ingress Controller. I deleted the Kubernetes Ingress objects first, which triggered the Ingress Controller to remove the ALB from AWS. Then `terraform destroy` completed.

**Result:** Full infrastructure teardown succeeded. I added a teardown checklist: always delete Kubernetes Ingress resources before `terraform destroy`.

---

### #21 — Orphaned ELB Security Group

**Severity:** Medium | **Category:** AWS VPC Cleanup

#### What Happened

After deleting the ALBs (#20), `terraform destroy` failed again on security group deletion:

```
Error: error deleting Security Group (sg-0abc123def456): DependencyViolation:
  resource sg-0abc123def456 has a dependent object
```

The ALB was gone but its security group remained as an orphan.

#### Root Cause

When the ALB Ingress Controller creates an ALB, it also creates a security group. When the ALB is deleted, the security group is **not automatically deleted**. Terraform doesn't know about it. When Terraform tries to delete the VPC's security groups, cross-references to the orphan block deletion.

#### Exact Fix

```bash
VPC_ID="vpc-0a1b2c3d4e5f67890"

# Find orphaned SGs created by ALB Ingress Controller (prefixed k8s-)
ORPHAN_SGS=$(aws ec2 describe-security-groups \
  --filters "Name=vpc-id,Values=$VPC_ID" \
  --query "SecurityGroups[?starts_with(GroupName,'k8s-')].GroupId" \
  --output text --region ap-south-1)

# Remove rules referencing each orphan from other SGs
for SG_ID in $ORPHAN_SGS; do
  REFS=$(aws ec2 describe-security-groups \
    --filters "Name=ip-permission.group-id,Values=$SG_ID" \
    --query "SecurityGroups[*].GroupId" \
    --output text --region ap-south-1)
  for REF_SG in $REFS; do
    aws ec2 revoke-security-group-ingress \
      --group-id $REF_SG --source-group $SG_ID \
      --region ap-south-1 || true
  done
  aws ec2 delete-security-group --group-id $SG_ID --region ap-south-1
done

terraform destroy -auto-approve
```

#### Interview Answer (STAR Format)

**Situation:** After deleting ALBs, `terraform destroy` failed again on a security group — orphaned by the ALB Ingress Controller.

**Task:** Remove the orphaned security group so Terraform could delete the VPC.

**Action:** Found SGs with `k8s-` name prefix (ALB Ingress Controller naming convention). Removed all cross-references to the orphan in other SGs, then deleted the orphan. Terraform destroy completed.

**Result:** Full VPC deletion succeeded. Added orphaned SG cleanup to the teardown runbook.

---

### #22 — Terraform State File Locked

**Severity:** High | **Category:** Terraform State Management

#### What Happened

Every `terraform plan` or `terraform apply` failed:

```
Error: Error locking state: Error acquiring the state lock: ConditionalCheckFailedException:
  Lock Info:
    ID:        a1b2c3d4-e5f6-7890-abcd-ef1234567890
    Who:       jenkins@jenkins-container
    Created:   2026-03-15 09:45:22 UTC
```

#### Root Cause

Terraform uses DynamoDB to lock the state file during operations. A previous Jenkins pipeline run was killed mid-apply (Jenkins container restarted). The lock was never released. The DynamoDB entry remains indefinitely.

#### Exact Fix

```bash
# Step 1: Confirm no one is actively running Terraform
# (Check with team + check Jenkins active builds)

# Step 2: Force-unlock with the Lock ID from the error message
terraform force-unlock a1b2c3d4-e5f6-7890-abcd-ef1234567890

# Step 3: Verify state is intact
terraform state list
terraform plan
```

Or delete from DynamoDB directly:
```bash
aws dynamodb delete-item \
  --table-name finbank-terraform-locks \
  --key '{"LockID": {"S": "finbank-terraform-state-<YOUR_AWS_ACCOUNT_ID>/finbank/terraform.tfstate"}}' \
  --region ap-south-1
```

#### Interview Answer (STAR Format)

**Situation:** After a Jenkins container restart during a Terraform apply on FinBank, all subsequent Terraform commands failed with "state locked."

**Task:** Release the stale lock so Terraform operations could resume.

**Action:** Confirmed no active Terraform process was running. The lock was stale from the killed process. I ran `terraform force-unlock` with the Lock ID from the error message, then ran `terraform state list` to verify the state file was intact, then `terraform plan` to confirm accurate infrastructure state.

**Result:** Lock released, operations resumed. Added a rule: never restart the Jenkins container while a Terraform stage is running.

---

### #23 — Variable Specified Twice

**Severity:** Low | **Category:** Terraform CLI

#### What Happened

`terraform apply` in Jenkins failed:

```
Error: Duplicate value for variable "db_password"
  A value for var.db_password was already set using a -var flag.
  Variables may not be set by multiple mechanisms in a single plan/apply.
```

#### Root Cause

`db_password` was set both in `terraform.tfvars` AND as a `-var` flag in the Jenkinsfile. Terraform rejects this as ambiguous.

#### Exact Fix

Remove from `terraform.tfvars`, pass secrets only via environment variable:

```bash
# In Jenkins pipeline environment block:
export TF_VAR_db_password="${DB_PASSWORD}"    # Terraform reads TF_VAR_* automatically
terraform apply -var-file="terraform.tfvars"  # no -var flag needed for db_password
```

#### Interview Answer (STAR Format)

**Situation:** Jenkins Terraform pipeline failed with "variable specified twice" for `db_password`.

**Task:** Fix the configuration so Terraform accepted the variable without conflict.

**Action:** Found `db_password` set in both `terraform.tfvars` and as a `-var` flag in Jenkinsfile. Removed it from tfvars and switched to `TF_VAR_db_password` env var in Jenkins credentials — Terraform reads it automatically, no `-var` flag needed, no conflict.

**Result:** Terraform apply ran without variable conflicts. Sensitive values now live only in Jenkins credentials.

---

### #24 — Staging Database Missing After Rebuild

**Severity:** High | **Category:** Multi-Environment Terraform

#### What Happened

Every infrastructure rebuild (`terraform destroy && apply`) caused staging backend to fail:

```
Unknown database 'finbank_staging_db'
java.sql.SQLException: Unknown database 'finbank_staging_db'
```

Dev database always existed. Staging had to be manually created after every rebuild.

#### Root Cause

Terraform only created `finbank_dev_db`. The staging database (`finbank_staging_db`) was originally created manually and was never added to Terraform state. Every rebuild starts fresh — only what's in Terraform gets created.

#### Exact Fix

Add staging DB creation to Terraform:

```hcl
# modules/rds/main.tf
resource "null_resource" "create_staging_db" {
  depends_on = [aws_db_instance.finbank]

  provisioner "local-exec" {
    command = <<-EOT
      mysql -h ${aws_db_instance.finbank.address} \
        -u ${var.db_username} \
        -p${var.db_password} \
        -e "CREATE DATABASE IF NOT EXISTS finbank_staging_db CHARACTER SET utf8mb4 COLLATE utf8mb4_unicode_ci;"
    EOT
  }

  triggers = {
    rds_endpoint = aws_db_instance.finbank.endpoint
  }
}
```

#### Interview Answer (STAR Format)

**Situation:** Every FinBank infrastructure rebuild left the staging database missing, causing staging backend to crash on every fresh deploy.

**Task:** Ensure the staging database is always created automatically as part of Terraform apply.

**Action:** The staging database was originally created manually, never in Terraform. I added a `null_resource` with a `local-exec` provisioner that runs `CREATE DATABASE IF NOT EXISTS finbank_staging_db` after the RDS instance is ready. The `IF NOT EXISTS` makes it idempotent.

**Result:** Both dev and staging databases are now created automatically after every `terraform apply`. No more manual steps in the rebuild process.

---

### #25 — Double Port in JDBC URL

**Severity:** High | **Category:** Terraform Outputs, Spring Boot Configuration

#### What Happened

After using a Terraform output value to build the database connection string, the backend failed:

```
Caused by: java.net.UnknownHostException:
  finbank-dev-mysql.cbmkgsmi8kk5.ap-south-1.rds.amazonaws.com:3306
  (No such host: finbank-dev-mysql.cbmkgsmi8kk5.ap-south-1.rds.amazonaws.com:3306)
```

The actual JDBC URL being used:
```
jdbc:mysql://finbank-dev-mysql.cbmkgsmi8kk5.ap-south-1.rds.amazonaws.com:3306:3306/finbank_dev_db
```

Two `:3306` in the URL.

#### Root Cause

Terraform has two different RDS output variables:

```hcl
output "db_endpoint" {
  value = aws_db_instance.finbank.endpoint
  # = "finbank-dev-mysql.cbmkgsmi8kk5.ap-south-1.rds.amazonaws.com:3306"
  # ↑ includes the port
}

output "db_host" {
  value = aws_db_instance.finbank.address
  # = "finbank-dev-mysql.cbmkgsmi8kk5.ap-south-1.rds.amazonaws.com"
  # ↑ hostname only, no port
}
```

The Helm values used `db_endpoint` (hostname:3306), and the JDBC URL template also appended `:3306` — resulting in `:3306:3306`.

#### How to Diagnose

```bash
terraform output db_endpoint
# finbank-dev-mysql.cbmkgsmi8kk5.ap-south-1.rds.amazonaws.com:3306   ← includes port

terraform output db_host
# finbank-dev-mysql.cbmkgsmi8kk5.ap-south-1.rds.amazonaws.com         ← hostname only

kubectl exec -it deployment/finbank-backend -n finbank-dev -- env | grep DB_HOST
# If it shows hostname:3306 — that's the bug
```

#### Exact Fix

Use `db_host` (the `address` attribute) when the port is set separately:

```bash
DB_HOST=$(terraform output -raw db_host)
# Output: finbank-dev-mysql.cbmkgsmi8kk5.ap-south-1.rds.amazonaws.com

sed -i "s|DB_HOST:.*|DB_HOST: \"${DB_HOST}\"|" helm/finbank-backend/values.yaml
```

Result:
```
jdbc:mysql://finbank-dev-mysql.cbmkgsmi8kk5.ap-south-1.rds.amazonaws.com:3306/finbank_dev_db  ✓
```

#### Prevention

- Name Terraform outputs clearly: `db_host_with_port` vs `db_host_only`.
- Never use `endpoint` output when the port is being appended separately in the URL template.

#### Interview Answer (STAR Format)

**Situation:** After using a Terraform output for the FinBank database connection, the backend failed with "No such host: hostname:3306" — the port was being treated as part of the hostname.

**Task:** Fix the JDBC URL so the database connection worked.

**Action:** I printed the JDBC URL from the pod's environment and found `hostname:3306:3306/db` — port appeared twice. Terraform's `db_endpoint` output includes `:3306`; `db_host` is hostname only. I switched to `db_host` everywhere the hostname is used separately from port.

**Result:** JDBC URL became well-formed and backend connected to RDS successfully. Renamed Terraform outputs to `db_host_with_port` and `db_host_only` for future clarity.

---

## Category E — AWS Networking

---

### #26 — ALB Health Checks Fail: Missing IAM Policy

**Severity:** Critical | **Category:** AWS IAM, ALB Ingress Controller

#### What Happened

After deploying the AWS Load Balancer Controller and creating Ingress resources, the ALB was created but all health checks were failing:

```
Warning  FailedDeployModel  reconcile error:
  operation error Elastic Load Balancing v2: CreateTargetGroup,
  AccessDeniedException: User is not authorized to perform: elasticloadbalancing:CreateTargetGroup
```

#### Root Cause

The AWS Load Balancer Controller requires a specific IAM policy with 50+ permissions to manage ALBs, target groups, security groups, and listeners. The IRSA role was created but the official policy was not attached — either missing or only partially applied.

#### How to Diagnose

```bash
# Check controller logs
kubectl logs -n kube-system -l app.kubernetes.io/name=aws-load-balancer-controller | grep -i error

# Check what policies are attached to the IRSA role
ROLE_NAME=$(kubectl get sa aws-load-balancer-controller -n kube-system \
  -o jsonpath='{.metadata.annotations.eks\.amazonaws\.com/role-arn}' | cut -d'/' -f2)
aws iam list-attached-role-policies --role-name $ROLE_NAME
```

#### Exact Fix (Step by Step)

```bash
# Step 1: Download the official policy
curl -O https://raw.githubusercontent.com/kubernetes-sigs/aws-load-balancer-controller/v2.7.2/docs/install/iam_policy.json

# Step 2: Create the policy
aws iam create-policy \
  --policy-name AWSLoadBalancerControllerIAMPolicy \
  --policy-document file://iam_policy.json \
  --region ap-south-1

# Step 3: Attach to IRSA role
aws iam attach-role-policy \
  --role-name finbank-dev-alb-controller-role \
  --policy-arn arn:aws:iam::<YOUR_AWS_ACCOUNT_ID>:policy/AWSLoadBalancerControllerIAMPolicy

# Step 4: Restart controller
kubectl rollout restart deployment/aws-load-balancer-controller -n kube-system
```

#### Prevention

- Manage the ALB Controller IAM policy in Terraform using the official policy JSON.
- After every EKS rebuild, verify policy attachment before deploying any Ingress resources.

#### Interview Answer (STAR Format)

**Situation:** FinBank ALB was provisioned but all target group health checks failed. Controller logs showed AccessDeniedException for `CreateTargetGroup`.

**Task:** Give the Load Balancer Controller the permissions it needs to fully manage ALBs.

**Action:** Checked the IRSA role — the official ALB Controller policy was not attached. Downloaded the official policy JSON from kubernetes-sigs GitHub, created it in IAM, attached it to the IRSA role. After restarting the controller, it completed the ALB, listener, and target group setup.

**Result:** ALB health checks went green and traffic started reaching pods. IAM policy creation moved into Terraform module for automatic provisioning on every rebuild.

---

### #27 — Backend Ingress Routes to Wrong Path

**Severity:** High | **Category:** Kubernetes Ingress, ALB Configuration

#### What Happened

After ALB was working, API calls to `/api/v1/users` returned 404 through the ALB even though the backend pod responded correctly when called directly inside the cluster:

```bash
# Direct pod call — WORKS
kubectl exec -it deployment/finbank-frontend -n finbank-dev -- \
  curl http://finbank-backend:8080/api/v1/users
# Returns: [{"id":1,"name":"Test User"}]

# Through ALB — FAILS
curl https://finbank-alb-123456789.ap-south-1.elb.amazonaws.com/api/v1/users
# Returns: 404 Not Found
```

#### Root Cause

The Ingress used `pathType: Exact` for `/api/v1/*`. Exact path type means the URL must literally equal `/api/v1/*` — not `/api/v1/users`. Only the literal string `/api/v1/*` (with an asterisk) would match. No real request would ever match this.

```yaml
# WRONG
- path: /api/v1/*
  pathType: Exact    # literal string match only
```

#### Exact Fix

```yaml
# helm/finbank-backend/templates/ingress.yaml — CORRECT
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: finbank-ingress
  namespace: finbank-dev
  annotations:
    kubernetes.io/ingress.class: alb
    alb.ingress.kubernetes.io/scheme: internet-facing
    alb.ingress.kubernetes.io/target-type: ip
    alb.ingress.kubernetes.io/healthcheck-path: /actuator/health
spec:
  rules:
  - http:
      paths:
      - path: /api/v1
        pathType: Prefix      # ← matches /api/v1 AND /api/v1/anything
        backend:
          service:
            name: finbank-backend
            port:
              number: 8080
      - path: /api/analytics
        pathType: Prefix
        backend:
          service:
            name: finbank-analytics
            port:
              number: 8000
      - path: /
        pathType: Prefix      # ← catch-all for frontend, MUST be last
        backend:
          service:
            name: finbank-frontend
            port:
              number: 80
```

More specific paths (`/api/v1`, `/api/analytics`) must come before the catch-all (`/`). ALB evaluates top-to-bottom and uses the first match.

#### Interview Answer (STAR Format)

**Situation:** FinBank ALB was running but API calls to `/api/v1/users` returned 404 through it while direct pod-to-pod calls worked.

**Task:** Fix Ingress routing so `/api/v1/*` correctly reaches the backend service.

**Action:** Checked Ingress config and found `pathType: Exact` on `/api/v1/*`. Exact path type requires literal URL match — no real request would ever send `/api/v1/*`. Changed all paths to `pathType: Prefix` and reordered so `/api/v1` and `/api/analytics` come before the frontend catch-all `/`.

**Result:** All API routes worked correctly through ALB. The ordering lesson: specific paths before catch-all, Prefix instead of Exact for API routing.

---

### #28 — Subnet IDs Change After EKS Rebuild

**Severity:** Medium | **Category:** AWS Infrastructure, EKS

#### What Happened

After rebuilding the EKS cluster, the ALB Ingress Controller failed to create the ALB:

```
Error creating ALB: InvalidSubnet: The subnet ID 'subnet-0abc123def456' does not exist
```

The Ingress YAML had hardcoded subnet IDs from the previous cluster. Those subnets were deleted with the old VPC.

#### Root Cause

Every `terraform destroy && apply` cycle creates new VPC and subnet resources with new IDs. Any Kubernetes manifest that hardcodes these IDs becomes stale after a rebuild.

#### Exact Fix

Use subnet tags for auto-discovery instead of hardcoding IDs:

```hcl
# In Terraform — tag subnets for ALB discovery
resource "aws_subnet" "public" {
  tags = {
    "kubernetes.io/cluster/finbank-dev" = "shared"
    "kubernetes.io/role/elb"            = "1"    # tells ALB controller: use this for internet-facing ALBs
  }
}

resource "aws_subnet" "private" {
  tags = {
    "kubernetes.io/cluster/finbank-dev" = "shared"
    "kubernetes.io/role/internal-elb"   = "1"
  }
}
```

In the Ingress YAML — remove the hardcoded `subnets` annotation entirely. The ALB controller reads the tags and auto-discovers the correct subnets:

```yaml
# Before (brittle):
annotations:
  alb.ingress.kubernetes.io/subnets: "subnet-0abc123,subnet-0def456"

# After (resilient — no subnet annotation needed):
annotations:
  alb.ingress.kubernetes.io/scheme: internet-facing
  alb.ingress.kubernetes.io/target-type: ip
  # subnets discovered via kubernetes.io/role/elb tag on VPC subnets
```

#### Prevention

- Never hardcode VPC-specific resource IDs (subnet IDs, SG IDs, VPC IDs) in Kubernetes manifests.
- Always use tags for resource discovery in dynamic environments.

#### Interview Answer (STAR Format)

**Situation:** After rebuilding the FinBank EKS cluster, the ALB Ingress Controller failed because hardcoded subnet IDs in the Ingress YAML pointed to subnets that no longer existed.

**Task:** Make the Ingress configuration resilient to EKS rebuilds without requiring manual ID updates.

**Action:** Removed the hardcoded `subnets` annotation from the Ingress YAML and added `kubernetes.io/role/elb = 1` tags to the public subnet Terraform resources. The ALB controller reads these tags to auto-discover which subnets to use — no IDs needed in the manifest.

**Result:** Subsequent rebuilds provisioned the ALB in the correct subnets automatically. This pattern should be used for any AWS resource that Kubernetes discovers from EC2 tags.

---

### #29 — IMDS Hop Limit Blocks ALB Controller

**Severity:** High | **Category:** AWS EC2 Metadata, EKS Networking

#### What Happened

The AWS Load Balancer Controller kept failing with authentication errors even though IRSA was correctly configured:

```
level=error msg="unable to retrieve credentials from EC2 instance metadata service:
  request send failed: Get http://169.254.169.254/latest/meta-data/iam/security-credentials/:
  context deadline exceeded"
```

#### Root Cause

AWS IMDS (Instance Metadata Service) at `169.254.169.254` lets EC2 instances get their IAM credentials. By default, IMDS has a **hop limit of 1** — only one network hop allowed.

In EKS, a pod's request travels: pod → container runtime → node → IMDS. That's 2 hops. With a limit of 1, the request gets dropped at the container runtime boundary. The pod cannot reach IMDS.

#### How to Diagnose

```bash
# Check the current hop limit on EKS nodes
NODE_ID=$(aws ec2 describe-instances \
  --filters "Name=tag:aws:eks:cluster-name,Values=finbank-dev" \
  --query "Reservations[0].Instances[0].InstanceId" \
  --output text --region ap-south-1)

aws ec2 describe-instances \
  --instance-ids $NODE_ID \
  --query "Reservations[0].Instances[0].MetadataOptions.HttpPutResponseHopLimit" \
  --region ap-south-1
# If output is 1 → this is the bug, should be 2
```

#### Exact Fix (Step by Step)

**Option A — Fix existing nodes via CLI:**

```bash
INSTANCE_IDS=$(aws ec2 describe-instances \
  --filters "Name=tag:aws:eks:cluster-name,Values=finbank-dev" \
             "Name=instance-state-name,Values=running" \
  --query "Reservations[*].Instances[*].InstanceId" \
  --output text --region ap-south-1)

for INSTANCE_ID in $INSTANCE_IDS; do
  aws ec2 modify-instance-metadata-options \
    --instance-id $INSTANCE_ID \
    --http-put-response-hop-limit 2 \
    --http-endpoint enabled \
    --region ap-south-1
done

kubectl rollout restart deployment/aws-load-balancer-controller -n kube-system
```

**Option B — Set in Terraform node group launch template (permanent fix):**

```hcl
resource "aws_launch_template" "eks_nodes" {
  metadata_options {
    http_endpoint               = "enabled"
    http_put_response_hop_limit = 2        # ← allows pods to reach IMDS
    http_tokens                 = "required"   # IMDSv2 only (security best practice)
  }
}
```

#### Prevention

Always set `http_put_response_hop_limit = 2` in the EKS node group launch template in Terraform. This is required for any pod that uses IMDS-based credentials.

#### Interview Answer (STAR Format)

**Situation:** The AWS Load Balancer Controller in FinBank couldn't get IAM credentials — "context deadline exceeded" reaching `169.254.169.254` — even though IRSA was correctly configured.

**Task:** Allow the controller pod to reach the EC2 instance metadata service for credentials.

**Action:** IMDS has a hop limit of 1 by default. Pods are 2 hops from IMDS (pod → node → metadata service). I increased the hop limit to 2 on all node instances using `aws ec2 modify-instance-metadata-options --http-put-response-hop-limit 2`. I also added this to the EKS node group launch template in Terraform so it applies automatically on every rebuild.

**Result:** Controller retrieved credentials through IMDS successfully and began managing ALBs. This is now item 1 on the EKS cluster setup checklist.

---

## Category F — GitOps, ArgoCD & Application

---

### #30 — ArgoCD Repo Credentials Dropped in zsh

**Severity:** High | **Category:** ArgoCD, Shell Scripting

#### What Happened

Running `argocd repo add` appeared to succeed but ArgoCD could not clone the private GitHub repo:

```
FATA[0002] rpc error: code = Unknown desc = authentication required
  permission to Anshujee/FinBank-DevOps-Notes.git denied to anonymous
```

The command had been run with `--username` and `--password` flags and returned no error. But ArgoCD stored corrupted credentials.

#### Root Cause

The GitHub PAT contained special characters (`@`, `!`, `#`) which zsh interprets as shell metacharacters. When passed as a `--password` argument without proper quoting, zsh corrupts the password string before it reaches the ArgoCD CLI. ArgoCD stores the corrupted string. Authentication fails on every sync.

#### How to Diagnose

```bash
# Check if stored credentials work
argocd repo list
argocd repo get https://github.com/Anshujee/FinBank-DevOps-Notes.git
# If STATUS shows ConnectionFailed → credentials are corrupted
```

#### Exact Fix (Step by Step)

**Option A — Single quotes around password:**

```bash
argocd repo add https://github.com/Anshujee/FinBank-DevOps-Notes.git \
  --username Anshujee \
  --password 'ghp_aBcDeFgH@123!XYZ#token'    # single quotes: no shell interpretation
```

**Option B (recommended) — Kubernetes Secret (GitOps-native approach):**

```yaml
# argocd-repo-secret.yaml
apiVersion: v1
kind: Secret
metadata:
  name: finbank-repo-creds
  namespace: argocd
  labels:
    argocd.argoproj.io/secret-type: repository
type: Opaque
stringData:
  type: git
  url: https://github.com/Anshujee/FinBank-DevOps-Notes.git
  username: Anshujee
  password: ghp_aBcDeFgH@123!XYZ#token    # no shell interpretation — stored directly in YAML
```

```bash
kubectl apply -f argocd-repo-secret.yaml
```

ArgoCD automatically reads Secrets labeled `argocd.argoproj.io/secret-type: repository`.

**Verify:**

```bash
argocd repo list
# STATUS column should show: Successful
```

#### Prevention

- Always use Kubernetes Secrets for ArgoCD repository credentials.
- After adding a repo, always verify `argocd repo list` shows `Successful` — never assume it worked.

#### Interview Answer (STAR Format)

**Situation:** After running `argocd repo add` to connect ArgoCD to the private FinBank GitHub repository, ArgoCD showed "unauthenticated" and could not clone the repo.

**Task:** Successfully connect ArgoCD to the private GitHub repository.

**Action:** Traced the issue to zsh interpreting `@`, `!`, `#` in the GitHub PAT as metacharacters, corrupting the password before it reached ArgoCD CLI. Fixed by using a Kubernetes Secret with the `argocd.argoproj.io/secret-type: repository` label — ArgoCD reads these directly without shell interpretation.

**Result:** ArgoCD showed `ConnectionSuccessful` and all ApplicationSet applications started syncing. ArgoCD repository credentials are now always managed as Kubernetes Secrets.

---

### #31 — OIDC ID Changed After EKS Rebuild

**Severity:** Critical | **Category:** AWS IRSA, EKS Identity

#### What Happened

After rebuilding the EKS cluster (terraform destroy + apply), all IRSA-dependent pods — External Secrets Operator, ALB Ingress Controller — started getting 403 errors:

```
InvalidIdentityToken: No OpenIDConnect provider found in your account for this URL:
  oidc.eks.ap-south-1.amazonaws.com/id/NEWOIDCID12345
WebIdentityErr: failed to retrieve credentials
```

Everything worked before the rebuild. Only the cluster was recreated.

#### Root Cause

Every EKS cluster has a unique OIDC provider URL: `oidc.eks.ap-south-1.amazonaws.com/id/UNIQUEID`. This ID is **different for every EKS cluster creation**.

IRSA trust policies on IAM roles reference this OIDC URL to verify the pod's identity. After a rebuild, the new cluster has a new OIDC ID. The old IAM trust policies still reference the OLD ID. AWS rejects every role assumption attempt with "No OpenIDConnect provider found."

#### How to Diagnose

```bash
# Get the CURRENT cluster's OIDC URL
aws eks describe-cluster \
  --name finbank-dev \
  --query "cluster.identity.oidc.issuer" \
  --output text --region ap-south-1
# Example: https://oidc.eks.ap-south-1.amazonaws.com/id/NEWID12345

# Check what OIDC URL is in the trust policy of an IRSA role
aws iam get-role \
  --role-name finbank-dev-external-secrets-role \
  --query "Role.AssumeRolePolicyDocument" | grep oidc
# If it shows the OLD ID → trust policy is stale
```

#### Exact Fix (Step by Step)

**If IRSA is managed by Terraform (from #17 — the correct approach):**

Terraform uses `data.aws_iam_openid_connect_provider.eks.url` which is dynamic. Running `terraform apply` after a cluster rebuild automatically detects the new OIDC URL and updates all trust policies:

```bash
terraform apply    # automatically updates OIDC references in all trust policies
kubectl rollout restart deployment/external-secrets -n external-secrets
kubectl rollout restart deployment/aws-load-balancer-controller -n kube-system
```

**If IRSA was created manually — update trust policy for each role:**

```bash
# Get the new OIDC URL (without https://)
NEW_OIDC_URL=$(aws eks describe-cluster \
  --name finbank-dev \
  --query "cluster.identity.oidc.issuer" \
  --output text --region ap-south-1 | sed 's|https://||')

# Register the new OIDC provider in IAM
aws iam create-open-id-connect-provider \
  --url https://$NEW_OIDC_URL \
  --client-id-list sts.amazonaws.com \
  --thumbprint-list 9e99a48a9960b14926bb7f3b02e22da2b0ab7280

# Update trust policy for each IRSA role
ROLE="finbank-dev-external-secrets-role"
NEW_TRUST=$(cat <<EOF
{
  "Version": "2012-10-17",
  "Statement": [{
    "Effect": "Allow",
    "Principal": {"Federated": "arn:aws:iam::<YOUR_AWS_ACCOUNT_ID>:oidc-provider/$NEW_OIDC_URL"},
    "Action": "sts:AssumeRoleWithWebIdentity",
    "Condition": {
      "StringEquals": {
        "${NEW_OIDC_URL}:sub": "system:serviceaccount:external-secrets:external-secrets",
        "${NEW_OIDC_URL}:aud": "sts.amazonaws.com"
      }
    }
  }]
}
EOF
)
aws iam update-assume-role-policy --role-name $ROLE --policy-document "$NEW_TRUST"

# Restart pods to pick up new credentials
kubectl rollout restart deployment/external-secrets -n external-secrets
```

#### Prevention

- Always manage IRSA with Terraform — it auto-updates OIDC references after every rebuild.
- Add to post-rebuild checklist: verify `aws iam get-role --role-name ... --query Role.AssumeRolePolicyDocument` shows the current cluster's OIDC ID.

#### Interview Answer (STAR Format)

**Situation:** After rebuilding the FinBank EKS cluster from scratch, all pods using IRSA failed with "InvalidIdentityToken: No OpenIDConnect provider found."

**Task:** Restore IRSA functionality for External Secrets Operator and ALB Ingress Controller.

**Action:** Every new EKS cluster gets a new unique OIDC provider ID. All IRSA trust policies still referenced the old ID — AWS rejects role assumption because the old OIDC provider no longer exists. Since IRSA was managed by Terraform, running `terraform apply` automatically updated all trust policies with the new OIDC URL. For any manually-created roles, I update the trust policy JSON with the new OIDC ARN.

**Result:** All IRSA-dependent pods worked after `terraform apply` and rollout restart. This is now step 3 of the post-rebuild checklist: verify OIDC trust policies are current.

---

### #32 — ArgoCD ApplicationSet Crashes on Install

**Severity:** High | **Category:** ArgoCD Configuration

#### What Happened

After installing ArgoCD and applying the ApplicationSet resource that manages all 9 applications, ArgoCD rejected it:

```bash
kubectl apply -f argocd/applicationset.yaml
# Error from server: error when creating "argocd/applicationset.yaml":
#   the server could not find the requested resource
#   (post applicationsets.argoproj.io)
```

#### Root Cause

The `ApplicationSet` CRD (Custom Resource Definition) was not installed. This happens with:
- Older ArgoCD versions (pre-2.0) that don't include the ApplicationSet controller
- Incomplete ArgoCD installation (only server installed, not the full bundle)
- Wrong `apiVersion` in the YAML that triggers schema validation failure

#### How to Diagnose

```bash
# Check if the CRD exists
kubectl get crd applicationsets.argoproj.io
# "Error from server (NotFound)" → CRD is missing

# Check ArgoCD version
kubectl get pods -n argocd -l app.kubernetes.io/name=argocd-server \
  -o jsonpath='{.items[0].spec.containers[0].image}'

# Check all ArgoCD CRDs
kubectl get crd | grep argoproj
```

#### Exact Fix (Step by Step)

```bash
# Reinstall ArgoCD with the full official manifest (includes ApplicationSet CRD)
kubectl delete namespace argocd
kubectl create namespace argocd
kubectl apply -n argocd \
  -f https://raw.githubusercontent.com/argoproj/argo-cd/stable/manifests/install.yaml

# Wait for pods
kubectl wait --for=condition=ready pod \
  -l app.kubernetes.io/name=argocd-server -n argocd --timeout=180s

# Verify CRD exists now
kubectl get crd applicationsets.argoproj.io

# Apply the ApplicationSet
kubectl apply -f argocd/applicationset.yaml
```

**Correct ApplicationSet YAML structure:**

```yaml
apiVersion: argoproj.io/v1alpha1    # correct API version for ArgoCD 2.x
kind: ApplicationSet
metadata:
  name: finbank-apps
  namespace: argocd
spec:
  generators:
  - list:
      elements:
      - service: backend
        namespace: finbank-dev
        values: values.yaml
      - service: backend
        namespace: finbank-stage
        values: values-staging.yaml
      - service: frontend
        namespace: finbank-dev
        values: values.yaml
      - service: frontend
        namespace: finbank-stage
        values: values-staging.yaml
      - service: analytics
        namespace: finbank-dev
        values: values.yaml
      - service: analytics
        namespace: finbank-stage
        values: values-staging.yaml
  template:
    metadata:
      name: 'finbank-{{service}}-{{namespace}}'
    spec:
      project: default
      source:
        repoURL: https://github.com/Anshujee/FinBank-DevOps-Notes.git
        targetRevision: main
        path: 'helm/finbank-{{service}}'
        helm:
          valueFiles:
          - '{{values}}'
      destination:
        server: https://kubernetes.default.svc
        namespace: '{{namespace}}'
      syncPolicy:
        automated:
          prune: true
          selfHeal: true
        syncOptions:
        - CreateNamespace=true
```

#### Interview Answer (STAR Format)

**Situation:** After installing ArgoCD in FinBank cluster, applying the ApplicationSet resource failed with "the server could not find the requested resource" — the ApplicationSet API type didn't exist.

**Task:** Get the ApplicationSet working so ArgoCD could manage all 9 applications (3 services × 3 environments) from a single configuration.

**Action:** Ran `kubectl get crd applicationsets.argoproj.io` — NotFound. The ApplicationSet CRD was not installed. This indicated an incomplete ArgoCD installation. I reinstalled using the official `stable` manifest which includes the ApplicationSet controller and all required CRDs. After installation, the ApplicationSet YAML applied successfully and all 9 ArgoCD applications were created.

**Result:** All 9 apps (finbank-backend-finbank-dev, finbank-backend-finbank-stage, etc.) appeared in ArgoCD and began syncing their Helm charts from Git. Any change to a Helm values file now automatically propagates to the correct environment.

---

### #33 — Registration API Returns Validation Failed

**Severity:** Medium | **Category:** Application API, Database Schema

#### What Happened

While testing the FinBank registration flow, POST `/api/v1/auth/register` returned HTTP 400:

```json
{
  "status": 400,
  "error": "Bad Request",
  "message": "Validation failed for fields: [phone, dateOfBirth]",
  "path": "/api/v1/auth/register"
}
```

Request being sent:
```json
{
  "email": "test@finbank.com",
  "password": "SecurePass123!",
  "firstName": "Test",
  "lastName": "User"
}
```

#### Root Cause

The `users` table has two `NOT NULL` columns without defaults:

```sql
phone       VARCHAR(15) NOT NULL,
date_of_birth DATE NOT NULL
```

These are mapped to Java entity fields with `@NotNull` / `@Column(nullable = false)`. Spring Validator rejects the request before it reaches the database layer because the required fields are missing from the request body.

This is correct behavior for a banking application — phone and date of birth are required for KYC (Know Your Customer) compliance.

#### How to Diagnose

```bash
# Check the table schema
mysql -h finbank-dev-mysql.cbmkgsmi8kk5.ap-south-1.rds.amazonaws.com \
  -u finbank_admin -p<YOUR_DB_PASSWORD> finbank_dev_db \
  -e "DESCRIBE users;"
# Look for NOT NULL columns with no DEFAULT value

# Test with all fields
curl -X POST http://localhost:8080/api/v1/auth/register \
  -H "Content-Type: application/json" \
  -d '{
    "email": "test@finbank.com",
    "password": "SecurePass123!",
    "firstName": "Test",
    "lastName": "User",
    "phone": "+91-9876543210",
    "dateOfBirth": "1990-05-15"
  }'
```

#### Exact Fix

Include all required fields in registration requests:

```json
POST /api/v1/auth/register
{
  "email": "test@finbank.com",
  "password": "SecurePass123!",
  "firstName": "Test",
  "lastName": "User",
  "phone": "+91-9876543210",
  "dateOfBirth": "1990-05-15"
}
```

Document all required fields in an OpenAPI spec:

```yaml
# openapi.yaml
paths:
  /api/v1/auth/register:
    post:
      requestBody:
        required: true
        content:
          application/json:
            schema:
              type: object
              required: [email, password, firstName, lastName, phone, dateOfBirth]
              properties:
                email:
                  type: string
                  format: email
                phone:
                  type: string
                  example: "+91-9876543210"
                dateOfBirth:
                  type: string
                  format: date
                  example: "1990-05-15"
```

#### Prevention

- Document all required API fields in an OpenAPI spec committed to the repo.
- Add API contract tests in Jenkins that test both happy path (all fields) and validation failures (missing fields).

#### Interview Answer (STAR Format)

**Situation:** During functional testing of FinBank's registration flow, the API returned HTTP 400 "Validation failed: phone, dateOfBirth" even with email, password, and name in the request.

**Task:** Understand why registration was failing and fix the request or schema.

**Action:** Checked the database schema — `phone` and `date_of_birth` are `NOT NULL` without defaults, mapped to `@NotNull` Java entity fields. Spring Validator correctly rejected the incomplete request. Since FinBank is a banking application requiring KYC, phone and date of birth are legitimately mandatory. I updated all test requests and created an OpenAPI spec documenting required vs optional fields.

**Result:** Registration worked with all required fields. The OpenAPI spec gives frontend developers and testers a clear reference for the full request body format.

---

## Quick Reference: Diagnostic Commands

```bash
# === KUBERNETES ===
# Check pod status — find anything not Running
kubectl get pods -A | grep -v Running | grep -v Completed

# Check why a pod is crashing
kubectl describe pod <pod-name> -n <namespace> | tail -20
kubectl logs <pod-name> -n <namespace> --previous

# Check env vars inside a running pod
kubectl exec -it deployment/<name> -n <namespace> -- env | grep -i <search>

# Check HPA status — TARGETS should show a %, not <unknown>
kubectl get hpa -A
kubectl describe hpa <name> -n <namespace>

# Check PDB — Current Healthy should be > 0
kubectl describe pdb <name> -n <namespace>

# === TERRAFORM ===
terraform state list                                    # see all tracked resources
terraform force-unlock <LOCK_ID>                        # release stale lock
terraform output                                        # see all output values
terraform plan -target=module.rds                       # plan specific module only

# === AWS ===
# EKS OIDC URL (changes after every rebuild!)
aws eks describe-cluster --name finbank-dev \
  --query "cluster.identity.oidc.issuer" --output text --region ap-south-1

# What's inside a VPC (before terraform destroy)
aws ec2 describe-instances --filters "Name=vpc-id,Values=<VPC_ID>"
aws elbv2 describe-load-balancers --region ap-south-1

# IRSA — check if role has policies attached
aws iam list-attached-role-policies --role-name <role-name>

# RDS — connect and check tables
mysql -h finbank-dev-mysql.cbmkgsmi8kk5.ap-south-1.rds.amazonaws.com \
  -u finbank_admin -p<YOUR_DB_PASSWORD> finbank_dev_db -e "SHOW TABLES;"

# === ARGOCD ===
argocd app list                    # all app statuses
argocd app get <app-name>          # details on one app
argocd app sync <app-name>         # force sync
argocd repo list                   # check repo connection status
```

---

## STAR Interview Answer Templates

Use these patterns for any troubleshooting question in an interview:

**Opening — Situation:**
> "During the FinBank project — a multi-microservice banking platform on AWS EKS with GitOps deployment via ArgoCD — I encountered..."

**Middle — Task + Action:**
> "I needed to... I diagnosed by running [command]. I found [root cause]. I fixed it by [step-by-step actions]. The key commands were [commands]."

**Closing — Result:**
> "After the fix [outcome]. I also automated this to prevent recurrence by [prevention step] and documented it in [where]."

**Metrics to mention:**
- 3 microservices: Spring Boot backend, React frontend, FastAPI analytics
- EKS: Kubernetes 1.31, t3.medium nodes, ap-south-1 region
- GitOps: ArgoCD with 9 applications (3 services × 3 environments)
- CI/CD: Jenkins with 9-stage backend pipeline
- IaC: Terraform with S3 state backend + DynamoDB locking
- Security: Trivy scanning, IRSA, External Secrets Operator, AWS Secrets Manager

---

*FinBank DevOps Troubleshooting Guide — 33 challenges documented*
