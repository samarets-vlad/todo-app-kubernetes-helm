# Todo App — Kubernetes & Helm on k3s

A production-ready deployment of a full-stack Todo application on a **single-node k3s VPS**, demonstrating real-world DevOps practices: Helm charts, observability stack, automated backups, HTTPS via cert-manager, and Helm hooks.

---

## Architecture

Internet
│
▼
Traefik Ingress (HTTPS + cert-manager)
│
├──► Frontend (nginx, 2 replicas)
│ │
├──► Backend (Python Flask, 2 replicas)
│ │
└──► MySQL Database (PVC: local-path)
│
CronJob Backup ──► PVC

Monitoring: Prometheus + Loki + Grafana + Promtail + Blackbox + Node Exporter

text

## Tech Stack

| Layer | Technology |
|---|---|
| Kubernetes | k3s v1.34+ |
| Ingress | Traefik (built-in k3s) |
| TLS | cert-manager + Let's Encrypt |
| Package manager | Helm v3 |
| Backend | Python Flask (Docker: `vladdisslav/todo-backend:v1.0.0`) |
| Frontend | nginx + HTML |
| Database | MySQL 8.0 |
| Monitoring | Prometheus, Grafana, Loki, Promtail, Blackbox Exporter, Node Exporter |
| Backup | Kubernetes CronJob → PVC |

## Repository Structure

.
├── backend/ # Backend source code + Dockerfile
│ ├── app.py
│ ├── Dockerfile
│ └── requirements.txt
└── helm-charts/
├── backend/ # Flask API chart
├── database/ # MySQL chart
├── frontend/ # nginx + HTML chart
├── ingress/ # Traefik Ingress chart
├── backup/ # CronJob backup chart
└── monitoring/ # Full observability stack

text

## Prerequisites

- VPS with Ubuntu 22.04+
- Domain with A record pointing to VPS IP
- k3s installed (see below)
- Helm v3

## Quick Start

### 1. Install k3s

```bash
curl -sfL https://get.k3s.io | sh -
mkdir -p ~/.kube
sudo cp /etc/rancher/k3s/k3s.yaml ~/.kube/config
sudo chown $USER:$USER ~/.kube/config
echo 'export KUBECONFIG=~/.kube/config' >> ~/.bashrc
source ~/.bashrc
2. Install Helm
bash
curl https://raw.githubusercontent.com/helm/helm/main/scripts/get-helm-3 | bash
3. Install cert-manager
bash
kubectl apply -f https://github.com/cert-manager/cert-manager/releases/latest/download/cert-manager.yaml
kubectl wait --namespace cert-manager \
  --for=condition=ready pod \
  --selector=app.kubernetes.io/instance=cert-manager \
  --timeout=120s
4. Create namespace
bash
kubectl create namespace todo-app
5. Deploy the stack
bash
# Database
helm install database helm-charts/database \
  --namespace todo-app \
  --set auth.rootPassword="YOUR_ROOT_PASSWORD" \
  --set auth.password="YOUR_DB_PASSWORD"

# Backend
helm install backend helm-charts/backend \
  --namespace todo-app \
  --set mysql.password="YOUR_DB_PASSWORD"

# Frontend
helm install frontend helm-charts/frontend \
  --namespace todo-app

# Ingress (replace with your domain)
helm install ingress helm-charts/ingress \
  --namespace todo-app \
  --set host="your-domain.com"

# Backup
helm install backup helm-charts/backup \
  --namespace todo-app

# Monitoring
helm install monitoring helm-charts/monitoring \
  --namespace todo-app \
  --set grafana.adminPassword="YOUR_GRAFANA_PASSWORD" \
  --set certManager.email="your@email.com"
6. Verify deployment
bash
kubectl get pods -n todo-app
kubectl get ingress -n todo-app
Expected output:

text
NAME                        READY   STATUS    RESTARTS
backend-xxx                 1/1     Running   0
backend-xxx                 1/1     Running   0
frontend-xxx                1/1     Running   0
frontend-xxx                1/1     Running   0
mysql-xxx                   1/1     Running   0
7. Test locally (without domain)
bash
# Add to /etc/hosts
echo "YOUR_VPS_IP todo.local" | sudo tee -a /etc/hosts

# Test
curl -k https://todo.local
curl -k https://todo.local/api
Helm Hooks
On every helm install, a post-install Job automatically:

Waits for the backend to be ready (initContainer)

Creates initial onboarding tasks via the API

Backup
The backup chart creates a daily CronJob (00:00 UTC) that dumps the MySQL database to a PVC.

bash
# Check backup job
kubectl get cronjob -n todo-app
kubectl get pvc -n todo-app
Monitoring
Service	Access
Grafana	http://YOUR_VPS_IP:NODEPORT
Prometheus	ClusterIP (internal)
Loki	ClusterIP (internal)
bash
# Get Grafana NodePort
kubectl get svc -n todo-app | grep grafana
Security Notes
All secrets are passed via --set flags, never stored in values.yaml

See values.example.yaml in each chart for required secret fields

Backend runs as non-root user (UID 1000)

Database accessible only within cluster via ClusterIP

Local Development
bash
# Lint all charts
helm lint helm-charts/backend helm-charts/frontend helm-charts/database

# Render templates without deploying
helm template backend helm-charts/backend --set mysql.password="test"
