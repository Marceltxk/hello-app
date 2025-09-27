# Projeto CI/CD com GitHub Actions e ArgoCD


- **Objetivo**: Automatizar o ciclo completo de desenvolvimento, build, deploy e execução de uma aplicação FastAPI usando GitHub Actions para CI/CD, Docker Hub como registry, e ArgoCD para entrega contínua em Kubernetes local com Rancher Desktop.

## Pré-requisitos Atendidos
- Conta no GitHub (repo público)
- Conta no Docker Hub com token de acesso
- Rancher Desktop com Kubernetes habilitado
- kubectl configurado corretamente
- ArgoCD instalado no cluster local
- Git, Python 3 e Docker instalados

---

## Etapa 1: Criar a Aplicação FastAPI

### 1.1 Repositório da Aplicação
Criado repositório: `hello-app`

### 1.2 Estrutura de Arquivos Criados

#### main.py
```python
from fastapi import FastAPI
import uvicorn

app = FastAPI()

@app.get("/")
async def root():
    return {"message": "Hello World"}

if __name__ == "__main__":
    uvicorn.run(app, host="0.0.0.0", port=8000)
```

#### requirements.txt
```txt
fastapi==0.104.1
uvicorn[standard]==0.24.0
```

#### Dockerfile
```dockerfile
FROM python:3.11-slim

WORKDIR /app

COPY requirements.txt .
RUN pip install --no-cache-dir -r requirements.txt

COPY . .

EXPOSE 8000

CMD ["python", "main.py"]
```

### 1.3 Comandos para Upload Inicial
```bash
git add .
git commit -m "Initial commit for hello-app application"
git push origin main
```

## Etapa 2: Criar o GitHub Actions (CI/CD)

### 2.1 Estrutura do Workflow
Criado arquivo: `.github/workflows/ci-cd.yml`

```yaml
name: CI/CD Pipeline

on:
  push:
    branches: [ main ]
  pull_request:
    branches: [ main ]

env:
  DOCKER_IMAGE: marcelchiarelo/hello-app

jobs:
  build-and-push:
    runs-on: ubuntu-latest
    
    steps:
    - name: Checkout code
      uses: actions/checkout@v4
      
    - name: Set up Docker Buildx
      uses: docker/setup-buildx-action@v3
      
    - name: Login to Docker Hub
      uses: docker/login-action@v3
      with:
        username: ${{ secrets.DOCKER_USERNAME }}
        password: ${{ secrets.DOCKER_PASSWORD }}
        
    - name: Generate image tag
      id: tag
      run: echo "IMAGE_TAG=v$(date +%Y%m%d-%H%M%S)-${GITHUB_SHA:0:7}" >> $GITHUB_OUTPUT
        
    - name: Build and push Docker image
      uses: docker/build-push-action@v5
      with:
        context: .
        push: true
        tags: |
          ${{ env.DOCKER_IMAGE }}:${{ steps.tag.outputs.IMAGE_TAG }}
          ${{ env.DOCKER_IMAGE }}:latest
          
    - name: Update manifest repository
      if: github.ref == 'refs/heads/main'
      run: |
        # Configurar SSH
        mkdir -p ~/.ssh
        echo "${{ secrets.SSH_PRIVATE_KEY }}" > ~/.ssh/id_rsa
        chmod 600 ~/.ssh/id_rsa
        ssh-keyscan github.com >> ~/.ssh/known_hosts
        
        # Configurar Git
        git config --global user.name "GitHub Actions"
        git config --global user.email "actions@github.com"
        
        # Clonar repositório de manifestos
        git clone git@github.com:marcelchiarelo/hello-manifests.git temp-manifests
        cd temp-manifests
        
        # Atualizar imagem no deployment
        sed -i "s|image: ${{ env.DOCKER_IMAGE }}:.*|image: ${{ env.DOCKER_IMAGE }}:${{ steps.tag.outputs.IMAGE_TAG }}|g" deployment.yaml
        
        # Commit e push das alterações
        git add deployment.yaml
        git commit -m "Update image tag to ${{ steps.tag.outputs.IMAGE_TAG }}"
        git push origin main
```

### 2.2 Secrets Configurados no GitHub
- `DOCKER_USERNAME`
- `DOCKER_PASSWORD` 
- `SSH_PRIVATE_KEY`


## Etapa 3: Repositório Git com os Manifestos do ArgoCD

### 3.1 Repositório de Manifestos
Criado repositório: `hello-manifests`

### 3.2 Arquivos de Manifesto Criados

#### deployment.yaml
```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: hello-app
  namespace: default
spec:
  replicas: 2
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
        image: marcelchiarelo/hello-app:latest
        ports:
        - containerPort: 8000
        resources:
          requests:
            memory: "128Mi"
            cpu: "100m"
          limits:
            memory: "256Mi"
            cpu: "200m"
        readinessProbe:
          httpGet:
            path: /
            port: 8000
          initialDelaySeconds: 10
          periodSeconds: 5
        livenessProbe:
          httpGet:
            path: /
            port: 8000
          initialDelaySeconds: 15
          periodSeconds: 10
```

#### service.yaml
```yaml
apiVersion: v1
kind: Service
metadata:
  name: hello-app-service
  namespace: default
spec:
  selector:
    app: hello-app
  ports:
    - protocol: TCP
      port: 80
      targetPort: 8000
      name: http
  type: ClusterIP
```

### 3.3 Upload dos Manifestos
```bash
git add .
git commit -m "Initial Kubernetes manifests"
git push origin main
```


## Etapa 4: Criar App no ArgoCD

### 4.1 Instalação do ArgoCD
```bash
kubectl create namespace argocd
kubectl apply -n argocd -f https://raw.githubusercontent.com/argoproj/argo-cd/stable/manifests/install.yaml
```

### 4.2 Acesso ao ArgoCD
```bash
kubectl port-forward svc/argocd-server -n argocd 8080:443
```

### 4.3 Configuração da Aplicação no ArgoCD
- **Application Name**: hello-app
- **Project**: default
- **Sync Policy**: Automatic
- **Repository URL**: https://github.com/marcelchiarelo/hello-manifests.git
- **Revision**: HEAD
- **Path**: . (root do repositório)
- **Cluster URL**: https://kubernetes.default.svc
- **Namespace**: default

**Screenshot do ArgoCD com a aplicação criada:**
```
[INSERIR SCREENSHOT DO ARGOCD AQUI]
```

---

## Etapa 5: Evidências de Funcionamento

### 5.1 Build e Push da Imagem no Docker Hub

**Evidência do GitHub Actions executando com sucesso:**
```
[INSERIR SCREENSHOT DO GITHUB ACTIONS AQUI]
```

**Evidência da imagem no Docker Hub:**
```
[INSERIR SCREENSHOT DO DOCKER HUB AQUI]
```

**Print do comando docker push:**
```bash
docker push marcelchiarelo/hello-app:latest
```

![Docker push](images/docker_push.png)



### 5.2 Atualização Automática dos Manifestos

**Print do histórico de commits no repositório de manifestos:**
```bash
git log --oneline -5
```
```
[INSERIR PRINT DO GIT LOG AQUI]
```

**Print mostrando a imagem atualizada no deployment:**
```bash
cat deployment.yaml | grep image
```
```
[INSERIR PRINT DO GREP IMAGE AQUI]
```

### 5.3 Aplicação Rodando no Kubernetes

**Print do kubectl get pods:**
```bash
kubectl get pods -l app=hello-app -o wide
```

![Get pods](images/get_pods.png)


**Print do status dos deployments:**
```bash
kubectl get deployments
```
```
[INSERIR PRINT DO KUBECTL GET DEPLOYMENTS AQUI]
```

**Print do status dos services:**
```bash
kubectl get services
```
```
[INSERIR PRINT DO KUBECTL GET SERVICES AQUI]
```

### 5.4 ArgoCD Sincronizado

**Print do status da aplicação no ArgoCD:**
```bash
kubectl get applications -n argocd
```
```
[INSERIR PRINT DO KUBECTL GET APPLICATIONS AQUI]
```

**Screenshot do ArgoCD mostrando aplicação sincronizada:**

![ArgoCD Sincronizado](images/argo_CD_sync.png)


### 5.5 Teste da Aplicação

**Print do port-forward:**
```bash
kubectl port-forward service/hello-app-service 8080:80
```
```
[INSERIR PRINT DO PORT-FORWARD AQUI]
```

**Resposta da aplicação via curl:**
```bash
curl http://localhost:8080/
```

![Curl response](images/curl_response.png)

**Resposta da aplicação via navegador:**

![Browser response](images/browser_response.png)

---

## Etapa 6: Teste do Pipeline Completo

### 6.1 Alteração do Código
Modificação realizada no arquivo `main.py`:
```python
@app.get("/")
async def root():
    return {"message": "Hello DevOps World!"}  # Mensagem alterada
```

### 6.2 Commit da Alteração
```bash
git add .
git commit -m "Update message to test CI/CD pipeline"
git push origin main
```

**Print do commit da alteração:**
```
[INSERIR PRINT DO COMMIT AQUI]
```

### 6.3 Evidências da Atualização Automática

**Print do GitHub Actions executando novamente:**
```
[INSERIR SCREENSHOT DO NOVO BUILD AQUI]
```

**Print dos pods sendo reiniciados:**
```bash
kubectl get events --sort-by=.metadata.creationTimestamp | grep hello-app | tail -10
```
```
[INSERIR PRINT DOS EVENTS AQUI]
```

**Print da nova resposta da aplicação:**
```bash
curl http://localhost:8080/
```
```
[INSERIR PRINT DA NOVA RESPOSTA AQUI]
```

---

## Resultados Obtidos

### Entregáveis Concluídos

1. **Link do repositório Git com a aplicação FastAPI + Dockerfile + GitHub Actions**
   - Repositório: [LINK DA APLICAÇÃO]

2. **Link do repositório com os manifests (deployment.yaml, service.yaml)**
   - Repositório: [LINK DOS MANIFESTOS]

3. **Evidência de build e push da imagem no Docker Hub**
   - Screenshots e prints fornecidos acima

4. **Evidência de atualização automática dos manifestos com a nova tag da imagem**
   - Git log e prints dos commits automáticos fornecidos

5. **Captura de tela do ArgoCD com a aplicação sincronizada**
   - Screenshots fornecidos acima

6. **Print do kubectl get pods com a aplicação rodando**
   - Print fornecido acima

7. **Print da resposta da aplicação via curl ou navegador**
   - Print fornecido acima

---

## Conclusão

O projeto foi implementado com sucesso, demonstrando um pipeline completo de CI/CD que:

- **Automatiza o build** da aplicação FastAPI em container Docker
- **Publica automaticamente** as imagens no Docker Hub
- **Atualiza automaticamente** os manifestos Kubernetes via GitHub Actions
- **Sincroniza automaticamente** o deploy via ArgoCD seguindo práticas GitOps
- **Permite atualizações contínuas** da aplicação de forma totalmente automatizada

O pipeline implementado segue as melhores práticas de DevOps e é representativo do que é utilizado em ambientes de produção em empresas modernas.

---

## Comandos de Limpeza (Opcional)

Para remover o ambiente após os testes:

```bash
# Remover aplicação do ArgoCD
kubectl delete application hello-app -n argocd

# Remover recursos do Kubernetes
kubectl delete deployment hello-app
kubectl delete service hello-app-service

# Remover ArgoCD (opcional)
kubectl delete namespace argocd
```# Test new SSH key
