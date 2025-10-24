# 🚀 **Projeto CI/CD com GitHub Actions**

---

## ***📝 Descrição do Projeto***
O objetivo principal é implementar um pipeline completo de **Integração Contínua e Entrega Contínua (CI/CD)** utilizando:

* **GitHub Actions** para automação de build e push da imagem Docker;
* **Docker Hub** como repositório de imagens;
* **ArgoCD** para entrega contínua em **Kubernetes local (Rancher Desktop)**.

A aplicação desenvolvida é uma API simples com **FastAPI**, que retorna uma mensagem de saudação.

> 🔗 Este projeto é composto por **dois repositórios GitHub**:
>
> * **Aplicação e pipeline CI/CD:** [projeto-ci-cd_hello-app](https://github.com/vicamaral14/projeto-ci-cd_hello-app)
> * **Manifests do ArgoCD:** [projeto-ci-cd_manifests](https://github.com/vicamaral14/projeto-ci-cd_manifests)

---

## ***⚙️ 1. Pré-requisitos***

Antes de começar, você precisa ter instalado e configurado:

| Ferramenta          | Função                         | Observação                        |
| ------------------- | ------------------------------ | --------------------------------- |
| **GitHub**          | Hospedar o código e a pipeline | Repositório público               |
| **Docker Hub**      | Armazenar imagens Docker       | Crie um token de acesso           |
| **Rancher Desktop** | Cluster Kubernetes local       | Habilite o Kubernetes             |
| **kubectl**         | Gerenciar o cluster            | Verifique com `kubectl get nodes` |
| **ArgoCD**          | Automação GitOps de deploy     | Deve estar instalado no cluster   |
| **Python 3**        | Rodar a aplicação FastAPI      | Recomendado Python 3.10+          |
| **Docker**          | Build e execução de containers | Necessário para a imagem          |

---

## ***🧱 2. Criando a Aplicação FastAPI***

1. Crie o repositório [projeto-ci-cd_hello-app](https://github.com/vicamaral14/projeto-ci-cd_hello-app).

2. Adicione o arquivo `main.py`:

```python
from fastapi import FastAPI

app = FastAPI()

@app.get("/")
async def root():
    return {"message": "Hello World"}
```

3. Crie o arquivo **Dockerfile** no mesmo diretório:

```dockerfile
FROM python:3.10-slim
WORKDIR /app
COPY . .
RUN pip install fastapi uvicorn
CMD ["uvicorn", "main:app", "--host", "0.0.0.0", "--port", "8080"]
```

4. Teste localmente:

```bash
docker build -t hello-app:latest .
docker run -d -p 8080:8080 hello-app
```

Acesse no navegador:
👉 [http://localhost:8080/](http://localhost:8080/)

---

## ***⚙️ 3. Configurando o GitHub Actions (CI/CD)***

1. No repositório da aplicação, crie a pasta:

```bash
.github/workflows
```

2. Dentro dela, adicione o arquivo `ci-cd.yml` com o seguinte conteúdo:

```yaml
name: CI/CD Pipeline

on:
  push:
    branches:
      - main

jobs:
  build:
    runs-on: ubuntu-latest

    steps:
      - name: Checkout código
        uses: actions/checkout@v3

      - name: Login no Docker Hub
        uses: docker/login-action@v2
        with:
          username: ${{ secrets.DOCKER_USERNAME }}
          password: ${{ secrets.DOCKER_PASSWORD }}

      - name: Build e Push da imagem
        run: |
          IMAGE_NAME=${{ secrets.DOCKER_USERNAME }}/hello-app
          TAG=$(date +%s)
          docker build -t $IMAGE_NAME:$TAG .
          docker push $IMAGE_NAME:$TAG
          echo "IMAGE_TAG=$TAG" >> $GITHUB_ENV

      - name: Atualizar manifests no outro repositório
        uses: actions/checkout@v3
        with:
          repository: vicamaral14/projeto-ci-cd_manifests
          token: ${{ secrets.SSH_PRIVATE_KEY }}
          path: manifests

      - name: Atualizar tag da imagem
        run: |
          cd manifests
          sed -i "s#image:.*#image: ${{ secrets.DOCKER_USERNAME }}/hello-app:${{ env.IMAGE_TAG }}#g" deployment.yaml
          git config user.name "github-actions"
          git config user.email "actions@github.com"
          git commit -am "Atualiza imagem para tag ${{ env.IMAGE_TAG }}"
          git push
```

3. No GitHub, adicione os seguintes **Secrets** em
   *Settings → Secrets → Actions*:

   * `DOCKER_USERNAME`
   * `DOCKER_PASSWORD`
   * `SSH_PRIVATE_KEY`
     
```` bash
Muito cuidado com as credenciais, pois se não estiverem corretas o argo apontara erro
````
---

## ***🧾 4. Repositório de Manifests (ArgoCD)***

Crie o repositório [projeto-ci-cd_manifests](https://github.com/vicamaral14/projeto-ci-cd_manifests).

Adicione os arquivos:

**deployment.yaml**

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: hello-app
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
          image: seuusuario/hello-app:latest
          ports:
            - containerPort: 8080
```

**service.yaml**

```yaml
apiVersion: v1
kind: Service
metadata:
  name: hello-service
spec:
  type: ClusterIP
  selector:
    app: hello-app
  ports:
    - protocol: TCP
      port: 8080
      targetPort: 8080
```

---

## ***🔄 5. Criar o Aplicativo no ArgoCD***

1. Acesse o painel do ArgoCD:
   👉 `http://localhost:8080`
2. Clique em **NEW APP** e preencha:

   * **Application Name:** hello-app
   * **Project:** default
   * **Repository URL:** [projeto-ci-cd_manifests](https://github.com/vicamaral14/projeto-ci-cd_manifests)
   * **Path:** `./`
   * **Cluster:** [https://kubernetes.default.svc](https://kubernetes.default.svc)
   * **Namespace:** default
3. Clique em **Create** e depois em **Sync**.

---

## ***🔍 6. Testando o Deploy***

Verifique se o pod foi criado corretamente:

```bash
kubectl get pods
kubectl get svc
```

Faça o port-forward:

```bash
kubectl port-forward service/hello-service 8080:8080
```

Acesse no navegador:
👉 [http://localhost:8080/](http://localhost:8080/)

---

## ***🧪 7. Testando o Pipeline (CI/CD)***

1. Edite o arquivo `main.py`:

```python
return {"message": "Nova mensagem! Deploy automatizado com sucesso!"}
```

2. Faça commit e push:

```bash
git add .
git commit -m "Atualiza mensagem da aplicação"
git push origin main
```

3. A pipeline será executada automaticamente:

   * 🔧 Build e push da nova imagem para o Docker Hub
   * 🔄 Atualização automática dos manifests ([projeto-ci-cd_manifests](https://github.com/vicamaral14/projeto-ci-cd_manifests))
   * 🚀 Deploy automatizado via ArgoCD

---

## ***📸 8. Evidências Esperadas***

| Evidência                                | Descrição                                |
| ---------------------------------------- | ---------------------------------------- |
| ✅ **Workflow do GitHub Actions**         | Print mostrando execução com sucesso     |
| ✅ **Docker Hub**                         | Print com a nova imagem e tag atualizada |
| ✅ **Commit no repositório de manifests** | Mostrando atualização da imagem          |
| ✅ **ArgoCD sincronizado**                | Print do app atualizado                  |
| ✅ **kubectl get pods**                   | Mostrando pod rodando                    |
| ✅ **Resposta via navegador**             | Mensagem “Hello Compass UOL!” exibida    |

---
