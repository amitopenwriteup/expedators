# Lab: Enable Argo CD on AKS (Azure Portal GitOps)

**Cluster:** `k8s-amit-test`
**Registry:** `amitack`
**Duration:** ~30 minutes
**Level:** Intermediate

## Objectives

By the end of this lab you will be able to:

- Enable Argo CD on an AKS cluster from the Azure portal
- Configure Workload Identity so Argo CD can reach Azure services (such as ACR) without stored secrets
- Configure ingress with an Azure DNS zone and subdomain
- Verify the Argo CD installation with `kubectl`

---

## Prerequisites

| Requirement | Notes |
|---|---|
| AKS cluster | `k8s-amit-test` is running |
| OIDC issuer + Workload Identity | Enabled on the cluster (see Step 0) |
| Azure Container Registry | `amitack` exists |
| Azure DNS zone | Required if you choose the **Azure DNS** provider |
| Permissions | Contributor on the cluster resource group. **Owner or User Access Administrator** on the ACR if role assignments need to be created |
| Tools | Azure CLI and `kubectl` (Cloud Shell works) |

> **Note:** If `az aks update --attach-acr amitack` fails with
> `Could not create a role assignment for ACR. Are you an Owner on this subscription?`,
> your account lacks `Microsoft.Authorization/roleAssignments/write`. Ask an Owner to run the role assignment for you (see [Troubleshooting](#troubleshooting)).

---

## Step 0: Prepare the cluster

Set variables and make sure the OIDC issuer and workload identity are enabled.

```bash
RG=<your-resource-group>
AKS=k8s-amit-test

az aks show -g $RG -n $AKS \
  --query "{oidc:oidcIssuerProfile.enabled, wi:securityProfile.workloadIdentity.enabled}" -o table
```

If either value is `false`/empty, enable them:

```bash
az aks update -g $RG -n $AKS --enable-oidc-issuer --enable-workload-identity
```

Get credentials for `kubectl`:

```bash
az aks get-credentials -g $RG -n $AKS --overwrite-existing
kubectl get nodes
```

---

## Step 1: Open the GitOps blade

1. Sign in to the [Azure portal](https://portal.azure.com).
2. Go to **Kubernetes center > Clusters > `k8s-amit-test`**.
3. In the left menu, under **Settings**, select **GitOps**.

You will see the empty GitOps page with two options:

- **Create Flux configuration**: built-in, Azure-native GitOps
- **Enable Argo CD**: visual dashboard and advanced sync controls

4. Select **Enable Argo CD**.

---

## Step 2: Basics tab

Configure the following:

| Setting | Value |
|---|---|
| Enable SSO | Unchecked (leave off for this lab) |
| Enable Workload identity | **Checked** |
| Managed identity | `aks-amit-agentpool` |
| Azure Container Registry | `amitack` |

**What this does:** Workload Identity lets Argo CD authenticate to Azure services using a federated managed identity, so no registry passwords or secrets are stored in the cluster. Selecting an ACR gives Argo CD access to pull from it.

> **Tip:** `aks-amit-agentpool` is the kubelet (node pool) identity. For production, use a dedicated user-assigned managed identity for Argo CD instead, and grant it only `AcrPull` on `amitack`.

Select **Next** to continue to the Ingress tab.

---

## Step 3: Ingress tab

Argo CD needs a hostname so you can reach the UI.

| Setting | Value |
|---|---|
| DNS provider | **Azure DNS** |
| Azure DNS Zone | Select your zone, e.g. `example.com` |
| Subdomain | e.g. `argocd` |

The resulting URL will be `https://<subdomain>.<zone>`, for example `https://argocd.example.com`.

- **Azure DNS:** records are created and updated automatically.
- **3rd party provider:** you must manually create DNS records pointing to the ingress IP after deployment.

> **Both fields are required.** If you don't have a DNS zone, create one first:
>
> ```bash
> az network dns zone create -g $RG -n <yourdomain.com>
> ```

---

## Step 4: Review + Deploy

1. Select the **Review + Deploy** tab.
2. Confirm the values from Steps 2 and 3.
3. Select **Deploy**.
4. Wait for the deployment to finish. This can take several minutes.

---

## Step 5: Verify the installation

Check the extension and pods:

```bash
az k8s-extension list -g $RG -c $AKS -t managedClusters -o table

kubectl get ns
kubectl get pods -n argocd
kubectl get ingress -n argocd
```

**Expected:** all Argo CD pods (server, repo-server, application-controller, redis, etc.) are `Running`, and the ingress shows an address.

> The namespace may differ depending on the extension defaults. Use `kubectl get pods -A | grep -i argo` to find it.

Get the initial admin password:

```bash
kubectl -n argocd get secret argocd-initial-admin-secret \
  -o jsonpath="{.data.password}" | base64 -d; echo
```

Open `https://<subdomain>.<zone>` and sign in with user `admin` and the password above.

If DNS or the ingress isn't ready, use port-forward as a fallback:

```bash
kubectl port-forward svc/argocd-server -n argocd 8080:443
# then browse to https://localhost:8080
```

---

## Step 6: Deploy a sample app (optional)

Create an Application that syncs a public sample repo:

```bash
cat <<'EOF' | kubectl apply -f -
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: guestbook
  namespace: argocd
spec:
  project: default
  source:
    repoURL: https://github.com/argoproj/argocd-example-apps.git
    targetRevision: HEAD
    path: guestbook
  destination:
    server: https://kubernetes.default.svc
    namespace: default
  syncPolicy:
    automated:
      prune: true
      selfHeal: true
EOF

kubectl get applications -n argocd
kubectl get pods -n default
```

In the Argo CD UI you should see `guestbook` as **Synced** and **Healthy**.

---

## Troubleshooting

| Symptom | Cause | Fix |
|---|---|---|
| `Could not create a role assignment for ACR` | Your account can't write role assignments | Have an Owner run the command below, or use an image pull secret |
| Managed identity dropdown is empty | No identities visible to you | Check Reader access on the identity's resource group |
| Azure DNS Zone dropdown is empty | No DNS zone in the subscription | Create a zone, or choose **3rd party provider** |
| Ingress has no address | Ingress controller still provisioning | Wait a few minutes, then `kubectl get svc -A` and look for a LoadBalancer IP |
| Pods `ImagePullBackOff` from ACR | Kubelet identity lacks `AcrPull` | Assign the role (below) |
| Deployment fails in the portal | Workload identity or OIDC not enabled | Repeat Step 0 |

**Grant `AcrPull` (run as an Owner):**

```bash
KUBELET_ID=$(az aks show -g $RG -n $AKS --query identityProfile.kubeletidentity.objectId -o tsv)
ACR_ID=$(az acr show -n amitack --query id -o tsv)
az role assignment create --assignee $KUBELET_ID --role AcrPull --scope $ACR_ID
```

**Check your own access:**

```bash
az role assignment list --assignee $(az ad signed-in-user show --query id -o tsv) --all -o table
```

---

## Cleanup

Remove the sample app and the Argo CD extension to avoid charges:

```bash
kubectl delete application guestbook -n argocd

az k8s-extension delete -g $RG -c $AKS -t managedClusters -n argocd --yes
```

> The extension name may differ. Use the output of `az k8s-extension list` from Step 5.

Delete the DNS zone only if you created it for this lab:

```bash
az network dns zone delete -g $RG -n <yourdomain.com> --yes
```

---

## Summary

You enabled Argo CD on AKS from the portal, wired it to ACR using Workload Identity, exposed it through an Azure DNS-backed ingress, and verified it with `kubectl`.

**Next steps:**

- Enable SSO with Microsoft Entra ID
- Use a dedicated managed identity with least privilege
- Compare with the Flux configuration option on the same GitOps blade
