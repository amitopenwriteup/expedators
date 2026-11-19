# Lab: Install ArgoCD on AKS and Expose It with a LoadBalancer

## Overview

Install ArgoCD in an AKS cluster, expose the `argocd-server` service with an Azure `LoadBalancer`, and log in to the UI. An optional step shows how to add a node NSG rule for the NodePort range.

Replace these placeholders with your own values wherever they appear:

| Placeholder | Meaning | How to find it |
|---|---|---|
| `<RESOURCE_GROUP>` | Resource group of the AKS cluster | `az aks list -o table` |
| `<AKS_CLUSTER_NAME>` | Name of the AKS cluster | `az aks list -o table` |
| `<NODE_RESOURCE_GROUP>` | Auto-created `MC_...` node resource group | `az aks show -g <RESOURCE_GROUP> -n <AKS_CLUSTER_NAME> --query nodeResourceGroup -o tsv` |
| `<NODE_NSG_NAME>` | Network security group of the nodes | `az network nsg list -g <NODE_RESOURCE_GROUP> -o table` |
| `<EXTERNAL-IP>` | Public IP of the `argocd-server` service | `kubectl -n argocd get svc argocd-server` |

---

## Prerequisites

- A running AKS cluster and `kubectl` access (see the AKS lab for `az aks get-credentials`)
- Azure CLI signed in to the right subscription

Confirm the connection:

```bash
kubectl get nodes
```

All nodes must be **Ready**.

---

## Part 1: Install ArgoCD

### Step 1: Create the namespace

```bash
kubectl create namespace argocd
```

### Step 2: Apply the ArgoCD manifest

```bash
kubectl apply -n argocd --server-side --force-conflicts \
  -f https://raw.githubusercontent.com/argoproj/argo-cd/stable/manifests/install.yaml
```

- `--server-side` applies the manifest with server-side apply. This is needed because some ArgoCD CRDs are too large for a normal client-side apply.
- `--force-conflicts` takes over any fields owned by another manager, so re-running the command doesn't fail.

### Step 3: Watch the pods start

```bash
kubectl -n argocd get pods -w
```

Wait until every pod shows **Running** and `READY` is full (for example `1/1`). Press **Ctrl+C** to stop watching.

Typical pods:

| Pod | Purpose |
|---|---|
| `argocd-server` | API server and web UI |
| `argocd-repo-server` | Clones Git repos and renders manifests |
| `argocd-application-controller` | Compares live state with Git and syncs |
| `argocd-applicationset-controller` | Generates Applications from templates |
| `argocd-dex-server` | SSO identity broker |
| `argocd-redis` | Cache |
| `argocd-notifications-controller` | Notifications |

---

## Part 2: Expose ArgoCD with a LoadBalancer

By default `argocd-server` is a `ClusterIP` service, reachable only inside the cluster.

### Step 1: List the services

```bash
kubectl get svc -n argocd
```

`argocd-server` shows `TYPE ClusterIP` and no external IP.

### Step 2: Change the service type to LoadBalancer

Option A: edit the service interactively:

```bash
kubectl edit svc argocd-server -n argocd
```

Change `type: ClusterIP` to `type: LoadBalancer`, then save and exit.

Option B: patch it in one command (no editor):

```bash
kubectl -n argocd patch svc argocd-server -p '{"spec": {"type": "LoadBalancer"}}'
```

> Edit `argocd-server` by name. Running `kubectl edit svc -n argocd` with no name opens every service in the namespace at once, which makes it easy to change the wrong one.

### Step 3: Wait for the external IP

```bash
kubectl get svc argocd-server -n argocd --watch
```

`EXTERNAL-IP` shows `<pending>` first. When a public IP appears, press **Ctrl+C**:

```text
NAME            TYPE           CLUSTER-IP    EXTERNAL-IP     PORT(S)                      AGE
argocd-server   LoadBalancer   <CLUSTER-IP>   <EXTERNAL-IP>   80:30338/TCP,443:30591/TCP   15m
```

> Replace `<EXTERNAL-IP>` with the EXTERNAL-IP value from **your own** output.

- A public IP means an **external** load balancer: https://external-ip

---



> **Always quote the `*`.** Unquoted, the shell replaces it with filenames from the current folder (in Cloud Shell: `clouddrive`), and Azure fails with `SecurityRuleInvalidAddressPrefix`.
>
> Use a specific source range when you can. `"*"` allows any source that can reach the nodes.


## Part 4: Log In to ArgoCD

### Step 1: Open the UI

Browse to `https://<EXTERNAL-IP>` (for example `https://<EXTERNAL-IP>`). ArgoCD uses a self-signed certificate by default, so accept the browser warning.

Quick check from the shell:

```bash
curl -k https://<EXTERNAL-IP>
```

### Step 2: Get the initial admin password

```bash
kubectl -n argocd get secret argocd-initial-admin-secret \
  -o jsonpath="{.data.password}" | base64 -d; echo
```

### Step 3: Sign in

- **Username:** `admin`
- **Password:** the value from Step 2

Change the password afterwards (**User Info** → **Update Password**) and delete the initial secret once you no longer need it:

```bash
kubectl -n argocd delete secret argocd-initial-admin-secret
```

---

## Part 5: (Recommended) Restrict Access to Your IP

The load balancer makes the login page reachable from the internet. Limit it to your public IP:

```bash
kubectl -n argocd patch svc argocd-server \
  -p '{"spec":{"loadBalancerSourceRanges":["<your-public-ip>/32"]}}'
```

AKS updates the network rules for you. Test from that IP afterwards.

To keep ArgoCD private instead, add the annotation `service.beta.kubernetes.io/azure-load-balancer-internal: "true"` to the service. It then gets a private `10.x` IP, reachable only from the VNet or a VPN.

---

---

## Next Steps

- Configure GitHub SSO through Dex (see the ArgoCD SSO lab). Use the load balancer URL **without a trailing slash**.
- Deploy a first Application from a Git repository.
- Use a DNS name and a real TLS certificate instead of the raw IP.

---

## Cleanup

```bash
kubectl delete namespace argocd
```

Deleting the namespace also removes the `argocd-server` service, and the Azure load balancer frontend for it is removed automatically.
