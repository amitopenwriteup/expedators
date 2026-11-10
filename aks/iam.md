# Lab: Identity and Access Management (IAM) on AKS

A hands-on lab on who can do what in AKS. You will enable Entra ID with Azure RBAC, grant and test access at cluster and namespace scope, use Kubernetes RBAC alongside it, give nodes access to a registry through the kubelet identity, give pods access to Key Vault with workload identity, and diagnose common access failures.

**Level:** Intermediate to advanced | **Time:** 3 to 4 hours | **Cost:** Small but real (2 nodes, a registry, a Key Vault). Delete everything at the end.

## What you will learn

- The identities in an AKS environment and what each is used for
- Why creating a cluster does not make you a Kubernetes admin when Azure RBAC is on
- The difference between getting credentials (Azure permission) and acting inside the cluster (Kubernetes permission)
- Namespace-scoped Azure RBAC roles, and Kubernetes `Role` and `RoleBinding` with Entra identities
- How the kubelet identity pulls from ACR
- How workload identity lets a pod read Key Vault with no secret
- Disabling local accounts
- Diagnosing `Forbidden`, `401` and federated-credential errors

## Identity map

| Identity | Used by | Typical purpose |
|---|---|---|
| Your Entra user | You | Sign in, run `az` and `kubectl` |
| Cluster identity (managed identity) | AKS control plane | Manage Azure resources such as load balancers and routes |
| Kubelet identity (managed identity) | Nodes | Pull images from ACR |
| Workload identity (user-assigned managed identity) | Application pods | Access Key Vault, Storage and other Azure services |
| Kubernetes service account | Pods | Call the Kubernetes API; also the token source for workload identity |

Two separate permission layers apply:

| Layer | Question it answers | Examples |
|---|---|---|
| **Azure RBAC (management plane)** | Can this identity manage the AKS resource and fetch credentials? | Contributor, Cluster User Role |
| **Authorization inside the cluster** | Can this identity read pods, create deployments, read secrets? | Azure RBAC roles for Kubernetes, or Kubernetes RBAC |

## Prerequisites

- An Azure subscription with **Owner**, or **Contributor plus User Access Administrator**, on the subscription or resource group
- Permission in Microsoft Entra ID to create an app registration or service principal
- Azure CLI, `kubectl`, `kubelogin` and `jq` (all available in **Azure Cloud Shell**, bash)

> Commands use bash. Names are examples. I have not run this lab against a live subscription. Role assignments can take 1 to 5 minutes to take effect, so wait and retry when a new permission seems ignored. CLI flags change between versions, so check `--help` if a command fails.

---

## Lab setup

### Variables and resource group

```bash
RG=aks-iam-lab-rg
LOCATION=eastus
CLUSTER=aks-iam-lab
SUFFIX=$RANDOM
ACR=aksiamacr$SUFFIX
KV=aks-iam-kv-$SUFFIX
UAMI=wi-identity

az group create --name $RG --location $LOCATION
```

### Create the cluster with Entra ID, Azure RBAC and workload identity

```bash
az aks create \
  --resource-group $RG \
  --name $CLUSTER \
  --node-count 2 \
  --node-vm-size Standard_D2s_v5 \
  --enable-aad \
  --enable-azure-rbac \
  --enable-oidc-issuer \
  --enable-workload-identity \
  --generate-ssh-keys
```

This takes 5 to 10 minutes.

### Tools and shared variables

```bash
az aks install-cli
NODE_RG=$(az aks show -g $RG -n $CLUSTER --query nodeResourceGroup -o tsv)
CLUSTER_ID=$(az aks show -g $RG -n $CLUSTER --query id -o tsv)
ME=$(az ad signed-in-user show --query id -o tsv)
TENANT_ID=$(az account show --query tenantId -o tsv)
```

**Checkpoint:** `echo $CLUSTER_ID $ME` prints two values.

---

## Part 1: Inventory the identities

```bash
az aks show -g $RG -n $CLUSTER \
  --query "{clusterIdentity:identity.type, aad:aadProfile.managed, azureRbac:aadProfile.enableAzureRbac, localAccountsDisabled:disableLocalAccounts, oidcIssuer:oidcIssuerProfile.issuerUrl}" \
  -o jsonc

az aks show -g $RG -n $CLUSTER --query identityProfile.kubeletidentity -o jsonc

az identity list -g $NODE_RG -o table
```

List what the kubelet identity is allowed to do:

```bash
KUBELET_OBJ_ID=$(az aks show -g $RG -n $CLUSTER --query identityProfile.kubeletidentity.objectId -o tsv)
az role assignment list --assignee $KUBELET_OBJ_ID --all -o table
```

### What to observe

- The cluster identity type is `SystemAssigned`.
- The kubelet identity is a managed identity created in the node resource group.
- `azureRbac` is `true`, so Azure RBAC decides access inside the cluster.
- The kubelet identity has no ACR role yet (ACR does not exist yet).

### Questions

1. Which identity would you give access to a storage account that your app code reads?
2. Why is the kubelet identity different from the cluster identity?

---

## Part 2: Authentication vs authorization

### 2.1 Get credentials

```bash
az aks get-credentials -g $RG -n $CLUSTER --overwrite-existing
kubelogin convert-kubeconfig -l azurecli
```

Getting credentials works because your account has a management plane role.

### 2.2 Try to use the cluster

```bash
kubectl get pods -A
```

Expected: a `Forbidden` error similar to *User does not have access to the resource in Azure. Update role assignment to allow access.* You are authenticated, but nothing authorizes you inside the cluster.

### 2.3 Grant yourself Kubernetes admin

```bash
az role assignment create \
  --assignee $ME \
  --role "Azure Kubernetes Service RBAC Cluster Admin" \
  --scope $CLUSTER_ID
```

Wait 1 to 5 minutes, then retry:

```bash
kubectl get pods -A
kubectl get nodes
```

### 2.4 Create namespaces and sample objects

```bash
kubectl create namespace iamlab
kubectl create deployment web --image=nginx:1.27 -n iamlab
kubectl create secret generic demo-secret --from-literal=password=s3cret -n iamlab
kubectl get all,secret -n iamlab
```

**Checkpoint:** You received `Forbidden` first, then succeeded after the role assignment.

### Questions

1. Why was `get-credentials` allowed but `kubectl get pods` denied?
2. Would a user with only Contributor on the cluster be able to use `kubectl`? Why or why not?

---

## Part 3: Namespace-scoped Azure RBAC for Kubernetes

You will create a service principal to act as a second user, and give it read-only access to one namespace. A separate Azure CLI profile and kubeconfig keep it apart from your own login.

### 3.1 Create the test identity

```bash
SP_JSON=$(az ad sp create-for-rbac --name aks-iam-lab-sp -o json)
SP_APP_ID=$(echo $SP_JSON | jq -r .appId)
SP_SECRET=$(echo $SP_JSON | jq -r .password)
SP_OBJ_ID=$(az ad sp show --id $SP_APP_ID --query id -o tsv)
echo $SP_APP_ID $SP_OBJ_ID
```

Check that it has no role assignments yet:

```bash
az role assignment list --assignee $SP_APP_ID --all -o table
```

(If the CLI assigned a default role, remove it with `az role assignment delete --assignee $SP_APP_ID`.)

### 3.2 A helper to run commands as the service principal

```bash
as_sp() { AZURE_CONFIG_DIR=/tmp/azsp KUBECONFIG=/tmp/kube-sp "$@"; }

as_sp az login --service-principal -u $SP_APP_ID -p $SP_SECRET --tenant $TENANT_ID -o none
as_sp az account show --query user.name -o tsv
```

### 3.3 Grant management plane and namespace roles

```bash
az role assignment create \
  --assignee $SP_APP_ID \
  --role "Azure Kubernetes Service Cluster User Role" \
  --scope $CLUSTER_ID

az role assignment create \
  --assignee $SP_APP_ID \
  --role "Azure Kubernetes Service RBAC Reader" \
  --scope "$CLUSTER_ID/namespaces/iamlab"
```

Wait 1 to 5 minutes for the roles to take effect.

### 3.4 Connect as the service principal

```bash
as_sp az aks get-credentials -g $RG -n $CLUSTER --file /tmp/kube-sp --overwrite-existing
as_sp kubelogin convert-kubeconfig -l azurecli
```

### 3.5 Test what it can and cannot do

```bash
as_sp kubectl get pods -n iamlab
as_sp kubectl get pods -n default
as_sp kubectl get secrets -n iamlab
as_sp kubectl run test --image=nginx:1.27 -n iamlab
```

### Expected results

| Command | Result | Reason |
|---|---|---|
| `get pods -n iamlab` | Allowed | Reader role on that namespace |
| `get pods -n default` | Forbidden | Role scope covers only `iamlab` |
| `get secrets -n iamlab` | Forbidden | The Reader role does not allow viewing secrets |
| `run test ...` | Forbidden | Reader cannot create objects |

### 3.6 Upgrade the role to Writer

```bash
az role assignment create \
  --assignee $SP_APP_ID \
  --role "Azure Kubernetes Service RBAC Writer" \
  --scope "$CLUSTER_ID/namespaces/iamlab"

# wait 1 to 5 minutes, then:
as_sp kubectl run test --image=nginx:1.27 -n iamlab
as_sp kubectl get pods -n iamlab
as_sp kubectl get pods -n default
```

Expected: creating a pod in `iamlab` now works, and `default` is still forbidden.

**Checkpoint:** The service principal can read in `iamlab`, then write in `iamlab`, and never touch `default`.

### Questions

1. What is the advantage of scoping a role to a namespace?
2. Which built-in roles map to read-only, write, namespace admin and cluster admin?

---

## Part 4: Kubernetes RBAC with an Entra identity

Kubernetes `Role` and `RoleBinding` objects still work when Azure RBAC is on. You will give the service principal the right to edit ConfigMaps in `default`.

### 4.1 Find the username Kubernetes sees

```bash
SP_K8S_USER=$(as_sp kubectl auth whoami -o jsonpath='{.status.userInfo.username}')
echo $SP_K8S_USER
```

(If `kubectl auth whoami` is not available in your kubectl version, decode the token or check the API server audit log instead.)

### 4.2 Create the Role and RoleBinding as admin

```bash
cat <<EOF | kubectl apply -f -
apiVersion: rbac.authorization.k8s.io/v1
kind: Role
metadata:
  name: configmap-editor
  namespace: default
rules:
- apiGroups: [""]
  resources: ["configmaps"]
  verbs: ["get", "list", "create", "update", "delete"]
---
apiVersion: rbac.authorization.k8s.io/v1
kind: RoleBinding
metadata:
  name: sp-configmap-editor
  namespace: default
subjects:
- kind: User
  name: $SP_K8S_USER
  apiGroup: rbac.authorization.k8s.io
roleRef:
  kind: Role
  name: configmap-editor
  apiGroup: rbac.authorization.k8s.io
EOF
```

### 4.3 Test

```bash
as_sp kubectl create configmap from-sp --from-literal=a=b -n default
as_sp kubectl get configmap from-sp -n default
as_sp kubectl auth can-i create configmaps -n default
as_sp kubectl get pods -n default
```

Expected: ConfigMap actions succeed; `get pods -n default` is still forbidden.

### 4.4 Inspect the bindings

```bash
kubectl get rolebinding -n default
kubectl describe rolebinding sp-configmap-editor -n default
```

**Checkpoint:** The same identity gets different rights from Azure RBAC (namespace `iamlab`) and Kubernetes RBAC (namespace `default`).

### Questions

1. When would you choose Kubernetes RBAC over Azure RBAC for Kubernetes? (custom roles, GitOps-managed permissions, fine-grained verbs)
2. When is Azure RBAC the better fit? (central governance across many clusters, same tooling as the rest of Azure)
3. For real teams, should you bind users directly or Entra groups? Why?

---

## Part 5: Kubelet identity and ACR

### 5.1 Create a registry and import an image

```bash
az acr create -g $RG -n $ACR --sku Basic
ACR_ID=$(az acr show -g $RG -n $ACR --query id -o tsv)
az acr import -n $ACR --source docker.io/library/nginx:1.27 --image nginx:1.27
```

### 5.2 Try to pull without permission

```bash
kubectl run acr-test --image=$ACR.azurecr.io/nginx:1.27 -n iamlab
kubectl get pod acr-test -n iamlab
kubectl describe pod acr-test -n iamlab | grep -A8 Events
```

Expected: `ErrImagePull` or `ImagePullBackOff`, with an unauthorized or `401` message from the registry.

### 5.3 Grant the kubelet identity AcrPull

```bash
az role assignment create \
  --assignee-object-id $KUBELET_OBJ_ID \
  --assignee-principal-type ServicePrincipal \
  --role AcrPull \
  --scope $ACR_ID
```

Wait a minute or two, then recreate the pod:

```bash
kubectl delete pod acr-test -n iamlab
kubectl run acr-test --image=$ACR.azurecr.io/nginx:1.27 -n iamlab
kubectl get pod acr-test -n iamlab -w
```

Expected: the pod reaches `Running`.

You could do the same in one step with `az aks update -g $RG -n $CLUSTER --attach-acr $ACR`.

**Checkpoint:** The pull failed with `401`, then worked after assigning AcrPull to the kubelet identity.

### Questions

1. Why is the role given to the kubelet identity and not to the cluster identity or your own user?
2. Why is this safer than an image pull secret with a registry password?

---

## Part 6: Workload identity to Key Vault

A pod will read a Key Vault secret with no stored credential.

### 6.1 Check the cluster has OIDC and workload identity

```bash
az aks show -g $RG -n $CLUSTER --query "{oidc:oidcIssuerProfile.enabled, issuer:oidcIssuerProfile.issuerUrl, wi:securityProfile.workloadIdentity.enabled}" -o jsonc
ISSUER=$(az aks show -g $RG -n $CLUSTER --query oidcIssuerProfile.issuerUrl -o tsv)
```

### 6.2 Create a Key Vault and a secret

```bash
az keyvault create -g $RG -n $KV --location $LOCATION --enable-rbac-authorization true
KV_ID=$(az keyvault show -g $RG -n $KV --query id -o tsv)

az role assignment create --assignee $ME --role "Key Vault Secrets Officer" --scope $KV_ID
# wait 1 to 2 minutes
az keyvault secret set --vault-name $KV --name demo --value "hello-from-keyvault"
```

### 6.3 Create the managed identity and give it read access

```bash
az identity create -g $RG -n $UAMI
UAMI_CLIENT_ID=$(az identity show -g $RG -n $UAMI --query clientId -o tsv)
UAMI_PRINCIPAL_ID=$(az identity show -g $RG -n $UAMI --query principalId -o tsv)

az role assignment create \
  --assignee-object-id $UAMI_PRINCIPAL_ID \
  --assignee-principal-type ServicePrincipal \
  --role "Key Vault Secrets User" \
  --scope $KV_ID
```

### 6.4 Federate the identity with a Kubernetes service account

The subject must match `system:serviceaccount:<namespace>:<serviceaccount>` exactly.

```bash
az identity federated-credential create \
  --name aks-wi-fic \
  --identity-name $UAMI \
  --resource-group $RG \
  --issuer $ISSUER \
  --subject system:serviceaccount:iamlab:wi-sa \
  --audience api://AzureADTokenExchange
```

### 6.5 Create the service account and the pod

```bash
cat <<EOF | kubectl apply -f -
apiVersion: v1
kind: ServiceAccount
metadata:
  name: wi-sa
  namespace: iamlab
  annotations:
    azure.workload.identity/client-id: $UAMI_CLIENT_ID
---
apiVersion: v1
kind: Pod
metadata:
  name: wi-test
  namespace: iamlab
  labels:
    azure.workload.identity/use: "true"
spec:
  serviceAccountName: wi-sa
  containers:
  - name: cli
    image: mcr.microsoft.com/azure-cli
    command: ["sleep", "3600"]
EOF

kubectl wait --for=condition=Ready pod/wi-test -n iamlab --timeout=300s
```

### 6.6 Read the secret from inside the pod

```bash
kubectl exec wi-test -n iamlab -- printenv | grep AZURE

kubectl exec wi-test -n iamlab -- sh -c '
  az login --service-principal -u $AZURE_CLIENT_ID -t $AZURE_TENANT_ID \
    --federated-token "$(cat $AZURE_FEDERATED_TOKEN_FILE)" --allow-no-subscriptions -o none &&
  az keyvault secret show --vault-name '"$KV"' --name demo --query value -o tsv'
```

Expected: the environment variables `AZURE_CLIENT_ID`, `AZURE_TENANT_ID`, `AZURE_FEDERATED_TOKEN_FILE` (and others) are present, and the command prints `hello-from-keyvault`.

**Checkpoint:** The pod read the secret without any password, key or client secret.

### Questions

1. Where does the pod's token come from, and what makes Entra ID trust it?
2. What would happen if you deleted the federated credential?
3. Why is this better than putting a secret in a Kubernetes Secret?

---

## Part 7: Local accounts

Local accounts use static admin credentials that bypass Entra ID.

```bash
az aks get-credentials -g $RG -n $CLUSTER --admin --file /tmp/admin-kubeconfig --overwrite-existing
echo "admin credentials written"

az aks update -g $RG -n $CLUSTER --disable-local-accounts
az aks get-credentials -g $RG -n $CLUSTER --admin --file /tmp/admin-kubeconfig2 --overwrite-existing
```

Expected: the last command fails with a message that static credentials are not allowed because local accounts are disabled.

Confirm normal Entra sign in still works:

```bash
kubectl get nodes
```

Clean up the admin file:

```bash
rm -f /tmp/admin-kubeconfig /tmp/admin-kubeconfig2
```

**Checkpoint:** Admin credentials are no longer available, but your Entra-based access still works.

### Questions

1. What risk do local accounts create? (shared, long-lived credentials with no per-user audit)
2. What is your break-glass plan when local accounts are disabled? (a break-glass Entra group, tested in advance)

---

## Part 8: Troubleshooting scenarios

### Scenario A: Federated credential subject mismatch

Create a second annotated service account that has **no** matching federated credential:

```bash
cat <<EOF | kubectl apply -f -
apiVersion: v1
kind: ServiceAccount
metadata:
  name: wi-sa2
  namespace: iamlab
  annotations:
    azure.workload.identity/client-id: $UAMI_CLIENT_ID
---
apiVersion: v1
kind: Pod
metadata:
  name: wi-test2
  namespace: iamlab
  labels:
    azure.workload.identity/use: "true"
spec:
  serviceAccountName: wi-sa2
  containers:
  - name: cli
    image: mcr.microsoft.com/azure-cli
    command: ["sleep", "3600"]
EOF
kubectl wait --for=condition=Ready pod/wi-test2 -n iamlab --timeout=300s

kubectl exec wi-test2 -n iamlab -- sh -c '
  az login --service-principal -u $AZURE_CLIENT_ID -t $AZURE_TENANT_ID \
    --federated-token "$(cat $AZURE_FEDERATED_TOKEN_FILE)" --allow-no-subscriptions -o none'
```

**Find it:** The login fails with an error saying no matching federated identity record was found (an `AADSTS` error). The service account name in the token subject is `wi-sa2`, but the federated credential only trusts `wi-sa`.

**Check the credential:**

```bash
az identity federated-credential list --identity-name $UAMI -g $RG -o table
```

**Solution:**

```bash
az identity federated-credential create \
  --name aks-wi-fic2 \
  --identity-name $UAMI \
  --resource-group $RG \
  --issuer $ISSUER \
  --subject system:serviceaccount:iamlab:wi-sa2 \
  --audience api://AzureADTokenExchange
```

Wait a minute, then repeat the login command.

### Scenario B: Key Vault access removed

```bash
az role assignment delete \
  --assignee $UAMI_PRINCIPAL_ID \
  --role "Key Vault Secrets User" \
  --scope $KV_ID
```

After a minute or two, run the secret read from Part 6.6 again.

**Find it:** The login works, but reading the secret returns `Forbidden` (the caller does not have secrets get permission).

**Solution:** Restore the role and retry:

```bash
az role assignment create \
  --assignee-object-id $UAMI_PRINCIPAL_ID \
  --assignee-principal-type ServicePrincipal \
  --role "Key Vault Secrets User" \
  --scope $KV_ID
```

### Scenario C: Missing Cluster User Role

```bash
az role assignment delete \
  --assignee $SP_APP_ID \
  --role "Azure Kubernetes Service Cluster User Role" \
  --scope $CLUSTER_ID

as_sp az aks get-credentials -g $RG -n $CLUSTER --file /tmp/kube-sp2 --overwrite-existing
```

**Find it:** The command fails with `AuthorizationFailed`. The identity lacks the permission `Microsoft.ContainerService/managedClusters/listClusterUserCredential/action`, even though it may still hold in-cluster roles.

**Solution:**

```bash
az role assignment create \
  --assignee $SP_APP_ID \
  --role "Azure Kubernetes Service Cluster User Role" \
  --scope $CLUSTER_ID
```

Wait, then run the `get-credentials` command again.

### Scenario D: A change that "does not work" (propagation and tokens)

You granted a role, but the old error remains. Check, in order:

1. Wait 1 to 5 minutes; role assignments and token claims take time to refresh.
2. Make sure the assignment is at the right scope: `az role assignment list --assignee <id> --all -o table`.
3. Refresh the token: `az account get-access-token` again, or sign out and in (`az logout`, `az login`), and re-run `kubelogin convert-kubeconfig -l azurecli`.
4. Confirm the principal in the error message is the one you assigned (user, group or service principal).

**Checkpoint:** You fixed a federated credential mismatch, a missing Key Vault role and a missing management plane role, and you can describe the signs of each.

---

## IAM troubleshooting flow

1. **Which layer failed?** `AuthorizationFailed` from `az` usually means a management plane role. `Forbidden` from `kubectl` means an in-cluster role.
2. **Who is the caller?** `az account show`, `kubectl auth whoami`, and the identity named in the error.
3. **At what scope is the role?** Subscription, resource group, cluster, namespace, or the target resource.
4. **Is it a pod identity problem?** Check the pod label, service account annotation, federated credential issuer and subject, and the role on the target resource.
5. **Wait and refresh tokens** before changing more.

---

## Knowledge check

1. What is the difference between `Azure Kubernetes Service Cluster User Role` and `Azure Kubernetes Service RBAC Cluster Admin`?
2. Why does a cluster creator not automatically get Kubernetes admin rights when Azure RBAC is enabled?
3. Which identity pulls images from ACR, and what role does it need?
4. What three things must line up for workload identity to work?
5. A Reader at namespace `iamlab` cannot see secrets. Is that a bug?
6. Why disable local accounts?

**Suggested answers**

1. The first only lets a user download kubeconfig (management plane). The second grants full admin inside the cluster.
2. In-cluster access is controlled by separate role assignments; owning the Azure resource does not grant it.
3. The kubelet identity, with `AcrPull` on the registry.
4. The pod label and annotated service account, a federated credential whose issuer and subject match, and a role for the identity on the target resource.
5. No, the Reader role intentionally excludes secrets.
6. They are shared static credentials that bypass Entra ID and per-user auditing.

---

## Cleanup

```bash
# Remove the test identity
az ad app delete --id $SP_APP_ID

# Remove local files and the helper
rm -rf /tmp/azsp /tmp/kube-sp /tmp/kube-sp2
unset -f as_sp

# Delete Azure resources
az group delete --name $RG --yes --no-wait

# After the group is deleted, purge the soft-deleted Key Vault so its name is freed
az keyvault purge --name $KV
```

Also remove any stray role assignments you made at subscription scope. Run `az role assignment list --assignee $ME --all -o table` and review them.

## Completion checklist

- [ ] Cluster created with Entra ID, Azure RBAC and workload identity
- [ ] Identities inventoried (cluster, kubelet)
- [ ] Forbidden error seen, then fixed with a Cluster Admin role
- [ ] Service principal tested with Reader then Writer at namespace scope
- [ ] Kubernetes `Role` and `RoleBinding` tested with an Entra identity
- [ ] Image pull failed, then worked after assigning AcrPull to the kubelet identity
- [ ] Pod read a Key Vault secret using workload identity
- [ ] Local accounts disabled and admin credentials blocked
- [ ] Four troubleshooting scenarios diagnosed
- [ ] Test identity, vault and resource group removed

## Role quick reference

| Purpose | Role | Scope |
|---|---|---|
| Download kubeconfig | Azure Kubernetes Service Cluster User Role | Cluster |
| Manage the AKS resource | Azure Kubernetes Service Contributor Role | Cluster or resource group |
| Cluster admin in Kubernetes | Azure Kubernetes Service RBAC Cluster Admin | Cluster |
| Namespace admin | Azure Kubernetes Service RBAC Admin | Namespace |
| Read and write in a namespace | Azure Kubernetes Service RBAC Writer | Namespace |
| Read-only in a namespace | Azure Kubernetes Service RBAC Reader | Namespace |
| Pull images | AcrPull | Registry (kubelet identity) |
| Read secrets in a vault | Key Vault Secrets User | Key Vault (workload identity) |

Namespace scope format: `<cluster-resource-id>/namespaces/<namespace>`.

## Notes

- Role names, CLI flags and token behaviour can change between releases. Check the current AKS and Entra ID documentation.
- Prefer Entra groups over individual users in real environments, and use the narrowest scope that works.
- Clusters, registries and vaults incur charges. Always finish with the cleanup step.
