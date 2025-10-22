# ArgoCD + Argo Rollouts GitOps Repo (Minikube)

This repository contains a ready-made GitOps layout with **Kustomize** overlays that demonstrates both **Canary** and **Blue-Green** deployments using **Argo Rollouts**, and is intended to be used with **ArgoCD** in a Minikube cluster. It also includes an `analysisTemplate` that integrates with **Prometheus** for automated metric-based promotion/rollback.

---

## Repo tree

```
argocd-argo-rollouts-repo/
├── base/
│   ├── canary-rollout.yaml
│   ├── canary-service.yaml
│   ├── bluegreen-rollout.yaml
│   ├── bluegreen-service.yaml
│   ├── ingress.yaml
│   ├── analysis-prometheus.yaml
│   └── kustomization.yaml
├── overlays/
│   └── dev/
│       ├── kustomization.yaml
│       ├── patch-image-canary.yaml
│       ├── patch-image-bluegreen.yaml
│       ├── patch-service-canary.yaml
│       ├── patch-ingress.yaml
│       └── patch-analysis.yaml
└── argocd/
    └── app.yaml
```

---

> **Note:** The full contents of each file are below. Apply them directly to your repo. The overlays show how ArgoCD + Kustomize will detect updates to *any* of the tracked YAMLs (image, services, ingress, analysis) and Argo Rollouts will perform progressive delivery.

---

## Commands and Setup Guide

Below is the complete command set — from installation and verification to cleanup.

### 🧱 1. Start Minikube

```bash
minikube start --nodes=3 --cpus=4 --memory=8192 --driver=docker
kubectl get nodes
```

### 🧱 2. Enable Ingress Controller

```bash
minikube addons enable ingress
kubectl get pods -n ingress-nginx
```

### 🧱 3. Install ArgoCD

```bash
kubectl create namespace argocd
kubectl apply -n argocd -f https://raw.githubusercontent.com/argoproj/argo-cd/stable/manifests/install.yaml
kubectl get pods -n argocd
```

Access UI:

```bash
kubectl port-forward svc/argocd-server -n argocd 8080:443
```

Get initial password:

```bash
kubectl -n argocd get secret argocd-initial-admin-secret -o jsonpath="{.data.password}" | base64 -d
```

### 🧱 4. Install Argo Rollouts

```bash
kubectl create namespace argo-rollouts
kubectl apply -n argo-rollouts -f https://github.com/argoproj/argo-rollouts/releases/latest/download/install.yaml
kubectl get pods -n argo-rollouts
kubectl get crds | grep rollout
```

CLI install:

```bash
curl -LO https://github.com/argoproj/argo-rollouts/releases/latest/download/kubectl-argo-rollouts-linux-amd64
chmod +x kubectl-argo-rollouts-linux-amd64
sudo mv kubectl-argo-rollouts-linux-amd64 /usr/local/bin/kubectl-argo-rollouts
```

Verify plugin:

```bash
kubectl argo rollouts version
```

### 🧱 5. Install Prometheus for Metrics

```bash
helm repo add prometheus-community https://prometheus-community.github.io/helm-charts
helm repo update
helm install prometheus prometheus-community/kube-prometheus-stack --namespace monitoring --create-namespace
kubectl get pods -n monitoring
```

### 🧱 6. Deploy ArgoCD Application

```bash
kubectl apply -f argocd/app.yaml -n argocd
kubectl get applications -n argocd
```

Sync from ArgoCD UI or CLI:

```bash
argocd app list
argocd app sync myapp-dev
```

### 🧱 7. Verify Rollouts and Analysis

Check rollout resources:

```bash
kubectl get rollout
kubectl describe rollout myapp-canary
kubectl argo rollouts get rollout myapp-canary --watch
```

Check analysis:

```bash
kubectl get analysistemplates
kubectl get analysisruns
kubectl describe analysisrun <name>
```

Check services and ingress:

```bash
kubectl get svc
kubectl get ingress
```

Check Prometheus integration:

```bash
kubectl port-forward svc/prometheus-operated -n monitoring 9090:9090
# Open http://localhost:9090 and verify query from analysis template
```

### 🧱 8. Observe Canary / BlueGreen Rollouts

Trigger rollout update (image tag change):

```bash
git add overlays/dev/patch-image-canary.yaml
git commit -m "Update image tag"
git push origin main
```

ArgoCD auto-syncs → rollout starts → monitor:

```bash
kubectl argo rollouts get rollout myapp-canary --watch
```

Promote / abort manually (if autoPromotion=false):

```bash
kubectl argo rollouts promote myapp-bluegreen
kubectl argo rollouts abort myapp-bluegreen
```

View rollout dashboard:

```bash
kubectl argo rollouts dashboard
# Open http://localhost:3100
```

### 🧱 9. Cleanup and Uninstall

Stop rollout and delete app:

```bash
kubectl delete application myapp-dev -n argocd
```

Delete Argo Rollouts:

```bash
kubectl delete namespace argo-rollouts
```

Delete ArgoCD:

```bash
kubectl delete namespace argocd
```

Delete Prometheus:

```bash
helm uninstall prometheus -n monitoring
kubectl delete namespace monitoring
```

Delete app resources:

```bash
kubectl delete namespace default --grace-period=0 --force
```

Stop and delete Minikube:

```bash
minikube stop
minikube delete --all --purge
```

Check cleanup:

```bash
kubectl get all --all-namespaces
minikube status
```

---

All commands above ensure a full install, verification, usage, and clean removal of ArgoCD + Argo Rollouts setup for GitOps progressive delivery on Minikube.
