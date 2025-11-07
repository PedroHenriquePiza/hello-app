#  Projeto DevOps – Pipeline CI/CD completo com GitHub Actions, Docker Hub, Kubernetes e ArgoCD  


Este projeto implementa um pipeline DevOps completo utilizando:

 FastAPI  
 Docker  
 Docker Hub  
 GitHub Actions  
 GitHub Personal Access Token (PAT)  
 Kubernetes (Rancher Desktop)  
 ArgoCD com GitOps  
 Pull Requests automáticos  

O objetivo é:  
**Criar uma aplicação → empacotar com Docker → enviar ao Docker Hub → atualizar manifestos automaticamente → ArgoCD sincroniza → deploy automático no Kubernetes.**

---

#  0) Pré-requisitos 

###  Ferramentas necessárias no Windows:
- **PowerShell** 
- **Git** → https://git-scm.com/download/win  
- **Rancher Desktop**  → https://rancherdesktop.io  
- Conta no **GitHub**
- Conta no **Docker Hub**

---

#  1) Estrutura de Repositórios

Você terá **dois repositórios separados**:

1) **hello-app** → código da aplicação  
2) **hello-manifests** → deployment.yaml e service.yaml (usado pelo ArgoCD)

---

#  2) Criando o Repositório hello-app (Aplicação)

No PowerShell:

```powershell
mkdir C:\Users\(SEU USUARIO)\projects -ErrorAction SilentlyContinue
cd C:\Users\(SEU USUARIO)\projects
mkdir hello-app
cd hello-app
```

### Criar **main.py**
```powershell
notepad main.py
```

Cole:
```python
from fastapi import FastAPI

app = FastAPI()

@app.get("/")
async def root():
    return {"message": "Hello World!"}
```

### Criar **requirements.txt**
```powershell
notepad requirements.txt
```

```
fastapi
uvicorn[standard]
```

### Criar **Dockerfile**
```powershell
notepad Dockerfile
```

```
FROM python:3.11-slim
WORKDIR /app
COPY requirements.txt .
RUN pip install --no-cache-dir -r requirements.txt
COPY ./main.py .
CMD ["uvicorn", "main:app", "--host", "0.0.0.0", "--port", "8000"]
```

### Configurar o Git:
```powershell
git init
git add .
git commit -m "Initial hello-app FastAPI"
git branch -M main
```

No GitHub → criar repo **hello-app**

Conectar:
```powershell
git remote add origin https://github.com/<SEU_GH_USER>/hello-app.git
git push -u origin main
```

---

#  3) Criando o Repositório hello-manifests 

```powershell
cd C:\Users\(SEU USUARIO)\projects
mkdir hello-manifests
cd hello-manifests
```

### Criar **deployment.yaml**
```powershell
notepad deployment.yaml
```

Cole (mude `<DOCKER_USER>`):
```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: hello-app
  namespace: default
  labels:
    app: hello-app
spec:
  replicas: 1
  selector:
    matchLabels:
      app: hello-app
  template:
    metadata:
      labels:
        app: hello-app
    spec:
      containers:
      - name: hello-app
        image: <DOCKER_USER>/hello-app:latest
        ports:
        - containerPort: 8000
        readinessProbe:
          httpGet:
            path: /
            port: 8000
          initialDelaySeconds: 2
          periodSeconds: 5
```

### Criar **service.yaml**
```powershell
notepad service.yaml
```

```
apiVersion: v1
kind: Service
metadata:
  name: hello-app
  namespace: default
spec:
  selector:
    app: hello-app
  ports:
    - protocol: TCP
      port: 8080
      targetPort: 8000
```

Inicializar repo:
```powershell
git init
git add .
git commit -m "Add hello app manifests"
git branch -M main
git remote add origin https://github.com/<SEU_GH_USER>/hello-manifests.git
git push -u origin main
```

---

#  4) Instalando o ArgoCD no Kubernetes

```powershell
kubectl create namespace argocd
kubectl apply -n argocd -f https://raw.githubusercontent.com/argoproj/argo-cd/stable/manifests/install.yaml
```

Acessar:

```powershell
kubectl -n argocd port-forward svc/argocd-server 8080:443
```

Abrir o navegador em:

 **https://localhost:8080**

Login:
- **admin**
- senha:
```powershell
kubectl -n argocd get secret argocd-initial-admin-secret -o jsonpath="{.data.password}" |
% { [System.Text.Encoding]::UTF8.GetString([System.Convert]::FromBase64String($_)) }
```

---

#  5) Criando o App no ArgoCD

No ArgoCD → **NEW APP**

### CONFIGURAÇÃO:

**General**
- Application Name: `hello-app`
- Project: `default`

**Source**
- Repository URL:  
  `https://github.com/<SEU_GH_USER>/hello-manifests`
- Revision: `main`
- Path: `.`

**Destination**
- Cluster: `https://kubernetes.default.svc`
- Namespace: `default`

**Sync Policy**
- Manual 

Clique **Create** → **SYNC**

---

#  6) Testar se a aplicação roda no Kubernetes

```powershell
kubectl get pods -l app=hello-app
kubectl get svc hello-app
kubectl port-forward svc/hello-app 8080:8080
```

Acessar:

👉 http://localhost:8080/

---

#  7) Configurar CI/CD no repositório hello-app

## Criar secrets no GitHub:
Em **Settings → Secrets and Variables → Actions**:

 `DOCKER_USERNAME` (seu usuário Docker Hub)  
 `DOCKER_PASSWORD` (token do Docker Hub)  
 `MANIFESTS_REPO_PAT` (GitHub PAT com escopo repo)  

## Criar workflow:
```powershell
cd C:\Users\(SEU USUARIO)\projects\hello-app
mkdir .github -ErrorAction SilentlyContinue
mkdir .github\workflows -ErrorAction SilentlyContinue
notepad .github\workflows\ci-cd.yml
```

Cole (substituir `<SEU_GH_USER>` e `<DOCKER_USER>`):
```yaml
name: CI/CD Build → DockerHub → Update Manifests

on:
  push:
    branches: [ "main" ]
  workflow_dispatch:

env:
  IMAGE_REPO: ${{ secrets.DOCKER_USERNAME }}/hello-app

jobs:
  build-and-push:
    runs-on: ubuntu-latest

    steps:
      - name: Checkout app
        uses: actions/checkout@v3

      - name: Login to Docker Hub
        uses: docker/login-action@v2
        with:
          username: ${{ secrets.DOCKER_USERNAME }}
          password: ${{ secrets.DOCKER_PASSWORD }}

      - name: Build and push image
        uses: docker/build-push-action@v3
        with:
          context: .
          push: true
          tags: |
            ${{ env.IMAGE_REPO }}:latest
            ${{ env.IMAGE_REPO }}:${{ github.sha }}

      - name: Clone manifests repo and update image tag
        run: |
          git config --global user.email "actions@github.com"
          git config --global user.name "github-actions"

          git clone https://x-access-token:${{ secrets.MANIFESTS_REPO_PAT }}@github.com/<SEU_GH_USER>/hello-manifests.git manifests
          cd manifests

          BRANCH=update-image-${GITHUB_SHA::7}
          git checkout -b $BRANCH

          sed -i "s#image:.*#image: <DOCKER_USER>/hello-app:${GITHUB_SHA}#g" deployment.yaml

          git add deployment.yaml
          git commit -m "Update image tag to ${GITHUB_SHA}"
          git push https://x-access-token:${{ secrets.MANIFESTS_REPO_PAT }}@github.com/<SEU_GH_USER>/hello-manifests.git $BRANCH

      - name: Create Pull Request
        uses: peter-evans/create-pull-request@v4
        with:
          token: ${{ secrets.MANIFESTS_REPO_PAT }}
          path: manifests
          commit-message: "Update image tag to ${{ github.sha }}"
          title: "Update image tag to ${{ github.sha }}"
          body: "This PR updates the image tag for hello-app"
          branch: "update-image-${{ github.sha }}"
          base: "main"
```

---

#  8) Testando o Pipeline Completo 

### Alterar código:
```powershell
cd C:\Users\(SEU USUARIO)\projects\hello-app
notepad main.py
```

Mudar:
```python
return {"message": "Hello World  - Versão 2!"}
```

Salvar e enviar:
```powershell
git add main.py
git commit -m "Versão 2"
git push
```

### Acompanhar:
- GitHub → Actions: build e push da imagem
- GitHub → hello-manifests → Pull Requests → PR automático
- Fazer merge
- ArgoCD sincroniza automaticamente
- Kubernetes atualiza a aplicação

### Validar:
```powershell
kubectl port-forward svc/hello-app 8080:8080
```

Acessar:

👉 http://localhost:8080/

Deve aparecer:
```json
{"message": "Hello World - Versão 2!"}
```

