# AKS Automated Deployment Project: `k8s-amit-test`

A sample project that uses **AKS Automated Deployments** to build and deploy an app from a GitHub repository to the `k8s-amit-test` cluster through GitHub Actions.

---

## 1. Overview

| Item | Value |
|---|---|
| Cluster | `k8s-amit-test` (Azure Kubernetes Service) |
| Feature | Automated deployments (Azure Portal) |
| CI/CD | GitHub Actions (alternative: Azure DevOps pipelines) |
| Registry | Azure Container Registry (ACR) |
| App | Simple Node.js web app |

**Flow**

```
Developer push -> GitHub repo -> GitHub Actions (build image, push to ACR)
              -> Deploy to AKS (k8s-amit-test) -> Service / Public IP
```

---

## 2. Prerequisites

- Azure subscription with access to the `k8s-amit-test` cluster
- GitHub account and a new empty repository
- Azure CLI (`az`), `kubectl`, Docker, Git
- An Azure Container Registry attached to the cluster

```bash
export RG=<your-resource-group>
export AKS=k8s-amit-test

az login
az aks get-credentials --resource-group $RG --name $AKS

# Create ACR (skip if you have one) and attach it to AKS
az acr create -g $RG -n <ACR_NAME> --sku Basic
az aks update -g $RG -n $AKS --attach-acr <ACR_NAME>
```

---

## 3. Project Structure

```
aks-auto-deploy/
├── app/
│   ├── package.json
│   └── server.js
├── k8s/
│   ├── deployment.yaml
│   └── service.yaml
├── Dockerfile
├── .github/
│   └── workflows/
│       └── deploy.yml        # generated/managed by AKS Automated deployments
└── README.md
```

---

## 4. Application Code

> Add this application code (`app/` folder and `Dockerfile`) to your GitHub repo.

**app/package.json**

```json
{
  "name": "aks-auto-deploy",
  "version": "1.0.0",
  "main": "server.js",
  "scripts": { "start": "node server.js" },
  "dependencies": { "express": "^4.19.2" }
}
```

**app/server.js**

```javascript
const express = require("express");
const app = express();
const PORT = process.env.PORT || 3000;

app.get("/", (req, res) => res.send("Hello from AKS k8s-amit-test!"));
app.get("/health", (req, res) => res.status(200).send("OK"));

app.listen(PORT, () => console.log(`Listening on ${PORT}`));
```

**Dockerfile**

```dockerfile
FROM node:20-alpine
WORKDIR /usr/src/app
COPY app/package*.json ./
RUN npm install --omit=dev
COPY app/ .
EXPOSE 3000
CMD ["npm", "start"]
```

---

## 5. Kubernetes Manifests

> Add these manifests (`k8s/` folder) to your GitHub repo.

**k8s/deployment.yaml**

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: aks-auto-deploy
spec:
  replicas: 2
  selector:
    matchLabels:
      app: aks-auto-deploy
  template:
    metadata:
      labels:
        app: aks-auto-deploy
    spec:
      containers:
        - name: aks-auto-deploy
          image: <ACR_NAME>.azurecr.io/aks-auto-deploy:latest
          ports:
            - containerPort: 3000
          readinessProbe:
            httpGet:
              path: /health
              port: 3000
          resources:
            requests:
              cpu: 100m
              memory: 128Mi
            limits:
              cpu: 250m
              memory: 256Mi
```

**k8s/service.yaml**

```yaml
apiVersion: v1
kind: Service
metadata:
  name: aks-auto-deploy
spec:
  type: LoadBalancer
  selector:
    app: aks-auto-deploy
  ports:
    - port: 80
      targetPort: 3000
```

---

## 6. Set Up Automated Deployments (Azure Portal)

1. In the Azure Portal open **Kubernetes services -> k8s-amit-test**.
2. In the left menu select **Automated deployments**, then click **Create** (the wizard is titled *Deploy an application*).

**Step 1: Repository**

| Field | Value |
|---|---|
| Workflow name | `myapp` |
| Repository location | `GitHub` (authorize your account if prompted) |
| Repository source | `My repositories` |
| Repository | `aks-auto-deploy` |
| Branch | `main` |

**Step 2: Application**

*Image*

| Field | Value |
|---|---|
| Container configuration | `Existing Dockerfile` |
| Dockerfile | Click **Select** and choose `Dockerfile` from the repo |
| Dockerfile build context | `./` |
| Azure Container Registry | `amitack` |
| Azure Container Registry image | `(New) image-workflow-...` or **Create new** with your own name |

> The image you select (for example `amitack.azurecr.io/<image-name>`) must match the `image:` value in `k8s/deployment.yaml`. A mismatch makes the deployment fail.

*Deployment configuration*

| Field | Value |
|---|---|
| Deployment options | `Use existing Kubernetes manifest deployment files` |
| Manifest files | Click **Select**, open the `k8s` folder, tick `deployment.yaml` and `service.yaml` (shown as `./k8s/deployment.yaml` and `./k8s/service.yaml`), then confirm |
| Namespace | `default` |

Click **Next**.

**Step 3: Environment variables**

Add any variables your app needs (none are required for this sample), then click **Next**.

**Step 4: Review**

Check the settings, then click **Next**.

**Step 5: Deploy**

Click **Deploy**. AKS opens a **pull request** in your GitHub repo containing the workflow file. Approve and merge the pull request, and the workflow builds the image and deploys the app.
