# Lab: AKS Identity and RBAC with Microsoft Entra ID

**Goal:** Sign in to AKS with Entra ID, give access by **group**, limit a dev group to one namespace with Kubernetes Roles and RoleBindings, then disable the shared local admin accounts.

**Prereqs:** Running AKS cluster, Cloud Shell (Bash), permission to create Entra groups, and permission to assign roles on the cluster (Owner or User Access Administrator).

```
Entra ID groups                   Cluster access                  Kubernetes RBAC
aks-admins      ----------------> cluster-admin (admin group)     ClusterRoleBinding (set by AKS)
aks-dev-team    -- Cluster User Role (Azure) --> kubeconfig  -->  Role + RoleBinding in namespace dev-team
aks-viewers     -- Cluster User Role (Azure) --> kubeconfig  -->  ClusterRole "view" bound in dev-team

Azure role = "can I fetch credentials and sign in?"      Kubernetes RBAC = "what can I do inside the cluster?"
```

---

## Part 1: Pre-flight

```bash
RG=<your-rg>
AKS=<your-cluster>

az aks show -g $RG -n $AKS \
  --query "{aad:aadProfile, k8sRbac:enableRbac, localAccountsDisabled:disableLocalAccounts, sku:sku.name}" -o json
```

Expected (cluster without Entra integration):
```json
{ "aad": null, "k8sRbac": true, "localAccountsDisabled": false, "sku": "Base" }
```

| Result | Action |
|---|---|
| `aad` is `null` | Do Part 3 to enable Entra integration |
| `aad.managed` is `true` | Skip Part 3, but note the existing admin group in `adminGroupObjectIDs` |
| `aad.enableAzureRbac` is `true` | Azure RBAC for Kubernetes is also active, so Azure roles can grant extra access and results below may differ. AKS Automatic clusters usually have this on |
| `k8sRbac` is `false` | Kubernetes RBAC is off, so this lab cannot work on this cluster |

---

## Part 2: Create the Entra groups

```bash
ADMIN_GID=$(az ad group create --display-name aks-admins --mail-nickname aks-admins --query id -o tsv)
DEV_GID=$(az ad group create --display-name aks-dev-team --mail-nickname aks-dev-team --query id -o tsv)
VIEW_GID=$(az ad group create --display-name aks-viewers --mail-nickname aks-viewers --query id -o tsv)

echo $ADMIN_GID $DEV_GID $VIEW_GID
```

Add yourself to the admin group so you keep access:

```bash
az ad group member add --group aks-admins --member-id $(az ad signed-in-user show --query id -o tsv)
az ad group member list --group aks-admins --query "[].userPrincipalName" -o tsv
```

Expected:
```
<guid> <guid> <guid>
amit@cloudadmin.sbs
```

**Portal alternative:** Microsoft Entra ID → **Groups** → **New group** → type **Security** → name → **Create**. Open each group to copy its **Object ID**.

If the CLI returns `Insufficient privileges`, create the groups in the portal, or ask your Entra admin. If you reopen Cloud Shell later, re-read the IDs with `az ad group show --group aks-dev-team --query id -o tsv`.

---

## Part 3: Enable Entra integration on the cluster

Skip if `aad.managed` was already `true`.

```bash
az aks update -g $RG -n $AKS --enable-aad --aad-admin-group-object-ids $ADMIN_GID
```

Expected: JSON that includes
```json
"aadProfile": { "adminGroupObjectIDs": ["<admin-group-guid>"], "enableAzureRbac": false, "managed": true, "tenantId": "<tenant-guid>" }
```

Members of `aks-admins` now get `cluster-admin` through Entra.

---

## Part 4: Allow the groups to sign in (Azure role)

Fetching a kubeconfig needs the **Azure Kubernetes Service Cluster User Role** on the cluster. Assign it to the groups, not to individuals.

```bash
AKS_ID=$(az aks show -g $RG -n $AKS --query id -o tsv)

for GID in $ADMIN_GID $DEV_GID $VIEW_GID; do
  az role assignment create --assignee-object-id $GID --assignee-principal-type Group \
    --role "Azure Kubernetes Service Cluster User Role" --scope $AKS_ID --query roleDefinitionName -o tsv
done
```

Expected:
```
Azure Kubernetes Service Cluster User Role
Azure Kubernetes Service Cluster User Role
Azure Kubernetes Service Cluster User Role
```

This role only lets members download credentials. It gives no permissions inside the cluster.

**If you get `AuthorizationFailed` on `Microsoft.Authorization/roleAssignments/write`:** your account (for example Contributor) can manage the cluster but cannot assign roles. Assigning roles needs **Owner**, **User Access Administrator** or **Role Based Access Control Administrator** on the cluster or its resource group.

Check what you have:

```bash
az role assignment list --assignee $(az ad signed-in-user show --query id -o tsv) \
  --all --include-inherited --include-groups --query "[].{role:roleDefinitionName, scope:scope}" -o table
```

Your options:
- Ask a subscription Owner to run the loop above (send them the three group IDs from `echo $ADMIN_GID $DEV_GID $VIEW_GID`), or to grant you **User Access Administrator** on the resource group.
- **Skip this part.** Parts 5, 6, 7 and 9 do not need it, because Contributor already includes the permission to fetch cluster credentials, and you are already in the admin group. Only real sign-ins by developers in Part 8 depend on it (see the workaround there).
- The portal route (cluster → **Access control (IAM)** → **Add role assignment**) needs the same permission, so the button will be disabled for you.

---

## Part 5: Connect as an admin and verify

```bash
az aks get-credentials -g $RG -n $AKS --overwrite-existing
kubelogin convert-kubeconfig -l azurecli

kubectl get nodes
kubectl auth can-i '*' '*'
```

Expected:
```
NAME                             STATUS   ROLES    AGE   VERSION
aks-nppool-12345678-vmss000000   Ready    <none>   10d   v1.xx.x

yes
```

The token comes from your Entra sign-in, and the `yes` comes from your membership in `aks-admins`. If you see `Forbidden`, wait a minute for group membership to refresh, or run `az logout` and `az login` again.

---

## Part 6: Kubernetes RBAC limited to one namespace

```bash
kubectl create namespace dev-team
kubectl create namespace prod
```

The subject name in a RoleBinding for an Entra group is the group's **Object ID**.

```bash
cat <<EOF | kubectl apply -f -
apiVersion: rbac.authorization.k8s.io/v1
kind: Role
metadata: { name: dev-role, namespace: dev-team }
rules:
- apiGroups: [""]
  resources: [pods, pods/log, services, configmaps]
  verbs: [get, list, watch, create, update, patch, delete]
- apiGroups: ["apps"]
  resources: [deployments, replicasets]
  verbs: [get, list, watch, create, update, patch, delete]
---
apiVersion: rbac.authorization.k8s.io/v1
kind: RoleBinding
metadata: { name: dev-binding, namespace: dev-team }
subjects:
- kind: Group
  name: "$DEV_GID"
  apiGroup: rbac.authorization.k8s.io
roleRef:
  kind: Role
  name: dev-role
  apiGroup: rbac.authorization.k8s.io
---
apiVersion: rbac.authorization.k8s.io/v1
kind: RoleBinding
metadata: { name: viewers-binding, namespace: dev-team }
subjects:
- kind: Group
  name: "$VIEW_GID"
  apiGroup: rbac.authorization.k8s.io
roleRef:
  kind: ClusterRole
  name: view
  apiGroup: rbac.authorization.k8s.io
EOF

kubectl get role,rolebinding -n dev-team
```

Expected:
```
NAME                                      CREATED AT
role.rbac.authorization.k8s.io/dev-role   2026-11-19T10:00:00Z

NAME                                                   ROLE                AGE
rolebinding.rbac.authorization.k8s.io/dev-binding      Role/dev-role       5s
rolebinding.rbac.authorization.k8s.io/viewers-binding  ClusterRole/view    5s
```

Two patterns here:
- **Custom Role** (`dev-role`): you choose the exact resources and verbs. It has no `secrets`, so developers cannot read them.
- **Built-in ClusterRole `view`** bound with a **RoleBinding**: reuses a standard read-only role, but only inside `dev-team`.

---

## Part 7: Test the permissions (impersonation)

As cluster admin you can test what a group member could do, without a second account. `--as` is any user name and `--as-group` is the group Object ID.

**Developers (`aks-dev-team`):**

```bash
echo "dev: create deployments in dev-team"; kubectl auth can-i create deployments -n dev-team --as=dev-user --as-group=$DEV_GID
echo "dev: list pods in dev-team";          kubectl auth can-i list pods -n dev-team --as=dev-user --as-group=$DEV_GID
echo "dev: read secrets in dev-team";       kubectl auth can-i get secrets -n dev-team --as=dev-user --as-group=$DEV_GID
echo "dev: list pods in prod";              kubectl auth can-i list pods -n prod --as=dev-user --as-group=$DEV_GID
echo "dev: list nodes";                     kubectl auth can-i list nodes --as=dev-user --as-group=$DEV_GID
echo "dev: delete namespaces";              kubectl auth can-i delete namespaces --as=dev-user --as-group=$DEV_GID
```

Expected:
```
dev: create deployments in dev-team
yes
dev: list pods in dev-team
yes
dev: read secrets in dev-team
no
dev: list pods in prod
no
dev: list nodes
no
dev: delete namespaces
no
```

**Viewers (`aks-viewers`):**

```bash
echo "viewer: list pods in dev-team";       kubectl auth can-i list pods -n dev-team --as=view-user --as-group=$VIEW_GID
echo "viewer: create deployments";          kubectl auth can-i create deployments -n dev-team --as=view-user --as-group=$VIEW_GID
echo "viewer: read secrets";                kubectl auth can-i get secrets -n dev-team --as=view-user --as-group=$VIEW_GID
echo "viewer: list pods in prod";           kubectl auth can-i list pods -n prod --as=view-user --as-group=$VIEW_GID
```

Expected:
```
viewer: list pods in dev-team
yes
viewer: create deployments
no
viewer: read secrets
no
viewer: list pods in prod
no
```

**Admins (`aks-admins`):**

```bash
kubectl auth can-i '*' '*' --as=admin-user --as-group=$ADMIN_GID
```

Expected: `yes`

Optionally list every permission a group has in the namespace:

```bash
kubectl auth can-i --list -n dev-team --as=dev-user --as-group=$DEV_GID | head
```

---

###Part : Cleanup

```bash
kubectl delete namespace dev-team prod

# only if you completed Part 4
for GID in $DEV_GID $VIEW_GID; do
  az role assignment delete --assignee $GID --role "Azure Kubernetes Service Cluster User Role" --scope $AKS_ID
done

az ad group delete --group aks-dev-team
az ad group delete --group aks-viewers
```

If you created the test user in Part 8, delete it in the portal (Entra ID → Users → `aks-dev-user` → **Delete**).

Verify:

```bash
kubectl get ns
az ad group list --display-name aks- --query "[].displayName" -o tsv
```

Expected: no `dev-team` or `prod` namespaces, and only `aks-admins` in the group list.

Keep `aks-admins` and the Entra integration, since they are your cluster admin path. Local accounts stay disabled. To restore the previous setting, run `az aks update -g $RG -n $AKS --enable-local-accounts`.
