# Lab: Create and Operate an AKS Cluster from the Azure Portal (UI)

A hands-on lab using only the **Azure portal** (https://portal.azure.com). You will create an AKS cluster, connect to it, deploy an app, scale, monitor, control access, break and fix a workload, then clean up.

**Level:** Beginner | **Time:** 90 to 120 minutes | **Cost:** Small but real. Delete the resource group at the end (Part 12).

> The portal changes often. Labels may differ slightly from this guide. If you cannot find an option, use the portal search bar or look for the closest wording.

## What you will learn

- Create a resource group, container registry and AKS cluster in the portal
- Read the AKS creation tabs and choose sensible settings
- Connect with `kubectl` using Cloud Shell
- Deploy and expose an application from the portal
- Scale node pools, check upgrades, and use monitoring
- Assign access with Azure role-based access control (RBAC)
- Diagnose a failing workload using portal views

## Prerequisites

- An Azure subscription with permission to create resources (Contributor on the subscription or resource group)
- A modern browser signed in to the Azure portal
- Enough vCPU quota in your region (about 4 vCPUs to start)
- No local installs are needed; you will use **Azure Cloud Shell** (the `>_` icon at the top of the portal)

---

## Part 1: Create a resource group

1. In the portal search bar, type **Resource groups** and open it.
2. Select **Create**.
3. Fill in:

| Field | Value |
|---|---|
| Subscription | Your subscription |
| Resource group | `aks-lab-rg` |
| Region | A region near you (for example East US) |

4. Select **Review + create**, then **Create**.

**Checkpoint:** `aks-lab-rg` appears in the resource groups list.

---

## Part 2: Create a container registry (optional but recommended)

1. Search for **Container registries** and select **Create**.
2. Fill in:

| Field | Value |
|---|---|
| Resource group | `aks-lab-rg` |
| Registry name | A unique lowercase name, such as `akslabacr12345` |
| Location | Same region as the resource group |
| Pricing plan | **Basic** |

3. Select **Review + create**, then **Create**.

**Checkpoint:** The registry deployment shows **Your deployment is complete**.

---

## Part 3: Create the AKS cluster

1. Search for **Kubernetes services** and open it.
2. Select **Create**, then **Kubernetes cluster**.

### Tab: Basics

| Field | Value |
|---|---|
| Subscription | Your subscription |
| Resource group | `aks-lab-rg` |
| Cluster preset configuration | **Dev/Test** (low cost, good for labs) |
| Kubernetes cluster name | `aks-lab` |
| Region | Same region as the resource group |
| Availability zones | Leave default, or select zones if your region supports them |
| AKS pricing tier | **Free** for the lab (use Standard for production SLA) |
| Kubernetes version | Leave the default |
| Automatic upgrade | Enabled with patch (recommended) |
| Authentication and Authorization | **Local accounts with Kubernetes RBAC** for this lab |

> For real environments choose **Microsoft Entra ID authentication with Azure RBAC**.

### Tab: Node pools

1. Select the default pool (**agentpool**).
2. Set:

| Field | Value |
|---|---|
| Node size | `Standard_D2s_v5` (or the cheapest size shown that has 2 vCPUs) |
| Scale method | **Autoscale** |
| Minimum node count | 1 |
| Maximum node count | 3 |

3. Select **Update**.

### Tab: Networking

| Field | Value |
|---|---|
| Private cluster | Leave **disabled** for the lab |
| Network configuration | **Azure CNI Overlay** |
| Bring your own virtual network | Leave off (AKS creates one) |
| Network policy | **Azure** or **Cilium** (or None) |
| DNS name prefix | Leave default |

### Tab: Integrations

| Field | Value |
|---|---|
| Container registry | Select the registry from Part 2 |
| Azure Monitor (Container insights) | **Enabled** |
| Azure Monitor managed Prometheus / Managed Grafana | Optional; leave default |
| Azure Policy | Optional |

Selecting the registry here gives the cluster permission to pull images from it.

### Tab: Advanced and Tags

- Leave **Advanced** as default.
- On **Tags**, add `purpose = lab` so you can find resources later.

### Review and create

1. Select **Review + create**. Wait for **Validation passed**.
2. Select **Create**.
3. Wait 5 to 10 minutes for **Your deployment is complete**, then select **Go to resource**.

**Checkpoint:** The AKS **Overview** page shows **Status: Succeeded** (and power state Running).

---

## Part 4: Explore the cluster in the portal

On the `aks-lab` page, open each of these and note what you see.

| Menu item | What to look for |
|---|---|
| **Overview** | Status, Kubernetes version, API server address, node resource group |
| **Settings > Node pools** | The `agentpool` pool, node count, VM size, scale mode |
| **Settings > Networking** | Network plugin (Azure CNI Overlay), pod and service CIDRs |
| **Settings > Cluster configuration** | Kubernetes version, upgrade settings, authentication mode |
| **Kubernetes resources > Namespaces** | `default`, `kube-system`, `kube-public`, `kube-node-lease` |
| **Kubernetes resources > Workloads** | System workloads such as `coredns` and `metrics-server` |
| **Monitoring > Insights** | Cluster, node and container health |

**Task:** Find the **node resource group** (named like `MC_aks-lab-rg_aks-lab_<region>`). Open it and list the VM scale set, load balancer and public IP that AKS created for you.

---

## Part 5: Connect to the cluster

1. On the AKS **Overview** page, select **Connect**.
2. Open **Azure Cloud Shell** (the `>_` icon at the top of the portal) and choose **Bash**.
3. Run the two commands the Connect pane shows, which look like:

```bash
az account set --subscription "<subscription-id>"
az aks get-credentials --resource-group aks-lab-rg --name aks-lab --overwrite-existing
```

4. Check the cluster:

```bash
kubectl get nodes
kubectl get pods -A
```

**Checkpoint:** At least one node shows `Ready`, and system pods are `Running`.

---

## Part 6: Deploy an application from the portal

1. In the AKS menu, go to **Kubernetes resources > Workloads**.
2. Select **Create**, then **Apply a YAML**.
3. Paste this YAML:

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: hello-web
  namespace: default
spec:
  replicas: 2
  selector:
    matchLabels:
      app: hello-web
  template:
    metadata:
      labels:
        app: hello-web
    spec:
      containers:
      - name: web
        image: nginx:1.27
        ports:
        - containerPort: 80
---
apiVersion: v1
kind: Service
metadata:
  name: hello-web
  namespace: default
spec:
  type: LoadBalancer
  selector:
    app: hello-web
  ports:
  - port: 80
    targetPort: 80
```

4. Select **Add**.
5. Open **Workloads** and wait until `hello-web` shows **2/2** ready pods.
6. Open **Kubernetes resources > Services and ingresses**. When `hello-web` shows an **External IP**, copy it.
7. Paste `http://<external-ip>` into a new browser tab.

**Checkpoint:** You see the **Welcome to nginx!** page.

---

## Part 6b: Break and fix a workload from the portal

This part shows how to read problems in the UI.

### Break it

1. Go to **Workloads > Create > Apply a YAML**.
2. Paste:

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: broken-web
  namespace: default
spec:
  replicas: 1
  selector:
    matchLabels:
      app: broken-web
  template:
    metadata:
      labels:
        app: broken-web
    spec:
      containers:
      - name: web
        image: nginx:doesnotexist-1.99
```

3. Select **Add**.

### Diagnose it

1. Open **Workloads** and find `broken-web`. It shows `0/1` ready.
2. Select it, then open the **Pods** (or **Live logs / Events**) view.
3. Look for the status **ImagePullBackOff** or **ErrImagePull**.
4. Open the pod and read the **Events**. Find the message that the image `nginx:doesnotexist-1.99` was not found.

### Fix it

1. Select `broken-web` under **Workloads**, then **YAML**.
2. Change the image to `nginx:1.27`.
3. Select **Review + save**, then **Save**.
4. Return to **Workloads** and wait for `1/1` ready.

**Checkpoint:** `broken-web` is healthy. Root cause was a wrong image tag.

---

## Part 7: Scale the node pool

1. Go to **Settings > Node pools**.
2. Select `agentpool`, then **Scale node pool**.
3. Change **Scale method** to **Manual** and set **Node count** to **2**.
4. Select **Apply**. Watch the **Provisioning state** change.
5. Check from Cloud Shell:

```bash
kubectl get nodes
```

6. Switch back to **Autoscale** (min 1, max 3) when finished.

**Checkpoint:** Two nodes appear while the count is manual.

### Add a second node pool

1. On **Node pools**, select **Add node pool**.
2. Set **Name** to `userpool1`, **Mode** to **User**, **Node size** to a 2 vCPU size, **Scale method** Manual with **Node count** 1.
3. Select **Add**, then wait for it to finish.

---

## Part 8: Check for upgrades

1. Go to **Settings > Cluster configuration**.
2. Look at **Kubernetes version** and any **Upgrade version** option.
3. Check **Automatic upgrade** and **Node security channel** settings.
4. To set a maintenance window, look for **Settings > Maintenance** (or a similar entry) and review the options. Do not apply an upgrade in this lab unless you are ready to wait.

**Checkpoint:** You can explain the difference between a Kubernetes version upgrade and a node image upgrade.

---

## Part 9: Monitoring

1. Go to **Monitoring > Insights**.
2. Select the **Cluster**, **Nodes**, **Controllers** and **Containers** tabs.
3. Find CPU and memory usage for your `hello-web` containers.
4. Open **Monitoring > Logs**. In the query editor, run:

```kusto
KubePodInventory
| where TimeGenerated > ago(1h)
| summarize count() by PodStatus
```

5. Open **Monitoring > Metrics** and chart **Node CPU usage percentage**.
6. Create an alert: **Monitoring > Alerts > Create > Alert rule**. Select a node CPU metric, set a threshold of 80 percent, and review an action group (do not save if you do not want notifications).

**Checkpoint:** You can see pod and node data. Data may take several minutes to appear after cluster creation.

---

## Part 10: Control access (IAM)

1. On the AKS page, open **Access control (IAM)**.
2. Select **Add > Add role assignment**.
3. On **Role**, search for and select **Azure Kubernetes Service Cluster User Role**. Select **Next**.
4. On **Members**, choose **User, group, or service principal**, select **Select members**, and pick a test user or group.
5. Select **Review + assign**.
6. Open the **Role assignments** tab to see the new entry.

Useful roles for AKS:

| Role | Purpose |
|---|---|
| Azure Kubernetes Service Contributor Role | Manage the cluster resource |
| Azure Kubernetes Service Cluster User Role | Download kubeconfig |
| Azure Kubernetes Service RBAC Cluster Admin | Full Kubernetes admin (needs Azure RBAC enabled on the cluster) |
| Azure Kubernetes Service RBAC Reader | Read-only inside the cluster |

> Remove the test role assignment afterwards: **Role assignments**, select it, then **Remove**.

---

## Part 11: Stop and start the cluster

1. On the AKS **Overview**, select **Stop**. Confirm.
2. Wait for the power state to show **Stopped**. Compute charges stop; storage and IPs still cost.
3. Select **Start** and wait for **Running**.
4. Run `kubectl get nodes` in Cloud Shell to confirm nodes return.

---

## Part 12: Clean up

1. Search for **Resource groups** and open `aks-lab-rg`.
2. Select **Delete resource group**.
3. Type the resource group name to confirm and select **Delete**.
4. Also remove the node resource group check: it is deleted automatically with the cluster.

**Checkpoint:** `aks-lab-rg` no longer appears in your resource groups.

---

## Knowledge check

1. Which Azure resource group holds the VM scale set, load balancer and public IP created for your cluster?
2. What is the difference between **Free** and **Standard** AKS pricing tiers?
3. Why does selecting the container registry on the **Integrations** tab matter?
4. A pod shows `ImagePullBackOff`. Which portal view shows the reason?
5. What does the **Cluster User Role** let a user do, and what does it not let them do?
6. What does **Stop** save, and what costs continue?

**Suggested answers**

1. The node resource group (`MC_...`).
2. Standard adds an uptime SLA and larger-scale features; Free has no SLA.
3. It grants the nodes permission to pull images, so you do not need image pull secrets.
4. The workload or pod **Events** view.
5. It lets the user download kubeconfig; it does not grant permissions inside Kubernetes (that needs RBAC roles).
6. It saves node compute cost; disks, public IPs and some other resources still bill.

---

## Troubleshooting

| Problem | Likely cause and fix |
|---|---|
| Cannot find **Kubernetes services** | Use the portal search bar and type `Kubernetes services` |
| Validation fails on quota or VM size | Choose another VM size or region, or request quota |
| Registry name rejected | Names must be globally unique, lowercase letters and numbers |
| Connect pane commands fail | Run `az login` in Cloud Shell and set the right subscription |
| `EXTERNAL-IP` stays pending | Wait a few minutes; check public IP quota |
| Workloads page is empty or access denied | You may need a cluster role. Ask for the Cluster User Role plus Kubernetes access |
| No data in Insights | Wait 10 to 15 minutes; confirm Container insights was enabled |

---

## Completion checklist

- [ ] Resource group and container registry created
- [ ] AKS cluster created and showing Succeeded
- [ ] Connected with `kubectl` from Cloud Shell
- [ ] `hello-web` reachable on its external IP
- [ ] `broken-web` diagnosed and fixed from the portal
- [ ] Node pool scaled and a second pool added
- [ ] Monitoring reviewed and role assignment tested
- [ ] Resource group deleted

## Notes

- Names, regions and VM sizes are examples. Use values that fit your subscription.
- I have not run this lab against a live subscription, and portal labels and tab layouts change often. Check the current AKS portal quickstart if a step does not match.
- Running clusters, node VMs, load balancers and Log Analytics all incur charges. Always finish with Part 12.
