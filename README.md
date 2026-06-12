# azure-cicd-pipeline

Multi-stage Azure DevOps CI/CD pipeline deploying a Dockerized Node.js app to AKS.

## Pipeline Architecture

```
Code Push
    │
    ▼
┌─────────────────┐
│  Stage 1: Build │  → Install deps → Run tests → Build Docker image → Push to ACR
└────────┬────────┘
         │
         ▼
┌──────────────────────┐
│ Stage 2: Staging     │  → Pull image from ACR → Deploy to AKS (staging namespace)
└──────────┬───────────┘
           │  (main branch only)
           ▼
┌──────────────────────┐
│ Stage 3: Production  │  → Deploy to AKS (production namespace) → Health check
└──────────────────────┘
```

## Stack

| Component | Technology |
|-----------|-----------|
| CI/CD | Azure DevOps Pipelines |
| Containerization | Docker (multi-stage build) |
| Registry | Azure Container Registry (ACR) |
| Orchestration | Azure Kubernetes Service (AKS) |
| Infra provisioning | Terraform (see `/terraform`) |
| App | Node.js + Express |

## Repository Structure

```
azure-cicd-pipeline/
├── app/
│   ├── app.js               # Express app with /health endpoint
│   ├── package.json
│   └── Dockerfile           # Multi-stage, non-root user
├── k8s/
│   ├── deployment.yaml      # Rolling update, resource limits, probes
│   └── service.yaml         # LoadBalancer service
├── .azure-pipelines/
│   └── azure-pipelines.yml  # 3-stage pipeline: Build → Staging → Production
├── terraform/
│   └── README.md            # Links to terraform-azure-infra repo
└── README.md
```

## Setup

### Prerequisites
- Azure subscription
- Azure DevOps organization
- AKS cluster + ACR provisioned via [terraform-azure-infra](https://github.com/Lokesh0423/terraform-azure-infra)

### Steps

1. **Fork/clone this repo** and import into Azure DevOps

2. **Create service connection** in Azure DevOps:
   - Project Settings → Service Connections → New → Azure Resource Manager
   - Name it `Azure-Service-Connection`

3. **Update pipeline variables** in `azure-pipelines.yml`:
   ```yaml
   ACR_NAME: 'youracrname'
   AKS_CLUSTER: 'your-aks-cluster'
   AKS_RESOURCE_GROUP: 'your-resource-group'
   ```

4. **Create namespaces** in AKS:
   ```bash
   kubectl create namespace staging
   kubectl create namespace production
   ```

5. **Run the pipeline** — push to `develop` triggers Build + Staging. Push to `main` triggers all 3 stages including Production.

## Key Features

- **Multi-stage build** — Docker image with non-root user for security
- **Rolling updates** — zero-downtime deployments on AKS
- **Health checks** — liveness and readiness probes on `/health`
- **Resource limits** — CPU and memory constraints defined per pod
- **Branch strategy** — `develop` deploys to staging, `main` deploys to production
- **End-to-end infra** — pairs with [terraform-azure-infra](https://github.com/Lokesh0423/terraform-azure-infra) for full IaC story



**Multi-stage Docker builds** — Separating the builder and runtime stages keeps the final image lean and clean. Adding a non-root user felt like a small thing but it's the kind of security detail that matters in production.

**Kubernetes probes** — Liveness vs readiness took me a while to really get. Liveness restarts a broken container. Readiness stops traffic going to a container that isn't ready yet. Both matter, they're not the same thing.

**Branch strategy** — Tying `develop` to staging and `main` to production makes the pipeline self-documenting. The branch name tells you exactly where your code is going.

**Rolling updates** — Setting `maxUnavailable: 0` means zero downtime during deploys. Learned this the hard way understanding what happens when you don't set it.

**ACR + AKS connection** — The service connection in Azure DevOps is the glue between the pipeline and the cloud. Getting that auth flow right was the most fiddly part of the whole setup.

## Related

- [terraform-azure-infra](https://github.com/Lokesh0423/terraform-azure-infra) — AKS, VNet, ACR, Azure Monitor modules
