# Todo App — Kubernetes & Helm on k3s

A production-ready deployment of a full-stack Todo application on a **single-node k3s VPS**, demonstrating real-world DevOps practices: Helm charts, observability stack, automated backups, HTTPS via cert-manager, and Helm hooks.

---

## Tech Stack

| Layer | Technology |
|---|---|
| Kubernetes | k3s v1.34+ |
| Ingress | Traefik (built-in k3s) |
| TLS | cert-manager + Let's Encrypt |
| Package manager | Helm v3 |
| Backend | Python Flask — `vladdisslav/todo-backend:v1.0.0` |
| Frontend | nginx + HTML |
| Database | MySQL 8.0 |
| Monitoring | Prometheus, Grafana, Loki, Promtail, Blackbox, Node Exporter |
| Backup | Kubernetes CronJob → PVC |

---

## Repository Structure

```text
.
├── backend/               # Source code + Dockerfile
│   ├── app.py
│   ├── Dockerfile
│   └── requirements.txt
└── helm-charts/
    ├── backend/           # Flask API chart
    ├── database/          # MySQL chart
    ├── frontend/          # nginx chart
    ├── ingress/           # Traefik Ingress chart
    ├── backup/            # CronJob backup chart
    └── monitoring/        # Prometheus, Grafana, Loki stack
```

## Quick Start
### 1. Install k3s
```bash
curl -sfL https://get.k3s.io | sh -
mkdir -p ~/.kube
sudo cp /etc/rancher/k3s/k3s.yaml ~/.kube/config
sudo chown $USER:$USER ~/.kube/config
echo 'export KUBECONFIG=~/.kube/config' >> ~/.bashrc && source ~/.bashrc
```
### 2. Install Helm
```bash
curl https://raw.githubusercontent.com/helm/helm/main/scripts/get-helm-3 | bash
```
### 3. Install cert-manager
```bash
kubectl apply -f https://github.com/cert-manager/cert-manager/releases/latest/download/cert-manager.yaml
```
### 4. Deploy the stack
```bash
kubectl create namespace todo-app

helm install database helm-charts/database --namespace todo-app \
  --set auth.rootPassword="YOUR_ROOT_PASSWORD" \
  --set auth.password="YOUR_DB_PASSWORD"

helm install backend helm-charts/backend --namespace todo-app \
  --set mysql.password="YOUR_DB_PASSWORD"

helm install frontend helm-charts/frontend --namespace todo-app

helm install ingress helm-charts/ingress --namespace todo-app \
  --set host="your-domain.com"

helm install backup helm-charts/backup --namespace todo-app

helm install monitoring helm-charts/monitoring --namespace todo-app \
  --set grafana.adminPassword="YOUR_GRAFANA_PASSWORD" \
  --set certManager.email="your@email.com"
```
### 5. Verify
```bash
kubectl get pods,svc,ingress -n todo-app
```
## Helm Hooks
On every helm install a post-install Job automatically:

Waits for the backend to be ready (initContainer with retry loop)

Creates initial onboarding tasks via the REST API

## Database Backup
Daily CronJob (00:00 UTC) dumps MySQL to a PVC.

```bash
kubectl get cronjob -n todo-app
kubectl get pvc -n todo-app
```
## Monitoring
Service	Access
Grafana	http://YOUR_VPS_IP:NODEPORT
Prometheus	ClusterIP (internal)
Loki	ClusterIP (internal)
## Security
Secrets passed via --set, never stored in values.yaml

See values.example.yaml in each chart for required fields

Backend runs as non-root (UID 1000)

Database accessible only within cluster (ClusterIP)

## Local Development
```bash
# Lint all charts
helm lint helm-charts/backend helm-charts/frontend helm-charts/database

# Render templates without deploying
helm template backend helm-charts/backend --set mysql.password="test"