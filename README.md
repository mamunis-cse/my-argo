# My Argo CD Frontend Project

This repository demonstrates a **declarative GitOps workflow using Argo CD** to deploy a frontend application on Kubernetes.

---

## 📁 Project Structure

```
my-argo/
├── frontend/
│   └── deployment.yaml # Kubernetes Deployment + Service
└── README.md           # Project documentation
```

---

## ⚡ Features

- Deploys frontend app using **Kubernetes Deployment & Service**
- Fully managed with **Argo CD GitOps**
- Supports **automatic sync, self-healing, and pruning**
- Works in **home/lab environments** with NodePort

---

## 🛠️ Prerequisites

- Kubernetes cluster (minikube, kubeadm, or cloud)
- `kubectl` configured to access your cluster
- Argo CD installed in `argocd` namespace
- GitHub repository with frontend manifests

---

## 🚀 Setup Guide

### 1️⃣ Install Argo CD

```bash
kubectl create namespace argocd
kubectl apply -n argocd -f https://raw.githubusercontent.com/argoproj/argo-cd/stable/manifests/install.yaml
```

### 2️⃣ Expose Argo CD (Optional NodePort)

```bash
kubectl edit svc argocd-server -n argocd
```

- Change type: `ClusterIP` → `NodePort`  
- Add `nodePort: 30099`  
- Access UI: `http://<NODE_IP>:30099`  

**Default credentials:**

```bash
kubectl -n argocd get secret argocd-initial-admin-secret -o jsonpath="{.data.password}" | base64 --decode
```

### 3️⃣ Create Argo CD Project

Create `ecom-site-project.yaml`:

```yaml
apiVersion: argoproj.io/v1alpha1
kind: AppProject
metadata:
  name: ecom-site
  namespace: argocd
spec:
  sourceRepos:
    - '*'
  destinations:
    - namespace: '*'
      server: 'https://kubernetes.default.svc'
  clusterResourceWhitelist:
    - group: '*'
      kind: '*'
```

Apply it:

```bash
kubectl apply -f ecom-site-project.yaml
```

### 4️⃣ Add Git Repo Secrets

Create `argo-secret.yaml`:

```yaml
apiVersion: v1
kind: Secret
metadata:
  name: ecom-deployment-repo
  namespace: argocd
  labels:
    argocd.argoproj.io/secret-type: repository
stringData:
  type: git
  url: https://github.com/your-username/my-argo.git
---
apiVersion: v1
kind: Secret
metadata:
  name: ecom-repo-creds
  namespace: argocd
  labels:
    argocd.argoproj.io/secret-type: repo-creds
stringData:
  type: git
  url: https://github.com/your-username/my-argo
  username: your-username
  password: <GITHUB_PERSONAL_ACCESS_TOKEN>
```

Apply it:

```bash
kubectl apply -f argo-secret.yaml
```

> ⚠️ Replace `your-username` with your GitHub username  
> ⚠️ Replace `<GITHUB_PERSONAL_ACCESS_TOKEN>` with a valid GitHub token

### 5️⃣ Create ApplicationSet

Create `application-set.yaml`:

```yaml
apiVersion: argoproj.io/v1alpha1
kind: ApplicationSet
metadata:
  name: ecom-applicationset
  namespace: argocd
  annotations:
    pref.argocd.argoproj.io/defaultview: list
spec:
  generators:
  - list:
      elements:
      - appname: frontend
  template:
    metadata:
      name: 'ecom-dev-{{appname}}'
    spec:
      project: ecom-site
      source:
        repoURL: https://github.com/your-username/my-argo.git
        targetRevision: dev
        path: '{{appname}}'
      destination:
        server: https://kubernetes.default.svc
        namespace: my-demo
      syncPolicy:
        automated:
          selfHeal: true
          prune: true
        syncOptions:
          - PrunePropagationPolicy=foreground
          - resources-health-timeout-seconds=10
```

Apply it:

```bash
kubectl apply -f application-set.yaml
```

### ✅ Result

- Argo CD automatically deploys the frontend app from your Git repo  
- Git → Kubernetes sync is fully automated  
- Self-healing and pruning are enabled  
- Access the frontend app via NodePort (or configure Ingress/LoadBalancer)

---

### ⚙️ Notes

- NodePort is optional; use Ingress or LoadBalancer in production  
- Make sure the `dev` branch exists in your Git repo  
- Ensure `frontend/deployment.yaml` defines both Deployment and Service
