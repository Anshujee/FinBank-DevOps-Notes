# Mock Interview — Azure DevOps Engineer

> Personal revision notes based on real mock interview practice sessions.
> Use these for verbal practice — read the question aloud, close the answer, answer from memory, then compare.

---

## Interview Questions Covered

1. [Introduce yourself and walk through your current role](#q1-self-introduction-and-current-role)
2. [End-to-end: new Azure environment request via Terraform](#q2-end-to-end-infrastructure-provisioning-with-terraform)
3. [Terraform drift, state locking, and concurrent pipeline conflicts](#q3-terraform-drift-state-locking-and-concurrent-execution)
4. [Production AKS incident — CrashLoopBackOff after deployment](#q4-production-troubleshooting--crashloopbackoff-on-aks)
5. [Pods are healthy but users cannot access the application — traffic flow and troubleshooting](#q5-pods-healthy-but-users-cannot-access--traffic-flow-and-layer-by-layer-troubleshooting)

---

## Q1. Introduce yourself and walk me through your current role and responsibilities.

### Your Answer (Polished)

Good morning, sir. Thank you for giving me this opportunity.

My name is Anshu, and I have around 10 years of overall IT experience across production support, testing, networking, cloud infrastructure, and DevOps.

Currently, I am working with Infosys BPM Limited, where my responsibilities are primarily focused on cloud infrastructure and DevOps activities. I work closely with development and microservices teams to understand their infrastructure requirements and provision and manage the required Azure resources.

For infrastructure provisioning, we primarily use Terraform as Infrastructure as Code. Rather than creating resources manually, we follow an automated approach where infrastructure changes are maintained as code, reviewed, and deployed through CI/CD pipelines. I am involved in analysing infrastructure requirements, modifying Terraform configurations, validating the changes, and supporting deployments across environments.

For configuration management and automation, I have worked with Ansible, and I also have experience with Docker and Kubernetes. I understand the process of containerising applications using Docker and deploying and managing containerised workloads on Azure Kubernetes Service.

From an operations perspective, I am also involved in troubleshooting infrastructure and application connectivity issues, and working with development and DevOps teams to identify whether problems are related to compute, networking, Kubernetes, configuration, or application components.

My earlier experience in production support and networking has also helped me in cloud operations — particularly when troubleshooting DNS, connectivity, load balancing, and infrastructure-related issues.

Overall, my experience is centred around Azure cloud infrastructure, Terraform, Ansible, Kubernetes, Docker, CI/CD automation, and production support. I am now looking for an opportunity where I can apply this experience in a more DevOps-focused environment and take greater ownership of automation, Kubernetes, and cloud infrastructure.

That is a brief overview of my experience and current responsibilities.

---

### What Makes This Answer Strong

- Opens professionally and confidently
- Gives a timeline (10 years) and a current employer — anchors you in a real context
- Covers every JD keyword: Terraform, Ansible, AKS, Docker, CI/CD, troubleshooting, networking
- The closing sentence explains *why you are here* — interviewers always want to know this
- Does not list every technology you have ever touched — stays focused on what is relevant to the role

### Tip for Delivery

Practice this until it takes 90–120 seconds. Do not read it. Key anchor points to memorise:
> 10 years → current role at Infosys → Terraform IaC → Ansible + Docker + AKS → troubleshooting background → why I want this role

---

## Q2. End-to-end Infrastructure Provisioning with Terraform

**Interviewer's Question:**

> A development team approaches you and says: "We need a new Azure environment — VNet, subnets, NSGs, an AKS cluster, and supporting resources. It needs to be deployed through Terraform." Walk me through how you would handle this end-to-end, from receiving the request until the infrastructure is successfully deployed. Also explain how you would structure the Terraform code and manage the state.

### Your Answer (Polished)

Once we receive the infrastructure request — typically through a Jira ticket or a formal change request — my first step is to understand the complete requirement. This includes the target environment, Azure region, VNet CIDR and subnet design, NSG rules, AKS node pool sizing, access requirements, and any service dependencies.

Next, I review our existing Terraform repository to determine whether we already have reusable modules that satisfy the requirement. We follow a modular Terraform approach, so instead of writing infrastructure from scratch, we reuse and extend standard modules wherever possible.

**Module structure:**

We maintain reusable modules for common components — networking, NSGs, and AKS, for example. Each module contains the required resources, input variables, and outputs. At the root level, we call these modules and pass environment-specific values through `tfvars` files or CI/CD pipeline variables.

```
terraform/
  modules/
    networking/     # VNet, subnets, route tables
    nsg/            # NSG rules and associations
    aks/            # AKS cluster, node pools
  environments/
    dev/
      main.tf       # calls modules
      dev.tfvars    # environment-specific values
    prod/
      main.tf
      prod.tfvars
```

**State management:**

We use a remote backend rather than storing the state locally. In Azure, Terraform state is stored in an Azure Storage Account Blob container using the `azurerm` backend configuration. We use a separate state file per environment — for example, `dev/terraform.tfstate` and `prod/terraform.tfstate` — so a change in one environment never risks the state of another. State files are never committed to Git because they can contain sensitive infrastructure information.

```hcl
terraform {
  backend "azurerm" {
    resource_group_name  = "rg-terraform-state"
    storage_account_name = "stterraformstate001"
    container_name       = "tfstate"
    key                  = "dev/aks/terraform.tfstate"
  }
}
```

**Pipeline execution:**

Once the code is ready, it goes through version control and a code review process. The CI/CD pipeline then handles the deployment.

The pipeline typically performs:
1. Code checkout
2. `terraform fmt` — formatting check
3. `terraform init` — initialise providers and backend
4. `terraform validate` — validate syntax and configuration
5. `terraform plan` — generate a plan and output it for review
6. Approval gate — an authorised engineer reviews the plan
7. `terraform apply` — executes the approved plan

We use `-out=plan.tfplan` to save the plan and pass it directly to apply, so what was reviewed is exactly what gets deployed.

Authentication to Azure is handled through the CI/CD identity — either a Service Principal or Workload Identity — and credentials are never hard-coded inside the Terraform code.

After deployment, I verify that the expected resources have been created, check AKS cluster status, review NSG and subnet configuration, and confirm basic application connectivity. Finally, I update the Jira ticket with the implementation evidence and hand the environment over to the requesting team.

---

### What Makes This Answer Strong

- Shows real ownership: Jira → design → code review → pipeline → handover
- Mentions the exact Terraform commands in the correct order
- Explains `plan -out` flag — shows depth that most candidates miss
- Separates state per environment — a genuine best practice
- Mentions that state files are not committed to Git — a security point interviewers notice
- Authentication via Service Principal — not hard-coded secrets

---

## Q3. Terraform Drift, State Locking, and Concurrent Execution

**Interviewer's Question:**

> You mentioned that your Terraform state is remotely managed. Suppose another engineer goes to the Azure Portal and manually changes an NSG rule that was originally created and managed by Terraform. The next day, you run `terraform plan`. What will happen? What is Terraform drift, how would you identify it, and how would you handle this situation without accidentally impacting production? Also — if two engineers or two pipelines try to modify the same Terraform infrastructure at the same time, what happens and how would you prevent conflicts?

### Your Answer (Polished)

**What is configuration drift:**

Configuration drift happens when the actual state of infrastructure in Azure no longer matches the desired state defined in our Terraform code. A manual change to an NSG rule through the Azure Portal is a classic example — Terraform managed that resource, but a direct change was made outside of Terraform.

**What happens when I run `terraform plan`:**

When I run `terraform plan`, Terraform first refreshes its view of the actual infrastructure by querying Azure through the provider. It then compares what Azure actually has against what our Terraform configuration says it should have.

For the manually changed NSG rule, the plan will show a proposed change to overwrite the manual modification and restore the configuration to what is defined in our Terraform code.

**Dedicated drift detection:**

Before deciding on any action, especially in production, I would first run:

```bash
terraform plan -refresh-only
```

This command is specifically designed to check for drift. It shows what has changed in the real infrastructure compared to our Terraform state — without proposing any resource modifications. It is a safe, read-only way to identify drift before taking action.

**How I handle it — two scenarios:**

*Scenario 1 — The manual change was intentional and valid:*
I would first check the Azure Activity Log to see who made the change, when, and whether it was associated with an approved incident or emergency change ticket. If the change is valid and needs to persist, I update the Terraform configuration to match the new desired state, raise a pull request, get it reviewed and approved, and then run the pipeline again to align the state with the code.

*Scenario 2 — The manual change was unauthorised or incorrect:*
After confirming through the Activity Log and discussing with the team, I would run `terraform plan` to verify the exact change Terraform proposes, get the required approval, and then run `terraform apply` to restore the resource to the correct configuration.

In either case, I would not run `terraform apply` in production without a reviewed and approved plan.

**Concurrent execution and state locking:**

Since our Terraform state is stored remotely in Azure Blob Storage using the `azurerm` backend, Azure Blob Storage uses a blob lease mechanism for state locking. When one Terraform operation acquires the lock, any other Terraform operation against the same state will fail with a lock error rather than running concurrently and corrupting the state.

This protects against two engineers or two CI/CD pipelines running `terraform apply` at the same time on the same environment.

If a stale lock exists because a previous run failed midway, I would confirm that no Terraform process is actively running before considering a force-unlock:

```bash
terraform force-unlock <lock-id>
```

This is used with caution — only when I am certain the previous operation is no longer running.

**Prevention through process:**

Beyond technical locking, I also try to prevent unauthorised manual changes through RBAC and least-privilege access policies, so that infrastructure changes go through IaC and the change management process rather than direct portal modifications.

---

### What Makes This Answer Strong

- `terraform plan -refresh-only` — shows awareness of a specific command for drift detection that most candidates do not know
- Checks Activity Log before acting — shows production maturity, not just technical knowledge
- Two clear scenarios: valid change vs invalid change — structured thinking
- `plan -out` + approval before apply for production — shows safety-first approach
- Explains blob lease mechanism for locking — specific, not generic
- `force-unlock` mentioned with appropriate caution — not reckless

---

## Q4. Production Troubleshooting — CrashLoopBackOff on AKS

**Interviewer's Question:**

> We have a production application running on AKS. Developers deployed a new version through the CI/CD pipeline. The deployment completed, but users are reporting that the application is unavailable. When you run `kubectl get pods -n production`, you see:
>
> ```
> NAME                       READY   STATUS             RESTARTS
> payment-api-7d89c8-x1      0/1     CrashLoopBackOff   6
> payment-api-7d89c8-x2      0/1     CrashLoopBackOff   6
> ```
>
> Walk me through exactly how you would troubleshoot this. What commands would you run? How would you identify the root cause? And how would you restore the service quickly while minimising production impact?

### Your Answer (Polished)

Since this is a production outage immediately following a new deployment, my first priority is to assess the impact and begin service restoration quickly, while investigating the root cause in parallel.

**Step 1 — Confirm the scope:**

```bash
kubectl get pods -n production
kubectl get deployment payment-api -n production
kubectl rollout status deployment/payment-api -n production
```

The `CrashLoopBackOff` status tells me the containers are starting, crashing, and being restarted by Kubernetes. With 6 restarts already, Kubernetes is backing off between restart attempts. This means the application itself is failing to start or crashing shortly after startup.

**Step 2 — Check Pod events and container state:**

```bash
kubectl describe pod payment-api-7d89c8-x1 -n production
```

I look at the Events section at the bottom and the container state details. This tells me the exit code of the last crash, any probe failures, image pull issues, resource limit violations, or scheduling problems. An exit code of `1` or `2` indicates an application error. An exit code of `137` means the container was killed — often OOMKilled due to memory limits.

**Step 3 — Check container logs:**

```bash
# Current logs (may be empty if container crashed immediately)
kubectl logs payment-api-7d89c8-x1 -n production

# Previous container logs — this is the most important one in CrashLoopBackOff
kubectl logs payment-api-7d89c8-x1 -n production --previous
```

The `--previous` flag shows the logs from the container instance that crashed. This is where I will most likely find the error message — a startup exception, a database connection failure, a missing environment variable, or a configuration error.

**Step 4 — Check recent cluster events:**

```bash
kubectl get events -n production --sort-by='.metadata.creationTimestamp' | tail -30
```

This shows recent warnings and errors across the namespace, which can surface issues not visible in individual pod logs — such as image pull failures, volume mount issues, or resource quota breaches.

**Step 5 — Check the deployment history:**

```bash
kubectl rollout history deployment/payment-api -n production
kubectl describe deployment payment-api -n production
```

Since the issue started immediately after a deployment, I compare the current revision with the previous revision to understand exactly what changed — image tag, environment variables, resource limits, ConfigMap or Secret references.

**Step 6 — Common root causes I check in this sequence:**

| Root Cause | How to Identify |
|---|---|
| Missing or incorrect Secret / ConfigMap | `kubectl describe pod` — MountVolume failed or environment variable missing |
| Database / Redis connectivity failure | Application logs show connection refused or timeout on startup |
| Incorrect environment variable | Logs show config parsing error or null reference |
| Liveness probe killing healthy container | `kubectl describe pod` — liveness probe failed events |
| OOMKilled — memory limit too low | Exit code 137 in `kubectl describe pod` |
| Incorrect image or startup command | `kubectl describe pod` — image pull error or wrong entrypoint |
| Application startup exception | Previous container logs show stack trace |

**Step 7 — Restore production service immediately:**

Because this is a production outage and the issue clearly started with the new deployment, I would perform a controlled rollback while the root cause investigation continues:

```bash
kubectl rollout undo deployment/payment-api -n production
```

This reverts to the previous known-stable revision.

**Step 8 — Verify recovery:**

```bash
kubectl rollout status deployment/payment-api -n production
kubectl get pods -n production
kubectl get endpoints payment-api -n production
```

I confirm that the Pods are Running and Ready, and that the Service endpoints are populated. A quick application health check confirms the service is responding before I communicate recovery to the team.

**Step 9 — Post-incident:**

Once production is stable, I work with the development team to perform a proper root-cause analysis using the container logs, deployment diff, and monitoring data. The fix is tested in dev and staging before being redeployed to production through the normal CI/CD process.

I document the incident with timeline, root cause, resolution steps, and preventive actions — for example, adding a startup probe to catch slow-starting applications before they are marked Ready, or improving environment variable validation at application startup.

---

### What Makes This Answer Strong

- Starts with "restore service first, investigate in parallel" — production maturity
- `--previous` flag on kubectl logs — the single most important command for CrashLoopBackOff; many candidates miss this
- Exit codes explained (1/2 = app error, 137 = OOMKilled) — shows depth
- The root cause table gives the interviewer confidence you have seen these problems before
- `kubectl rollout undo` is the right instinct — fast, safe, reversible
- Checks `kubectl get endpoints` after rollback — verifies the Service is actually serving traffic, not just that Pods are running
- Closes with documentation and prevention — shows end-to-end ownership

### The Commands in Order

```bash
# 1. Scope
kubectl get pods -n production
kubectl get deployment payment-api -n production

# 2. Pod detail and events
kubectl describe pod payment-api-7d89c8-x1 -n production

# 3. Logs — both current and previous
kubectl logs payment-api-7d89c8-x1 -n production
kubectl logs payment-api-7d89c8-x1 -n production --previous

# 4. Namespace events
kubectl get events -n production --sort-by='.metadata.creationTimestamp' | tail -30

# 5. Deployment history
kubectl rollout history deployment/payment-api -n production

# 6. Rollback
kubectl rollout undo deployment/payment-api -n production

# 7. Verify recovery
kubectl rollout status deployment/payment-api -n production
kubectl get pods -n production
kubectl get endpoints payment-api -n production
```

---

---

## Q5. Pods Healthy but Users Cannot Access — Traffic Flow and Layer-by-Layer Troubleshooting

**Interviewer's Question:**

> Assume your application Pods are now healthy:
>
> ```
> NAME                    READY   STATUS    RESTARTS
> payment-api-abc123      1/1     Running   0
> payment-api-def456      1/1     Running   0
> ```
>
> However, users still cannot access the application from the internet. The application is running on AKS, and traffic comes through an Ingress/Gateway before reaching the application Service and Pods. Walk me through the complete traffic flow from Internet → Azure → AKS → Ingress/Gateway → Service → Pod. Then explain how you would troubleshoot when Pods are healthy but external users cannot reach the application.

---

### What the Interviewer is Really Testing Here

This question is more advanced than the CrashLoopBackOff question. There, the problem was obvious — Pods were crashing. Here, Pods are perfectly healthy. The interviewer wants to know three things:

1. **Do you have a mental model of the full traffic path?** Can you trace a user's request from their browser all the way to the pod, naming every component on the way? This proves you understand how Kubernetes and Azure networking actually work together — not just individual tools in isolation.

2. **Do you troubleshoot in layers, from outside in?** A junior engineer might randomly check random things and get lucky. A senior engineer works systematically — confirm the outer layer is working, then move one layer inward, until you find where traffic is being dropped.

3. **Do you know what can fail at each layer?** Every hop in the chain is a potential point of failure. The interviewer wants to hear you name those failure modes.

If you can answer this question clearly, it signals that you understand the full stack — Azure networking, Kubernetes internals, Ingress, DNS, TLS — all working together.

---

### Part 1 — The Complete Traffic Flow

Before troubleshooting anything, it helps to draw the full path in your head. Here is exactly what happens when a user opens a browser and types `https://payment.mybank.com`:

```
User Browser (https://payment.mybank.com)
        │
        │  1. DNS Lookup
        ▼
Azure DNS / Public DNS
  "payment.mybank.com → 20.10.5.100"
        │
        │  2. HTTPS request to Public IP
        ▼
Azure Load Balancer  OR  Azure Application Gateway
  (Public IP: 20.10.5.100)
  - Load Balancer: Layer 4, forwards TCP traffic to AKS nodes
  - Application Gateway: Layer 7, reads HTTP headers, routes by host/path,
    can do SSL termination and WAF
  - NSG on this subnet must allow inbound 443 from Internet
        │
        │  3. Traffic forwarded to AKS nodes
        ▼
AKS Node (VM / VMSS instance)
  - Receives traffic on a NodePort (e.g. 31456)
  - NSG on the node subnet must allow traffic from Load Balancer
  - If using AGIC with Azure CNI: App Gateway routes directly to Pod IP
    (no NodePort needed)
        │
        │  4. Traffic hits Ingress Controller pod
        ▼
Ingress Controller Pod  (nginx / AGIC)
  - Runs inside AKS as a Deployment in its own namespace (e.g. ingress-nginx)
  - Exposed via a Kubernetes Service of type LoadBalancer (gets the public IP)
  - Reads Ingress resources to know the routing rules
  - Routes: hostname "payment.mybank.com" + path "/" → service "payment-api" port 8080
  - Handles TLS termination if configured (reads TLS secret)
        │
        │  5. Routed to Kubernetes Service
        ▼
Kubernetes Service  (ClusterIP: "payment-api", port 8080)
  - Internal virtual IP inside the cluster
  - kube-proxy maintains iptables or IPVS rules to forward to pod IPs
  - Uses label selector to pick which pods to send traffic to
  - Only sends traffic to pods that are Ready (readiness probe passing)
  - Has an Endpoints list — if empty, traffic is dropped here
        │
        │  6. Forwarded to application pod
        ▼
Application Pod  (payment-api-abc123)
  - Container receives the request on port 8080
  - Processes and returns response
  - Response travels the same path in reverse back to the user
```

This is the complete path. Each arrow is a potential failure point.

---

### Part 2 — What Can Fail at Each Layer

| Layer | Component | What Can Break |
|---|---|---|
| 1 | DNS | Domain not pointing to correct IP, DNS not propagated, TTL too high |
| 2 | Azure Load Balancer / App Gateway | NSG blocking port 443, public IP unassigned, App Gateway backend showing unhealthy, SSL certificate expired or missing |
| 3 | AKS Nodes | NSG blocking Load Balancer → Node traffic, Node in NotReady state |
| 4 | Ingress Controller | Ingress controller pods not running, no external IP assigned (still Pending), wrong Ingress class annotation, wrong hostname or path in Ingress rule, TLS secret missing or expired |
| 5 | Kubernetes Service | Label selector not matching pod labels, Service port wrong, Endpoints list empty |
| 6 | Pod | Pod is Running but Readiness probe failing (excluded from Endpoints), app listening on wrong port, Network Policy blocking Ingress → Pod traffic |

---

### Part 3 — Troubleshooting: Layer by Layer, Outside In

The correct approach is to start from the outside (user's internet) and work inward, confirming each layer before moving deeper.

**Step 1 — Confirm DNS is resolving correctly**

```bash
nslookup payment.mybank.com
dig payment.mybank.com
```

The IP returned must match the public IP of your Azure Load Balancer or Application Gateway. If it shows an old IP or a different IP, the DNS record is wrong or has not propagated yet.

---

**Step 2 — Check the Ingress Controller service has an external IP**

```bash
kubectl get svc -n ingress-nginx
```

Expected output:
```
NAME                    TYPE           CLUSTER-IP     EXTERNAL-IP    PORT(S)
ingress-nginx-ctrl      LoadBalancer   10.0.100.50    20.10.5.100    80:31800/TCP,443:31801/TCP
```

If `EXTERNAL-IP` shows `<pending>`, Azure has not yet assigned a public IP to the Load Balancer. This could be an Azure quota issue or a delay. The application will be unreachable until this resolves.

---

**Step 3 — Check the Ingress resource rules**

```bash
kubectl get ingress -n production
kubectl describe ingress payment-api-ingress -n production
```

Look at the output of `describe`. It shows the hostname, path, and backend service name and port. Common mistakes:

- `host: payment.mybank.com` is correct but traffic is arriving on `www.payment.mybank.com` — hostname mismatch
- Path is `/api` but user is hitting `/` — path mismatch
- Backend service name is misspelled or in wrong namespace
- `ingressClassName` annotation is wrong or missing — Ingress exists but no controller is picking it up

---

**Step 4 — Check if the Ingress Controller pods are healthy**

```bash
kubectl get pods -n ingress-nginx
kubectl logs -n ingress-nginx deployment/ingress-nginx-controller | tail -50
```

If the Ingress controller pods are not Running, no traffic routing happens at all — all requests will get a connection refused or timeout. The controller logs will also show whether it successfully loaded the Ingress rules for your application.

---

**Step 5 — Check the Service and its Endpoints**

```bash
kubectl get svc payment-api -n production
kubectl get endpoints payment-api -n production
```

The Endpoints output is the most important check at this layer:

```
NAME          ENDPOINTS                     AGE
payment-api   10.244.1.5:8080,10.244.2.3:8080   5d
```

If Endpoints shows `<none>` — the Service has no pods to send traffic to. This means either:
- The Service's label selector does not match the pod labels
- All pods exist but none are passing the Readiness probe (so they are excluded from Endpoints)
- The pods are in a different namespace than the Service

Check the selector:
```bash
kubectl describe svc payment-api -n production
# Look at: Selector field
kubectl get pods -n production --show-labels
# Confirm the pod labels match the selector exactly
```

---

**Step 6 — Check pod Readiness specifically**

This is the key insight: a pod can show `1/1 Running` and still not receive traffic if its Readiness probe is failing. A pod is only added to Service Endpoints when it is both Running AND Ready.

```bash
kubectl get pods -n production
# READY column must show 1/1, not 0/1

kubectl describe pod payment-api-abc123 -n production
# Look for: Readiness probe failed events
# Example: "Readiness probe failed: HTTP probe failed with statuscode: 503"
```

If the Readiness probe is failing, the pod is Running but is intentionally excluded from traffic. Fix the underlying health check issue or temporarily fix the readiness probe path if it is misconfigured.

---

**Step 7 — Test connectivity from inside the cluster**

This is a powerful isolation technique. If I can reach the application from inside the cluster but not from outside, the problem is at the Ingress or Azure layer, not the application itself.

```bash
# Spin up a temporary debug pod inside the cluster
kubectl run debug --image=curlimages/curl --rm -it --restart=Never \
  -- curl -v http://payment-api.production.svc.cluster.local:8080/health
```

- If this **works**: application and Service are fine. Problem is at Ingress, Azure networking, or DNS.
- If this **fails**: problem is at Service or Pod level — check Endpoints and readiness.

---

**Step 8 — Check TLS certificate if HTTPS is being used**

```bash
# If using cert-manager
kubectl get certificate -n production
kubectl describe certificate payment-api-tls -n production
# Status.Conditions should show: Ready = True
# If Ready = False, cert-manager failed to issue/renew the certificate
```

A common production issue: the TLS certificate expires silently. The Ingress controller tries to terminate TLS but has an expired certificate, so HTTPS requests fail with a handshake error even though all pods and services are healthy.

---

**Step 9 — Check Azure Application Gateway backend health (if using AGIC)**

If you are using Azure Application Gateway as the Ingress:

```bash
az network application-gateway show-backend-health \
  --resource-group rg-prod \
  --name agw-prod
```

In the Azure Portal, go to Application Gateway → Backend Health. If it shows Unhealthy, App Gateway's health probe is failing — it is not sending traffic to the AKS nodes even if the nodes are fine. Common cause: the health probe is hitting the wrong path or port, or NSG is blocking the App Gateway probe IP range from reaching the nodes.

---

**Step 10 — Check Network Policies**

```bash
kubectl get networkpolicy -n production
kubectl describe networkpolicy -n production
```

If Network Policies exist in the namespace, they might be blocking traffic from the Ingress controller namespace to the application namespace. A Network Policy that says "only allow traffic from namespace X" will silently drop traffic from the Ingress controller if the Ingress controller is in a different namespace.

---

### Part 4 — Your Full Answer (As Spoken in Interview)

When the interviewer asks this question, here is how to structure your spoken answer:

"First, let me trace the traffic path, because that is the best way to understand where the problem could be.

When a user accesses the application, their request goes through DNS to resolve to our Azure public IP. From there, it hits either our Azure Load Balancer or Application Gateway at the edge, depending on our setup. The Azure Load Balancer forwards TCP traffic to the AKS nodes on a NodePort, where our Ingress Controller pod picks it up. The Ingress Controller reads the Ingress rules and routes to the correct Kubernetes Service based on hostname and path. The Service then forwards to a healthy pod based on its label selector — only pods that are passing the Readiness probe are included in the Service Endpoints.

Since the Pods show Running and Ready, I know the problem is somewhere in the outer layers — Ingress, Azure networking, DNS, or Service configuration.

I would troubleshoot from the outside in.

First, I would verify DNS is pointing to the correct public IP. Then I would check the Ingress controller service to confirm it has an external IP assigned. I would then inspect the Ingress resource itself to verify the hostname, path, and backend service name are correct. I would check the Kubernetes Service Endpoints to confirm it is populated — if Endpoints shows none, no traffic can reach the pods regardless of their health. I would test from inside the cluster using a debug pod to isolate whether the problem is inside or outside the cluster. And for HTTPS, I would check the TLS certificate status.

If we are using Azure Application Gateway as the Ingress, I would also check the backend health in the portal, because App Gateway has its own health probe that is separate from Kubernetes readiness probes."

---

### What Makes This Answer Strong

- Opens with the complete traffic flow — this is the core of what the interviewer is testing
- Explains both Load Balancer and Application Gateway paths — shows awareness of both setups
- The outside-in troubleshooting approach shows senior-level thinking — not random guessing
- `kubectl get endpoints` is the most important command here — many candidates forget it
- The debug pod technique (`kubectl run debug --rm -it`) is a real production skill that impresses interviewers
- TLS certificate check — a silent production killer that candidates often overlook
- Application Gateway backend health — shows Azure-specific depth beyond just Kubernetes

---

## Quick Revision Summary

| Question | Core Points to Remember |
|---|---|
| Self Introduction | 10 yrs → current role → Terraform + AKS + Ansible → troubleshooting background → why this role |
| New Environment End-to-End | Jira → design → modular Terraform → separate state per env → pipeline with approval gate → verify |
| Drift + State Locking | `plan -refresh-only` for drift check → Activity Log → update code or restore → blob lease locking → force-unlock with caution |
| CrashLoopBackOff | `describe pod` → `logs --previous` → exit codes → rollback first → then RCA |
| Pods healthy, users can't access | Full traffic path: DNS → Azure edge → Ingress Controller → Service → Pod → troubleshoot outside-in → check Endpoints first |

---

*Personal mock interview notes — for revision and GitHub reference.*
