# T24 DevOps Pipeline — Containerised Deployment on Azure

An end-to-end CI/CD pipeline that builds a containerised T24-style application, pushes it to Azure Container Registry, and deploys it to Azure Kubernetes Service via Helm — fully automated through GitHub Actions. This demonstrates the modern deployment pattern used for Temenos T24/Transact on cloud-native infrastructure.

## What this project demonstrates

This is a working reference implementation of the two T24 deployment models and the complete build-to-deploy automation chain:

- Containerised application build (represents a T24 Transact image)
- Automated image build tagged with the git commit SHA for full traceability
- Push to Azure Container Registry (ACR)
- Helm-based deployment to AKS with rolling updates
- GitHub Actions pipeline triggered on code changes
- Environment-driven configuration via Helm values

## Architecture

```
Developer commits change
        |
        v
GitHub (source of truth)
        |
        v  (GitHub Actions triggers on app/** change)
CI/CD Pipeline:
|-- Azure login (service principal)
|-- docker build -> image tagged with git SHA
|-- docker push -> Azure Container Registry
|-- helm upgrade -> deploy to AKS
|-- verify -> pods and service status
        |
        v
AKS Cluster (application running, LoadBalancer exposed)
```

## Pipeline flow

The GitHub Actions workflow (`.github/workflows/t24-cicd.yml`) runs on every push that touches `app/` or `t24-deploy/`, or can be triggered manually:

1. Checkout the repository
2. Authenticate to Azure using a service principal stored in GitHub Secrets
3. Build the Docker image, tagging it with the git commit SHA
4. Push the image to ACR
5. Retrieve AKS credentials
6. Deploy via `helm upgrade`, injecting the new image tag
7. Verify pods and service are healthy

## T24 Deployment Models

This project illustrates the containerised deployment path (Type 2). In a real T24 estate there are two deployment types:

**Type 2 - Image / code change (demonstrated here)**
Code or JAR changes are baked into the container image. A change means rebuilding the image, pushing it to the registry, and rolling out new pods via Helm. This is what the pipeline in this repo automates.

**Type 1 - DSF / configuration change**
Configuration and metadata - T24 versions, enquiries, menus, parameters - live in the database, not the image. These are packaged using the Temenos Deployment Service Framework (DSF) or applied via OFS, and deployed to the target environment's database without rebuilding the image. In a mature pipeline, config is also stored in Git and the pipeline detects whether a change is code or config and routes to the correct deployment path.

## Image layering (T24 preimage pattern)

For T24, custom changes typically layer on top of the large base image rather than rebuilding everything:

```
t24-base (core + ~1000 JARs)
     |
     v  FROM t24-base + team WAR changes
t24-app:v2
     |
     v  FROM t24-app:v2 + new WAR change
t24-app:v3   (inherits everything before it)
```

Each new image builds `FROM` the previous tagged image, inheriting all prior changes. Meaningful image tagging (git SHA or version tags) is essential so builds reference the correct base and rollbacks target a known-good image. Layers are periodically flattened by rebasing from the base to avoid image sprawl.

## Artifact management with JFrog Artifactory

In an enterprise T24 setup, JFrog Artifactory serves as the central repository for all artifact types - Docker images, JARs, WARs, and Helm charts. Its key advantage is the promotion model: an artifact is built once and promoted through dev -> UAT -> production without rebuilding, guaranteeing that what was tested is exactly what ships. Combined with build metadata it provides the audit trail banking compliance requires.

## Project structure

```
t24-devops-pipeline/
|-- app/
|   |-- app.py            # Application (represents T24 Transact)
|   |-- Dockerfile        # Container build definition
|-- t24-deploy/           # Helm chart for AKS deployment
|   |-- Chart.yaml
|   |-- values.yaml
|   |-- templates/
|-- .github/workflows/
    |-- t24-cicd.yml      # CI/CD pipeline
```

## Prerequisites

- AKS cluster
- Azure Container Registry attached to AKS (AcrPull role)
- GitHub secret `AZURE_CREDENTIALS` (service principal with contributor access)

## Usage

The pipeline runs automatically on push. To deploy manually:

```bash
# Build and push
docker build -t <acr>.azurecr.io/t24-app:<tag> ./app
docker push <acr>.azurecr.io/t24-app:<tag>

# Deploy
helm upgrade t24-app ./t24-deploy \
  --namespace t24 --create-namespace \
  --set image.repository=<acr>.azurecr.io/t24-app \
  --set image.tag=<tag>
```

## Phased automation approach

In a regulated banking environment with restricted access and change control, this automation is best adopted incrementally: first version-control T24 code and config in Git for audit trails, then automate image builds and DSF packaging, and finally automate deployment to lower environments with approval gates for production. This respects banking change control while progressively reducing manual effort and deployment errors.

## Related repositories

- **t24-azure-terraform** - Terraform infrastructure for T24 on Azure (AKS, App Gateway, SQL, Key Vault, ACR, Storage)
- **t24-helm-chart** - Production Helm chart for T24 with R25 runbook specifications and ArgoCD GitOps
