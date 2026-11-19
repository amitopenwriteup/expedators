# AKS Operational Runbooks: Failed-State Recovery (Portal-Based Lab)

| | |
|---|---|
| **Format** | Hands-on lab, Azure portal only (no CLI required) |
| **Duration** | 90-120 minutes |
| **Level** | Intermediate |
| **Cost** | Low. 1 small AKS cluster; delete at the end |

---

## 1. Lab Objectives

By the end of this lab you will be able to:

1. Identify the difference between **provisioning state**, **power state**, and **node/Kubernetes health** in the portal.
2. Diagnose a cluster or node pool stuck in **Failed** state using Activity log, Diagnose and solve problems, and Resource health.
3. Recover a failed cluster / node pool by **reconciling** (re-applying a valid operation).
4. Recover a **NotReady** node by reimaging, and by cordon/drain/replace.
5. Resolve failures caused by **resource locks**, **quota**, and a **stopped cluster**.
6. Produce a short, reusable runbook entry for each failure type.

## 2. Key Concepts

| State type | Where to see it | Example values |
|---|---|---|
| **Provisioning state** (ARM) | Cluster > Overview, Node pools blade | Succeeded, Updating, Failed, Canceled |
| **Power state** | Cluster > Overview | Running, Stopped |
| **Node health** (Kubernetes) | Cluster > Kubernetes resources > Workloads / Node pools > Nodes; `kubectl get nodes` via Run command | Ready, NotReady |

Core principle: **a Failed provisioning state does not necessarily mean the workloads are down.** Always check whether workloads are still serving before choosing a recovery action.

Typical recovery order (least to most disruptive):

1. Read the error (Activity log / Diagnostics)
2. Fix the root cause (quota, lock, network, permissions)
3. Reconcile (re-apply the operation)
4. Reimage / replace the node or pool
5. Recreate the cluster (last resort)

## 3. Prerequisites

- An Azure subscription with **Contributor** (or Owner, needed for Exercise 4 locks) on a lab resource group.
- Ability to create AKS clusters (check **Subscriptions > Usage + quotas** for available regional vCPUs; you need about 8 vCPUs spare).
- Browser access to the Azure portal. No local tools needed.

> **Note:** Portal labels change over time. If a menu name differs slightly, search the blade using the portal's top search bar.

## 4. Lab Setup

### 4.1 Create the resource group
1. Portal > **Resource groups** > **Create**.
2. Name: `rg-aks-recovery-lab`. Region: your nearest region. **Review + create** > **Create**.

### 4.2 Create the AKS cluster
1. Portal > **Kubernetes services** > **Create** > **Create a Kubernetes cluster**.
2. **Basics**
   - Resource group: `rg-aks-recovery-lab`
   - Cluster name: `aks-recovery-lab`
   - Preset: **Dev/Test**
   - Node size: `Standard_D2s_v5` (or similar 2-vCPU size available to you)
   - Node count: **2**
3. **Node pools**: keep `agentpool` as the system pool. Click **Add node pool** and create a user pool named `userpool1` with 1 node, same size.
4. Leave networking defaults. **Review + create** > **Create**. Wait for deployment (5-10 minutes).

### 4.3 Deploy a sample workload
1. Open the cluster > **Kubernetes resources** > **Workloads** > **Create** > **Apply a YAML**.
2. Paste:

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: web-demo
spec:
  replicas: 4
  selector:
    matchLabels:
      app: web-demo
  template:
    metadata:
      labels:
        app: web-demo
    spec:
      containers:
      - name: nginx
        image: nginx:stable
        ports:
        - containerPort: 80
        resources:
          requests:
            cpu: 100m
            memory: 64Mi
---
apiVersion: v1
kind: Service
metadata:
  name: web-demo
spec:
  type: LoadBalancer
  selector:
    app: web-demo
  ports:
  - port: 80
    targetPort: 80
```

3. Click **Add**. Under **Services and ingresses**, wait for an external IP and confirm it serves the nginx page.

### 4.4 Record the baseline
Fill in this table. You will compare against it after every exercise.

| Item | Baseline value |
|---|---|
| Provisioning state | Succeeded |
| Power state | Running |
| Kubernetes version | ______ |
| Nodes Ready | ___ / 3 |
| web-demo pods Running | 4 / 4 |
| Node resource group (`MC_...`) | ______ |

---

## 5. Exercise 1: Cluster in Failed State After a Bad Operation

**Scenario:** A scale operation fails (quota or capacity), leaving the node pool and cluster in **Failed** provisioning state.

### 5.1 Inject the fault
1. Cluster > **Node pools** > select `userpool1` > **Scale node pool**.
2. Choose **Manual**, and set node count to a value that exceeds your regional vCPU quota (for a 2-vCPU VM, try a count that pushes you past your remaining quota; check **Usage + quotas** first).
3. **Apply**. The operation should fail.

> If you cannot exceed quota safely, alternatively add a new node pool using a VM size unavailable in your region/zone. Either produces a Failed provisioning state.

### 5.2 Observe
1. Cluster > **Overview**: note the **Provisioning state** (Failed) and error banner.
2. Cluster > **Node pools**: note which pool shows Failed.
3. Confirm workloads still serve (open the external IP). **Record:** are the workloads impacted? ______

### 5.3 Diagnose
1. Cluster > **Activity log**. Filter: **Timespan** = Last hour, **Status** = Failed. Open the failed `Create or Update Managed Cluster` / `Create or Update Agent Pool` event and read the **JSON** tab. Find the `statusMessage` / error code (examples: `QuotaExceeded`, `AllocationFailed`, `SkuNotAvailable`).
2. Cluster > **Diagnose and solve problems** > **Cluster and Control Plane Availability and Performance** (or search for the failure type). Review findings.
3. **Record the error code:** ______

### 5.4 Fix the root cause
- **QuotaExceeded:** Subscriptions > **Usage + quotas** > filter by region and VM family > **Request increase**, or choose a smaller node count/different VM family.
- **AllocationFailed / SkuNotAvailable:** choose a different VM size or availability zone configuration.

### 5.5 Reconcile
1. Node pools > `userpool1` > **Scale node pool** > set a **valid** count (for example, 1 or 2).
2. **Apply**. A successful operation moves the pool and cluster back to **Succeeded**.
3. If the cluster stays Failed even though the pool is healthy, perform a benign update to force reconciliation: Cluster > **Tags** > add `reconcile=true` > **Apply**, or Cluster > **Cluster configuration** > **Upgrade** tab > re-select the *current* Kubernetes version and save.

### 5.6 Verify
- Provisioning state: **Succeeded**
- Node pools: all **Succeeded**
- web-demo: 4/4 pods Running

### Runbook entry (complete this)

| Field | Your notes |
|---|---|
| Symptom | |
| First check | Activity log JSON error |
| Common causes | Quota, SKU capacity, policy, permissions |
| Fix | |
| Reconcile action | |
| Validation | |

---

## 6. Exercise 2: Node in NotReady State

**Scenario:** A worker node stops reporting to the control plane.

### 6.1 Inject the fault
1. Open the **node resource group** (`MC_rg-aks-recovery-lab_aks-recovery-lab_<region>`).
2. Open the **Virtual machine scale set** for `userpool1` > **Instances**.
3. Select the instance > **Stop** (deallocate is fine). Wait 3-5 minutes.

> Warning: Manual changes in the node resource group are unsupported for production. Use only in a lab to simulate a failure.

### 6.2 Observe
1. Cluster > **Node pools** > `userpool1` > **Nodes** (or Kubernetes resources > Workloads). The node shows **NotReady**.
2. Cluster > **Insights** > **Nodes** tab (if monitoring enabled): note CPU/memory gap.
3. Pods on that node may show **Unknown/Terminating** or be rescheduled to other nodes after the eviction timeout (about 5 minutes).

### 6.3 Diagnose
1. Cluster > **Diagnose and solve problems** > **Node Health** / **Node not ready** detectors.
2. Cluster > **Run command** (Kubernetes resources > Run command, or top toolbar). Run:

```
kubectl get nodes -o wide
kubectl describe node <node-name>
kubectl get pods -A -o wide | grep <node-name>
```

3. Read **Conditions** (Ready, MemoryPressure, DiskPressure, NetworkUnavailable) and **Events**.

### 6.4 Recover (least to most disruptive)

**Option A: Start the instance**
1. Node resource group > VMSS > Instances > select instance > **Start**.
2. Wait for the node to return to **Ready**.

**Option B: Reimage the instance**
1. VMSS > Instances > select instance > **Reimage**. Confirm.
2. Node re-registers with a clean OS disk. Local (non-persistent) data is lost.

**Option C: Cordon, drain, and replace**
1. Run command:

```
kubectl cordon <node-name>
kubectl drain <node-name> --ignore-daemonsets --delete-emptydir-data --force
```

2. Node pools > `userpool1` > **Scale node pool** to add 1 node (surge capacity), confirm the new node is Ready.
3. Delete the bad instance from the VMSS, then scale back to the desired count.

### 6.5 Verify
- All nodes **Ready**
- web-demo back to 4/4 Running (`kubectl get deploy web-demo`)
- External IP still serves traffic

### Runbook entry (complete this)

| Field | Your notes |
|---|---|
| Symptom | Node NotReady |
| Quick checks | describe node, VMSS instance state |
| Escalation ladder | Start > Reimage > Cordon/Drain/Replace |
| Data-loss risk | |

---

## 7. Exercise 3: Operations Blocked by a Resource Lock

**Scenario:** A `CanNotDelete` or `ReadOnly` lock on the node resource group or cluster causes updates, scale, or upgrades to fail and leaves the cluster Failed.

### 7.1 Inject the fault
1. Open the node resource group `MC_...` > **Locks** > **Add**.
2. Name: `lab-lock`, Lock type: **Read-only**. **OK**.
3. Cluster > **Node pools** > `userpool1` > **Scale node pool** to a different valid count > **Apply**. The operation fails.

### 7.2 Diagnose
1. Cluster > **Activity log** > open the failed event > JSON. Look for `ScopeLocked` / "cannot perform ... because the scope is locked".
2. Cluster > **Locks** and node resource group > **Locks**: list all locks.

### 7.3 Fix and reconcile
1. Delete `lab-lock` from the node resource group > **Locks**.
2. Retry the scale operation. Confirm provisioning state returns to **Succeeded**.
3. Best practice: avoid Read-only locks on AKS resources; if a lock is required, use CanNotDelete on the **cluster resource group**, not on the node resource group.

### Runbook entry (complete this)

| Field | Your notes |
|---|---|
| Error string | |
| Where to look | Activity log, Locks blades |
| Fix | |
| Prevention | |

---

## 8. Exercise 4: Stopped Cluster / Failed Start

**Scenario:** A cluster is stopped (cost saving) and fails to start, or users report the API server is unreachable.

### 8.1 Stop the cluster
1. Cluster > **Overview** > **Stop**. Wait until **Power state** = Stopped.
2. Observe: the API server is unavailable; workloads are not running.

### 8.2 Start the cluster
1. Cluster > **Overview** > **Start**.
2. Wait for **Power state** = Running and **Provisioning state** = Succeeded.

### 8.3 If the start fails
1. Check **Activity log** for the start operation error.
2. Common causes: quota consumed while stopped, expired service principal/identity credentials, policy deny, locked resource group.
3. Fix the cause, then click **Start** again. If the cluster shows Failed after the fix, reconcile as in Exercise 1 (Section 5.5).

> Notes: A cluster must be stopped/started as a whole; stopped clusters keep config and state but release compute billing. Check current limitations (such as Spot pools or minimum stop duration) in Microsoft documentation for your version.

### Runbook entry (complete this)

| Field | Your notes |
|---|---|
| Symptom | API server unreachable / Power state Stopped |
| Fix | |
| Validation | |

---

## 9. Exercise 5 (Challenge): Failed Upgrade

1. Cluster > **Cluster configuration** > **Upgrade**. Start an upgrade of the user node pool only (select node pool upgrade) to the next minor version.
2. While it runs, open **Activity log** and **Node pools** to watch surge nodes being created and drained.
3. Answer in your notes:
   - What does the portal show if the upgrade stalls on a drain (for example, a PodDisruptionBudget blocking)?
   - Which Run command output tells you which pod is blocking the drain? (Hint: `kubectl get pdb -A`, `kubectl get pods -A -o wide`.)
   - If the upgrade is **Canceled/Failed**, how do you resume? (Re-issue the same upgrade; AKS continues from where it stopped.)

---

## 10. Quick Decision Table

| Symptom in portal | Most likely cause | First action |
|---|---|---|
| Cluster **Failed**, workloads fine | Failed control-plane/node pool operation | Read Activity log JSON, fix cause, reconcile |
| Node pool **Failed** after scale | Quota / SKU / zone capacity | Request quota or change size; rescale |
| Node **NotReady** | VM stopped, kubelet or network issue | Start/Reimage; else cordon, drain, replace |
| Update fails with `ScopeLocked` | Resource lock | Remove or relocate lock |
| Power state **Stopped** | Manually stopped | **Start** the cluster |
| Upgrade stuck | PDB or draining issue | Inspect PDBs, adjust, retry upgrade |
| Cluster Failed with auth errors | Expired identity / permissions | Fix role assignments or credentials, reconcile |

## 11. Validation Checklist

- [ ] Cluster Provisioning state: **Succeeded**
- [ ] Cluster Power state: **Running**
- [ ] All node pools: **Succeeded**
- [ ] All nodes: **Ready**
- [ ] web-demo: 4/4 pods Running and reachable at the external IP
- [ ] No resource locks left on the node resource group
- [ ] Runbook entries completed for Exercises 1 to 4

## 12. Knowledge Check

1. A cluster shows **Failed** but the app is serving traffic. Is it an outage? What should you check first?
2. Where in the portal do you find the exact error code for a failed operation?
3. Why is a **Read-only** lock on the node resource group harmful to AKS?
4. Order these by increasing disruption: reimage node, start VM, cordon/drain/replace, recreate cluster.
5. What is a safe "no-op" way to force reconciliation of a Failed cluster from the portal?

<details>
<summary>Suggested answers</summary>

1. Not necessarily; the data plane can keep running. Check workload health, then read the Activity log to see which operation failed.
2. Cluster (or resource group) > **Activity log** > failed event > **JSON** tab.
3. AKS needs to create/modify resources (VMSS, NICs, disks, load balancers) in that group; a Read-only lock blocks those writes.
4. Start VM, reimage node, cordon/drain/replace, recreate cluster.
5. Add a tag, or re-apply the current Kubernetes version/scale setting, so a successful update replaces the failed state.
</details>

## 13. Cleanup

1. Remove any locks you created (Exercise 3).
2. Portal > **Resource groups** > `rg-aks-recovery-lab` > **Delete resource group** > type the name to confirm.
3. Confirm the `MC_...` node resource group is also deleted.

## 14. Runbook Template (copy for each incident type)

```
# Runbook: <Failure name>
Severity:
Detection (alert / symptom):
Impact (control plane / nodes / workloads):
Diagnose:
  1. Portal blade / Run command to check
  2. Error code to look for
Recover:
  1. Fix root cause
  2. Reconcile or replace
Verify:
  - Provisioning state, Power state, node Ready, app health
Rollback / escalation:
Owner / last tested:
```

## 15. References

- Azure Kubernetes Service documentation: Troubleshooting guides
- AKS: Stop and start a cluster
- AKS: Upgrade a cluster and node pools
- AKS: Resource lock and node resource group guidance
- Azure Portal: Activity log and Diagnose and solve problems
