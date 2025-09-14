# Phoenix Application GitOps Platform

## Project Overview

This project implements a complete GitOps-managed platform for a Phoenix web application on Kubernetes. All infrastructure and application configurations are managed through this Git repository using FluxCD as the GitOps operator.

## Architecture Diagram
┌─────────────────────────────────────────────────────────────┐
│                    GitHub Repository                         │
│              (jas-backgammon-gitops)                        │
└────────────────────┬────────────────────────────────────────┘
│ GitOps Sync (FluxCD)
▼
┌─────────────────────────────────────────────────────────────┐
│                    OrbStack Kubernetes                       │
├─────────────────────────────────────────────────────────────┤
│   ┌──────────────┐  ┌──────────────┐  ┌──────────────┐    │
│   │  FluxCD      │  │CloudNativePG │  │  Prometheus  │    │
│   │  Operators   │  │   Operator   │  │    Stack     │    │
│   └──────────────┘  └──────────────┘  └──────────────┘    │
│   ┌──────────────┐  ┌──────────────┐  ┌──────────────┐    │
│   │  PostgreSQL  │  │  Phoenix App │  │   Grafana    │    │
│   │   Cluster    │  │  Deployment  │  │  Dashboard   │    │
│   └──────────────┘  └──────────────┘  └──────────────┘    │
└─────────────────────────────────────────────────────────────┘

## Technology Stack

- **Kubernetes Distribution**: OrbStack (native macOS Kubernetes)
- **GitOps Tool**: FluxCD v2
- **Container Runtime**: Docker
- **Configuration Management**: Helm + Kustomize
- **Database**: PostgreSQL managed by CloudNativePG operator
- **Monitoring**: Prometheus + Grafana (kube-prometheus-stack)
- **Application**: Phoenix Framework (Elixir)

## Prerequisites

- OrbStack installed with Kubernetes enabled
- kubectl CLI configured for OrbStack context
- flux CLI installed: `brew install fluxcd/tap/flux`
- Docker for building container images
- GitHub account with personal access token (repo scope)
- Docker Hub account for image registry

## Complete Setup Instructions

### Step 1: Clone Repository and Initialize
```bash
git clone https://github.com/jasaulakh1988/jas-backgammon-gitops.git
cd jas-backgammon-gitops
Step 2: Verify OrbStack Kubernetes
bashkubectl config use-context orbstack
kubectl get nodes
Step 3: Bootstrap FluxCD
bashexport GITHUB_TOKEN=<your-github-token>
export GITHUB_USER=jasaulakh1988

flux bootstrap github \
  --owner=$GITHUB_USER \
  --repository=jas-backgammon-gitops \
  --branch=main \
  --path=./clusters/local \
  --personal
This command:

Installs Flux controllers in flux-system namespace
Configures Flux to watch this repository
Creates deploy key for repository access
Commits Flux manifests to clusters/local/flux-system

Step 4: Deploy Infrastructure Components
All infrastructure is automatically deployed by Flux from the Git repository:

CloudNativePG Operator: Manages PostgreSQL lifecycle
PostgreSQL Cluster: Single instance with phoenix_app database
Prometheus Stack: Metrics collection and alerting
Grafana: Visualization on port 30300

Step 5: Build and Deploy Phoenix Application
The Phoenix application Docker image was built from source:
bashcd apps/phoenix-app/docker
docker build --target prod -t jasaulakh/phoenix-app:v1.0.0 .
docker push jasaulakh/phoenix-app:v1.0.0
Deployment happens automatically via Flux using the custom Helm chart.
Helm Chart Configuration
The Phoenix app Helm chart (apps/phoenix-app/helm-chart/) exposes these configurable values:
ParameterDescriptionDefaultreplicaCountNumber of Phoenix pods2image.repositoryDocker image repositoryjasaulakh/phoenix-appimage.tagDocker image tagv1.0.0service.typeKubernetes service typeNodePortservice.nodePortExternal port for access30080env.MIX_ENVElixir environmentprodenv.PHX_HOSTPhoenix host configurationlocalhostdatabase.hostPostgreSQL hostpostgres-cluster.default.svc.cluster.localdatabase.nameDatabase namephoenix_appresources.limits.memoryMemory limit512Miresources.limits.cpuCPU limit500m
All Phoenix configuration from config/runtime.exs is exposed via environment variables in the deployment.
Secret Management
Secrets are managed securely:

Database credentials stored in Kubernetes Secret
SECRET_KEY_BASE configured via Helm values (should use external secret manager in production)
No hardcoded secrets in code or manifests
Credentials injected as environment variables at runtime

Monitoring Setup
Grafana Dashboard for CloudNativePG
Dashboard Choice: CloudNativePG official dashboard (ID: 20417)
Reasoning:

Official dashboard from CloudNativePG maintainers
Comprehensive PostgreSQL metrics including connections, transactions, and replication
Compatible with CloudNativePG operator metrics
Shows cluster health, backup status, and resource utilization

To import:

Access Grafana at http://localhost:30300 (admin/admin123)
Go to Dashboards → Import
Enter dashboard ID: 20417
Select Prometheus datasource

Prometheus Metrics
Prometheus automatically scrapes metrics from:

CloudNativePG PostgreSQL clusters (port 9187)
Phoenix application (if metrics endpoint configured)
Kubernetes components
Node exporters

Accessing Services

Phoenix Application: http://localhost:30080
Grafana Dashboard: http://localhost:30300

Username: admin
Password: admin123



Repository Structure
.
├── README.md                        # This file
├── clusters/
│   └── local/                      # Flux cluster configuration
│       ├── flux-system/            # Flux components (auto-generated)
│       ├── cloudnativepg.yaml      # Database operator Kustomization
│       ├── monitoring.yaml         # Prometheus stack Kustomization
│       └── phoenix-app.yaml        # Phoenix app Kustomization
├── infrastructure/
│   ├── cloudnativepg/              # PostgreSQL operator and cluster
│   │   ├── namespace.yaml
│   │   ├── helmrepository.yaml
│   │   ├── helmrelease.yaml
│   │   ├── postgres-cluster.yaml
│   │   └── kustomization.yaml
│   └── monitoring/                 # Prometheus and Grafana
│       ├── namespace.yaml
│       ├── helmrepository.yaml
│       ├── helmrelease.yaml
│       └── kustomization.yaml
├── apps/
│   └── phoenix-app/
│       ├── docker/                 # Phoenix source and Dockerfile
│       ├── helm-chart/             # Custom Helm chart
│       │   ├── Chart.yaml
│       │   ├── values.yaml
│       │   └── templates/
│       ├── helm-release.yaml
│       └── kustomization.yaml
└── scripts/                        # Helper scripts (if needed)
GitOps Workflow

Make changes to any YAML files in this repository
Commit and push to main branch
Flux automatically detects changes (within 10 minutes)
Changes are applied to cluster maintaining desired state

Force immediate sync:
bashflux reconcile source git flux-system
flux reconcile kustomization flux-system
Verification Commands
bash# Check Flux sync status
flux get all

# Check application pods
kubectl get pods -n default

# Check monitoring stack
kubectl get pods -n monitoring

# Check PostgreSQL cluster
kubectl get cluster -n default

# View Phoenix app logs
kubectl logs -n default -l app=phoenix-app

# Check Flux logs for issues
flux logs --all-namespaces
Bonus: Alerting Configuration
Alertmanager is configured to send alerts when pods are not ready:
Alert Rule: Pod not in Ready state for >1 minute
Notification: Webhook to Discord/Slack (configure in values)
To test alerting:
bash# Scale down Phoenix app to trigger alert
kubectl scale deployment phoenix-app --replicas=0 -n default
# Wait 1 minute for alert
# Scale back up
kubectl scale deployment phoenix-app --replicas=2 -n default
Troubleshooting
If Phoenix app is crashing:

Check database connectivity: kubectl exec -it postgres-cluster-1 -- psql -U phoenix_user -d phoenix_app
View application logs: kubectl logs -n default phoenix-app-<pod-id>
Verify environment variables: kubectl describe pod phoenix-app-<pod-id>
Check database credentials: kubectl get secret postgres-credentials -o yaml

Security Considerations

All components run with least privilege
Resource limits enforced on all containers
Network policies can be added for additional isolation
Secrets never committed to Git
RBAC configured for service accounts

Performance Tuning

Phoenix app configured with 2 replicas for high availability
PostgreSQL connection pooling configured
Resource requests/limits set for predictable performance
Monitoring alerts for resource exhaustion
