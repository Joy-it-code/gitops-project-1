# GitOps Configuration Management with ArgoCD, Helm, Kustomize, and Vault

## Project Overview

This mini project demonstrates GitOps-based configuration management using **ArgoCD**, **Helm**, **Kustomize**, and **Vault** for secret management. The setup simulates real-world GitOps workflows with automated deployments, environment-based configurations, and secure secret injection.


## Project Structure
```bash
gitops-project
├── helm-app/
│   └── my-app/                 
│       ├── Chart.yaml
│       ├── values.yaml
│       └── templates/
├── kustomize-app/
│   ├── base/                    
│   └── overlays/
│       ├── dev/
│       └── prod/
├── argocd-apps/                  
├── vault/                     
├── .gitignore
└── README.md
```

---


## Objectives
- Integrate **Helm** charts with ArgoCD

- Manage environments with **Kustomize overlays**

- Secure secrets using **Kubernetes, Vault,** and **AWS Secrets Manager**

- Customize **resource management** and **sync policies** in ArgoCD


---

## Tools Required

- Docker Desktop (w/ Kubernetes)

- kubectl

- Helm

- Kustomize

- ArgoCD CLI

- Vault CLI

- Git

- VS Code (with Kubernetes, YAML, and Git plugins)


## Verify Installed Tools



```bash
helm version
kubectl version --client
kustomize version
vault -v
aws --version
git --version
argocd version --client
```
![](./img/1a.installed.tools.png)




 ## 1: Project Folder Setup
```bash
mkdir -p gitops-project-1/{helm-app/my-app/{templates,charts},kustomize-app/my-app/{base,overlays/dev,overlays/prod},argocd-apps}
cd gitops-project-1
```


## 2: ArgoCD Setup on Local Kubernetes

### Check cluster nodes:
```bash
kubectl cluster-info
kubectl get nodes
```


###  Install ArgoCD
```bash
kubectl create namespace argocd
kubectl apply -n argocd -f https://raw.githubusercontent.com/argoproj/argo-cd/stable/manifests/install.yaml
```



### Access ArgoCD UI
```bash
kubectl port-forward svc/argocd-server -n argocd 8080:443
```
**Visit: `http://localhost:8080`**


### Login to ArgoCD CLI and UI

- Get the default admin password:
```bash
kubectl get secret argocd-initial-admin-secret -n argocd -o jsonpath="{.data.password}" | base64 -d && echo
argocd login localhost:8080 --username admin --password <paste-password>
```

**Login using:**

- Username: admin
- Password: (from the above command)


### Check ArgoCD pods:
```bash
kubectl get pods -n argocd
```

### Check node status:
```bash
kubectl get nodes
```
![](./img/1b.get.pods.nodes.png)


### Create and Push Repo to GitHub
```bash
git init
git remote add origin https://github.com/YOUR_USERNAME/gitops-project.git
git add .
git commit -m "Initial project structure"
git push -u origin main
```

### Create .gitignore File
```bash
*.log
*.tmp
*.bak
.vault-token
__pycache__/
.DS_Store
.idea/
*.swp
.env
```


## 3: HELM: APP DEPLOYMENT

### Helm App Setup
```bash
helm create helm-app/my-app
```

### Clean up default chart:
```bash
rm -rf helm-app/my-app/tests
```

### Update files:

`my-app/values.yaml`
```bash
replicaCount: 1
image:
  repository: nginx
  tag: latest
```


### Test Helm Chart Locally:
```bash
helm lint .
```


### Commit to Git
```bash
git init
git add .
git commit -m "first commit"
git branch -M main
git remote add origin <YOUR_REPO>
git push -u origin main
```

### Deploy Helm Chart in ArgoCD
```bash
mkdir -p argocd-apps && nano argocd-apps/helm-my-app.yaml
```

### Paste
```bash
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: helm-my-app
  namespace: argocd
spec:
  project: default
  source:
    repoURL: https://github.com/<your-user>/<repo>.git
    targetRevision: main
    path: helm-app/my-app
    helm:
      valueFiles:
        - values.yaml
  destination:
    server: https://kubernetes.default.svc
    namespace: default
  syncPolicy:
    automated:
      selfHeal: true
      prune: true
```

### Apply it:
```bash
kubectl apply -f argocd-apps/helm-my-app.yaml
```


### Test Helm App
```bash
kubectl get all -l app.kubernetes.io/name=my-app
```


### Verify the Application in ArgoCD
- Check it via CLI:
```bash
kubectl get applications -n argocd
```

- Or Open the ArgoCD UI
```bash
kubectl port-forward svc/argocd-server -n argocd 8080:443
```
**Check on Browser `https://localhost:8080`**


### push to GitHub
```bash
git add argocd-apps/helm-my-app.yaml
git commit -m "argocd-apps/helm-my-app.yaml"
git push origin main
```

## 4: Kustomize App Setup
