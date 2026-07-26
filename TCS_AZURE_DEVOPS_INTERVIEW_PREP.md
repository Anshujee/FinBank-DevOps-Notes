# TCS Azure DevOps Engineer — Complete Interview Preparation Guide

> **Role:** Azure DevOps Engineer — TCS
> **Interview Structure:** 2 Technical Rounds + HR Round
> **Prepared:** July 2026
> **Based on:** Real TCS interview experiences (Glassdoor, AmbitionBox, LinkedIn) + JD analysis

---

## How TCS Interviews This Role

```
Round 1 — Technical Interview 1 (45–60 min)
  → Basics to Intermediate
  → They start from your resume, then expand
  → Azure fundamentals, Kubernetes basics, CI/CD, Terraform
  → Expect: "Tell me about your current project" → then deep dive

Round 2 — Technical Interview 2 (45–60 min)
  → Advanced + Production Scenarios
  → Real troubleshooting questions
  → Istio, mTLS, NSG, AKS internals, GitLab advanced
  → Expect: "What happened when X broke in production?"

Round 3 — HR / MR Round (20–30 min)
  → Salary, notice period, relocation
  → Why TCS, career goals
  → Situational questions
```

---

## Table of Contents

1. [Azure Core Concepts](#1-azure-core-concepts)
2. [Azure Kubernetes Service — AKS](#2-azure-kubernetes-service--aks)
3. [Istio Service Mesh](#3-istio-service-mesh)
4. [Networking — NSG, Load Balancing, Ingress, Egress](#4-networking--nsg-load-balancing-ingress-egress)
5. [TLS and mTLS with SSL Certificates](#5-tls-and-mtls-with-ssl-certificates)
6. [Terraform on Azure — IaC](#6-terraform-on-azure--iac)
7. [Ansible — Configuration Management](#7-ansible--configuration-management)
8. [GitLab CI/CD Pipelines](#8-gitlab-cicd-pipelines)
9. [Python and Bash Scripting](#9-python-and-bash-scripting)
10. [GenAI in CI/CD Pipelines](#10-genai-in-cicd-pipelines)
11. [Real Production Troubleshooting Scenarios](#11-real-production-troubleshooting-scenarios)
12. [HR Round Questions](#12-hr-round-questions)

---

# 1. Azure Core Concepts

---

**Q1. Can you walk me through your current Azure infrastructure setup?**

> This is always the first question. They want to hear you talk naturally about what you have built. Practice this answer until it sounds natural.

**Answer:**
"In my current project, we have a multi-environment Azure setup — dev, staging, and production. The core infrastructure is built using Terraform. We use Azure Kubernetes Service for container orchestration, Azure Container Registry for storing Docker images, Azure Key Vault for secrets management, and Azure SQL Database for relational data storage. All environments are isolated in separate resource groups with their own VNets and NSG rules. The CI/CD pipelines run on GitLab, which builds and pushes images to ACR and then deploys to AKS using Helm charts."

---

**Q2. What is the difference between Azure regions and availability zones?**

**Answer:**
"An Azure region is a specific geographic location where Azure data centres are located — for example, East US, Central India, or Southeast Asia. Each region has multiple data centres.

An Availability Zone is a physically separate data centre within a single region. They have independent power, cooling, and networking. If one zone goes down, the others keep running.

In practice — if I deploy a VM or AKS node pool across 3 availability zones in the same region, even if one entire data centre fails, my application keeps running on the other two zones. This gives me high availability within a region."

---

**Q3. What is a Resource Group in Azure and how do you organize them?**

**Answer:**
"A Resource Group is a logical container that holds related Azure resources — like VMs, VNets, storage accounts, databases — all belonging to the same application or environment.

I organize resource groups by environment and application:
- `rg-myapp-dev` for development resources
- `rg-myapp-staging` for staging
- `rg-myapp-prod` for production

This approach makes it easy to apply different RBAC permissions, cost tracking, and policies per environment. For example, developers get Contributor access to dev resource group but only Reader access to production. It also makes cleanup easy — deleting a resource group deletes everything inside it in one operation."

---

**Q4. What is Azure RBAC and how does it work?**

**Answer:**
"RBAC stands for Role-Based Access Control. It controls who can do what with Azure resources.

It has three parts:
- **Security Principal** — who is being given access (a user, group, or service principal)
- **Role Definition** — what actions they can perform (Owner, Contributor, Reader, or custom roles)
- **Scope** — where the role applies (subscription level, resource group level, or individual resource level)

For example, in our team setup I assign:
- `Owner` to the infrastructure team at subscription level
- `Contributor` to developers at the dev resource group only
- `Reader` to auditors at subscription level
- A custom role with only `Microsoft.ContainerService/managedClusters/read` for a monitoring service that only needs to read AKS cluster status"

---

**Q5. What is a Service Principal in Azure and when do you use it?**

**Answer:**
"A Service Principal is like a user account but for applications and automation tools — not for humans. It is an identity that applications, services, and scripts use to authenticate with Azure.

I use Service Principals in:
- **Terraform** — when running `terraform apply` in GitLab CI/CD, Terraform authenticates using a Service Principal with Contributor access to the resource group
- **GitLab pipelines** — the pipeline uses a Service Principal stored as a CI/CD variable to push images to ACR or deploy to AKS
- **AKS** — the cluster itself uses a Service Principal (or Managed Identity, which is newer) to create load balancers, attach disks, etc.

The best practice today is to use **Managed Identity** instead of Service Principal where possible, because Managed Identity does not require managing credentials at all — Azure handles the token rotation automatically."

---

**Q6. What is Azure Managed Identity and how is it different from Service Principal?**

**Answer:**
"Both Managed Identity and Service Principal are used for application authentication, but Managed Identity is simpler and safer because:

| Feature | Service Principal | Managed Identity |
|---|---|---|
| Credential management | You manage client secrets/certs | Azure manages automatically |
| Secret rotation | Manual or automated | Automatic by Azure |
| Where it can be used | Anywhere | Only on Azure resources |
| Best for | External tools like Terraform, GitLab | Azure-hosted apps like AKS pods, VMs |

In practice, for an AKS pod that needs to read from Azure Key Vault, I use **Workload Identity** (which is built on Managed Identity). The pod gets a Kubernetes ServiceAccount linked to a Managed Identity, and it can call Key Vault without any stored credentials. This is exactly like AWS IRSA but on Azure."

---

# 2. Azure Kubernetes Service — AKS

---

**Q7. What is AKS and how does it differ from managing Kubernetes yourself?**

**Answer:**
"AKS is Azure's managed Kubernetes service. The key difference is that Azure manages the control plane — the API server, etcd, scheduler, and controller manager — for free. You only pay for the worker nodes (VMs).

When you manage Kubernetes yourself on VMs, you have to:
- Install and configure all control plane components
- Handle etcd backups
- Manage TLS certificates and rotate them
- Upgrade the control plane manually

With AKS, I just run `az aks upgrade --kubernetes-version 1.31` and Azure handles the rolling upgrade of control plane components. I still manage the worker node pools and the applications running on them."

---

**Q8. What are node pools in AKS and why would you use multiple node pools?**

**Answer:**
"A node pool is a group of VMs (nodes) with the same configuration inside an AKS cluster. You can have multiple node pools with different VM sizes, OS types, or configurations.

I use multiple node pools for:
- **System node pool** — runs AKS system components like CoreDNS, kube-proxy. Uses smaller VMs like Standard_D2s_v3.
- **Application node pool** — runs my application pods. Uses Standard_D4s_v3 or larger.
- **GPU node pool** — for ML workloads, uses NC-series VMs with NVIDIA GPUs.
- **Spot node pool** — uses Azure Spot VMs (70–80% cheaper) for non-critical batch jobs.

I use Kubernetes `nodeSelector` or `taints and tolerations` to make sure each pod lands on the right node pool. For example, GPU workloads get a taint `gpu=true:NoSchedule` on the GPU pool, and only pods with `tolerations: gpu=true` will schedule there."

---

**Q9. What networking options does AKS support and which did you use?**

**Answer:**
"AKS supports three main networking plugins:

**1. Kubenet (basic)**
- Nodes get VNet IPs, pods get IPs from a separate private range
- NAT is used for pod communication — simpler but limited scalability
- Good for small clusters or dev environments

**2. Azure CNI**
- Every pod gets a real VNet IP directly — no NAT
- Pods are directly reachable from other Azure services
- Needs more IP address planning because pods consume VNet IPs
- Good for production where pods need direct connectivity to Azure services like SQL, Redis

**3. Azure CNI Overlay (newest, 2024)**
- Pods get IPs from a separate overlay network, not from the VNet
- Saves VNet IP space (the big limitation of Azure CNI)
- Best of both worlds — direct connectivity without consuming VNet IPs

In my projects, I use **Azure CNI** for production because our pods need direct connectivity to Azure SQL and Redis. For dev, we use **kubenet** to save VNet IP space."

---

**Q10. How does AKS handle autoscaling?**

**Answer:**
"AKS has two levels of autoscaling:

**Horizontal Pod Autoscaler (HPA) — scales pods:**
HPA monitors CPU or memory metrics and adds/removes pod replicas.
```yaml
apiVersion: autoscaling/v2
kind: HorizontalPodAutoscaler
metadata:
  name: my-app-hpa
spec:
  scaleTargetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: my-app
  minReplicas: 2
  maxReplicas: 10
  metrics:
  - type: Resource
    resource:
      name: cpu
      target:
        type: Utilization
        averageUtilization: 70
```
When CPU goes above 70%, HPA adds pods. When it drops below, HPA removes pods.

**Cluster Autoscaler — scales nodes:**
When pods are pending because no node has enough resources, the Cluster Autoscaler adds a new node to the node pool. When nodes are underutilized, it removes them.

I enable it like this in Terraform:
```hcl
default_node_pool {
  enable_auto_scaling = true
  min_count           = 2
  max_count           = 10
}
```

**The flow:** More users → pods need more CPU → HPA adds pods → no node capacity → Cluster Autoscaler adds a node → pods schedule → traffic handled."

---

**Q11. How do you upgrade an AKS cluster safely without downtime?**

**Answer:**
"I follow this process for zero-downtime upgrades:

**Step 1 — Check available versions:**
```bash
az aks get-upgrades --resource-group rg-prod --name aks-prod --output table
```

**Step 2 — Upgrade control plane first:**
```bash
az aks upgrade --resource-group rg-prod --name aks-prod --kubernetes-version 1.31 --control-plane-only
```
This upgrades the API server without touching nodes.

**Step 3 — Upgrade node pools one at a time (cordon and drain):**
```bash
az aks nodepool upgrade --resource-group rg-prod --cluster-name aks-prod \
  --name nodepool1 --kubernetes-version 1.31
```
AKS cordon the old node (no new pods), drain it (move existing pods to other nodes), delete it, and create a new upgraded node. It does one node at a time — `max-surge` controls how many extra nodes are created during upgrade.

**Step 4 — Verify:**
```bash
kubectl get nodes
kubectl get pods --all-namespaces
```

**Key safety rules I follow:**
- Always upgrade dev → staging → prod, never jump directly to prod
- Make sure PodDisruptionBudgets are set so deployments allow graceful draining
- Do not upgrade more than one minor version at a time (e.g. 1.29 → 1.30, not 1.29 → 1.31)"

---

**Q12. How do you handle secrets in AKS? What is the Azure Key Vault integration?**

**Answer:**
"I never store secrets in Kubernetes YAML files or environment variables directly in the deployment spec. The approaches I use:

**Option 1 — Azure Key Vault with Secrets Store CSI Driver:**
The Secrets Store CSI Driver is an add-on that mounts Key Vault secrets directly as files or environment variables into pods.

```yaml
# SecretProviderClass — tells CSI driver which Key Vault to read
apiVersion: secrets-store.csi.x-k8s.io/v1
kind: SecretProviderClass
metadata:
  name: my-app-secrets
spec:
  provider: azure
  parameters:
    keyvaultName: kv-myapp-prod
    objects: |
      array:
        - objectName: db-password
          objectType: secret
```
The pod mounts this and gets the secret as an environment variable. The pod never talks to Key Vault directly — the CSI driver handles it using Workload Identity.

**Option 2 — External Secrets Operator (ESO):**
ESO runs as a controller in the cluster, reads from Key Vault, and creates native Kubernetes Secret objects. Pods then use regular Kubernetes Secrets without knowing about Key Vault.

I prefer ESO because it gives you a real Kubernetes Secret that tools like Helm and standard deployments can use without modification."

---

**Q13. Production scenario: Pods are stuck in Pending state. How do you debug?**

**Answer:**
"This is one of the most common production issues. I follow this exact process:

**Step 1 — Check the pod events:**
```bash
kubectl describe pod <pod-name> -n <namespace>
```
Look at the Events section at the bottom. Common messages:
- `0/3 nodes are available: insufficient memory` → node doesn't have enough RAM
- `0/3 nodes are available: node selector does not match` → wrong nodeSelector label
- `0/3 nodes are available: taints not tolerated` → pod missing a toleration
- `no persistent volume found for PVC` → storage not provisioned

**Step 2 — If it's a resource issue:**
```bash
kubectl top nodes   # Check node CPU/memory usage
kubectl get nodes   # Check if nodes are Ready
```
If nodes are full, check if Cluster Autoscaler is enabled and working:
```bash
kubectl get events -n kube-system | grep cluster-autoscaler
```

**Step 3 — If it's a scheduling issue:**
```bash
kubectl get events --sort-by='.metadata.creationTimestamp' -n <namespace>
```

**Step 4 — Common fixes:**
- Resource issue → increase node pool max count or add bigger VM SKU
- Taint/toleration issue → add correct toleration to pod spec
- PVC issue → check StorageClass exists and is the default one
- Image pull issue → check ACR credentials are valid, pod has image pull secret

In production, I also check if the issue happened after a recent deployment — usually a new deployment requests more resources than the old one."

---

# 3. Istio Service Mesh

---

**Q14. What is Istio and why would you use it in a Kubernetes cluster?**

**Answer:**
"Istio is a service mesh — it is a layer of infrastructure that sits between your microservices and handles all the network communication between them, without you changing any application code.

Without Istio, if I have 10 microservices talking to each other, I have to implement retry logic, timeouts, circuit breakers, encryption, and access control in each service's code separately. This is a huge amount of repeated work.

With Istio, I add a sidecar proxy called Envoy to every pod. This proxy intercepts all network traffic going in and out of the pod. Istio's control plane (called Istiod) manages all these proxies centrally.

What Istio gives me:
- **mTLS** — automatic encryption between every service-to-service call
- **Traffic management** — route 10% of traffic to new version, 90% to old (canary deployments)
- **Retries and circuit breaking** — automatically retry failed requests
- **Observability** — see exactly which service is calling which, and how fast
- **Access control** — allow only Service A to call Service B, block everything else

In a banking project where security is critical, Istio's automatic mTLS means no service can talk to another without a valid certificate — even inside the cluster."

---

**Q15. How does Istio inject the sidecar proxy into pods?**

**Answer:**
"Istio uses Kubernetes admission webhooks for automatic sidecar injection. When a pod is created in a namespace that has the label `istio-injection=enabled`, Kubernetes calls Istio's mutating admission webhook before actually creating the pod.

The webhook modifies the pod spec to add:
1. An `initContainer` called `istio-init` that sets up iptables rules to redirect all traffic through Envoy
2. A sidecar container called `istio-proxy` (the Envoy proxy)

```bash
# Enable automatic injection for a namespace
kubectl label namespace my-app istio-injection=enabled

# Verify sidecar is injected — pod should have 2 containers
kubectl get pods -n my-app
# NAME                    READY   STATUS    
# my-app-7d9f8-xvzp2     2/2     Running   ← 2/2 means app + Envoy sidecar
```

If I need to exclude a specific pod from injection (like a batch job or database client), I add:
```yaml
annotations:
  sidecar.istio.io/inject: "false"
```"

---

**Q16. What is mTLS and how does Istio implement it?**

**Answer:**
"TLS (regular) means only the server proves its identity to the client. Your browser trusts a website because the website shows a certificate.

mTLS (Mutual TLS) means BOTH sides authenticate each other. The client also shows a certificate to the server. This is critical for service-to-service communication inside a cluster — you want to be sure that when Service A calls Service B, Service B is actually Service B and not an attacker.

**How Istio does mTLS automatically:**

Each pod's Envoy sidecar is given a SPIFFE certificate by Istiod (the Istio control plane). SPIFFE is a standard for service identity — the certificate says 'I am service A in namespace prod.'

When Service A calls Service B:
1. Service A's Envoy says: 'Here is my certificate proving I am Service A'
2. Service B's Envoy says: 'Here is my certificate proving I am Service B'
3. Both verify each other's certificates against the Istio CA (Certificate Authority)
4. Only if both are verified does the actual request go through

**Configuring mTLS mode:**
```yaml
# STRICT mode — all traffic must use mTLS, plaintext rejected
apiVersion: security.istio.io/v1beta1
kind: PeerAuthentication
metadata:
  name: default
  namespace: my-app
spec:
  mtls:
    mode: STRICT
```

```yaml
# PERMISSIVE mode — accepts both mTLS and plaintext (for migration)
spec:
  mtls:
    mode: PERMISSIVE
```

I use STRICT mode in production and PERMISSIVE during migration when some old services are not yet on Istio."

---

**Q17. What is an Istio VirtualService and DestinationRule? Explain with an example.**

**Answer:**
"These are the two main Istio traffic management resources.

**VirtualService** — defines routing rules. 'When a request comes in, where should it go?'

**DestinationRule** — defines what happens at the destination. 'Once traffic arrives at a service, how should it be treated?'

**Real example — canary deployment:**

I have version v1 running in production. I want to test v2 with 10% of traffic first.

```yaml
# DestinationRule — defines two subsets (v1 and v2)
apiVersion: networking.istio.io/v1alpha3
kind: DestinationRule
metadata:
  name: my-app
spec:
  host: my-app    # the Kubernetes service name
  subsets:
  - name: v1
    labels:
      version: v1
  - name: v2
    labels:
      version: v2
```

```yaml
# VirtualService — route 90% to v1, 10% to v2
apiVersion: networking.istio.io/v1alpha3
kind: VirtualService
metadata:
  name: my-app
spec:
  hosts:
  - my-app
  http:
  - route:
    - destination:
        host: my-app
        subset: v1
      weight: 90
    - destination:
        host: my-app
        subset: v2
      weight: 10
```

When v2 is proven stable (error rate stays low in Grafana/Kiali), I shift weight to 0% v1 and 100% v2. If v2 has issues, I shift back to 100% v1 in seconds. No downtime, no risky deployments."

---

**Q18. What is an Istio Gateway? How is it different from a Kubernetes Ingress?**

**Answer:**
"Both manage traffic coming into the cluster from outside, but they work at different levels.

**Kubernetes Ingress:**
- Works with any Ingress controller (nginx, traefik, Azure App Gateway)
- Simple HTTP/HTTPS routing based on host and path
- Limited to Layer 7 HTTP routing rules
- No built-in mTLS for incoming traffic

**Istio Gateway:**
- Works with Istio's Ingress Gateway (an Envoy proxy at the edge)
- Controls which ports and protocols the gateway accepts
- Then you use a VirtualService to route to specific services
- Full Istio features — advanced routing, retries, circuit breaking, observability

```yaml
# Istio Gateway — opens port 443 on the ingress gateway
apiVersion: networking.istio.io/v1alpha3
kind: Gateway
metadata:
  name: my-app-gateway
spec:
  selector:
    istio: ingressgateway
  servers:
  - port:
      number: 443
      name: https
      protocol: HTTPS
    tls:
      mode: SIMPLE    # Simple TLS — server proves its identity
      credentialName: my-app-tls-secret   # Kubernetes secret with cert and key
    hosts:
    - myapp.example.com
```

```yaml
# VirtualService — routes traffic from Gateway to the service
apiVersion: networking.istio.io/v1alpha3
kind: VirtualService
metadata:
  name: my-app
spec:
  hosts:
  - myapp.example.com
  gateways:
  - my-app-gateway
  http:
  - route:
    - destination:
        host: my-app
        port:
          number: 8080
```"

---

**Q19. What is an Istio ServiceEntry and when do you use it?**

**Answer:**
"A ServiceEntry is used to register external services (services outside the mesh) with Istio so that traffic to them can be managed, monitored, and secured.

By default, Istio blocks or ignores all egress traffic to external hosts. If my pod tries to call an external API like `api.stripe.com`, Istio's Envoy sidecar can intercept it.

**Why I use ServiceEntry:**
1. To allow traffic to specific external endpoints (allowlist external services)
2. To apply Istio policies (retries, timeouts, circuit breaking) to external service calls
3. To see external service calls in the Istio observability tools (Kiali, Jaeger)

```yaml
# Allow traffic to an external payment API
apiVersion: networking.istio.io/v1alpha3
kind: ServiceEntry
metadata:
  name: payment-api
spec:
  hosts:
  - api.paymentprovider.com
  ports:
  - number: 443
    name: https
    protocol: HTTPS
  resolution: DNS
  location: MESH_EXTERNAL
```

With egress in REGISTRY_ONLY mode (strict), only services registered via ServiceEntry are reachable. This is a strong security control — it prevents any pod from calling arbitrary external URLs."

---

**Q20. Production scenario: mTLS is enabled but two services cannot communicate. How do you debug?**

**Answer:**
"I have hit this exact issue in production. Here is my step-by-step debugging process:

**Step 1 — Check if both pods have Istio sidecars:**
```bash
kubectl get pods -n my-app
# Should show 2/2 READY — if one shows 1/1, it has no sidecar and mTLS will fail
```

**Step 2 — Check PeerAuthentication policy:**
```bash
kubectl get peerauthentication -n my-app
kubectl describe peerauthentication default -n my-app
```
If STRICT mode is set but one service has no sidecar, requests are rejected.

**Step 3 — Check DestinationRule for traffic policy:**
```bash
kubectl get destinationrule -n my-app -o yaml
```
Look for `trafficPolicy.tls.mode`. If it says `DISABLE` but PeerAuthentication requires mTLS, they conflict.

**Step 4 — Use istioctl for proxy analysis:**
```bash
istioctl analyze -n my-app
# This tool automatically detects Istio configuration problems
```

**Step 5 — Check Envoy proxy logs on the failing pod:**
```bash
kubectl logs <pod-name> -c istio-proxy -n my-app | grep -i error
```

**Common root causes I have found:**
- New deployment forgot to enable istio-injection label on namespace
- DestinationRule had `mode: DISABLE` while PeerAuthentication had `mode: STRICT`
- Certificate expired — run `istioctl proxy-status` to see cert expiry
- Wrong namespace — PeerAuthentication applied to wrong namespace

**Fix:** Usually it is either missing sidecar injection or conflicting TLS mode settings between PeerAuthentication and DestinationRule."

---

# 4. Networking — NSG, Load Balancing, Ingress, Egress

---

**Q21. What is a Network Security Group (NSG) in Azure?**

**Answer:**
"An NSG is a firewall for Azure virtual networks. It contains a list of rules — each rule says allow or deny traffic based on source IP, destination IP, port, and protocol.

NSGs can be applied at two levels:
- **Subnet level** — affects all resources inside that subnet
- **NIC level** — affects a specific VM's network interface

Each rule has a priority number (100–4096). Lower number = higher priority. Rules are evaluated in order from lowest to highest priority. The first matching rule wins.

**Example NSG rules I set up for AKS:**
```
Priority 100 — Allow HTTPS (443) from Internet to AKS nodes → Allow
Priority 200 — Allow HTTP (80) from Internet to AKS nodes  → Allow
Priority 300 — Allow port 6443 from my VPN IP to AKS API server → Allow
Priority 400 — Deny all inbound from Internet → Deny

Priority 100 — Allow all outbound to Internet → Allow (for pulling images)
```

**Important:** Azure always has hidden default rules at priority 65000+ that allow intra-VNet traffic and deny all internet traffic by default. Your explicit rules override these."

---

**Q22. What is the difference between Azure Load Balancer, Application Gateway, and Azure Front Door?**

**Answer:**
"These are all different tools for distributing traffic, but they work at different layers and for different use cases.

| Feature | Load Balancer | Application Gateway | Azure Front Door |
|---|---|---|---|
| OSI Layer | Layer 4 (TCP/UDP) | Layer 7 (HTTP/HTTPS) | Layer 7 (Global) |
| Routing | IP + Port based | URL path, hostname based | Global, multi-region |
| SSL Termination | No | Yes | Yes |
| WAF (Web Firewall) | No | Yes | Yes |
| Geographic distribution | No | No | Yes (global) |
| Use case | Non-HTTP workloads, internal | Web apps within a region | Global web apps |

**When I use each one:**
- **Azure Load Balancer** — for TCP traffic like databases inside a VNet, or when I need simple IP-based load balancing at high speed
- **Application Gateway v2** — for HTTP/HTTPS web applications within a region, especially when I need WAF, URL-based routing, or SSL offloading. AKS uses this as the AGIC (Application Gateway Ingress Controller)
- **Azure Front Door** — for globally distributed applications where users are in multiple countries and I want them routed to the nearest region automatically

**In AKS context:** When I create a Kubernetes Service of type `LoadBalancer`, Azure automatically provisions an Azure Load Balancer. When I use AGIC (Application Gateway Ingress Controller), my Kubernetes Ingress resources are translated into Application Gateway routing rules."

---

**Q23. Explain Kubernetes Ingress vs Egress in the context of your project.**

**Answer:**
"In Kubernetes:

**Ingress** means traffic coming INTO the cluster from outside — users hitting your application from their browser.

**Egress** means traffic going OUT of the cluster — your pods calling external APIs, databases, or services.

**Ingress in my setup:**
I use an Ingress Controller (either nginx or Azure Application Gateway Ingress Controller). The Ingress resource defines routing rules:
```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: my-app-ingress
  annotations:
    kubernetes.io/ingress.class: nginx
    nginx.ingress.kubernetes.io/ssl-redirect: 'true'
spec:
  tls:
  - hosts:
    - myapp.example.com
    secretName: my-app-tls
  rules:
  - host: myapp.example.com
    http:
      paths:
      - path: /api
        pathType: Prefix
        backend:
          service:
            name: backend-service
            port:
              number: 8080
      - path: /
        pathType: Prefix
        backend:
          service:
            name: frontend-service
            port:
              number: 80
```

**Egress in my setup:**
When pods call external services (Azure SQL, external APIs), the traffic goes through the node's NAT or through Azure Firewall if configured.

For Istio egress, I configure an Egress Gateway + ServiceEntry:
- Egress Gateway = dedicated Istio proxy that handles all outbound traffic
- ServiceEntry = registers which external hosts are allowed
- All external calls from pods go through the Egress Gateway, where I can log, monitor, and apply policies"

---

**Q24. How do you configure TLS termination for an Ingress in AKS?**

**Answer:**
"There are two ways I use TLS with Kubernetes Ingress:

**Option 1 — Manual TLS secret:**
```bash
# Create a Kubernetes secret with the certificate and key
kubectl create secret tls my-app-tls \
  --cert=path/to/fullchain.pem \
  --key=path/to/privkey.pem \
  -n my-app
```
Then reference it in the Ingress:
```yaml
spec:
  tls:
  - hosts:
    - myapp.example.com
    secretName: my-app-tls
```

**Option 2 — cert-manager (automated, this is what I use in production):**
cert-manager is a Kubernetes add-on that automatically requests, renews, and manages TLS certificates from Let's Encrypt or other Certificate Authorities.

```yaml
# cert-manager ClusterIssuer — tells cert-manager to use Let's Encrypt
apiVersion: cert-manager.io/v1
kind: ClusterIssuer
metadata:
  name: letsencrypt-prod
spec:
  acme:
    server: https://acme-v02.api.letsencrypt.org/directory
    email: admin@mycompany.com
    privateKeySecretRef:
      name: letsencrypt-prod
    solvers:
    - http01:
        ingress:
          class: nginx
```

```yaml
# Ingress with cert-manager annotation — automatically gets certificate
metadata:
  annotations:
    cert-manager.io/cluster-issuer: letsencrypt-prod
spec:
  tls:
  - hosts:
    - myapp.example.com
    secretName: my-app-tls   # cert-manager creates this automatically
```

cert-manager monitors certificate expiry and renews them 30 days before they expire. I never have to manually renew a certificate."

---

# 5. TLS and mTLS with SSL Certificates

---

**Q25. Explain the difference between TLS and mTLS in simple terms.**

**Answer:**
"Think of it like identity verification at a building entrance.

**TLS (regular)** — like a bank branch:
The bank proves its identity to you (shows its name on the door, has a certificate). You as a customer don't need to prove who you are to enter the building.

This is how your browser works — the website shows an SSL certificate. You trust the website. But the website doesn't verify who you are at the TLS level (login is handled separately at the application layer).

**mTLS (Mutual TLS)** — like a high-security government building:
You need to show your ID to the security guard. The security guard also shows you their ID badge. Both sides verify each other before anyone enters.

**In my microservices:**
- Service A calls Service B using mTLS
- Service A shows its certificate: 'I am service-a in namespace prod'
- Service B shows its certificate: 'I am service-b in namespace prod'
- Both verify through the same CA (Istio's CA in our case)
- If Service A's certificate is expired or invalid, Service B rejects the call immediately

This means even if an attacker gets inside the Kubernetes cluster and creates a rogue pod, that pod cannot call Service B because it doesn't have a valid certificate from our CA."

---

**Q26. How do you rotate SSL certificates in a production environment without downtime?**

**Answer:**
"This depends on where the certificate is:

**For Ingress/Load Balancer certificates (cert-manager):**
cert-manager handles this automatically. It monitors expiry and renews 30 days before expiration. The new certificate replaces the old one in the Kubernetes Secret. The Ingress Controller reloads the certificate with zero downtime.

**For manually managed certificates:**
```bash
# Step 1 — Generate new certificate (or get from CA)
# Step 2 — Update the Kubernetes secret with the new cert
kubectl create secret tls my-app-tls \
  --cert=new-fullchain.pem \
  --key=new-privkey.pem \
  --dry-run=client -o yaml | kubectl apply -f -
# The --dry-run + apply trick updates existing secret without deleting it first

# Step 3 — Verify the nginx ingress controller picked it up
kubectl logs -n ingress-nginx deployment/ingress-nginx-controller | tail -20
```

**For Istio service-to-service mTLS certificates:**
Istio manages these automatically. It issues certificates with 24-hour expiry and rotates them automatically without any intervention needed.

**For Azure Application Gateway:**
```bash
# Update cert in Key Vault
az keyvault certificate import --vault-name kv-prod --name my-cert --file new-cert.pfx

# Application Gateway reads from Key Vault — update happens on next sync
az network application-gateway ssl-cert update \
  --resource-group rg-prod \
  --gateway-name agw-prod \
  --name my-cert \
  --key-vault-secret-id <new-secret-id>
```
The Application Gateway does a graceful reload — existing connections are not dropped."

---

# 6. Terraform on Azure — IaC

---

**Q27. How do you manage Terraform state for Azure infrastructure in a team?**

**Answer:**
"This is critical to get right because if two people run `terraform apply` at the same time on the same state file, you can corrupt your infrastructure.

**I use Azure Blob Storage as the remote backend:**
```hcl
terraform {
  backend "azurerm" {
    resource_group_name  = "rg-terraform-state"
    storage_account_name = "stterraformstate001"
    container_name       = "tfstate"
    key                  = "prod/aks/terraform.tfstate"
  }
}
```

This gives me:
- **Remote storage** — state is in Azure Blob, not on anyone's laptop
- **State locking** — Azure Blob uses blob leases as a locking mechanism. If one pipeline is running, another pipeline trying to acquire the lock will wait or fail safely
- **Versioning** — I enable versioning on the storage account so I can roll back to a previous state if something goes wrong
- **Encryption** — Azure encrypts the state at rest automatically

**State file organization:**
I use separate state files per environment and per module:
```
tfstate/
  dev/networking/terraform.tfstate
  dev/aks/terraform.tfstate
  dev/databases/terraform.tfstate
  prod/networking/terraform.tfstate
  prod/aks/terraform.tfstate
  prod/databases/terraform.tfstate
```
This way, a change to AKS does not risk affecting the networking state."

---

**Q28. What is `terraform import` and when do you use it?**

**Answer:**
"`terraform import` is used when you have existing Azure resources that were NOT created by Terraform and you want to bring them under Terraform management.

**Real scenario where I used this:**
Our team manually created an AKS cluster directly in Azure Portal as a quick proof of concept. Later, we decided to manage everything with Terraform. But if I run `terraform apply` with the AKS resource, Terraform will try to create a new cluster and fail because the name is already taken.

**Solution — terraform import:**
```bash
# Step 1 — Write the Terraform resource block first
resource "azurerm_kubernetes_cluster" "main" {
  name                = "aks-prod"
  resource_group_name = "rg-prod"
  location            = "East US"
  # ... other config
}

# Step 2 — Import the existing resource into Terraform state
terraform import azurerm_kubernetes_cluster.main \
  /subscriptions/6a6cb5d4/resourceGroups/rg-prod/providers/Microsoft.ContainerService/managedClusters/aks-prod

# Step 3 — Run terraform plan — it will show differences between your HCL and real state
terraform plan
# Adjust your HCL until plan shows "No changes"
```

**When NOT to use terraform import:**
- Do not import resources you plan to delete soon
- Do not import resources that have complex configuration that is hard to replicate in HCL
- For large environments, use `terraformer` tool which can auto-generate HCL from existing Azure resources"

---

**Q29. Production scenario: Terraform apply fails midway. The state file is partially updated. What do you do?**

**Answer:**
"This has happened to me and it is stressful but manageable.

**Step 1 — Do NOT run terraform apply again immediately.**
A partial apply means some resources were created, some were not. Running apply again might try to create duplicates.

**Step 2 — Run terraform plan to understand the current state:**
```bash
terraform plan
```
This shows what Terraform thinks exists (in state) vs what actually exists in Azure.

**Step 3 — For resources that partially created but are in broken state:**
```bash
terraform refresh
# This syncs the state with real Azure (reads current reality)
terraform plan
# Now plan shows the true delta
```

**Step 4 — If a resource is in state but not in Azure (apply failed after writing to state but before creating in Azure):**
```bash
# Remove the resource from state
terraform state rm azurerm_kubernetes_cluster.main

# Then re-run apply — Terraform will create it fresh
terraform apply
```

**Step 5 — If a resource is in Azure but not in state (apply failed before writing to state):**
```bash
terraform import azurerm_kubernetes_cluster.main <azure-resource-id>
```

**Prevention:**
- Always use `-target` for risky changes to limit blast radius
- Enable soft-delete on the state storage account so you can restore previous state
- Use `terraform plan -out=plan.tfplan` and `terraform apply plan.tfplan` so the apply executes exactly what was planned"

---

**Q30. What is the difference between `count` and `for_each` in Terraform?**

**Answer:**
"Both are used to create multiple instances of a resource, but they work differently.

**`count` — creates resources by number:**
```hcl
resource "azurerm_resource_group" "rg" {
  count    = 3
  name     = "rg-app-${count.index}"
  location = "East US"
}
# Creates: rg-app-0, rg-app-1, rg-app-2
```
Problem with count: resources are indexed by number. If you delete `rg-app-1`, Terraform renumbers — `rg-app-2` becomes index 1 and Terraform destroys and recreates it.

**`for_each` — creates resources by unique key:**
```hcl
resource "azurerm_resource_group" "rg" {
  for_each = toset(["dev", "staging", "prod"])
  name     = "rg-app-${each.key}"
  location = "East US"
}
# Creates: rg-app-dev, rg-app-staging, rg-app-prod
```
With for_each, each resource has a stable key. Deleting 'staging' only deletes rg-app-staging. The other two are not touched.

**When to use which:**
- Use `count` when creating N identical resources (e.g. 3 identical VMs)
- Use `for_each` when creating resources with different configurations identified by a unique key (environments, regions, etc.)
- In practice I use `for_each` almost exclusively because it is safer"

---

# 7. Ansible — Configuration Management

---

**Q31. What is Ansible and how is it different from Terraform?**

**Answer:**
"Both are infrastructure automation tools but they do different things:

**Terraform** — creates and manages infrastructure (VMs, VNets, AKS clusters, databases). It is for provisioning — making resources exist.

**Ansible** — configures existing infrastructure (installs software, manages files, configures services on VMs that already exist). It is for configuration management.

A simple way to think about it:
- Terraform builds the house (creates the VM)
- Ansible furnishes and configures the house (installs nginx, sets up configs, creates users)

Ansible is agentless — it connects to servers over SSH (or WinRM for Windows) and pushes configuration. No agent software needs to be installed on the target servers.

**I use both together:**
```
Terraform creates VM →
Ansible playbook runs on the VM →
configures nginx, installs app dependencies, sets environment variables →
App is ready to serve traffic
```"

---

**Q32. Write an Ansible playbook to install and configure nginx on multiple servers.**

**Answer:**
```yaml
# install-nginx.yml
---
- name: Install and configure nginx
  hosts: webservers          # Group of servers from inventory
  become: true               # Run as root (sudo)
  vars:
    nginx_port: 80
    server_name: myapp.example.com

  tasks:
    - name: Update apt cache
      apt:
        update_cache: yes
        cache_valid_time: 3600    # Only update if cache is older than 1 hour

    - name: Install nginx
      apt:
        name: nginx
        state: present            # present = install if not there

    - name: Copy nginx config file
      template:
        src: templates/nginx.conf.j2    # Jinja2 template with variables
        dest: /etc/nginx/sites-available/myapp
        owner: root
        group: root
        mode: '0644'
      notify: Restart nginx             # Triggers handler if this task changes anything

    - name: Enable the site
      file:
        src: /etc/nginx/sites-available/myapp
        dest: /etc/nginx/sites-enabled/myapp
        state: link

    - name: Ensure nginx is started and enabled
      service:
        name: nginx
        state: started
        enabled: yes              # Start on boot

  handlers:
    - name: Restart nginx
      service:
        name: nginx
        state: restarted
        # Handler only runs if a task that notifies it actually made a change
        # This prevents unnecessary restarts
```

```
# Run the playbook:
ansible-playbook -i inventory/production install-nginx.yml

# Dry run — show what would change without making changes:
ansible-playbook -i inventory/production install-nginx.yml --check
```"

---

**Q33. What are Ansible roles and why do you use them?**

**Answer:**
"An Ansible role is a way to organize a playbook into a reusable, structured directory. Instead of having one huge playbook file, a role breaks it into separate files for tasks, variables, templates, and handlers.

```
roles/
  nginx/
    tasks/
      main.yml       # All tasks for nginx
    handlers/
      main.yml       # Handlers (restart nginx etc.)
    templates/
      nginx.conf.j2  # Jinja2 template files
    vars/
      main.yml       # Role-specific variables
    defaults/
      main.yml       # Default values (can be overridden)
    files/
      index.html     # Static files to copy
```

**Why I use roles:**
- Reusability — I write the nginx role once and use it across 10 different playbooks
- Team collaboration — different engineers can work on different roles
- Ansible Galaxy — I can download community roles instead of writing from scratch

```yaml
# Using roles in a playbook
- name: Configure web servers
  hosts: webservers
  roles:
    - nginx           # Uses roles/nginx/
    - certbot         # Uses roles/certbot/
    - app-config      # Uses roles/app-config/
```"

---

# 8. GitLab CI/CD Pipelines

---

**Q34. Explain the structure of a GitLab CI/CD pipeline. How is it different from Jenkins?**

**Answer:**
"A GitLab CI/CD pipeline is defined in a file called `.gitlab-ci.yml` at the root of the repository. GitLab reads this file and runs the pipeline automatically on push, merge request, or schedule.

**Basic structure:**
```yaml
stages:
  - build
  - test
  - security
  - push
  - deploy

variables:
  IMAGE_NAME: $CI_REGISTRY_IMAGE/my-app
  IMAGE_TAG: $CI_COMMIT_SHORT_SHA    # GitLab built-in variable

build:
  stage: build
  script:
    - docker build -t $IMAGE_NAME:$IMAGE_TAG .
  only:
    - develop
    - main

test:
  stage: test
  script:
    - mvn test
  artifacts:
    reports:
      junit: target/surefire-reports/*.xml

trivy-scan:
  stage: security
  script:
    - trivy image --exit-code 1 --severity HIGH,CRITICAL $IMAGE_NAME:$IMAGE_TAG

push-to-acr:
  stage: push
  script:
    - docker login $CI_REGISTRY -u $CI_REGISTRY_USER -p $CI_REGISTRY_PASSWORD
    - docker push $IMAGE_NAME:$IMAGE_TAG
  only:
    - main

deploy-to-aks:
  stage: deploy
  script:
    - az aks get-credentials --resource-group rg-prod --name aks-prod
    - helm upgrade --install my-app ./helm/my-app
        --set image.tag=$IMAGE_TAG
        --namespace production
  environment:
    name: production
  only:
    - main
  when: manual    # Requires manual approval for production deploy
```

**How GitLab differs from Jenkins:**
- GitLab CI is built into GitLab — no separate server to install and maintain
- Configuration is in the repository as code — not in a separate Jenkins UI
- GitLab uses `Runners` (lightweight agents) that execute jobs — can be shared or project-specific
- GitLab has built-in secret management (CI/CD Variables with masking)
- GitLab has Environments feature for tracking deployments per environment
- Jenkins is more flexible with plugins but requires more maintenance"

---

**Q35. What is a GitLab Runner and what types are there?**

**Answer:**
"A GitLab Runner is an agent that picks up CI/CD jobs from GitLab and executes them. GitLab itself does not run the jobs — Runners do.

**Types of Runners:**

**1. Shared Runners** — provided by GitLab.com for all projects. Good for open source. Not suitable for private infrastructure because you don't control the environment.

**2. Group Runners** — registered at the GitLab group level, available to all projects in that group.

**3. Project-specific Runners** — registered for a single project. Most secure for production pipelines.

**Executor types (how the runner executes jobs):**
- **Shell executor** — runs directly on the runner machine. Simple but no isolation.
- **Docker executor** — each job runs in a fresh Docker container. Clean environment per job.
- **Kubernetes executor** — each job creates a pod in a Kubernetes cluster. Scales automatically.

**In my setup:**
I use a Kubernetes executor with a runner deployed inside our AKS cluster. Each CI job spawns a fresh pod, runs the job, and the pod is destroyed. This gives perfect isolation between jobs and scales with the cluster's capacity.

```toml
# config.toml for Kubernetes executor runner
[[runners]]
  name = "aks-runner"
  executor = "kubernetes"
  [runners.kubernetes]
    namespace = "gitlab-runner"
    image = "ubuntu:22.04"
    privileged = true    # Needed for Docker-in-Docker builds
```"

---

**Q36. What branching strategy do you follow in GitLab and how does it connect to CI/CD?**

**Answer:**
"I follow **GitFlow** for mature projects and **Trunk-Based Development** for fast-moving projects. Let me explain what I use for production:

**GitFlow (what I use for most projects):**
```
main ────────────────────────────────────── production deploys
  │
release/1.2 ──────── final testing ──────── staging deploys
  │
develop ─────────────────────────────────── dev/integration deploys
  │        │
feature/  hotfix/
login     fix-payment-bug
```

**CI/CD mapping to branches:**
```yaml
deploy-dev:
  stage: deploy
  script:
    - helm upgrade my-app ./helm --namespace finbank-dev
  only:
    - develop    # Dev deploy happens on every push to develop

deploy-staging:
  stage: deploy
  script:
    - helm upgrade my-app ./helm --namespace finbank-stage
  only:
    - /^release\/.*$/    # Staging deploy happens on release branches

deploy-production:
  stage: deploy
  script:
    - helm upgrade my-app ./helm --namespace finbank-prod
  when: manual           # Production deploy requires manual approval
  only:
    - main               # Only from main branch
```

**Hotfix flow:**
If there is a critical bug in production:
1. Create `hotfix/fix-critical-payment-bug` from `main`
2. Fix and test locally
3. Merge to `main` → auto-deploys to production
4. Merge back to `develop` so the fix is in future releases

**Trunk-Based Development (for startups/fast teams):**
Everyone commits to `main` (or short-lived feature branches merged within 1-2 days). Relies on feature flags to hide unfinished features from users. Simpler but requires strong automated testing."

---

**Q37. Production scenario: The GitLab pipeline passes but the deployment fails silently. How do you debug?**

**Answer:**
"Silent failures in CD pipelines are the worst because everything looks green but the application is broken. Here is how I find it:

**Step 1 — Check helm deployment status:**
```bash
helm list -n production
# If status shows 'failed' or 'pending-upgrade', that is the problem
helm status my-app -n production
```

**Step 2 — Check Kubernetes rollout:**
```bash
kubectl rollout status deployment/my-app -n production
# If it times out, the new pods are not becoming Ready
kubectl get pods -n production
# Look for pods in CrashLoopBackOff or Error state
```

**Step 3 — Check logs of failing pods:**
```bash
kubectl logs <failing-pod-name> -n production --previous
# --previous shows logs from the crashed container before restart
```

**Step 4 — Check deployment events:**
```bash
kubectl describe deployment my-app -n production
# Look at Events section for image pull errors, resource issues, etc.
```

**Step 5 — Check if liveness/readiness probes are failing:**
```bash
kubectl describe pod <pod-name> -n production
# Events section will show: 'Liveness probe failed'
# or 'Back-off restarting failed container'
```

**Common real causes I have found:**
- New image has a startup bug that passes tests but crashes in production config
- Readiness probe path changed but Kubernetes health check was not updated
- New deployment needs a new environment variable that was not added to Key Vault
- ACR image pull failed because the service principal's secret expired
- Helm values file had wrong image tag because the GitLab variable was not set

**Fix for GitLab pipeline:** Add a post-deploy verification step:
```yaml
verify-deployment:
  stage: verify
  script:
    - kubectl rollout status deployment/my-app -n production --timeout=5m
    - kubectl get pods -n production | grep my-app | grep Running
  after_script:
    - if [ $CI_JOB_STATUS == 'failed' ]; then
        kubectl rollout undo deployment/my-app -n production;
      fi
```"

---

# 9. Python and Bash Scripting

---

**Q38. Write a Python script to list all AKS clusters across all resource groups in an Azure subscription.**

**Answer:**
```python
#!/usr/bin/env python3
"""
List all AKS clusters in an Azure subscription with their status and node counts.
Uses azure-mgmt-containerservice SDK.
Install: pip install azure-mgmt-containerservice azure-identity
"""

from azure.identity import DefaultAzureCredential
from azure.mgmt.containerservice import ContainerServiceClient
import os

def list_aks_clusters():
    # DefaultAzureCredential automatically picks up:
    # - Environment variables (AZURE_CLIENT_ID, etc.)
    # - Managed Identity (when running on Azure)
    # - Azure CLI credentials (when running locally)
    credential = DefaultAzureCredential()
    subscription_id = os.environ["AZURE_SUBSCRIPTION_ID"]
    
    client = ContainerServiceClient(credential, subscription_id)
    
    print(f"{'CLUSTER NAME':<30} {'RESOURCE GROUP':<25} {'LOCATION':<15} {'K8S VERSION':<12} {'STATUS'}")
    print("-" * 100)
    
    # List all managed clusters across all resource groups
    clusters = client.managed_clusters.list()
    
    for cluster in clusters:
        name = cluster.name
        rg = cluster.id.split('/')[4]   # Extract RG from resource ID
        location = cluster.location
        version = cluster.kubernetes_version
        status = cluster.provisioning_state
        
        print(f"{name:<30} {rg:<25} {location:<15} {version:<12} {status}")

if __name__ == "__main__":
    list_aks_clusters()
```

---

**Q39. Write a Bash script to check if all pods in a namespace are running and alert if any are not.**

**Answer:**
```bash
#!/bin/bash
# check-pods-health.sh
# Usage: ./check-pods-health.sh production
# Sends alert if any pod is not in Running state

NAMESPACE=${1:-default}
ALERT_EMAIL="devops@company.com"
FAILED_PODS=()

echo "Checking pod health in namespace: $NAMESPACE"
echo "Timestamp: $(date)"
echo "---"

# Get all pods and their statuses
while IFS= read -r line; do
    POD_NAME=$(echo "$line" | awk '{print $1}')
    STATUS=$(echo "$line" | awk '{print $3}')
    READY=$(echo "$line" | awk '{print $2}')
    
    if [[ "$STATUS" != "Running" && "$STATUS" != "Completed" ]]; then
        FAILED_PODS+=("$POD_NAME (Status: $STATUS, Ready: $READY)")
        echo "FAILED: $POD_NAME — Status: $STATUS"
    else
        echo "OK:     $POD_NAME — Status: $STATUS Ready: $READY"
    fi
done < <(kubectl get pods -n "$NAMESPACE" --no-headers 2>/dev/null)

echo "---"

# If there are failed pods, send alert
if [[ ${#FAILED_PODS[@]} -gt 0 ]]; then
    echo "ALERT: ${#FAILED_PODS[@]} pod(s) are not healthy!"
    
    MESSAGE="Pod Health Alert — Namespace: $NAMESPACE\n\nFailed Pods:\n"
    for pod in "${FAILED_PODS[@]}"; do
        MESSAGE+="  - $pod\n"
    done
    
    # Send email alert (requires mailutils)
    echo -e "$MESSAGE" | mail -s "[ALERT] Pod failure in $NAMESPACE" "$ALERT_EMAIL"
    
    # Exit with error code — can be used in CI/CD to fail the pipeline
    exit 1
else
    echo "All pods are healthy in namespace: $NAMESPACE"
    exit 0
fi
```

---

**Q40. Write a Python script to restart all failed pods in a namespace.**

**Answer:**
```python
#!/usr/bin/env python3
"""
Find and restart all pods in CrashLoopBackOff or Error state
by deleting them (Deployment controller recreates them automatically)
"""

from kubernetes import client, config
import sys

def restart_failed_pods(namespace):
    # Load kubeconfig (from ~/.kube/config or in-cluster config)
    try:
        config.load_incluster_config()    # Running inside a pod
    except config.ConfigException:
        config.load_kube_config()          # Running locally
    
    v1 = client.CoreV1Api()
    
    # Get all pods in namespace
    pods = v1.list_namespaced_pod(namespace=namespace)
    
    failed_statuses = ["CrashLoopBackOff", "Error", "OOMKilled", "ImagePullBackOff"]
    restarted = []
    
    for pod in pods.items:
        pod_name = pod.metadata.name
        
        # Check container statuses
        if pod.status.container_statuses:
            for container in pod.status.container_statuses:
                waiting = container.state.waiting
                if waiting and waiting.reason in failed_statuses:
                    print(f"Deleting failed pod: {pod_name} (Reason: {waiting.reason})")
                    
                    # Delete the pod — the Deployment will recreate it
                    v1.delete_namespaced_pod(
                        name=pod_name,
                        namespace=namespace,
                        body=client.V1DeleteOptions(grace_period_seconds=0)
                    )
                    restarted.append(pod_name)
                    break
    
    if restarted:
        print(f"\nRestarted {len(restarted)} pods:")
        for pod in restarted:
            print(f"  - {pod}")
    else:
        print("No failed pods found.")
    
    return len(restarted)

if __name__ == "__main__":
    namespace = sys.argv[1] if len(sys.argv) > 1 else "default"
    restart_failed_pods(namespace)
```

---

# 10. GenAI in CI/CD Pipelines

---

**Q41. The JD mentions 'Automate CI/CD pipelines using GenAI concepts.' What does this mean and how do you implement it?**

**Answer:**
"This is a newer concept that TCS and other companies are actively exploring. Here are the practical ways GenAI is being integrated into CI/CD:

**1. AI-Powered Code Review in Pipeline:**
Tools like GitHub Copilot for PR reviews, or custom LLM integrations that automatically review code changes, suggest fixes, and flag security issues as part of the GitLab pipeline.

**2. Intelligent Test Generation:**
GenAI tools like GitHub Copilot or Amazon CodeWhisperer generate unit tests for new code automatically. The CI pipeline runs these generated tests alongside manually written tests.

**3. Pipeline Configuration Generation:**
Using LLMs (like the Claude or OpenAI API) to generate GitLab CI YAML configurations based on project type, tech stack, and requirements.

**4. Root Cause Analysis of Failures:**
When a pipeline fails, an AI agent reads the logs and suggests the likely cause and fix. This reduces mean time to recovery.

**Practical implementation in a GitLab pipeline:**
```yaml
ai-pipeline-analysis:
  stage: .post      # Runs after all other stages
  script:
    - |
      if [ "$CI_JOB_STATUS" == "failed" ]; then
        # Call LLM API with the failed job logs
        FAILED_LOGS=$(cat job_logs.txt)
        curl -X POST https://api.anthropic.com/v1/messages \
          -H "x-api-key: $ANTHROPIC_API_KEY" \
          -H "content-type: application/json" \
          -d "{
            \"model\": \"claude-sonnet-4-6\",
            \"max_tokens\": 1024,
            \"messages\": [{
              \"role\": \"user\",
              \"content\": \"Analyze this CI/CD failure and suggest a fix: $FAILED_LOGS\"
            }]
          }" > ai_analysis.json
        cat ai_analysis.json
      fi
  when: always
  artifacts:
    paths:
      - ai_analysis.json
    expire_in: 1 week
```

**5. Security scanning with AI:**
Tools like Snyk or Semgrep are adding AI capabilities to understand the context of vulnerabilities — not just flagging the issue but explaining the business impact and priority."

---

# 11. Real Production Troubleshooting Scenarios

---

**Scenario 1: AKS Cluster is not accessible — kubectl commands are timing out.**

**Answer:**
"This happened to us once. Here is my exact debugging process:

**Step 1 — Check AKS cluster status in Azure:**
```bash
az aks show --resource-group rg-prod --name aks-prod --query provisioningState
# If it shows 'Failed' or 'Updating', Azure has a control plane issue
```

**Step 2 — Check Azure Service Health:**
Go to Azure Portal → Service Health → check if there is an active incident in your region. If yes, wait for Azure to resolve it.

**Step 3 — Check if it is a networking issue:**
```bash
# Can you reach the API server endpoint?
curl -k https://<aks-api-server-ip>:443/healthz
```
If no response, check:
- NSG rules blocking port 443 to the API server
- VPN/firewall changes that might have blocked access
- Check if `endpoint_public_access` was accidentally disabled in Terraform

**Step 4 — Refresh kubeconfig:**
```bash
az aks get-credentials --resource-group rg-prod --name aks-prod --overwrite-existing
kubectl cluster-info
```

**Step 5 — Check node health:**
```bash
kubectl get nodes
# If nodes show NotReady — check VM health in Azure Portal
az vmss list-instances --resource-group <node-rg> --name <vmss-name>
```

**Root cause in our case:** A new NSG rule was applied to the subnet that blocked port 443 outbound from the nodes to the API server. The cluster nodes could not communicate with the control plane. Fix: removed the blocking NSG rule."

---

**Scenario 2: Istio mTLS enabled — one service suddenly can't call another after a routine deployment.**

**Answer:**
"After a deployment, Service A was getting 503 errors calling Service B. Everything was working before. This is what I did:

**Step 1 — Check if new deployment has sidecar:**
```bash
kubectl get pods -n production | grep service-b
# Showed 1/1 instead of 2/2 — the new deployment did NOT have the Istio sidecar!
```

**Root cause:** The new deployment was applied to a new namespace that did not have `istio-injection=enabled`. The old namespace had it, but the deployment was moved.

**Step 2 — Verify namespace label:**
```bash
kubectl get namespace production --show-labels
# istio-injection label was missing
```

**Fix:**
```bash
kubectl label namespace production istio-injection=enabled
kubectl rollout restart deployment/service-b -n production
# New pods now show 2/2 with Envoy sidecar
```

**Prevention added:** Added a GitLab CI check that verifies the target namespace has Istio injection enabled before deploying:
```bash
# In deploy stage
INJECTION=$(kubectl get namespace $NAMESPACE --show-labels | grep istio-injection=enabled)
if [ -z "$INJECTION" ]; then
  echo "ERROR: Namespace $NAMESPACE does not have Istio injection enabled"
  exit 1
fi
```"

---

**Scenario 3: Terraform apply is destroying and recreating a production database every time.**

**Answer:**
"This is a dangerous situation — Terraform planning to destroy a production RDS or Azure SQL database on every apply.

**Step 1 — Run plan and read carefully:**
```bash
terraform plan 2>&1 | tee plan.txt
grep -A 5 'must be replaced' plan.txt
```
Look at what change is forcing recreation. Common culprits:
- `db_name` changed
- `location` changed
- `admin_login` changed (some resources require recreation for username changes)
- Terraform version upgrade changed how a resource is processed

**Step 2 — Use lifecycle prevent_destroy:**
```hcl
resource "azurerm_mssql_server" "main" {
  name                = "sql-prod"
  resource_group_name = azurerm_resource_group.main.name
  location            = azurerm_resource_group.main.location

  lifecycle {
    prevent_destroy = true
    # Terraform will error if it tries to destroy this resource
    # Forces you to consciously remove this block before destroying
  }
}
```

**Step 3 — Use ignore_changes for fields that change at runtime:**
```hcl
lifecycle {
  ignore_changes = [
    administrator_login_password,    # Changed by ops team manually
    tags["LastModifiedBy"]           # Tag changed by Azure Policy
  ]
}
```

**Fix in our case:** The `location` field was set using a variable that was changed from `East US` to `eastus` (same location, different format). Azure accepted both but Terraform saw them as different, triggering recreation. Fix: standardized the location format in the variable and ran `terraform apply -refresh-only` to update state without making changes."

---

**Scenario 4: GitLab pipeline fails during Docker build with 'no space left on device'.**

**Answer:**
"This happens when the GitLab Runner's disk fills up with old Docker images and build cache.

**Immediate fix:**
```bash
# SSH into the runner VM
docker system prune -af --volumes
# This removes: all stopped containers, unused images, build cache, unused volumes
# WARNING: Also removes volumes — check what volumes exist first
docker volume ls
docker system prune -af    # Without --volumes if you want to keep volumes
```

**Check disk usage:**
```bash
df -h
du -sh /var/lib/docker/*    # Find what is consuming space
```

**Long-term fix — add cleanup step to GitLab CI:**
```yaml
cleanup:
  stage: .post    # Always runs last
  script:
    - docker image prune -af --filter "until=24h"    # Remove images older than 24h
    - docker container prune -f
    - docker builder prune -af --filter "until=48h"
  when: always    # Run even if pipeline fails
  allow_failure: true
```

**Better long-term fix — use Docker BuildKit with a registry cache:**
Instead of caching locally on the runner, push the build cache to ACR:
```yaml
build:
  script:
    - |
      docker buildx build \
        --cache-from type=registry,ref=$CI_REGISTRY_IMAGE:cache \
        --cache-to type=registry,ref=$CI_REGISTRY_IMAGE:cache,mode=max \
        --push \
        -t $IMAGE_NAME:$IMAGE_TAG .
```
This moves the cache to the registry — runner disk stays clean."

---

**Scenario 5: An AKS node is NotReady — pods on it are stuck in Terminating state.**

**Answer:**
"A production node went NotReady. Pods on it stopped responding but showed as Terminating — they were stuck.

**Step 1 — Understand the state:**
```bash
kubectl get nodes
# NAME           STATUS     ROLES    AGE
# aks-node-001   NotReady   <none>   5d

kubectl describe node aks-node-001
# Events show: 'NodeNotReady' — kubelet stopped posting node status
# Conditions show: 'MemoryPressure=True' — node is out of memory
```

**Step 2 — Force delete stuck Terminating pods:**
Normal `kubectl delete pod` respects the graceful termination period. If the node is dead, pods can be stuck forever.
```bash
# Force delete — removes from etcd immediately without waiting for the pod to stop
kubectl delete pod <pod-name> -n production --grace-period=0 --force
```

**Step 3 — Cordon the bad node:**
```bash
kubectl cordon aks-node-001
# Prevents any new pods from scheduling to this node while you investigate
```

**Step 4 — Drain remaining pods off:**
```bash
kubectl drain aks-node-001 --ignore-daemonsets --delete-emptydir-data --force
# Moves all pods to other healthy nodes
```

**Step 5 — Fix the node or replace it:**
If MemoryPressure — check which pod was consuming all memory (OOM killed):
```bash
kubectl get events -n production --sort-by='.metadata.creationTimestamp' | grep OOM
```

**Step 6 — If node is unrecoverable:**
```bash
kubectl delete node aks-node-001
# Azure VMSS will provision a replacement node automatically
```

**Prevention:** Set memory requests and limits on all pods. A pod with no limits can consume all node memory and cause this exactly."

---

# 12. HR Round Questions

---

**Q42. Tell me about yourself.**

**Answer:**
"I am a DevOps engineer with experience in designing and managing cloud-native infrastructure on both AWS and Azure. My expertise is in Kubernetes, Terraform, CI/CD pipelines, and DevSecOps.

In my recent project, FinBank, I built a complete production-grade banking application infrastructure on AWS — this includes EKS clusters, RDS MySQL, ElastiCache Redis, Secrets Manager with IRSA for zero-credential security, and a full Jenkins CI/CD pipeline with ArgoCD GitOps deployment across three environments.

Alongside that, I have worked on Azure with AKS, Azure Pipelines, Azure Container Registry, and Key Vault in another project called AzureShop.

I am now looking to join TCS to work on enterprise-scale Azure DevOps projects where I can apply these skills in a real MNC environment and continue growing toward a senior/architect role."

---

**Q43. Why do you want to join TCS?**

**Answer:**
"TCS works with large enterprise clients across banking, insurance, healthcare, and retail — exactly the domains where DevOps and cloud-native skills are used at real scale. I want to be in an environment where I am solving infrastructure problems for systems that handle millions of transactions, not just hundreds.

I also know that TCS invests in certifications and training. The scale of projects at TCS would expose me to real production challenges that you simply cannot simulate in smaller setups. That experience will be the foundation of my long-term career in cloud infrastructure."

---

**Q44. What is your expected salary?**

**Answer:**
"Based on my current skill set — Kubernetes, Terraform, Azure, GitLab CI/CD, Istio, and the two production projects I have built — I am looking at a range of ₹8–12 LPA. I am open to discussion based on the role's full scope and growth path. My priority is the right project and the right team, not just the number."

---

**Q45. Where do you see yourself in 3 years?**

**Answer:**
"In 3 years I see myself as a Senior DevOps Engineer or Platform Engineer, someone who owns the infrastructure decisions for a large project — designing multi-environment setups, improving pipeline reliability, and mentoring junior engineers.

I plan to have my CKA and AWS DevOps Engineer Professional certifications by then. I also want to go deeper into DevSecOps and platform engineering, which are the directions the industry is moving. TCS's exposure to enterprise-scale projects would accelerate that path significantly."

---

## Quick Revision — Top 10 Things TCS ALWAYS Asks

```
1. Walk me through your current project architecture
   → Practice describing FinBank/AzureShop clearly in 3 minutes

2. What is the difference between AKS and self-managed Kubernetes?
   → Azure manages control plane, you manage nodes

3. How does mTLS work in Istio?
   → Both sides exchange SPIFFE certificates issued by Istio CA

4. How do you manage Terraform state in a team?
   → Azure Blob backend with locking + separate state per environment

5. Explain your CI/CD pipeline end to end
   → GitLab → build → test → scan → push to ACR → Helm → AKS

6. What happens when a pod is in CrashLoopBackOff?
   → kubectl describe pod → kubectl logs --previous → check probes and env vars

7. What is the difference between a Load Balancer and Application Gateway?
   → L4 vs L7, TCP/UDP vs HTTP with WAF and URL routing

8. What is NSG and where do you apply it?
   → Subnet or NIC level firewall rules, stateless, priority-based

9. How do you rotate SSL certificates?
   → cert-manager for automation, manual via kubectl create secret --dry-run

10. What is the difference between count and for_each in Terraform?
    → count uses index (risky deletion), for_each uses stable key
```

---

*Prepared for TCS Azure DevOps Interview — July 2026*
*Based on: Glassdoor TCS interview experiences, real production scenarios, JD analysis*

---

Sources used in research:
- [TCS DevOps Interview Questions — Glassdoor](https://www.glassdoor.com/Interview/TCS-Devops-Engineer-Interview-Questions-EI_IE3211746.0,3_KO4,19.htm)
- [AKS Interview Questions 2026 — ProjectPractical](https://www.projectpractical.com/azure-kubernetes-service-aks-interview-questions-and-answers/)
- [Real-World AKS Scenarios — Medium](https://mihirpopat.medium.com/7-real-world-aks-scenario-based-questions-youll-encounter-on-the-job-39362e2177af)
- [Istio Interview Questions 2026 — scmGalaxy](https://www.scmgalaxy.com/tutorials/top-50-istio-interview-questions-with-answers/)
- [Service Mesh Questions — CloudSoft](https://cloudsoftsol.com/interview-questions/service-mesh-interview-questions-2025-istio-linkerd-real-world-scenarios/)
- [Terraform Interview Questions 2026 — DataCamp](https://www.datacamp.com/blog/terraform-interview-questions)
- [CI/CD Real Scenarios — ThinkCloudly](https://thinkcloudly.com/blog/real-world-ci-cd-scenarios-asked-in-devops-interviews/)
- [Azure Networking Questions — KodeKloud](https://kodekloud.com/blog/azure-interview-questions/)
- [DevOps Interview Questions — NotHarshhaa GitHub](https://github.com/NotHarshhaa/DevOps-Interview-Questions)
- [GitLab CI/CD Questions 2026 — igmGuru](https://www.igmguru.com/blog/ci-cd-interview-questions)
