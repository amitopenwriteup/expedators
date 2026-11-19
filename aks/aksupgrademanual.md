# Lab: Upgrade an AKS Cluster from 1.36.5 to 1.37.1 (Control Plane First, Then Node Pool) using Azure Portal

## Lab Overview

| Item | Value |
|---|---|
| Cluster name | `k8s-amit` |
| Node pool | `agentpool` (System mode, 2 nodes, Ubuntu Linux, Standard_D4ds_v5) |
| Current version | 1.36.5 |
| Target version | 1.37.1 |
| Upgrade type | Minor version upgrade (1.36 -> 1.37) |
| Upgrade scope | Control plane only, then node pool separately |
| Automatic upgrade | Disabled |
| AKS pricing tier | Free |
| Tool | Azure Portal |

**Objective:** Upgrade the AKS control plane to Kubernetes 1.37.1 after running the upgrade readiness check, then upgrade the `agentpool` node pool to the same version and verify the cluster is healthy.

**Estimated time:** 30-45 minutes (a node pool upgrade can take up to 10 minutes per node).

## Prerequisites

- An Azure subscription with Contributor access on the AKS resource group.
- A running AKS cluster on version 1.36.5.
- Enough vCPU quota and subnet IPs for surge nodes during the node pool upgrade.
- Workloads that can tolerate nodes being drained (replicas > 1, sensible PodDisruptionBudgets).

> AKS allows upgrades one minor version at a time. 1.36.x -> 1.37.x is valid. The node pool version must never be newer than the control plane, which is why the control plane is upgraded first.

---

## Part 1: Upgrade the Control Plane

### Step 1: Open the AKS cluster

1. Sign in to https://portal.azure.com
2. Search for **Kubernetes services**.
3. Click your cluster **k8s-amit**.

### Step 2: Check the current version

1. In the left menu, under **Settings**, click **Upgrades**.
2. Verify the **Kubernetes version** shows `1.36.5`.
3. Note that **Automatic upgrade type** and **AKS pricing tier** (`Free`) are shown on the same page.

### Step 3: Open the upgrade panel

1. Under the Kubernetes version, click **Upgrade version**.
2. The **Upgrade Kubernetes version** panel opens on the right.

### Step 4: Configure the upgrade

Set the options as follows:

| Setting | Value | Notes |
|---|---|---|
| Automatic upgrade | `Disabled` | You control when upgrades happen |
| Kubernetes Version | `1.37.1` | Select from the dropdown |
| Upgrade scope | **Upgrade control plane only** | Node pools are upgraded separately in Part 2 |
| Force upgrade | Unchecked | Only use if a normal upgrade is blocked and you accept the risk |

### Step 5: Run the upgrade readiness check

1. Next to **Upgrade readiness**, click **Check**.
2. Wait for the result. Microsoft recommends running this check before every upgrade to catch potential issues.
3. Confirm the message reads: **"Cluster is ready for upgrade for this version."**
4. If the check reports issues (for example deprecated API usage), fix them first and run **Check** again.

### Step 6: Start the control plane upgrade

1. Click **Save**.
2. The Upgrades page shows the banner **"This cluster is upgrading..."**.
3. Click **Refresh** periodically, and open **Activity log** to follow the operation.
4. Wait until the banner disappears.

### Step 7: Verify the control plane

1. Go to **Settings > Upgrades**.
2. Confirm the **Kubernetes version** now shows `1.37.1`.

> At this point the control plane is on 1.37.1 while the node pool is still on 1.36.5. This is supported temporarily. Continue to Part 2.

---

## Part 2: Upgrade the Node Pool

### Step 8: Open the node pool

1. In the left menu, under **Settings**, click **Node pools**.
2. Click the node pool **agentpool**.
3. On the **Overview** page, review the essentials:
   - **Provisioning state:** Succeeded
   - **Power state:** Running
   - **Mode:** System
   - **Kubernetes version:** `1.36.5`
   - **Target node count:** 2 nodes
   - **Ready nodes:** 2/2 nodes ready

### Step 9: Open the node pool upgrade panel

1. On the top toolbar, click **Upgrade Kubernetes**.
2. The **Upgrade node pool** panel opens on the right.
3. Confirm **Cluster version** shows `1.37.1`.

### Step 10: Configure the node pool upgrade

| Setting | Value | Notes |
|---|---|---|
| Node pool Kubernetes version | `1.37.1` | Must match the cluster version |
| Node drain timeout | `30` (minutes) | Default. Allowed range is 0 minutes to 24 hours |
| Skip undrainable nodes | Checked | Nodes that cannot be drained are skipped and quarantined instead of failing the upgrade |

> The upgrade rolls out in stages so applications keep running. Expect up to about 10 minutes per node.

> Tip: run a test operation first to estimate how long your node pool takes to drain, and adjust the **Node drain timeout** if needed.

### Step 11: Start the node pool upgrade

1. Click **Save**.
2. Open **Overview** on the node pool and click **Refresh** to watch progress.
3. Open **Nodes** to watch each node being replaced with a new one on 1.37.1.
4. Wait until **Provisioning state** shows **Succeeded**.

---

## Part 3: Verify the Upgrade

1. Go to **Settings > Upgrades** and confirm the cluster version is `1.37.1`.
2. Go to **Settings > Node pools > agentpool** and confirm:
   - **Kubernetes version:** `1.37.1`
   - **Provisioning state:** Succeeded
   - **Ready nodes:** 2/2 nodes ready
3. Go to **Kubernetes resources > Workloads** and confirm all pods are Running.
4. Go to **Kubernetes resources > Services and ingresses** and test your applications.
5. If you skipped undrainable nodes, check for quarantined nodes and fix the blocking cause (PDBs, stuck pods), then re-run the node pool upgrade.

