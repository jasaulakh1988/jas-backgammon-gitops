**Phoenix Application GitOps Platform - Complete Setup Guide
Project Overview
This project implements a production-ready GitOps platform for a Phoenix web application using Kubernetes, FluxCD, PostgreSQL (CloudNativePG), and full observability with Prometheus/Grafana.
**

<img width="560" height="451" alt="image" src="https://github.com/user-attachments/assets/a3f0acdd-4c93-415a-8ee7-1ca642d5bc11" />

**1. Prerequisites**

macOS with OrbStack installed
kubectl CLI
flux CLI (brew install fluxcd/tap/flux)
Docker Desktop
GitHub account with personal access token
Docker Hub account

**Step 1: Kubernetes Cluster Setup with OrbStack
1.1 Install and Configure OrbStack**

# Install OrbStack (if not already installed)
brew install --cask orbstack

# Start OrbStack and enable Kubernetes
# Open OrbStack app → Settings → Kubernetes → Enable

# Set kubectl context
kubectl config use-context orbstack

# Verify cluster is running
kubectl get nodes
kubectl get ns

<img width="710" height="252" alt="image" src="https://github.com/user-attachments/assets/a1ec83d0-2d56-4869-b611-54db848bdc02" />

**Step 2: Bootstrap FluxCD for GitOps
2.1 Prepare GitHub Repository**

# Fork or create repository
git clone https://github.com/jasaulakh1988/jas-backgammon-gitops.git
cd jas-backgammon-gitops

# Set environment variables
export GITHUB_TOKEN=<your-github-token>
export GITHUB_USER=jasaulakh1988

**2.2 Install FluxCD**

# Bootstrap Flux
flux bootstrap github \
  --owner=$GITHUB_USER \
  --repository=jas-backgammon-gitops \
  --branch=main \
  --path=./clusters/local \
  --personal

# Verify Flux installation
flux check
kubectl get pods -n flux-system

<img width="764" height="139" alt="image" src="https://github.com/user-attachments/assets/e59d60fe-bbc9-4959-8b1a-31eb7eacdd9e" />

**2.3 Verify Flux Components**

# Check all Flux resources
flux get all
kubectl get gitrepository -n flux-system
kubectl get kustomization -n flux-system
<img width="1504" height="666" alt="image" src="https://github.com/user-attachments/assets/eda07636-f5b1-4bef-97eb-a61eaf6fff81" />

**Step 3: Deploy CloudNativePG Operator and PostgreSQL
3.1 Deploy CloudNativePG Operator via GitOps**

# The operator is deployed automatically via Flux
# Check deployment status
kubectl get pods -n cnpg-system
kubectl get crds | grep cnpg

<img width="734" height="191" alt="image" src="https://github.com/user-attachments/assets/8fce4017-5511-43a5-b74c-6825cbd4aa55" />


**3.2 Deploy PostgreSQL Cluster**
# PostgreSQL cluster is created via GitOps
# Verify cluster creation
kubectl get cluster -n default
kubectl get pods -n default | grep postgres
kubectl get svc | grep postgres
<img width="826" height="202" alt="image" src="https://github.com/user-attachments/assets/588956f2-8e42-4d9c-aa76-dbcc0a84bb68" />

**3.3 Verify Database Connection**
# Test database connection
kubectl exec -it postgres-cluster-1 -- psql -U phoenix_user -d phoenix_app


# Run test query
\dt
\q

**Step 4: Deploy Monitoring Stack (Prometheus + Grafana)**
**4.1 Deploy kube-prometheus-stack**

# Check Prometheus stack deployment
kubectl get pods -n monitoring
kubectl get svc -n monitoring
<img width="1054" height="353" alt="image" src="https://github.com/user-attachments/assets/e2017f34-4d22-4820-9afb-2ee9404e78a9" />

**4.2 Access Grafana Dashboard**

# Grafana is exposed on NodePort 30300
open http://localhost:30300

# Credentials:
# Username: admin
# Password: admin123

<img width="1495" height="852" alt="image" src="https://github.com/user-attachments/assets/42026906-9e97-4de6-9dea-079468d0dc0b" />
<img width="1490" height="860" alt="image" src="https://github.com/user-attachments/assets/2f2e49af-f6d4-4ac4-a613-104a6b0fff87" />
<img width="1495" height="846" alt="image" src="https://github.com/user-attachments/assets/4df521f5-957b-4386-a1da-ed5ba09ee929" />
<img width="1501" height="642" alt="image" src="https://github.com/user-attachments/assets/a71266ec-176c-4dae-9fe9-2ffcd96c064d" />
<img width="1500" height="865" alt="image" src="https://github.com/user-attachments/assets/f1a7c75a-00c6-48b3-82b4-7755a7172b8d" />
<img width="1501" height="863" alt="image" src="https://github.com/user-attachments/assets/75207d0a-f0cd-4e6e-8d3b-c4dca99b09a9" />
<img width="1480" height="870" alt="image" src="https://github.com/user-attachments/assets/5615282c-2062-4066-a25d-68dcbdca47e0" />
<img width="1497" height="860" alt="image" src="https://github.com/user-attachments/assets/1f176783-20a8-47e8-b58a-f3c35882337e" />

**
Step 5: Build and Deploy Phoenix Application
5.1 Build Docker Image**
cd apps/phoenix-app/docker

# Add req dependency to mix.exs
vim mix.exs
# Add: {:req, "~> 0.5.0"}

# Update dependencies
docker run --rm -v $(pwd):/app -w /app elixir:1.18.4-slim \
  sh -c "apt-get update && apt-get install -y ca-certificates && \
  mix local.hex --force && mix deps.get"

# Build production image
docker build --target prod -t jasaulakh/phoenix-app:v1.1.0 .
docker push jasaulakh/phoenix-app:v1.1.0

<img width="1503" height="724" alt="image" src="https://github.com/user-attachments/assets/07ffda90-540e-426b-a2e7-dcfaf41a8185" />

**5.2 Deploy Phoenix App via Helm**

# Update Helm chart values
cd ../helm-chart
vim values.yaml  # Update image tag to v1.1.0

# Commit and push for GitOps
git add .
git commit -m "Deploy Phoenix app v1.1.0"
git push

# Force Flux reconciliation
flux reconcile source git flux-system
flux reconcile kustomization phoenix-app

<img width="1173" height="928" alt="image" src="https://github.com/user-attachments/assets/1de19eee-ce70-4ae2-97a2-2c34490d93d9" />

**5.3 Verify Application**
# Check pod status
kubectl get pods -n default -l app=phoenix-app
kubectl logs -n default -l app=phoenix-app --tail=20

# Access application
curl http://localhost:30080

<img width="924" height="239" alt="image" src="https://github.com/user-attachments/assets/bd65a96f-c99e-4d27-986e-0adfd3ab5859" />
<img width="1508" height="840" alt="image" src="https://github.com/user-attachments/assets/5f44b3fa-52fd-4c91-98ac-800db53f4e7b" />


**Step 6: Configure Metrics Collection**
**6.1 Enable PostgreSQL Metrics**

#Check PodMonitor creation
kubectl get podmonitor -A

# Verify metrics endpoint
kubectl port-forward postgres-cluster-1 9187:9187
curl localhost:9187/metrics | grep cnpg_collector_up

<img width="1497" height="843" alt="image" src="https://github.com/user-attachments/assets/356d9855-4532-4adb-80fa-19656270076b" />

**Step 7: Verify Complete Setup**

# Flux resources
flux get all

# Application resources
kubectl get all -n default

# Monitoring resources
kubectl get all -n monitoring

# Database operator
kubectl get all -n cnpg-system

<img width="776" height="208" alt="image" src="https://github.com/user-attachments/assets/11679383-f473-492d-b081-dd3a2f2257f3" />

**Step 8: GitOps Workflow Demonstration
8.1 Making Changes**

# Example: Scale Phoenix app
vim apps/phoenix-app/helm-chart/values.yaml
# Change replicaCount: 3

git add .
git commit -m "Scale Phoenix app to 3 replicas"
git push

# Watch reconciliation
flux get kustomization phoenix-app --watch
kubectl get pods -n default -l app=phoenix-app --watch

**Troubleshooting Commands**
# Check Flux logs
flux logs --all-namespaces

# Check Phoenix app logs
kubectl logs -n default -l app=phoenix-app

# Check PostgreSQL logs
kubectl logs postgres-cluster-1

# Check events
kubectl get events --all-namespaces --sort-by='.lastTimestamp'

# Force reconciliation
flux reconcile source git flux-system
flux reconcile kustomization --all

**Repository Structure**

jas-backgammon-gitops/
├── clusters/local/           # Flux cluster configuration
├── infrastructure/           # Infrastructure components
│   ├── cloudnativepg/       # PostgreSQL operator & cluster
│   └── monitoring/          # Prometheus & Grafana
├── apps/                    # Application deployments
│   └── phoenix-app/         # Phoenix application
│       ├── docker/          # Dockerfile & source
│       └── helm-chart/      # Custom Helm chart
└── README.md               # This file


**# To completely remove the setup**
flux uninstall --namespace=flux-system
kubectl delete namespace monitoring
kubectl delete namespace cnpg-system
kubectl delete namespace default





