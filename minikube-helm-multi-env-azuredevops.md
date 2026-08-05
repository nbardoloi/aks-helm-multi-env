# Helm Multi-Env on Minikube + Docker Hub via Azure DevOps

Adapted from the AKS/ACR tutorial pattern, swapping:
- **ACR → Docker Hub**
- **AKS → Minikube** (local cluster)

The key implication: **Azure DevOps hosted agents cannot reach a local minikube cluster.** You must run a **self-hosted agent** on the machine (or network) where minikube lives. Everything below assumes that.

---

## 1. Repo layout

```
myapp/
├── src/                        # your app source
├── Dockerfile
├── charts/
│   └── myapp/
│       ├── Chart.yaml
│       ├── values.yaml
│       ├── values-dev.yaml
│       ├── values-prod.yaml
│       └── templates/
│           ├── deployment.yaml
│           ├── service.yaml
│           └── _helpers.tpl
└── azure-pipelines.yml
```

## 2. Dockerfile (example)

```dockerfile
FROM node:20-alpine
WORKDIR /app
COPY src/ .
RUN npm install
EXPOSE 8080
CMD ["node", "server.js"]
```

## 3. Helm chart

**charts/myapp/Chart.yaml**
```yaml
apiVersion: v2
name: myapp
description: Multi-env demo app
version: 0.1.0
appVersion: "1.0"
```

**charts/myapp/values.yaml** (defaults)
```yaml
replicaCount: 1

image:
  repository: yourdockerhubuser/myapp   # <-- Docker Hub repo, not ACR
  tag: latest
  pullPolicy: IfNotPresent

service:
  type: NodePort        # minikube-friendly; use LoadBalancer only with `minikube tunnel`
  port: 8080

env: dev

resources: {}
```

**charts/myapp/values-dev.yaml**
```yaml
env: dev
replicaCount: 1
image:
  tag: dev
```

**charts/myapp/values-qa.yaml**
```yaml
env: qa
replicaCount: 1
image:
  tag: qa
```

**charts/myapp/values-prod.yaml**
```yaml
env: prod
replicaCount: 2
image:
  tag: prod
```

**charts/myapp/templates/deployment.yaml**
```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: {{ .Release.Name }}-{{ .Values.env }}
  namespace: {{ .Values.env }}
spec:
  replicas: {{ .Values.replicaCount }}
  selector:
    matchLabels:
      app: {{ .Release.Name }}
      env: {{ .Values.env }}
  template:
    metadata:
      labels:
        app: {{ .Release.Name }}
        env: {{ .Values.env }}
    spec:
      containers:
        - name: {{ .Release.Name }}
          image: "{{ .Values.image.repository }}:{{ .Values.image.tag }}"
          imagePullPolicy: {{ .Values.image.pullPolicy }}
          env:
            - name: ENV
              value: {{ .Values.env }}
          ports:
            - containerPort: 8080
```

**charts/myapp/templates/service.yaml**
```yaml
apiVersion: v1
kind: Service
metadata:
  name: {{ .Release.Name }}-{{ .Values.env }}-svc
  namespace: {{ .Values.env }}
spec:
  type: {{ .Values.service.type }}
  selector:
    app: {{ .Release.Name }}
    env: {{ .Values.env }}
  ports:
    - port: {{ .Values.service.port }}
      targetPort: 8080
```

Create the namespaces once (or template a `namespace.yaml` — but simplest to precreate):
```bash
kubectl create namespace dev
kubectl create namespace qa
kubectl create namespace prod
```

---

## 4. Docker Hub setup

1. Create a **public or private repo** on Docker Hub: `yourdockerhubuser/myapp`.
2. Create an **Access Token**: Docker Hub → Account Settings → Security → New Access Token. Use this instead of your password anywhere auth is needed.

---

## 5. Minikube setup (on the agent machine)

```bash
minikube start --driver=docker
minikube addons enable ingress   # optional
kubectl create namespace dev
kubectl create namespace qa
kubectl create namespace prod
```

If you want the pipeline's `docker build` to reuse minikube's Docker daemon (skips a registry round-trip for local testing), you *can* do `eval $(minikube docker-env)` — but for a real CI flow, **push to Docker Hub and let Kubernetes pull it**, so keep the normal `docker build && docker push` path. That also makes `imagePullPolicy` behave correctly.

---

## 6. Azure DevOps setup

### 6.1 Self-hosted agent (required)
Since minikube is local:
1. Azure DevOps → Organization Settings → Agent Pools → Add pool (or use `Default`) → New agent.
2. Download and configure the agent **on the machine running minikube**:
   ```bash
   mkdir myagent && cd myagent
   tar zxvf vsts-agent-linux-x64-<version>.tar.gz
   ./config.sh   # paste your org URL + PAT
   ./run.sh
   ```
3. This machine now needs: `docker`, `kubectl`, `helm`, and a working `minikube` with the right context (`kubectl config current-context` should show `minikube`).

### 6.2 Service connections
- **Docker Registry service connection** (Project Settings → Service connections → New → Docker Registry → Docker Hub): give it your Docker Hub username + access token. Name it e.g. `dockerhub-connection`.
- **No Kubernetes service connection needed** if the self-hosted agent already has `kubectl`/`helm` pointed at minikube's context — the pipeline just runs `helm` commands directly on that agent, same as running them from your terminal. (This is the main structural simplification vs. AKS, where you'd normally need an AKS service connection or `az aks get-credentials`.)

---

## 7. azure-pipelines.yml

```yaml
trigger:
  branches:
    include:
      - main

pool: 'Default'   # your self-hosted pool with the minikube agent

variables:
  dockerHubConnection: 'dockerhub-connection'
  imageRepo: 'yourdockerhubuser/myapp'

stages:
  - stage: Build
    jobs:
      - job: BuildAndPush
        steps:
          - task: Docker@2
            displayName: Build and push dev image
            inputs:
              containerRegistry: $(dockerHubConnection)
              repository: $(imageRepo)
              command: buildAndPush
              Dockerfile: '**/Dockerfile'
              tags: |
                dev
                $(Build.BuildId)

  - stage: DeployDev
    dependsOn: Build
    jobs:
      - job: HelmDeployDev
        steps:
          - script: |
              helm upgrade --install myapp ./charts/myapp \
                -f ./charts/myapp/values-dev.yaml \
                --set image.tag=$(Build.BuildId) \
                --namespace dev
            displayName: 'Helm deploy to dev (minikube)'

  - stage: DeployQA
    dependsOn: DeployDev
    condition: succeeded()
    jobs:
      - deployment: HelmDeployQA
        environment: 'qa'   # optional approval gate before QA
        strategy:
          runOnce:
            deploy:
              steps:
                - task: Docker@2
                  displayName: Tag and push qa image
                  inputs:
                    containerRegistry: $(dockerHubConnection)
                    repository: $(imageRepo)
                    command: buildAndPush
                    Dockerfile: '**/Dockerfile'
                    tags: |
                      qa
                      $(Build.BuildId)
                - script: |
                    helm upgrade --install myapp ./charts/myapp \
                      -f ./charts/myapp/values-qa.yaml \
                      --set image.tag=$(Build.BuildId) \
                      --namespace qa
                  displayName: 'Helm deploy to qa (minikube)'

  - stage: DeployProd
    dependsOn: DeployQA
    condition: succeeded()
    jobs:
      - deployment: HelmDeployProd
        environment: 'production'   # gives you a manual approval gate in Azure DevOps
        strategy:
          runOnce:
            deploy:
              steps:
                - task: Docker@2
                  displayName: Tag and push prod image
                  inputs:
                    containerRegistry: $(dockerHubConnection)
                    repository: $(imageRepo)
                    command: buildAndPush
                    Dockerfile: '**/Dockerfile'
                    tags: |
                      prod
                      $(Build.BuildId)
                - script: |
                    helm upgrade --install myapp ./charts/myapp \
                      -f ./charts/myapp/values-prod.yaml \
                      --set image.tag=$(Build.BuildId) \
                      --namespace prod
                  displayName: 'Helm deploy to prod (minikube)'
```

Notes:
- Using a **deployment job** with `environment: 'production'` lets you configure an approval check in Azure DevOps (Pipelines → Environments → production → Approvals), giving you the manual gate between dev and prod that the original AKS tutorial likely used with separate stages/approvals.
- `$(Build.BuildId)` gives each build a unique, traceable image tag instead of relying only on `latest`.

---

## 8. Verifying deployments

```bash
kubectl get pods -n dev
kubectl get svc -n dev
minikube service myapp-dev-svc -n dev   # opens it in your browser (NodePort)

kubectl get pods -n qa
minikube service myapp-qa-svc -n qa

kubectl get pods -n prod
minikube service myapp-prod-svc -n prod
```

---

## 9. Summary of what changed vs. the AKS/ACR tutorial

| AKS/ACR original | Minikube/Docker Hub version |
|---|---|
| ACR service connection | Docker Registry (Docker Hub) service connection |
| `az acr build` / `docker push <acr>.azurecr.io/...` | `docker push yourdockerhubuser/myapp` |
| Microsoft-hosted agent + AKS service connection / `az aks get-credentials` | **Self-hosted agent** on the machine running minikube, using its local kubeconfig context directly |
| AKS namespaces `dev`/`qa`/`prod` | Minikube namespaces `dev`/`qa`/`prod` (same Helm/values pattern) |
| Multi-stage pipeline: Dev → QA → Prod with approvals | Same structure — `DeployDev` → `DeployQA` → `DeployProd`, each an Azure DevOps `environment` for approval gates |
| Public LoadBalancer / Ingress with real DNS | `NodePort` + `minikube service`, or Ingress + `minikube tunnel` if you want it |

If you share the actual folder structure/values files from that repo (or paste them in), I can map this 1:1 instead of the generic scaffold above.
