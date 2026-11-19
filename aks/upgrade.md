Here is the lab in Markdown, based on your screenshots (cluster `k8s-amit`, upgrading from 1.35.7 to 1.36.5):

````markdown
# Lab: Upgrade an AKS Cluster from 1.35.7 to 1.36.5 using Azure Portal

## Lab Overview

| Item | Value |
|---|---|
| Cluster name | `k8s-amit` |
| Current version | 1.35.7 |
| Target version | 1.36.5 |
| Upgrade type | Minor version upgrade (1.35 -> 1.36) |
| Automatic upgrade type | Patch |
| AKS pricing tier | Free |
| Fleet Manager | None |
| Tool | Azure Portal |

**Objective:** Upgrade the AKS control plane and node pools to Kubernetes 1.36.5 and verify the cluster is healthy.

**Estimated time:** 20-40 minutes, depending on the number of nodes.

## Prerequisites

- An Azure subscription with Contributor access on the AKS resource group.
- A running AKS cluster on version 1.35.7.
- Enough vCPU quota and subnet IPs for surge nodes during the upgrade.

> AKS allows upgrades one minor version at a time. 1.35.7 -> 1.36.x is valid, but you cannot skip from 1.35 straight to 1.37.

---

## Step 1: Open the AKS cluster

1. Sign in to https://portal.azure.com
2. Search for **Kubernetes services**.
3. Click your cluster **k8s-amit**.

## Step 2: Check the current version

1. In the left menu, under **Settings**, click **Upgrades**.
2. Verify the details shown on the page:
   - **Kubernetes version:** `1.35.7`
   - **Automatic upgrade type:** `Patch`
   - **Fleet Manager:** `None`
   - **AKS pricing tier:** `Free`

![Upgrades page before upgrade](images/image1.png)

## Step 3: Pre-upgrade checks

1. Open **Overview** and confirm the cluster **Status** is *Succeeded/Running*.
2. Open **Settings > Node pools** and confirm all nodes are **Ready**.
3. Open **Kubernetes resources > Workloads** and confirm pods are Running.
4. Click **View the Kubernetes changelog** and **View the AKS changelog** (links on the Upgrades page) and review them for breaking changes.
5. Open **Diagnose and solve problems** and run the upgrade readiness checks.

## Step 4: Start the upgrade

1. On the **Upgrades** page, click **Upgrade version** under the Kubernetes version.
2. Choose the target version: **1.36.5**.
3. Choose the upgrade scope:
   - **Upgrade control plane + all node pools** (recommended for this lab), or
   - **Upgrade control plane only** (then upgrade node pools separately from the **Node pools** blade).
4. Click **Save**.

## Step 5: Monitor the upgrade

1. The Upgrades page shows the banner **"This cluster is upgrading..."**.
2. The **Upgrade version** link is greyed out and the pricing tier is locked while the upgrade runs.
3. Click **Refresh** periodically.
4. Open **Activity log** to follow the upgrade operation.
5. Open **Settings > Node pools** to watch each node pool move to the new version.

![Cluster upgrading](images/image2.png)

> Do not make other changes to the cluster while the upgrade is in progress.

## Step 6: Verify the upgrade

1. When the banner disappears, go to **Settings > Upgrades**.
2. Confirm **Kubernetes version** now shows `1.36.5`.
3. Go to **Settings > Node pools** and confirm all pools show `1.36.5` and a status of **Succeeded**.
4. Go to **Kubernetes resources > Workloads** and confirm all pods are Running.
5. Check **Kubernetes resources > Services and ingresses** and test your applications.

## Step 7: Post-upgrade tasks (optional)

- **Update the node image** under **Node pools > (select pool) > Update node image**.
- **Enable long-term support (LTS)** from the checkbox on the Upgrades page if you need extended support (this requires the Premium tier).
- **Set a maintenance window** under **Settings > Maintenance** so future upgrades run at a planned time.
- Keep the **Automatic upgrade type** as `Patch` so the cluster receives patch releases automatically.

## Troubleshooting

| Issue | Possible cause | Fix |
|---|---|---|
| Upgrade fails to start | Version skipped or not available in region | Pick the next minor version from the list |
| Upgrade stuck on node drain | PodDisruptionBudget blocks eviction | Relax the PDB or scale up replicas |
| Surge node creation fails | vCPU quota or subnet IP shortage | Increase quota or free IPs, then retry |
| Pods failing after upgrade | Removed or deprecated APIs | Check the changelog and update manifests |

> AKS does not support downgrading. If the upgrade fails, check the **Activity log**, fix the cause, and retry.

## Lab Completion Checklist

- [ ] Verified starting version 1.35.7
- [ ] Completed pre-upgrade checks
- [ ] Started the upgrade to 1.36.5
- [ ] Monitored until "This cluster is upgrading..." cleared
- [ ] Verified version 1.36.5 on the cluster and all node pools
- [ ] Confirmed workloads are healthy
````

Save your two screenshots in an `images/` folder, named `image1.png` and `image2.png`, so the image links in the lab work.
