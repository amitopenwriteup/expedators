# Lab: ArgoCD SSO with GitHub via Dex (on AKS)

## Overview

Integrate ArgoCD with GitHub OAuth using Dex as the identity broker. After completing this lab, users will log in to ArgoCD using their GitHub account.

In this lab ArgoCD runs on AKS and is exposed by a `LoadBalancer` service, so the ArgoCD URL is the service's **EXTERNAL-IP** (example: `https://<EXTERNAL-IP>`).

---

## Prerequisites

- ArgoCD installed and running on the cluster (namespace `argocd` in this lab)
- `kubectl` access to the cluster
- A GitHub account (used to create the OAuth app)
- ArgoCD reachable from your browser (public IP or DNS name)

---

## Part 0: Find the ArgoCD URL

Get the external IP of the `argocd-server` service:

```bash
kubectl -n argocd get svc argocd-server
```

```text
NAME            TYPE           CLUSTER-IP    EXTERNAL-IP     PORT(S)                      AGE
argocd-server   LoadBalancer   <CLUSTER-IP>   <EXTERNAL-IP>   80:30338/TCP,443:30591/TCP   15m
```

> Replace every `<EXTERNAL-IP>` in this lab with the EXTERNAL-IP value from **your own** output.

- A **public** EXTERNAL-IP means an external load balancer. A `10.x` address means an internal one.
- You reach ArgoCD at `https://<EXTERNAL-IP>`. You do **not** need to open the NodePorts (`30338`, `30591`).
- ArgoCD uses a self-signed certificate by default, so accept the browser warning.

Set a variable used in the rest of this lab:

```bash
export ARGOCD_URL="https://<EXTERNAL-IP>"     # NO trailing slash
```

> **No trailing slash, ever.** `https://<EXTERNAL-IP>/` makes ArgoCD build the Dex issuer as `https://<EXTERNAL-IP>//api/dex`, which fails with `404 page not found`.

> An IP address works for a lab. For anything longer-lived, use a DNS name with a real certificate. If the IP changes, update the `url`, the `redirectURI` and the GitHub OAuth App.

---

## Part 1: GitHub Side — Create an OAuth App

### Step 1: Go to GitHub OAuth Apps

Open `https://github.com/settings/developers` → **OAuth Apps**

### Step 2: Register a New OAuth App

Click **"New OAuth App"** and fill in:

| Field | Value |
|---|---|
| **Application name** | `ArgoCD` (or any name) |
| **Homepage URL** | `https://<EXTERNAL-IP>` |
| **Authorization callback URL** | `https://<EXTERNAL-IP>/api/dex/callback` |

> The callback URL **must** end with `/api/dex/callback` (this is what Dex listens on) and must not contain `//` before `api`.

Click **"Register application"**.

### Step 3: Copy the Credentials

After registration, GitHub shows you:

- **Client ID** → copy this (e.g., `Ov23liABC1234567890`)
- Click **"Generate a new client secret"**
- **Client Secret** → copy immediately (shown only once!)

---

## Part 2: Kubernetes Side — Store the Secret

Store the GitHub client secret in the `argocd-secret` Kubernetes Secret:

```bash
# Base64-encode the client secret (-w0 keeps it on one line)
echo -n "YOUR_GITHUB_CLIENT_SECRET" | base64 -w0

# Patch the argocd-secret
kubectl patch secret argocd-secret \
  -n argocd \
  --type merge \
  -p '{"data": {"dex.github.clientSecret": "<BASE64_ENCODED_SECRET>"}}'
```

Verify:

```bash
kubectl get secret argocd-secret -n argocd -o jsonpath='{.data.dex\.github\.clientSecret}' | base64 -d; echo
```

---

## Part 3: ArgoCD Side — Configure Dex

Edit the `argocd-cm` ConfigMap:

```bash
kubectl edit configmap argocd-cm -n argocd
```

Add the following under `data:`:

```yaml
data:
  url: https://<EXTERNAL-IP>            # Must match your ArgoCD URL, NO trailing slash

  dex.config: |
    connectors:
    - type: github
      id: github
      name: GitHub
      config:
        clientID: Ov23liABC1234567890        # From GitHub OAuth App
        clientSecret: $dex.github.clientSecret  # References the K8s secret
        redirectURI: https://<EXTERNAL-IP>/api/dex/callback
```

> `dex.config: |` must be indented under `data:` and the connector block indented under it. Bad indentation stops Dex from starting.

Check the saved values (no trailing slash):

```bash
kubectl -n argocd get cm argocd-cm -o jsonpath='{.data.url}'; echo
```

To fix the URL quickly without editing:

```bash
kubectl -n argocd patch cm argocd-cm --type merge \
  -p '{"data":{"url":"https://<EXTERNAL-IP>"}}'
```

---

## Part 4: Restart ArgoCD Components

Apply changes by restarting the relevant pods:

```bash
kubectl -n argocd rollout restart deployment argocd-server argocd-dex-server
kubectl -n argocd rollout status deployment argocd-server
kubectl -n argocd rollout status deployment argocd-dex-server
```

Or watch the pods:

```bash
kubectl get pods -n argocd -w
```

---

## Part 5: Verify Dex Before Logging In

Check that ArgoCD can reach Dex and that the issuer has no `//`:

```bash
curl -k $ARGOCD_URL/api/dex/.well-known/openid-configuration
```

Expected: JSON that includes

```text
"issuer": "https://<EXTERNAL-IP>/api/dex"
```

- **404** → ArgoCD isn't reaching Dex. Re-check the `url` (trailing slash), the Dex pod logs and the `dex.config` indentation.
- **Connection refused / timeout** → check that the `argocd-server` service has an EXTERNAL-IP and is reachable from your machine.

---

## Part 6: Test the Login

1. Open `https://<EXTERNAL-IP>` in your browser (accept the certificate warning)
2. Click **"LOG IN VIA GITHUB"**
3. GitHub asks you to authorize the OAuth App → click **"Authorize"**
4. You are redirected back to ArgoCD and logged in 
> If you also use the local `admin` account, the initial password is:
> ```bash
> kubectl -n argocd get secret argocd-initial-admin-secret -o jsonpath="{.data.password}" | base64 -d; echo
> ```

---

## What You Need from GitHub — Summary Checklist

```
[ ] GitHub OAuth App created
[ ] Homepage URL set to ArgoCD URL (no trailing slash)
[ ] Callback URL set to: https://<argocd-url>/api/dex/callback
[ ] Client ID copied
[ ] Client Secret generated and copied (shown only once!)
```
