# Lab: Pod Disruption Budget (PDB) and Node Drain on AKS

**Cluster:** `k8s-amit-test`  |  **Goal:** add a second node, run a workload, protect it with a PDB, drain a node and watch how the PDB controls the drain.

---

## 1. Concepts (read first)

| Term | Meaning |
|---|---|
| **Voluntary disruption** | Planned actions: `kubectl drain`, node upgrades, node image upgrades, autoscaler scale-down. These go through the **Eviction API**. |
| **Involuntary disruption** | Node crash, VM failure, OOM kill, `kubectl delete pod`. A PDB **does not** protect against these. |
| **PDB** | A rule that says how many pods of an app must stay up (`minAvailable`) or how many may be down (`maxUnavailable`) during voluntary disruptions. |
| **Cordon** | Marks a node unschedulable. Existing pods keep running. |
| **Drain** | Cordon + evict every pod on the node. Each eviction is checked against PDBs. |

**How the PDB works during a drain**

```
kubectl drain node-1
   -> cordon node-1 (no new pods land here)
   -> for each pod on node-1: call Eviction API
         -> would this eviction break a PDB?
               NO  -> pod is evicted, ReplicaSet creates a replacement on another node
               YES -> eviction refused (HTTP 429), drain retries every ~5 seconds
```

The drain only moves to the next pod once the replacement pod is Ready and the PDB allows another disruption.

---

## 2. Prerequisites

```bash
export RG=<your-resource-group>
export AKS=k8s-amit-test

az login
az aks get-credentials --resource-group $RG --name $AKS
kubectl get nodes
```

You should see **one** node. Note its name, for example `aks-nodepool1-12345678-vmss000000`.

---

## 3. Step 1: Add a second node (Azure Portal)

1. Azure Portal -> **Kubernetes services** -> `k8s-amit-test`.
2. Left menu -> **Settings** -> **Node pools**.
3. Click your node pool (for example `nodepool1`).
4. Click **Scale node pool**.
5. Scale method: **Manual**, Node count: **2**. Click **Apply**.
   - If the cluster autoscaler is enabled, either set min count to 2 or switch to Manual for this lab.
6. Wait until provisioning finishes (2 to 5 minutes).

> If scaling fails, check your regional vCPU quota (Subscriptions -> Usage + quotas).

Verify:

```bash
kubectl get nodes -o wide
```

Expected: two nodes in `Ready` state. Save the names:

```bash
NODE1=<first-node-name>
NODE2=<second-node-name>
```

---

## 4. Step 2: Deploy a workload

Create `web.yaml`:

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: web
  namespace: default
spec:
  replicas: 4
  selector:
    matchLabels:
      app: web
  template:
    metadata:
      labels:
        app: web
    spec:
      topologySpreadConstraints:
        - maxSkew: 1
          topologyKey: kubernetes.io/hostname
          whenUnsatisfiable: ScheduleAnyway
          labelSelector:
            matchLabels:
              app: web
      containers:
        - name: nginx
          image: nginx:1.27
          ports:
            - containerPort: 80
          readinessProbe:
            httpGet:
              path: /
              port: 80
            initialDelaySeconds: 3
            periodSeconds: 3
          resources:
            requests:
              cpu: 50m
              memory: 64Mi
            limits:
              cpu: 100m
              memory: 128Mi
```

> `ScheduleAnyway` is deliberate. With `DoNotSchedule`, pods could get stuck Pending when one node is cordoned during the drain.

```bash
kubectl apply -f web.yaml
kubectl rollout status deployment/web
kubectl get pods -l app=web -o wide
```

Expected: 4 pods, roughly 2 on each node. If all landed on one node, delete the pods once (`kubectl delete pod -l app=web`) and let them reschedule.

---

## 5. Step 3: Create the PDB

Create `web-pdb.yaml`:

```yaml
apiVersion: policy/v1
kind: PodDisruptionBudget
metadata:
  name: web-pdb
  namespace: default
spec:
  minAvailable: 3
  selector:
    matchLabels:
      app: web
```

```bash
kubectl apply -f web-pdb.yaml
kubectl get pdb web-pdb
```

Expected:

```
NAME      MIN AVAILABLE   MAX UNAVAILABLE   ALLOWED DISRUPTIONS   AGE
web-pdb   3               N/A               1                     5s
```

**Reading it:** 4 replicas minus 3 required = **1 pod** may be evicted at a time.

> Use either `minAvailable` or `maxUnavailable` in one PDB, not both. `maxUnavailable: 1` gives the same result here and keeps working if you change the replica count.

---

## 6. Step 4: Drain a node and watch the PDB work

Open **two terminals**.

**Terminal 1: watch pods**

```bash
kubectl get pods -l app=web -o wide -w
```

**Terminal 2: drain**

```bash
kubectl drain $NODE1 --ignore-daemonsets --delete-emptydir-data --timeout=300s
```

What you will see:

1. `node/<NODE1> cordoned`
2. Pods on NODE1 are evicted **one at a time**. A replacement starts on NODE2, becomes Ready, and only then is the next pod evicted.
3. At no point do fewer than 3 `web` pods run as Ready.
4. `node/<NODE1> drained`

Verify:

```bash
kubectl get pods -l app=web -o wide     # all 4 pods on NODE2
kubectl get nodes                       # NODE1 shows Ready,SchedulingDisabled
kubectl get pdb web-pdb
```

Bring the node back:

```bash
kubectl uncordon $NODE1
```

Pods do not move back automatically. To rebalance: `kubectl rollout restart deployment/web`.

---

---

## 8. Cleanup

```bash
kubectl delete -f web-pdb.yaml
kubectl delete -f web.yaml
kubectl uncordon $NODE1
```

Scale the node pool back to 1 in the portal (Node pools -> Scale node pool -> 1) to stop paying for the extra node.
