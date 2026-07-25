# Anshu's Complete DevOps Career Growth Plan — 2026 to 2031

> **Prepared for:** Anshu (DevOps Jee)
> **Date:** July 2026
> **Current Status:** FinBank + AzureShop projects completed. Ready for first DevOps job.
> **Goal:** From first DevOps job → Senior DevOps Engineer / Architect
> **Target Market:** India (Bangalore, Hyderabad, Pune, Gurugram, Mumbai)

---

## What You Already Have (Your Starting Advantage)

Most people applying for DevOps jobs in India have only done YouTube tutorials or Udemy courses. You have actually **built and deployed two real projects** from scratch on real cloud platforms. This is your biggest advantage.

### FinBank Project (AWS Stack) ✅
| Tool / Technology | What You Did |
|---|---|
| **Docker** | Containerized Spring Boot backend, React frontend, Python analytics |
| **Jenkins** | 9-stage CI/CD pipeline — Checkout → Build → Test → SonarQube → Trivy → ECR Push → Helm Update |
| **AWS EKS** | Created and managed full Kubernetes cluster with worker nodes |
| **Terraform** | Wrote all 6 modules — VPC, EKS, RDS, ECR, ElastiCache, Secrets Manager |
| **Helm + ArgoCD** | GitOps deployment across 3 environments (dev / staging / prod) |
| **AWS RDS** | MySQL 8.0 on private subnet with backups, encryption, parameter groups |
| **AWS ElastiCache** | Redis cluster for session management and JWT blacklisting |
| **AWS ECR** | Private Docker registries with lifecycle policies and image scanning |
| **AWS Secrets Manager + ESO** | IRSA-based secret injection — zero hardcoded credentials in pods |
| **Prometheus + Grafana** | Full monitoring stack with alerts |
| **SonarQube + Trivy** | Code quality and container security scanning in pipeline |
| **ALB Ingress** | AWS Application Load Balancer for routing traffic to services |

### AzureShop Project (Azure Stack) ✅
| Tool / Technology | What You Did |
|---|---|
| **Azure Terraform IaC** | Full infrastructure as code on Azure |
| **Azure Pipelines CI/CD** | Complete CI/CD pipeline with YAML |
| **Azure Container Registry (ACR)** | Private Docker registry on Azure |
| **Azure Key Vault** | Secrets management on Azure |
| **AKS (Azure Kubernetes Service)** | Kubernetes on Azure |

**What this means:** You can genuinely say "I have worked with AWS AND Azure" in interviews. Very few freshers can say this.

---

## The 5-Stage Growth Plan

```
Stage 1 → Stage 2 → Stage 3 → Stage 4 → Stage 5
0-6 mo    6mo-2yr   2yr-4yr   4yr-7yr   7yr-12yr

₹5-9 LPA  ₹10-18    ₹18-32    ₹32-55    ₹55-1Cr+
```

---

# STAGE 1: Get Your First Job (0 – 6 Months)

## Goal: Land first DevOps role at ₹5–9 LPA

This stage is all about one thing — **getting hired**. You already have the knowledge. Now you need to package it correctly and apply systematically.

---

### Step 1.1 — Fix Your GitHub Profile (Week 1–2)

Your GitHub is your portfolio. Recruiters and engineers look at this before calling you.

**What to do:**

**1. Make Infra_FinBank repository public and write a strong README**
```
The README must include:
✅ Architecture diagram (copy from the guide)
✅ Tech stack list (Terraform, EKS, ArgoCD, Helm, Jenkins, Prometheus...)
✅ What each module does (VPC, EKS, RDS, ElastiCache, ECR, Secrets Manager)
✅ How to run it (terraform apply command)
✅ Screenshots of Grafana dashboard, ArgoCD UI, Jenkins pipeline
```

**2. Pin these repositories on your GitHub profile:**
- `FinBank-backend` — Spring Boot + Docker
- `Infra_FinBank` — Terraform + Helm + ArgoCD
- `FinBank-DevOps-Notes` — your study guide (this file!)
- `AzureShop` (if public) — shows Azure knowledge

**3. Write a good GitHub profile README**
```markdown
# Hi, I'm Anshu — DevOps Engineer

I build cloud-native infrastructure on AWS and Azure.

🔧 Stack: Terraform | Kubernetes | Docker | Jenkins | ArgoCD | Helm
☁️ Cloud: AWS (EKS, RDS, ECR, ElastiCache, Secrets Manager) | Azure (AKS, ACR, Key Vault)
📊 Monitoring: Prometheus | Grafana
🔒 Security: SonarQube | Trivy | ESO + IRSA

🏗️ Projects:
- FinBank — Production banking app on AWS EKS with full DevOps pipeline
- AzureShop — Microservices e-commerce on AKS with Azure Pipelines
```

---

### Step 1.2 — Build Your Resume (Week 1–2)

**Resume rules for DevOps in India:**
- Keep it **1 page** (fresher) or **2 pages** (if you have internship experience)
- No photos, no date of birth, no fancy colors
- Use simple clean format — ATS (Applicant Tracking System) must be able to read it

**Resume Structure:**
```
NAME | Phone | Email | GitHub Link | LinkedIn Link | Location

SUMMARY (3 lines)
DevOps engineer with hands-on experience building cloud-native banking infrastructure
on AWS using Terraform, EKS, Jenkins, ArgoCD, and Helm. Experienced with complete
CI/CD pipelines, GitOps workflows, and multi-environment Kubernetes deployments.

SKILLS
Cloud:        AWS (EKS, ECR, RDS, ElastiCache, Secrets Manager, ALB, IAM)
              Azure (AKS, ACR, Key Vault, Azure Pipelines)
IaC:          Terraform (6 custom modules), Helm
Containers:   Docker, Kubernetes (EKS, AKS)
CI/CD:        Jenkins, ArgoCD, GitHub Actions
Monitoring:   Prometheus, Grafana
Security:     SonarQube, Trivy, IRSA, External Secrets Operator
Languages:    Bash, Python (basic), YAML
Databases:    MySQL (RDS), Redis (ElastiCache)

PROJECTS

FinBank — Cloud-Native Banking Platform on AWS
→ Built complete AWS infrastructure using Terraform (VPC, EKS, RDS, ECR, ElastiCache,
  Secrets Manager) across dev/staging/prod environments
→ Implemented 9-stage Jenkins CI/CD pipeline with SonarQube code analysis and Trivy
  container security scanning
→ Deployed 3 microservices (Spring Boot, React, FastAPI) using Helm + ArgoCD GitOps
  across 3 Kubernetes namespaces
→ Configured Prometheus + Grafana monitoring with custom dashboards and alerting
→ Implemented zero-credential security using AWS IRSA + External Secrets Operator

AzureShop — Microservices E-commerce on Azure
→ Built Azure infrastructure with Terraform (AKS, ACR, Key Vault, Azure SQL)
→ Implemented Azure Pipelines CI/CD for 3 microservices
→ Deployed to AKS with Kubernetes manifests across dev/staging/prod

EDUCATION
[Your Degree] | [College Name] | [Year]

CERTIFICATIONS (add as you get them)
```

---

### Step 1.3 — Set Up Job Application System (Week 2–3)

**Apply on ALL of these portals every single day:**

| Portal | Link | Best For |
|---|---|---|
| **Naukri.com** | naukri.com | Highest volume in India, most IT recruiters use this |
| **LinkedIn** | linkedin.com/jobs | MNCs, product companies, GCCs |
| **Foundit.in** | foundit.in | Good for mid-size companies |
| **Glassdoor** | glassdoor.co.in | With company reviews + salary info |
| **Instahyre** | instahyre.com | Startup and product companies |
| **AngelList / Wellfound** | wellfound.com | Startups — better pay for freshers |
| **Company career pages** | Direct apply | TCS, Infosys, HCLTech, Accenture direct |

**Daily application target: 15–20 applications per day**

**Keywords to search on job portals:**
```
DevOps Engineer
Cloud Engineer
Site Reliability Engineer (SRE)
Platform Engineer
Infrastructure Engineer
AWS DevOps
Kubernetes Engineer
CI/CD Engineer
DevOps Fresher
Junior DevOps
Cloud Infrastructure Engineer
```

---

### Step 1.4 — LinkedIn Profile Optimization (Week 2)

LinkedIn is where 70% of recruiters in India search for DevOps candidates.

**Must-do items:**
- ✅ Profile photo (professional, clear background)
- ✅ Headline: `DevOps Engineer | AWS | Kubernetes | Terraform | Jenkins | ArgoCD`
- ✅ About section: Same as resume summary but in first person
- ✅ Featured section: Add link to your GitHub FinBank project
- ✅ Skills section: Add all tools — recruiters search by skill keyword
- ✅ Turn on "Open to Work" — set it to visible to recruiters only
- ✅ Connect with 50+ DevOps engineers, recruiters, and hiring managers per week
- ✅ Post once a week about what you learned or built — visibility matters

---

### Step 1.5 — Interview Preparation (Week 3–6 — ongoing)

**How DevOps interviews work in India (3 rounds typical):**
```
Round 1: HR Screen (15 min) — background, salary expectation, notice period
Round 2: Technical Round 1 (45-60 min) — concept questions, scenario questions
Round 3: Technical Round 2 / Manager (45-60 min) — deep dive, hands-on problem solving
```

**Most asked interview topics (memorize these):**

#### Docker Questions
```
Q: What is the difference between a Docker image and a container?
A: Image = the blueprint (like a recipe). Container = the running instance (like the cooked dish).
   You can run many containers from one image.

Q: What is a Dockerfile? Walk me through one.
A: A Dockerfile is a text file with instructions to build a Docker image.
   FROM = base image, RUN = execute commands, COPY = copy files, CMD = what runs when container starts.

Q: What is Docker Compose?
A: A tool to run multiple containers together using a YAML file. For example, run app + database + Redis
   all with one "docker-compose up" command.
```

#### Kubernetes Questions
```
Q: What is a Pod, Deployment, and Service? Explain the difference.
A: Pod = smallest unit, runs one container. Deployment = manages multiple pod replicas and handles
   rolling updates. Service = stable network address (IP + DNS) for a group of pods.

Q: What happens when a pod crashes in Kubernetes?
A: The Deployment controller detects it and creates a new pod automatically on a healthy node.
   The Service keeps routing traffic to healthy pods only.

Q: What is a ConfigMap vs a Secret?
A: ConfigMap = non-sensitive config (database host, log level). Secret = sensitive data (passwords,
   tokens). Secrets are base64 encoded and can be mounted as files or env variables.

Q: What is Helm and why use it?
A: Helm is the package manager for Kubernetes. Instead of writing 63 separate YAML files
   (7 per service × 3 services × 3 environments), you write templates once and pass different
   values files per environment.
```

#### Terraform Questions
```
Q: What is Terraform state and why is it important?
A: State file (terraform.tfstate) is Terraform's memory — it maps your code to real AWS resources.
   Without it, Terraform doesn't know what it already created. We store it in S3 with DynamoDB
   locking to prevent two people applying at the same time.

Q: What is a Terraform module?
A: A reusable group of related resources. Like a function in programming — you define it once
   and call it multiple times with different inputs. In FinBank, we have 6 modules:
   VPC, EKS, RDS, ECR, ElastiCache, Secrets Manager.

Q: What is terraform plan vs terraform apply?
A: Plan = shows you what changes Terraform WILL make (preview, nothing created yet).
   Apply = actually creates/updates/deletes the resources.
```

#### CI/CD Questions
```
Q: Explain your CI/CD pipeline.
A: In FinBank, the Jenkins pipeline has 9 stages:
   1. Checkout code from GitHub
   2. Maven build (compile Java code)
   3. Unit tests
   4. SonarQube code quality analysis
   5. Docker build locally (for Trivy scan)
   6. Trivy container vulnerability scan
   7. Docker build + push to AWS ECR
   8. Update Helm values.yaml with new image tag
   9. Commit the tag update to GitHub → ArgoCD detects and deploys

Q: What is GitOps? How does ArgoCD implement it?
A: GitOps = Git is the single source of truth for what should be running in Kubernetes.
   ArgoCD watches the Git repository. When it detects a change (like a new image tag),
   it automatically syncs the Kubernetes cluster to match. No manual kubectl apply needed.
```

#### AWS Questions
```
Q: What is the difference between a public and private subnet?
A: Public subnet = resources get a public IP, can be reached from internet (ALB lives here).
   Private subnet = no public IP, not reachable from internet directly (EKS nodes, RDS, Redis).
   Private resources access internet outbound via NAT Gateway.

Q: What is IRSA?
A: IAM Roles for Service Accounts. Allows a specific Kubernetes pod to assume an IAM role
   and access AWS services (like Secrets Manager) without hardcoding any AWS credentials
   in the pod. Works through the OIDC trust bridge between EKS and AWS IAM.

Q: What is the difference between ECR and DockerHub?
A: Both store Docker images. ECR is AWS's private registry — images stay inside AWS,
   closer to EKS (faster pulls, no rate limits, no extra cost for data transfer within AWS).
   DockerHub is public — has pull rate limits and images travel over the internet to EKS.
```

---

### Stage 1 — Target Companies for First Job

**Tier A — Best for first job (high volume, take freshers):**
- TCS Digital / TCS iON
- Infosys — Systems Engineer / Digital roles
- HCLTech — Cloud/DevOps trainee
- Wipro — WILP or direct cloud roles
- Cognizant — GenC roles
- Accenture — Associate Software Engineer

**Tier B — Better pay, harder to get without experience:**
- Virtusa
- Mphasis
- Hexaware
- LTIMindtree
- Persistent Systems
- Birlasoft

**Tier C — Startups (best learning, equity upside):**
- Series A/B startups in Bangalore / Hyderabad
- Search: "DevOps fresher startup Bangalore" on LinkedIn

**Salary expectation at this stage: ₹5–9 LPA**

---

# STAGE 2: Junior DevOps Engineer (6 Months – 2 Years)

## Goal: Grow from ₹8 LPA → ₹14–18 LPA. Get your first certification.

You have a job. Now the goal is to learn fast, contribute beyond what is asked, and get your first AWS certification within 8–12 months of joining.

---

### Step 2.1 — Learn These on the Job (Month 1–12)

**A. Get deeply comfortable with real production**
- How does your company's pipeline work? Draw it on paper.
- When something breaks at 2AM (on-call), how is it fixed? Learn the runbooks.
- Read every Kubernetes YAML file in your company's repo. Understand each line.
- Understand the cost of your cloud infrastructure. Which services cost the most?

**B. Learn Python scripting properly**

Python is the #1 automation language for DevOps engineers. Even basic Python knowledge doubles your usefulness.

```python
# What to learn in Python for DevOps:
# 1. Reading/writing files
# 2. Making HTTP API calls (requests library)
# 3. Working with JSON and YAML
# 4. boto3 — AWS SDK for Python (talk to AWS from Python script)
# 5. Simple automation scripts (backup, cleanup, reporting)

# Example — list all EC2 instances using boto3:
import boto3
ec2 = boto3.client('ec2', region_name='ap-south-1')
instances = ec2.describe_instances()
for reservation in instances['Reservations']:
    for instance in reservation['Instances']:
        print(instance['InstanceId'], instance['State']['Name'])
```

**Free Python for DevOps resources:**
- Python for Everybody — Coursera (free to audit)
- Automate the Boring Stuff with Python — free at automatetheboringstuff.com
- Real Python — realpython.com (practical examples)

**C. Go deeper into Kubernetes**

You know the basics from FinBank. Now learn the harder parts:
```
What to learn next in Kubernetes:
→ RBAC — Role-Based Access Control (who can do what inside the cluster)
→ Network Policies — which pods can talk to which pods (security)
→ Resource Quotas and Limits — prevent one pod from eating all memory
→ Horizontal Pod Autoscaler (HPA) — auto-scale pods based on CPU/memory
→ Persistent Volumes (PV/PVC) — give pods access to storage that survives restarts
→ Custom Resource Definitions (CRDs) — how tools like ArgoCD extend Kubernetes
→ Kubernetes debugging — kubectl describe, kubectl logs, kubectl exec, events
```

**D. Learn Ansible (Infrastructure Configuration)**

Ansible is used for configuring servers — installing software, managing files, running scripts across many machines at once. It is required in 60% of DevOps job descriptions in India.

```yaml
# Ansible is simple — it uses YAML playbooks
# Example: Install nginx on 10 servers at once
- name: Install nginx
  hosts: webservers
  tasks:
    - name: Install nginx package
      apt:
        name: nginx
        state: present

# Run with: ansible-playbook install-nginx.yml
# It connects to all servers via SSH and runs the task
```

**What Ansible replaces:** Logging into 10 servers one by one and typing the same commands manually.

**Free Ansible resource:** Red Hat Ansible Basics — free on developer.redhat.com

---

### Step 2.2 — Get Your First Certification (Month 6–10)

#### Certification 1: AWS Certified Solutions Architect – Associate (SAA-C03)

**Why this first:**
- Most recognized AWS certification in India
- Adds ₹1.5–3 LPA to your salary immediately
- Required for most mid-level DevOps roles
- Validates your AWS knowledge from the FinBank project

**Cost:** ₹15,135 (including 18% GST) — exam fee only

**Study plan (8 weeks):**
```
Week 1-2: Core services — EC2, VPC, S3, IAM
Week 3-4: Databases — RDS, DynamoDB, ElastiCache
Week 5:   Compute — Lambda, ECS, EKS, Auto Scaling
Week 6:   Networking — Route 53, CloudFront, ALB, VPN, Direct Connect
Week 7:   Security — IAM roles, KMS, Secrets Manager, WAF, Shield
Week 8:   Practice exams — minimum 3 full mock tests before attempting
```

**Free/Paid study resources:**
- **Free:** AWS official skill builder — skillbuilder.aws (free tier available)
- **Free:** freeCodeCamp SAA-C03 course on YouTube (13 hours — excellent)
- **Paid (₹500):** Udemy — Stephane Maarek's SAA-C03 course (best paid resource)
- **Paid (₹1000):** Tutorial Dojo practice exams — most realistic mock tests

**You already know:** VPC, EKS, RDS, ECR, ElastiCache, Secrets Manager, IAM, ALB from FinBank. You are 40% prepared already.

---

### Step 2.3 — Build a Second Project (Month 8–18)

Having two projects shows a pattern of skill, not a one-time achievement.

**Project ideas at this stage:**

**Option A — DevSecOps Pipeline (highly in demand)**
```
Build a complete security-focused pipeline:
→ GitHub Actions (instead of Jenkins — shows flexibility)
→ SAST scanning with Semgrep or Checkmarx
→ Container scanning with Trivy (you know this)
→ Open Policy Agent (OPA) for Kubernetes policy enforcement
→ Secrets scanning with GitLeaks
→ Dependency scanning with OWASP Dependency Check
→ Deploy to EKS with all security gates enforced
```

**Option B — Multi-Cloud Setup (shows Azure + AWS)**
```
Take your FinBank app and:
→ Deploy backend on AWS EKS (you have this)
→ Deploy frontend on Azure AKS
→ Use Azure Front Door or AWS CloudFront for global routing
→ Show it on resume as "multi-cloud deployment"
```

**Option C — GitOps with GitHub Actions + ArgoCD**
```
Replace Jenkins with GitHub Actions:
→ Write GitHub Actions workflows (.github/workflows/*.yml)
→ Same pipeline stages as Jenkins but in GitHub native YAML
→ Push to ECR, update Helm values, ArgoCD deploys
→ Shows you know multiple CI/CD tools (Jenkins AND GitHub Actions)
```

---

### Step 2.4 — Where to Move After Stage 2

After 18–24 months with solid production experience + AWS SAA certification, target:

| Company Type | Example Companies | Expected Package |
|---|---|---|
| Product companies | Razorpay, CRED, Zepto, Groww, PhonePe | ₹18–28 LPA |
| GCCs | Goldman Sachs, JP Morgan, HSBC, Citi India | ₹16–25 LPA |
| Mid-size IT | Mphasis, Persistent, LTIMindtree | ₹13–18 LPA |
| Large IT upgrade | Infosys → HCL → Accenture → next tier | ₹12–16 LPA |

**How to move:** Apply 3 months before your 2-year mark. Recruiters value 2 years of solid experience much more than 18 months.

---

# STAGE 3: Mid-Level DevOps Engineer (2 – 4 Years)

## Goal: Grow from ₹18 LPA → ₹28–35 LPA. Become the go-to person on your team.

At this stage you are no longer learning basics — you are making **technical decisions** and becoming someone others depend on.

---

### Step 3.1 — Advanced Skills to Learn

**A. DevSecOps — Security in DevOps (Highest Demand Specialization)**

Security is the fastest growing area inside DevOps. Companies that get hacked (and banking companies especially) are investing heavily in "shift-left security" — catching problems in the pipeline before they reach production.

```
What DevSecOps covers:
→ SAST (Static Application Security Testing) — scan source code for vulnerabilities
→ DAST (Dynamic Application Security Testing) — test running application for vulnerabilities
→ Container security — Trivy (you know this), Falco (runtime threat detection)
→ Infrastructure security — tfsec or Checkov (scan Terraform code for misconfigurations)
→ Secrets scanning — GitLeaks, TruffleHog (find accidentally committed passwords)
→ Supply chain security — sign Docker images with Cosign, verify before deployment
→ Kubernetes security — OPA/Gatekeeper policies, Pod Security Standards
→ Compliance — SOC2, ISO 27001, PCI-DSS (important for banking)
```

**Why DevSecOps pays more:** A DevOps engineer who can also handle security is rarer and more valuable. Target: ₹5–10 LPA premium over a regular DevOps engineer.

---

**B. Platform Engineering (The Future of DevOps)**

Platform Engineering is what DevOps evolves into. Instead of each DevOps engineer manually helping developers deploy their apps, Platform Engineers build **self-service internal tools** so developers can deploy themselves without waiting for the DevOps team.

```
Platform Engineering tools to learn:
→ Backstage (Spotify's open-source internal developer portal)
→ Crossplane — manage cloud infrastructure from Kubernetes (like Terraform but in K8s)
→ Service Mesh — Istio or Linkerd (control network traffic between microservices)
→ Internal developer platforms (IDPs) — what companies like Flipkart, Swiggy build internally
```

Think of it this way: DevOps engineer = builds the car. Platform Engineer = builds the road system and highway that all cars drive on.

---

**C. Advanced Cloud — Cost Optimization**

Cloud bills at big companies can be ₹5–50 crore per month. Engineers who can **reduce cloud costs** without hurting performance get promoted and paid very well.

```
Cost optimization skills:
→ AWS Cost Explorer — analyze where money is being spent
→ Reserved Instances vs Spot Instances vs On-Demand (when to use which)
→ Right-sizing — identifying oversized EC2/RDS instances
→ Kubernetes resource requests and limits tuning — stop wasting RAM/CPU
→ S3 lifecycle policies — automatically move old files to cheaper storage classes
→ Identifying idle or orphaned resources (EIPs, unused EBS volumes, stopped EC2s)
→ FinOps — the practice of cloud financial management
```

**This skill alone can get you a ₹5 LPA raise** — companies love engineers who save them money.

---

**D. Service Mesh — Istio**

When you have 50+ microservices talking to each other, you need a way to:
- Control which service can talk to which
- Add retries and circuit breakers automatically
- Encrypt all service-to-service traffic (mTLS)
- Track latency and errors between services

Istio is the most popular service mesh. It adds a "sidecar proxy" (Envoy) to every pod that handles all network communication transparently.

```
What Istio does:
→ Traffic management — route 10% traffic to new version, 90% to old (canary deployment)
→ Security — automatic mTLS encryption between all services
→ Observability — detailed metrics, traces, logs for every service call
→ Fault injection — test how your system handles failures (chaos engineering)
```

---

### Step 3.2 — Certifications at Stage 3

#### Certification 2: Certified Kubernetes Administrator (CKA)

**Why CKA:**
- Most respected Kubernetes certification globally
- Linux Foundation exam — hands-on in a real terminal (not multiple choice)
- Proves you can actually operate Kubernetes, not just answer quiz questions
- Required or preferred in 40% of senior DevOps job descriptions in India

**Cost:** ~₹35,000 (exam fee, includes 2 attempts)

**What the exam tests:**
```
CKA Exam Domains:
→ Cluster architecture, installation, configuration (25%)
→ Workloads and scheduling (15%)
→ Services and networking (20%)
→ Storage (10%)
→ Troubleshooting (30%) ← the hardest and most important part
```

**Study resources:**
- Mumshad Mannambeth's CKA course on Udemy (₹500, best resource)
- KodeKloud practice labs (₹1500/month — worth every rupee for hands-on practice)
- killer.sh practice exam (included with CKA purchase — do this twice)

---

#### Certification 3: AWS DevOps Engineer – Professional (DOP-C02)

**Why AWS DevOps Pro:**
- Highest AWS certification for a DevOps engineer
- Proves you can architect complete DevOps solutions on AWS
- Most valued certification by product companies and GCCs
- Salary bump: ₹3–8 LPA at this level

**Cost:** ₹25,000 (Professional level exam fee)

**Prerequisite:** You should have SAA-Associate before attempting this.

**What the exam tests:**
```
DOP-C02 Exam Domains:
→ SDLC automation — CodePipeline, CodeBuild, CodeDeploy, Jenkins on AWS
→ Configuration management — CloudFormation, OpsWorks, Systems Manager
→ Monitoring and logging — CloudWatch, X-Ray, OpenSearch
→ Policies and standards — IAM, SCP, Config, Security Hub, GuardDuty
→ Incident and event response — auto-remediation, EventBridge
→ High availability and disaster recovery — multi-region, RTO/RPO
```

---

### Step 3.3 — Where to Apply at Stage 3

| Company | Role | Expected Package |
|---|---|---|
| **Razorpay** | Senior DevOps / SRE | ₹25–40 LPA |
| **CRED** | Senior Infrastructure Engineer | ₹28–45 LPA |
| **Zepto / Blinkit** | Platform Engineer | ₹22–38 LPA |
| **PhonePe** | SRE / Cloud Engineer | ₹25–42 LPA |
| **Groww / Smallcase** | DevOps Lead | ₹22–35 LPA |
| **Goldman Sachs GCC** | Senior Cloud Engineer | ₹28–45 LPA |
| **JP Morgan India** | Site Reliability Engineer | ₹25–40 LPA |
| **Atlassian India** | Platform Engineer | ₹30–50 LPA |
| **Freshworks** | Senior DevOps Engineer | ₹20–35 LPA |
| **Hasura / Chargebee** | Infrastructure Engineer | ₹22–38 LPA |

---

# STAGE 4: Senior DevOps / Tech Lead (4 – 7 Years)

## Goal: ₹35 LPA → ₹55 LPA. Lead a team. Own a platform.

At this stage you are not just doing — you are **designing** and **leading**. You make architecture decisions that affect the whole company. You mentor junior engineers.

---

### Step 4.1 — Leadership Skills (As Important as Technical)

This is where most technical people get stuck. They know everything technically but never get promoted because they cannot lead. Leadership skills you need:

```
→ Running architecture review meetings
→ Writing technical design documents (TDDs) — explain a solution before building it
→ Mentoring junior DevOps engineers
→ Working with product managers and developers (not just the DevOps team)
→ Incident management — staying calm when production is down, leading the war room
→ Presenting cloud cost reports to management
→ Hiring — interviewing and evaluating DevOps candidates
→ Sprint planning and task estimation
```

---

### Step 4.2 — Technical Areas at Senior Level

**A. SRE (Site Reliability Engineering)**
```
SRE concepts to master:
→ SLO (Service Level Objective) — "our API must respond in <200ms 99.9% of the time"
→ SLI (Service Level Indicator) — the actual measured metric
→ Error Budget — how much downtime/slowness is acceptable per month
→ Toil reduction — automating repetitive manual tasks
→ Chaos Engineering — intentionally breaking things to find weaknesses (Netflix-style)
→ Post-mortem culture — blame-free incident reviews to prevent repeat failures
```

**B. FinOps (Cloud Financial Operations)**
```
At senior level you own cloud costs:
→ Build cost dashboards showing cost per team, per service, per environment
→ Implement Kubernetes resource quotas per namespace (each team gets a budget)
→ Recommend Reserved Instance purchases (save 30–40% on compute)
→ Identify and eliminate waste automatically with Lambda functions
→ Report monthly cloud cost trends to CTO / VP Engineering
```

**C. Multi-Region Architecture**
```
Production-grade systems need to survive an entire AWS region going down:
→ Active-active multi-region — both regions serve traffic (complex but zero downtime)
→ Active-passive multi-region — one region is standby (simpler, some downtime)
→ Global databases — Aurora Global, DynamoDB Global Tables
→ DNS failover — Route 53 health checks auto-switch traffic between regions
→ Data replication — how to keep databases in sync across regions
```

---

### Step 4.3 — Certification at Stage 4

#### Certification 4: AWS Solutions Architect – Professional (SAP-C02)

This is the hardest AWS certification. Passing it puts you in the top 5% of AWS professionals in India.

**Cost:** ₹25,000

**What it tests:**
```
→ Designing solutions for organizational complexity
→ New solutions that are cost-optimized, resilient, and high-performing
→ Migrating complex multi-tier applications
→ Designing for new solutions across all AWS services
→ Continuous improvement of existing solutions
```

#### Certification 5 (Optional): CKAD or CKS

- **CKAD** (Certified Kubernetes Application Developer) — useful if moving toward platform engineering
- **CKS** (Certified Kubernetes Security Specialist) — if moving toward DevSecOps

---

### Step 4.4 — Specialization Paths at Stage 4

At this level, pick ONE specialization. Generalists plateau at ₹35–40 LPA. Specialists go to ₹55 LPA+.

| Specialization | What It Means | Salary Range |
|---|---|---|
| **SRE** | Reliability, uptime, SLOs, on-call engineering | ₹35–60 LPA |
| **Platform Engineer** | Build internal platforms, developer experience | ₹38–65 LPA |
| **DevSecOps Architect** | Security across the entire SDLC | ₹35–55 LPA |
| **Cloud Architect** | Design cloud solutions, cloud migrations | ₹40–75 LPA |
| **FinOps Engineer** | Cloud cost optimization and governance | ₹32–55 LPA |

---

# STAGE 5: Architect / Manager / Principal (7 – 12 Years)

## Goal: ₹55 LPA → ₹1 Crore+

At this level you are designing the technology strategy for an organization.

---

### What This Role Looks Like

```
→ Define the company's cloud strategy for the next 3 years
→ Evaluate and choose tools (which Kubernetes distribution, which CI/CD platform)
→ Set engineering standards that all teams must follow
→ Review architecture proposals from senior engineers
→ Work with CISO (Chief Information Security Officer) on compliance
→ Present to CTO and board on infrastructure reliability and cost
→ Build and manage a team of 10–30 DevOps/SRE/Platform engineers
→ Hire, evaluate performance, handle escalations
```

### Where Architects Work in India

| Company | Title | Package |
|---|---|---|
| Google India | Staff SRE / Principal Engineer | ₹80L – 1.5 Cr |
| Amazon India | Principal SDE / Principal DevOps | ₹70L – 1.2 Cr |
| Microsoft India | Principal Engineer / Cloud Architect | ₹65L – 1.1 Cr |
| Flipkart | VP Engineering / Principal Platform | ₹60L – 1 Cr |
| Razorpay / CRED | VP Engineering / CTO path | ₹60L – 1.5 Cr |
| GCCs (Goldman, JP Morgan) | Executive Director / MD (tech) | ₹80L – 1.5 Cr |

---

## Complete Salary Journey Summary

```
Stage 1 (0–6 months)    First job                     ₹5  – 9  LPA
Stage 2 (6mo–2 years)   Junior DevOps + AWS SAA cert  ₹10 – 18 LPA
Stage 3 (2–4 years)     Mid-level + CKA + AWS DOP     ₹18 – 35 LPA
Stage 4 (4–7 years)     Senior / Tech Lead + AWS SAP  ₹35 – 55 LPA
Stage 5 (7–12 years)    Architect / VP / Principal    ₹55 – 1.5 Cr LPA
```

---

## Complete Certification Roadmap

```
YEAR 1      AWS Solutions Architect – Associate (SAA-C03)   ₹15,135
YEAR 2      HashiCorp Terraform Associate                   ₹20,000
YEAR 2-3    Certified Kubernetes Administrator (CKA)        ₹35,000
YEAR 3-4    AWS DevOps Engineer – Professional (DOP-C02)    ₹25,000
YEAR 5      AWS Solutions Architect – Professional (SAP)    ₹25,000
OPTIONAL    Azure DevOps Expert (AZ-400)                    ₹15,000
OPTIONAL    CKS – Kubernetes Security Specialist            ₹35,000

Total investment over 5 years: ~₹1.7 Lakh
Return: From ₹6 LPA fresher → ₹35 LPA mid-level (in 4 years)
ROI: Extremely high
```

---

## Complete Skills Roadmap — What to Learn and When

```
NOW (Already Done)
✅ Linux fundamentals
✅ Docker and containerization
✅ Jenkins CI/CD pipeline
✅ Terraform (6 custom modules on AWS)
✅ Kubernetes (EKS) + Helm + ArgoCD
✅ AWS core services (VPC, EKS, RDS, ECR, ElastiCache, Secrets Manager, ALB, IAM)
✅ Azure (AKS, ACR, Key Vault, Azure Pipelines, Terraform)
✅ Prometheus + Grafana monitoring
✅ SonarQube + Trivy security scanning
✅ GitOps with ArgoCD

YEAR 1 (Next 12 Months)
→ Python scripting for automation (boto3, YAML parsing, REST APIs)
→ Ansible for configuration management
→ GitHub Actions (second CI/CD tool — most companies use this now)
→ AWS SAA-C03 Certification
→ Git advanced — branching strategies, rebasing, conflict resolution
→ Shell scripting (Bash) — deeper level

YEAR 2
→ Terraform Associate Certification
→ Advanced Kubernetes — RBAC, Network Policies, HPA, PV/PVC
→ CKA Certification
→ DevSecOps tools — tfsec, Checkov, Semgrep, OPA/Gatekeeper
→ Istio service mesh basics
→ GitLeaks, SBOM (Software Bill of Materials)

YEAR 3-4
→ AWS DevOps Engineer Professional Certification
→ Platform Engineering — Backstage, Crossplane
→ SRE practices — SLOs, SLIs, error budgets, chaos engineering (Chaos Monkey)
→ Cloud cost optimization — FinOps practices
→ Multi-region architecture design
→ Incident management and post-mortem culture

YEAR 5+
→ AWS Solutions Architect Professional
→ Architecture design patterns — event-driven, microservices at scale
→ Engineering leadership and team management
→ Cloud strategy and roadmap planning
→ FinOps governance at organizational level
```

---

## Daily / Weekly Routine to Stay Ahead

### Daily (30–45 minutes)

```
Morning (15 min):
→ Read one DevOps article or blog post
  Recommended: DevOps.com, The New Stack, AWS Blog, CNCF Blog

Evening (15-30 min):
→ Practice something hands-on (not just reading)
  Rotate between: Kubernetes labs, Terraform exercises, Python scripts, AWS console
```

### Weekly

```
→ Apply to 15-20 jobs on Naukri + LinkedIn (even after you get a job — always be aware of market)
→ Post one thing on LinkedIn about what you learned or built this week
→ Connect with 5-10 new DevOps engineers / recruiters on LinkedIn
→ Watch one conference talk: KubeCon, AWS re:Invent, HashiConf (all free on YouTube)
→ Read CNCF (Cloud Native Computing Foundation) newsletter
```

### Monthly

```
→ Complete one chapter/section of your current certification course
→ Review your resume — update with anything new you learned or built
→ Check job market — what new skills are appearing in job postings?
→ Check your LinkedIn profile views — if low, update your headline/about section
```

---

## Best Free Resources to Use Now

| Resource | What It Is | Link |
|---|---|---|
| **AWS Skill Builder** | Official AWS training, free tier available | skillbuilder.aws |
| **KodeKloud** | Best hands-on Kubernetes + DevOps labs | kodekloud.com |
| **Play with Kubernetes** | Free K8s playground in browser | labs.play-with-k8s.com |
| **Terraform Registry** | Official Terraform documentation | registry.terraform.io |
| **GitHub NotHarshhaa/DevOps-Interview-Questions** | 1100+ DevOps interview Q&A | github.com/NotHarshhaa |
| **freeCodeCamp YouTube** | Free AWS SAA, CKA courses (full length) | youtube.com/freecodecamp |
| **CNCF YouTube** | KubeCon talks — watch real engineers discuss real problems | youtube.com/cncf |
| **Stephane Maarek Udemy** | Best paid AWS courses (~₹500 in sale) | udemy.com |
| **Mumshad Mannambeth Udemy** | Best paid CKA course (~₹500 in sale) | udemy.com |
| **AWS re:Invent YouTube** | Real AWS architecture talks from AWS engineers | youtube.com |

---

## Top Communities to Join in India

```
→ DevOps India Slack workspace — slack.devopsindia.in
→ CNCF India Slack — communityinviter.com/apps/cloud-native/cncf
→ AWS User Group India — aws.amazon.com/developer/community/usergroups
→ LinkedIn: "DevOps Engineers India" group
→ Reddit: r/devops (global, very active)
→ Telegram: "DevOps Jobs India" channels — many active job postings
→ Discord: KodeKloud community — active engineers helping each other
```

---

## The Most Important Mindset Rules

```
Rule 1: Build, don't just watch.
   For every concept you learn, build something with it.
   Watching a Kubernetes tutorial ≠ knowing Kubernetes.
   Breaking and fixing a real cluster = knowing Kubernetes.

Rule 2: Document everything.
   Your FINBANK_COMPLETE_DEVOPS_GUIDE.md and this file are proof of this.
   Engineers who write clearly are valued more than those who cannot explain their work.

Rule 3: The market rewards depth, not breadth.
   Being good at Kubernetes + AWS is better than being OK at 15 tools.
   Go deep first, then expand.

Rule 4: Your project is your proof.
   When a recruiter asks "can you do Terraform?" — you say yes AND send the GitHub link.
   Most candidates say yes but have nothing to show. You have everything to show.

Rule 5: Certifications open doors, projects close them.
   Certifications get you the interview call. Your project knowledge gets you the offer.
   You need both.

Rule 6: Network actively.
   In India, 30-40% of jobs are filled through referrals before they are even posted publicly.
   Every person you connect with on LinkedIn is a potential referral into their company.

Rule 7: Always be applying.
   Even when you have a job, apply once a month. Know your market value.
   The best negotiating leverage is a competing offer.
```

---

## Your Immediate Action Items — This Week

```
Day 1 (Today):
☐ Make Infra_FinBank repository public on GitHub
☐ Write a proper README for FinBank project (architecture diagram + tech stack)

Day 2:
☐ Update LinkedIn headline and About section
☐ Turn on "Open to Work" visible to recruiters

Day 3:
☐ Draft your resume using the template in Stage 1
☐ Get it reviewed by a DevOps engineer on LinkedIn

Day 4:
☐ Create profiles on Naukri.com, Foundit.in, Instahyre
☐ Upload resume to all portals with proper keywords

Day 5:
☐ Apply to 20 jobs across all portals
☐ Send 10 LinkedIn connection requests to DevOps recruiters

Day 6-7:
☐ Start AWS SAA-C03 study (freeCodeCamp YouTube — free, 13 hours)
☐ Join CNCF India Slack + DevOps India communities
```

---

*Last updated: July 2026*
*Based on current India DevOps job market data — 35,000+ active openings, 30% YoY growth*

---

> You have already done the hardest part — you built real projects on real cloud infrastructure.
> Now it is time to show that work to the world and get paid for it.
