<img width="821" height="367" alt="image" src="https://github.com/user-attachments/assets/3b5417bb-8742-44ed-a903-4526a494c305" /># Deploy an Azure Kubernetes Service (AKS) Cluster Using the Azure Portal

Step-by-step guide to deploy an AKS cluster in the Azure portal, connect to it (Cloud Shell, PowerShell or Azure CLI), and run the sample **AKS Store** multi-container app.

> **Note:** These steps use default settings for **evaluation only**. Before deploying a production cluster, review the AKS baseline reference architecture.

> **Important:** Azure Linux 2.0 is no longer supported (since Nov 30, 2025). From **Oct 31, 2026** its node images are removed and node pools can't be scaled. Use Ubuntu Linux or migrate to `AzureLinux3`.

---

## 1. Prerequisites

- An Azure account (create a free account if needed).
- Basic understanding of Kubernetes concepts.
- An identity with the minimum permissions to create an AKS cluster.
- Familiarity with Azure Cloud Shell (optional but recommended).

---

## 2. Create the AKS Cluster

1. Sign in to the [Azure portal](https://portal.azure.com).
2. On the home page, select **Create a resource**.
3. Under **Categories**, select **Infrastructure Services > Azure Kubernetes Service (AKS)**.

### 2.1 Basics tab

| Section | Setting | Value |
|---|---|---|
| Project details | Subscription | Your Azure subscription |
| Project details | Resource group | **Create new** → `myResourceGroup` → **Ok** |
| Cluster details | Cluster preset configuration | **Dev/Test** |
| Cluster details | Kubernetes cluster name | `myAKSCluster` |
| Cluster details | Region | e.g. `East US 2` |
| Cluster details | Fleet manager | **None** |
| Cluster details | Availability zones | **None** (deselect any selected zones) |
| Cluster details | AKS pricing tier | **Free** |
| Cluster details | Kubernetes version | **1.35.7** |


Leave the remaining settings at their defaults and select **Next**.

> Tip: A new resource group makes cleanup easy and avoids affecting production or dev workloads.

### 2.2 Node pools tab (System mode)

### delete te default node pool, select and delete it

1. Select **Add node pool** → **Add a Virtual Machine Scale Set node pool**.
2. Configure:

| Setting | Value |
|---|---|
| Node pool name | `nplinux` |
| **Mode** | **System** |
| OS SKU | **Ubuntu Linux** |
| Availability zones | **None** |
| Enable Azure Spot instances | Unchecked |
| Node size | **Choose a size** → search `D2s_v5` → select **Standard D2s v5** (2 vCPUs, 8 GiB) → **Select** |
| Scale method | Autoscale (recommended) |
| Minimum node count | `1` |
| Maximum node count | `20` |

3. Leave other settings at default and select **Add**.

> **System vs. User mode:** System node pools host critical system pods (e.g. CoreDNS, metrics-server). User node pools mainly run your application pods. The original quickstart uses **User** mode; this guide uses **System** mode as requested. An AKS cluster must always have at least one System node pool.

### 2.3 Review + create

1. Select **Review + create** to run validation.
2. When validation passes, select **Create**.
3. Wait a few minutes, then select **Go to resource** (or open the resource group and select the AKS resource).

---

## 3. Connect to the Cluster

`kubectl` is the Kubernetes command-line client. It's preinstalled in Azure Cloud Shell. Choose **one** of the options below.

### Option A: Azure Cloud Shell

Open Cloud Shell with the **`>_`** button at the top of the Azure portal, then run:

```bash
az aks get-credentials --resource-group myResourceGroup --name myAKSCluster
```

### Option B: PowerShell (local)

1. Sign in to Azure:

   ```powershell
   Connect-AzAccount
   ```

2. (If needed) Select the subscription:

   ```powershell
   Set-AzContext -Subscription "<subscription-name-or-id>"
   ```

3. Install the Az.Aks module (first time only) and kubectl:

   ```powershell
   Install-Module -Name Az.Aks -Scope CurrentUser
   Install-AzAksCliTool        # installs kubectl (and kubelogin)
   ```

   > After installing, make sure the install folder is on your `PATH` (restart PowerShell if `kubectl` isn't recognized).

4. Download credentials and configure `kubectl`:

   ```powershell
   Import-AzAksCredential -ResourceGroupName myResourceGroup -Name myAKSCluster
   ```

### Option C: Azure CLI (local)

```bash
az login
az aks get-credentials --resource-group myResourceGroup --name myAKSCluster
```

> The Azure CLI also works inside a PowerShell window.

### Verify the connection

```bash
kubectl get nodes
```

Example output (the node status must be **Ready**):

```text
NAME                       STATUS   ROLES   AGE     VERSION
aks-nodepool1-31718369-0   Ready    agent   6m44s   v1.15.10
```

---

## 4. Test the Application

The front end is exposed to the internet through a Kubernetes LoadBalancer service. This can take a few minutes.

1. Check pod status; make sure all pods are **Running**:

   ```bash
   kubectl get pods
   ```

2. Watch for the public IP of `store-front`:

   ```bash
   kubectl get service store-front --watch
   ```

   `EXTERNAL-IP` initially shows `<pending>`:

   ```text
   NAME          TYPE           CLUSTER-IP    EXTERNAL-IP   PORT(S)        AGE
   store-front   LoadBalancer   10.0.100.10   <pending>     80:30025/TCP   4h4m
   ```

   Once a real IP appears, press **Ctrl+C** to stop watching:

   ```text
   NAME          TYPE           CLUSTER-IP    EXTERNAL-IP    PORT(S)        AGE
   store-front   LoadBalancer   10.0.100.10   20.62.159.19   80:30025/TCP   4h5m
   ```

3. Open a browser to the external IP (e.g. `http://20.62.159.19`) to see the Azure Store app.

---

## 5. Next Steps

- AKS tutorial series: a full code-to-deployment walkthrough.
- AKS solution guidance for production-ready designs.

*Source: Microsoft Learn, "Quickstart: Deploy an AKS cluster using Azure portal" (last updated 12/21/2025). Modified to use a System-mode node pool and to include PowerShell connection steps.*
