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
git add README.md
git commit -m "Initial project structure"
git branch -M main
git remote add origin https://github.com/YOUR_USERNAME/gitops-project.git
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

### Create Helm Chart 
```bash
helm create helm-app/my-app
```

### Clean up default chart:
```bash
rm -rf helm-app/my-app/tests
```

### Update files:

### Edit `helm-app/my-app/Chart.yaml`:

```bash
apiVersion: v2
name: my-app
description: A Helm chart for deploying My App
type: application
version: 0.1.0
appVersion: "1.0"
```


### Edit `helm-app/my-app/values.yaml`
```bash
replicaCount: 1

image:
  repository: nginx
  tag: "latest"
  pullPolicy: IfNotPresent

service:
  type: ClusterIP
  port: 80

ingress:
  enabled: false
  className: ""
  annotations: {}
  hosts:
    - host: my-app.local
      paths:
        - path: /
          pathType: Prefix
  tls: []

resources: {}

nodeSelector: {}

tolerations: []

affinity: {}

autoscaling:
  enabled: false
  minReplicas: 1
  maxReplicas: 3
  targetCPUUtilizationPercentage: 80
  targetMemoryUtilizationPercentage: 75
serviceAccount:
  create: true
  name: ""
```


### Edit `templates/deployment.yaml`
```bash
apiVersion: apps/v1
kind: Deployment
metadata:
  name: {{ .Release.Name }}
  labels:
    app: {{ .Chart.Name }}
spec:
  replicas: {{ .Values.replicaCount }}
  selector:
    matchLabels:
      app: {{ .Chart.Name }}
  template:
    metadata:
      labels:
        app: {{ .Chart.Name }}
    spec:
      containers:
        - name: {{ .Chart.Name }}
          image: "{{ .Values.image.repository }}:{{ .Values.image.tag }}"
          imagePullPolicy: {{ .Values.image.pullPolicy }}
          ports:
            - containerPort: {{ .Values.service.port }}
          resources: {{- toYaml .Values.resources | nindent 12 }}
```


### Edit `templates/service.yaml`
```bash
apiVersion: v1
kind: Service
metadata:
  name: {{ .Release.Name }}
spec:
  type: {{ .Values.service.type }}
  selector:
    app: {{ .Chart.Name }}
  ports:
    - port: {{ .Values.service.port }}
      targetPort: {{ .Values.service.port }}
```



### Edit `templates/ingress.yaml`
```bash
{{- if .Values.ingress.enabled }}
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: {{ .Release.Name }}
  annotations:
    {{- range $key, $value := .Values.ingress.annotations }}
    {{ $key }}: {{ $value | quote }}
    {{- end }}
spec:
  rules:
    {{- range .Values.ingress.hosts }}
    - host: {{ .host }}
      http:
        paths:
          {{- range .paths }}
          - path: {{ .path }}
            pathType: {{ .pathType }}
            backend:
              service:
                name: {{ $.Release.Name }}
                port:
                  number: {{ $.Values.service.port }}
          {{- end }}
    {{- end }}
{{- end }}
```





### Test Helm Chart Locally:
```bash
cd helm-app/my-app
helm lint .
helm package helm-app/my-app
```




### Commit to Git
```bash
git add helm-app/my-app
git commit -m "Add Helm chart for my-app"
git push
```

### Deploy Helm Chart in ArgoCD

### Create `argocd-apps/helm-my-app.yaml`:

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


### Push to GitHub
```bash
git add argocd-apps/helm-my-app.yaml
git commit -m "argocd-apps/helm-my-app.yaml"
git push origin main
```



### Verify the Application in ArgoCD
- Check it via CLI:
```bash
kubectl get applications -n argocd
```

### Verify Deployment
```bash
kubectl get deployments
kubectl get pods
kubectl get svc
```
![](./img/1c.get.app.svc.png)



- Or Open the ArgoCD UI
```bash
kubectl port-forward svc/argocd-server -n argocd 8080:443
```
**Check on Browser `https://localhost:8080`**
![](./img/1d.healthy.helm.pg1.png)
![](./img/1e.healthy.helm.pg2.png)


### Access NGINX app (helm-my-app):
```bash
kubectl port-forward svc/helm-my-app 8081:80
```
**Then go to:`http://localhost:8081`**
![](./img/1f.local.host.8081.png)



## 4: Kustomize App Setup

- Create Kustomize base:
```bash
mkdir -p kustomize-app/my-app/base
```

### `base/kustomization.yaml`
```bash
resources:
  - deployment.yaml
  - service.yaml
```

### base/deployment.yaml
```bash
apiVersion: apps/v1
kind: Deployment
metadata:
  name: kustomize-app
spec:
  replicas: 1
  selector:
    matchLabels:
      app: kustomize-app
  template:
    metadata:
      labels:
        app: kustomize-app
    spec:
      containers:
        - name: nginx
          image: nginx:latest
          ports:
            - containerPort: 80
```


### `base/service.yaml`
```bash
apiVersion: v1
kind: Service
metadata:
  name: kustomize-app
spec:
  type: ClusterIP
  selector:
    app: kustomize-app
  ports:
    - port: 80
      targetPort: 80
```

## 5: Add Overlays

### `overlays/dev/kustomization.yaml`
```bash
resources:
  - ../../base
patchesStrategicMerge:
  - patch.yaml
```

### `overlays/dev/patch.yaml`
```bash
apiVersion: apps/v1
kind: Deployment
metadata:
  name: kustomize-app
spec:
  replicas: 2
```

**Duplicate `overlays/dev` for `overlays/prod` and set `replicas: 3`**

### overlays/prod/kustomization.yaml
```bash
resources:
  - ../../base
patchesStrategicMerge:
  - patch.yaml
```

### overlays/prod/patch.yaml
```bash
apiVersion: apps/v1
kind: Deployment
metadata:
  name: kustomize-app
spec:
  replicas: 3
```

### Test and Commit Kustomize Configurations:

```bash
cd kustomize-app/my-app/base
kustomize build .
git add kustomize-app/my-app
git commit -m "Add Kustomize base and overlays"
git push
```


## Deploy Kustomize App via ArgoCD

- Create argocd-apps/kustomize-my-app-dev.yaml:

```bash
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: kustomize-my-app-dev
  namespace: argocd
spec:
  project: default
  source:
    repoURL: https://github.com/YOUR_USERNAME/gitops-project.git
    targetRevision: main
    path: kustomize-app/my-app/overlays/dev
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
kubectl apply -f argocd-apps/kustomize-my-app-dev.yaml
```



### Run in the Right Environment

### Enter WSL Ubuntu
```bash
wsl
```

### Verify Docker & Kind 
```bash
docker --version
kind version
```


### Create the Kind Cluster
```bash
kind create cluster
```


### Confirm it’s running
```bash
kubectl get nodes
```


### Load your Docker image
```bash
mkdir -p ~/images
cd ~/images
docker save nginx:1.25.2 -o nginx.tar
kind load docker-image nginx:1.25.2
```



### Update Deployment to Use That Image
```bash
image: nginx:1.25.2
```



### Push to GitHub
```bash
git add .
git commit -m "Use nginx:1.25.2 for Kustomize app"
git push
```


### Confirm in ArgoCD:
```bash
kubectl get applications -n argocd
```



### Verify Deployment 
```bash
kubectl get pods
kubectl logs <pod-name>
```
![](./img/2a.get.pod.kust.png)


Or

###  Port Forward to Test Locally:
- Port Forward One of Your Running Pods

- Forward from helm-my-app:
```bash
kubectl port-forward pod/helm-my-app-684797d6d4-cwvkt 8080:80
```
**Then open:`http://localhost:8080`**
![](./img/2b.port.forward.helm.png)



- Forward from `kustomize-app:`
```bash
kubectl port-forward pod/kustomize-app-6f45c44745-hg4r4 8081:80
```
**Then go to: `http://localhost:8081`**
![](./img/2c.port.forward.kust.png)



### ArgoCD Sync Check

- Check sync status:
```bash
kubectl get applications -n argocd
```
![](./img/2d.get.argocd.health.png)


### Log into ArgoCD UI:
```bash
kubectl port-forward svc/argocd-server -n argocd 8080:443
```
**Then open: `https://localhost:8080`**



### Push to GitHub

```bash
git status
git add .
git commit -m "Deploy nginx with ArgoCD and port-forward enabled"
git push origin main
```


## 6: Secrets Management in Kubernetes
- Create Kubernetes Secret:
```bash
kubectl create secret generic my-secret --from-literal=password=mypassword
```

### Reference in Deployment Manifest:
```bash
env:
  - name: DB_PASSWORD
    valueFrom:
      secretKeyRef:
        name: my-secret
        key: password
```


### Validate and Push to GitHub:
```bash
cd helm-app/my-app
helm lint .
cd ../..
git add .
git commit -m "Use Vault reference for secrets"
git push origin main
```