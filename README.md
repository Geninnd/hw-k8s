# Hello World App

Este repositório contém tudo que você precisa para rodar o **Hello World App**, criar a imagem Docker e integrar com o Jenkins, ArgoCD e Kubernetes.

## 0. Instalar ferramentas

- **Docker e dependências**:
```bash
sudo apt update
sudo apt install -y git docker.io curl unzip
sudo systemctl start docker
sudo systemctl enable docker
sudo usermod -aG docker $USER
newgrp docker
```

- **kubectl**:
```bash
curl -LO "https://dl.k8s.io/release/$(curl -L -s https://dl.k8s.io/release/stable.txt)/bin/linux/amd64/kubectl"
sudo install -o root -g root -m 0755 kubectl /usr/local/bin/
```

- **ArgoCD CLI**:
```bash
curl -sLO https://github.com/argoproj/argo-cd/releases/latest/download/argocd-linux-amd64
sudo install -o root -g root -m 0755 argocd-linux-amd64 /usr/local/bin/argocd
```

- **ngrok** (para testes locais de webhook):
```bash
curl -sLo ngrok.zip https://bin.equinox.io/c/4VmDzA7iaHb/ngrok-stable-linux-amd64.zip
unzip ngrok.zip && sudo mv ngrok /usr/local/bin
```

- **Jenkins**:
```bash
docker pull jenkins/jenkins:lts
```

### Verificar instalações
```bash
# Verificar versões
docker --version
kubectl version --client
argocd version --client
ngrok version
docker inspect jenkins/jenkins:lts | grep -i version
```

## 1. Clonar repositórios

```bash
git clone https://github.com/Geninnd/hw-app.git
cd hw-app
git clone https://github.com/Geninnd/hw-k8s.git ../hw-k8s
```

## 2. Registry Docker local

```bash
docker run -d -p 5000:5000 --name registry registry:2
```

## 3. Jenkins (com Docker)

```bash
docker run -d \
  -p 8080:8080 \
  -p 50000:50000 \
  -v jenkins_home:/var/jenkins_home \
  -v /var/run/docker.sock:/var/run/docker.sock \
  --name jenkins jenkins/jenkins:lts
```

### 3.1 Configurar Jenkins

1. Acesse: [http://localhost:8080](http://localhost:8080)
2. **Obter senha inicial**:

   ```bash
   docker logs jenkins | grep 'Initial Admin Password'
   ```
3. Siga o wizard:

   * Instale plugins recomendados: **GitHub**, **Pipeline**, **Docker Pipeline**
   * Crie um usuário admin

#### 3.1.1 Como obter o Personal Access Token (PAT) no GitHub

1. No GitHub, clique no seu avatar → **Settings**

2. Vá em **Developer settings** → **Personal access tokens** → **Tokens (classic)**

3. Clique em **Generate new token**

4. Defina validade e marque **Scopes**:

   * **repo**
   * **admin:repo_hook**

5. Clique em **Generate token** e **copie** o valor (você verá só uma vez)

6. Adicione **Credentials** no Jenkins:

   * **Manage Jenkins → Credentials → System → Global → Add Credentials**
   * **Kind**: Secret text
   * **Secret**: cole o PAT copiado
   * **ID**: `github-token`

7. Configure GitHub server:

   * **Manage Jenkins → Configure System → GitHub**
   * **Jenkins URL**: `http://localhost:8080`
   * **Credentials**: selecione `github-token`
   * **Test Connection** → OK

## 4. Pipeline Jenkins

1. Crie job **Pipeline** `hw-app`.
2. Em **Build Triggers**, marque **GitHub hook trigger for GITScm polling**.
3. Em **Pipeline → Definition: Pipeline script from SCM**:

   * SCM: **Git**
   * Repository URL: `https://github.com/Geninnd/hw-app.git`
   * Credentials: a mesma que colocou na etapa 6
   * Branch: ***/main***
4. Salve.

## 5. Expor Jenkins com Ngrok

```bash
ngrok http http://localhost:8080"
```

Copie a URL **HTTPS** que será exibida no terminal (ex: `https://2a69-187-56-94-247.ngrok-free.app`).

## 6. Configurar Webhook do GitHub

No GitHub de `seu-usuário/hw-app` → **Settings → Webhooks → Add webhook**:

* Payload URL: `https://2a69-187-56-94-247.ngrok-free.app/github-webhook/`
* Content type: `application/json`
* Events: `Just the push event`
* **Sem secret** (não é necessário para uma aplicação local)

> ⚠️ O webhook só funciona com Ngrok ativo.

## 7. Configurar ArgoCD e kubectl

1. Acesse a UI do ArgoCD: abra no navegador `http://<ARGOCD_SERVER>`
2. Faça login como `admin` e sua senha
3. No canto superior esquerdo, clique em **+ New App**
4. Preencha:

   * **Application Name**: `hw-app`
   * **Project**: `default`
   * **Repository URL**: `https://github.com/Geninnd/hw-k8s.git`
   * **Revision**: `main`
   * **Path**: `.`
   * **Cluster**: `https://kubernetes.default.svc`
   * **Namespace**: `default`
5. Em **Sync Policy**, marque **Automated** (Sync & Prune)
6. Clique em **Create**

---
A partir de agora, ao **push** na branch `main` do `hw-app`, o Jenkins será acionado via Ngrok, buildará e enviará a imagem ao registry. O ArgoCD detectará a mudança no repositório de manifests `hw-k8s` e sincronizará automaticamente o deployment no cluster Kubernetes.
