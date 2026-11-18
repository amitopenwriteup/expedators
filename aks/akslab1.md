# AKS Lab: Create a Cluster, Connect, and Verify Networking

Hands-on lab to create an Azure Kubernetes Service (AKS) cluster in the Azure portal, connect to it (Cloud Shell, PowerShell or Azure CLI), and verify the cluster, its network configuration and pod-to-pod connectivity.

> **Note:** These steps use default settings for **evaluation only**. Review the AKS baseline reference architecture before building a production cluster.

> **Important:** Azure Linux 2.0 is no longer supported (since Nov 30, 2025). From **Oct 31, 2026** its node images are removed and node pools can't be scaled. This lab uses **Ubuntu Linux**.

---

## 1. Lab Environment

| Item | Value |
|---|---|
| Resource group | `akg-rg-amit` |
| AKS cluster name | `akscluster-amit` |
| Node resource group (auto-created) | `MC_akg-rg-amit_akscluster-amit_southindia` |
| Node pool | `nppool` (System mode, Ubuntu Linux, autoscale min 1 / max 20) |
| Node OS image | Ubuntu 24.04.5 LTS (kernel `6.8.0-1073-azure`) |
| Container runtime | `containerd://2.3.7-4` |
| Kubernetes version | `v1.36.5` |
| Lab namespace | `netlab` |

> **Careful with names:** the resource group is `akg-rg-amit` (with a **g**). Using `aks-rg-amit` returns `ResourceGroupNotFound`.

---

## 2. Prerequisites

- An Azure account and subscription.
- An identity with the minimum permissions to create an AKS cluster.
- Basic Kubernetes knowledge and familiarity with Azure Cloud Shell.

---

## 3. Create the AKS Cluster

1. Sign in to the [Azure portal](https://portal.azure.com).
2. Select **Create a resource > Infrastructure Services > Azure Kubernetes Service (AKS)**.

### 3.1 Basics tab

| Section | Setting | Value |
|---|---|---|
| Project details | Subscription | Your Azure subscription |
| Project details | Resource group | **Create new** → `akg-rg-amit` → **Ok** |
| Cluster details | Cluster preset configuration | **Dev/Test** |
| Cluster details | Kubernetes cluster name | `akscluster-amit` |
| Cluster details | Region | **South India** |
| Cluster details | Fleet manager | **None** |
| Cluster details | Availability zones | **None** |
| Cluster details | AKS pricing tier | **Free** |

Leave the rest at default and select **Next**.

### 3.2 Node pools tab (System mode)

Select **Add node pool → Add a Virtual Machine Scale Set node pool** and configure:

| Setting | Value |
|---|---|
| Node pool name | `nppool` |
| **Mode** | **System** |
| OS SKU | **Ubuntu Linux** |
| Availability zones | **None** |
| Azure Spot instances | Unchecked |
| Node size | **Standard D2s v5** (2 vCPUs, 8 GiB) |
| Scale method | Autoscale |
| Minimum / Maximum node count | `1` / `20` |

Select **Add**.

> **System vs. User mode:** System node pools host critical system pods (CoreDNS, metrics-server, konnectivity, etc.). A cluster must always have at least one System pool. In this lab `nppool` is the only node pool, so all system pods and lab workloads run on it.

### 3.3 Review + create

Select **Review + create**, then **Create**. When deployment finishes, select **Go to resource**.

---

## 4. Connect to the Cluster

Choose **one** option.

### Option A: Azure Cloud Shell (used in this lab)

Open Cloud Shell with the **`>_`** button in the portal, then:

```bash
az account set --subscription <subscription-id>
az aks get-credentials --resource-group akg-rg-amit --name akscluster-amit --overwrite-existing
```

Expected output:

```text
Merged "akscluster-amit" as current context in /home/amit/.kube/config
```

`--overwrite-existing` replaces any stale entry for the same cluster in your kubeconfig.

### Option B: PowerShell (local)

```powershell
Connect-AzAccount
Set-AzContext -Subscription "<subscription-id>"

# First time only
Install-Module -Name Az.Aks -Scope CurrentUser
Install-AzAksCliTool        # installs kubectl (and kubelogin)

Import-AzAksCredential -ResourceGroupName akg-rg-amit -Name akscluster-amit -Force
```

> Restart PowerShell if `kubectl` isn't recognized after installing. `-Force` overwrites an existing kubeconfig entry.

### Option C: Azure CLI (local)

```bash
az login
az account set --subscription <subscription-id>
az aks get-credentials --resource-group akg-rg-amit --name akscluster-amit --overwrite-existing
```

---

## 5. Verify the Cluster

### 5.1 Nodes

```bash
kubectl get no
```

```text
NAME                       STATUS   ROLES    AGE   VERSION
aks-nppool-18427379-vms1   Ready    <none>   24m   v1.36.5
aks-nppool-18427379-vms2   Ready    <none>   19m   v1.36.5
```

Both nodes must be **Ready**. `ROLES <none>` is normal on AKS. The second node was added by the autoscaler; later in the lab (at ~78 min) only `vms1` remains, because the autoscaler scaled back down to the minimum of 1.

### 5.2 System pods

```bash
kubectl get po -A
```

All pods in `kube-system` should be **Running**. The `eraser-aks-nppool-...` pod showing **Completed (0/3)** is expected. It is a one-time image-cleanup job.

| Pod(s) | Purpose |
|---|---|
| `azure-cns`, `azure-ip-masq-agent` | Azure CNI networking and IP masquerading |
| `kube-proxy` | Service routing on each node |
| `coredns`, `coredns-autoscaler` | Cluster DNS and its autoscaler |
| `konnectivity-agent`, `konnectivity-agent-autoscaler` | Control plane ↔ node tunnel |
| `metrics-server` | Resource metrics (`kubectl top`) |
| `cloud-node-manager` | Node lifecycle integration with Azure |
| `csi-azuredisk-node`, `csi-azurefile-node` | Azure Disk / Azure Files storage drivers |
| `ama-metrics-*` | Azure Monitor managed Prometheus metrics |
| `retina-agent` | Container network observability (Retina) |
| `azure-wi-webhook-controller-manager` | Workload Identity webhook |
| `eraser-*` | Image cleaner (removes unused images) |

---

## 6. Inspect the Network Configuration

Set variables (use the correct resource group name):

```bash
export RG="akg-rg-amit"
export AKS="akscluster-amit"
```

Query the network profile:

```bash
az aks show -g $RG -n $AKS \
  --query "networkProfile.{plugin:networkPlugin, mode:networkPluginMode, dataplane:networkDataplane, policy:networkPolicy, podCidr:podCidr, serviceCidr:serviceCidr, dnsServiceIp:dnsServiceIp, outboundType:outboundType}" \
  -o yaml
```

```yaml
dataplane: azure
dnsServiceIp: 10.0.0.10
mode: overlay
outboundType: loadBalancer
plugin: azure
podCidr: 10.244.0.0/16
policy: none
serviceCidr: 10.0.0.0/16
```

| Field | Value | Meaning |
|---|---|---|
| `plugin` / `mode` | `azure` / `overlay` | **Azure CNI Overlay**: pods get IPs from a private overlay range, not from the VNet subnet |
| `dataplane` | `azure` | Azure dataplane (not Cilium) |
| `policy` | `none` | **No network policy engine**, so `NetworkPolicy` objects aren't enforced |
| `podCidr` | `10.244.0.0/16` | Pod IP range |
| `serviceCidr` | `10.0.0.0/16` | ClusterIP range |
| `dnsServiceIp` | `10.0.0.10` | CoreDNS service IP (inside `serviceCidr`) |
| `outboundType` | `loadBalancer` | Egress via the Azure load balancer's public IP |

### Node resource group

AKS creates a second, managed resource group (`MC_...`) holding the cluster's infrastructure.

```bash
export NODE_RG=$(az aks show -g $RG -n $AKS --query nodeResourceGroup -o tsv)
echo $NODE_RG
az resource list -g $NODE_RG -o table
```

```text
MC_akg-rg-amit_akscluster-amit_southindia
```

| Resource | Type |
|---|---|
| Public IP (GUID name) | `Microsoft.Network/publicIPAddresses` (outbound) |
| `kubernetes` | `Microsoft.Network/loadBalancers` |
| `akscluster-amit-agentpool` | `Microsoft.ManagedIdentity/userAssignedIdentities` |
| `aks-agentpool-36027734-nsg` | `Microsoft.Network/networkSecurityGroups` |
| `aks-vnet-36027734` | `Microsoft.Network/virtualNetworks` |
| `aks-nppool-...-vms1` | `Microsoft.Compute/virtualMachines` (node) |
| `aks-nppool-...-vmsnic-1-...` | `Microsoft.Network/networkInterfaces` |
| `aks-nppool-...-vms1_disk1_...` | `Microsoft.Compute/disks` (OS disk) |
| VM extensions (`cse-agent-1`, `computeAksLinuxBilling`, `AKSLinuxExtension`) | `Microsoft.Compute/virtualMachines/extensions` |

> **Do not modify** resources in the `MC_` group manually. AKS manages them.
> The disk's resource group appears in upper case (`MC_AKG-RG-AMIT_...`). That is a normal Azure display quirk.

---

## 7. Pod Networking Test (`netlab`)

### 7.1 Create the namespace and check node IPs

```bash
kubectl create namespace netlab
kubectl get nodes -o wide
```

```text
NAME                       STATUS   ROLES    AGE   VERSION   INTERNAL-IP   EXTERNAL-IP   OS-IMAGE             KERNEL-VERSION             CONTAINER-RUNTIME
aks-nppool-18427379-vms1   Ready    <none>   78m   v1.36.5   10.224.0.4    <none>        Ubuntu 24.04.5 LTS   6.8.0-1073-azure (amd64)   containerd://2.3.7-4
```

The node has a VNet IP (`10.224.0.4`) and no external IP.

### 7.2 Deploy 3 nginx replicas

```bash
kubectl -n netlab create deployment web --image=nginx --replicas=3
kubectl -n netlab rollout status deployment/web
kubectl -n netlab get pods -o wide
```

```text
deployment "web" successfully rolled out
NAME                   READY   STATUS    RESTARTS   AGE   IP             NODE
web-7887448d46-9klqn   1/1     Running   0          5s    10.244.0.52    aks-nppool-18427379-vms1
web-7887448d46-f8vxk   1/1     Running   0          5s    10.244.0.188   aks-nppool-18427379-vms1
web-7887448d46-mjd7d   1/1     Running   0          5s    10.244.0.42    aks-nppool-18427379-vms1
```

Pod IPs come from the overlay `podCidr` (`10.244.0.0/16`), while the node uses the VNet range (`10.224.x.x`).

### 7.3 Test pod-to-pod connectivity

```bash
kubectl -n netlab run client --image=busybox:1.36 --restart=Never -- sleep 3600
kubectl -n netlab wait --for=condition=Ready pod/client --timeout=60s

POD_IP=$(kubectl -n netlab get pod -l app=web -o jsonpath='{.items[0].status.podIP}')
kubectl -n netlab exec client -- wget -qO- --timeout=3 http://$POD_IP | head -n 4
```

```text
pod/client created
pod/client condition met
<!DOCTYPE html>
<html>
<head>
<title>Welcome to nginx!</title>
```

The nginx welcome page confirms the `client` pod can reach a `web` pod directly by pod IP.

> **Limitation:** at the time of this test the cluster had a single node, so all pods shared one node. This proves same-node connectivity only. To test **cross-node** traffic, scale the node pool to 2+ nodes (or use `--overrides` / pod anti-affinity) and repeat.

### 7.4 Check the DaemonSets

```bash
kubectl -n kube-system get daemonset
```

```text
NAME                             DESIRED   CURRENT   READY   UP-TO-DATE   AVAILABLE   NODE SELECTOR              AGE
ama-metrics-node                 1         1         1       1            1           <none>                     95m
ama-metrics-win-node             0         0         0       0            0           <none>                     95m
azure-cns                        1         1         1       1            1           <none>                     99m
azure-cns-win                    0         0         0       0            0           <none>                     99m
azure-ip-masq-agent              1         1         1       1            1           <none>                     99m
cloud-node-manager               1         1         1       1            1           <none>                     99m
cloud-node-manager-windows       0         0         0       0            0           <none>                     99m
csi-azuredisk-node               1         1         1       1            1           <none>                     99m
csi-azuredisk-node-win           0         0         0       0            0           <none>                     99m
csi-azurefile-node               1         1         1       1            1           <none>                     99m
csi-azurefile-node-win           0         0         0       0            0           <none>                     99m
kube-proxy                       1         1         1       1            1           <none>                     99m
retina-agent                     1         1         1       1            1           <none>                     96m
retina-agent-win                 0         0         0       0            0           kubernetes.io/os=windows   96m
windows-kube-proxy-initializer   0         0         0       0            0           <none>                     99m
```

Linux DaemonSets (`azure-cns`, `azure-ip-masq-agent`, `kube-proxy`, `cloud-node-manager`, `csi-*-node`, `ama-metrics-node`, `retina-agent`) show **DESIRED = CURRENT = READY = 1**, one per node. The `*-win` / `windows-*` DaemonSets show **0** because there are no Windows nodes.

---

## 8. Next Steps

- Enable a network policy engine (the cluster currently has `policy: none`) and test restricting traffic between pods.
- Expose `web` with a `ClusterIP` service, then a `LoadBalancer` service, and test DNS and external access.
- Scale to 2+ nodes and test cross-node pod traffic.
- Clean up when finished by deleting the `akg-rg-amit` resource group.

*Based on Microsoft Learn, "Quickstart: Deploy an AKS cluster using Azure portal", adapted with a System-mode node pool, PowerShell connection steps, and the output from this lab.*
