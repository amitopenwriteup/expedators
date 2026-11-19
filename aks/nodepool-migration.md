# Lab: Two Node Pools, Labels, nodeSelector and Workload Migration

**Goal:** Add a second node pool from the portal, label both pools, run a workload on one pool with a `nodeSelector`, then migrate it to the other pool with zero downtime and back.

**Prereqs:** Running AKS cluster, Cloud Shell (Bash) or local `kubectl`, quota for 1 extra small node.

```
Existing pool (nppool)    label pool=blue   <---- migrate ---->   New pool (greenpool)   label pool=green

Deployment web:  nodeSelector pool=blue  ->  change to pool=green  ->  rolling update moves the pods
```

---

## Part 1: Pre-flight

```bash
RG=<your-rg>
AKS=<your-cluster>

az aks get-credentials -g $RG -n $AKS --overwrite-existing

az aks nodepool list -g $RG --cluster-name $AKS \
  --query "[].{name:name, mode:mode, type:type, vm:vmSize, count:count}" -o table

kubectl get nodes -o custom-columns='NODE:.metadata.name,AGENTPOOL:.metadata.labels.agentpool,POOL:.metadata.labels.pool'
```

Expected:
```
Name    Mode    Type              Vm                Count
------  ------  ----------------  ----------------  -----
nppool  System  Virtual machines  Standard_D2as_v5  2

NODE                              AGENTPOOL   POOL
aks-nppool-12345678-vmss000000    nppool      <none>
aks-nppool-12345678-vmss000001    nppool      <none>
```

The `type` value can differ on your cluster. It does not matter for this lab.

---

## Part 2: Create the second node pool with a label (Azure portal)

1. Portal → your AKS cluster → **Settings** → **Node pools** → **+ Add node pool**.
2. **Node pool tab**
   - Node pool name: `greenpool` (lowercase letters and digits, max 12)
   - Mode: **User**
   - OS type: **Linux**
   - Node size: **Standard_D2as_v5** (or any small size)
   - Scale method: **Manual**, node count **1** (Autoscale min 1, max 2 also works)
3. **Optional settings tab** → **Labels**: add key `pool`, value `green`. Leave taints empty.
4. **Add**. Provisioning takes 3 to 5 minutes.

Labels and field names can differ slightly between portal versions.

Verify:

```bash
az aks nodepool list -g $RG --cluster-name $AKS --query "[].{name:name, mode:mode, count:count, labels:nodeLabels}" -o table

kubectl get nodes -o custom-columns='NODE:.metadata.name,AGENTPOOL:.metadata.labels.agentpool,POOL:.metadata.labels.pool'
```

Expected:
```
Name        Mode    Count   Labels
----------  ------  ------  ----------------
nppool      System  2
greenpool   User    1       {'pool': 'green'}

NODE                                AGENTPOOL    POOL
aks-nppool-12345678-vmss000000      nppool       <none>
aks-nppool-12345678-vmss000001      nppool       <none>
aks-greenpool-23456789-vmss000000   greenpool    green
```

---

## Part 3: Label the existing pool

Pick one option. Option A or B persists the label on the pool; option C is the fallback.

**Option A: portal.** Node pools → `nppool` → **Configuration** → add label `pool=blue` → **Save**, if your portal lets you edit labels there.

**Option B: CLI.**

```bash
az aks nodepool update -g $RG --cluster-name $AKS -n nppool --labels pool=blue
```

On some pool types (for example Virtual machines pools) this can fail with the same `VirtualMachinesProfile` error you saw with app routing. If it does, use option C.

**Option C: label the nodes directly (always works).**

```bash
kubectl label nodes -l agentpool=nppool pool=blue
```

This label lives on the node objects only. Nodes added later by the autoscaler, an upgrade or a reimage will **not** carry it, and pods selecting `pool=blue` would stay Pending on them. It is fine for a lab.

Shortcut: AKS already labels every node `agentpool=<pool name>`, so you could select on `agentpool=nppool` and skip labeling. This lab uses custom labels to practice them.

Verify:

```bash
kubectl get nodes -L pool
```

Expected:
```
NAME                                STATUS   ROLES    AGE   VERSION   POOL
aks-greenpool-23456789-vmss000000   Ready    <none>   5m    v1.xx.x   green
aks-nppool-12345678-vmss000000      Ready    <none>   10d   v1.xx.x   blue
aks-nppool-12345678-vmss000001      Ready    <none>   10d   v1.xx.x   blue
```

---

## Part 4: Run a workload on the blue pool

```bash
kubectl create namespace migrate

cat <<'EOF' | kubectl apply -f -
apiVersion: apps/v1
kind: Deployment
metadata: { name: web, namespace: migrate }
spec:
  replicas: 3
  selector: { matchLabels: { app: web } }
  template:
    metadata: { labels: { app: web } }
    spec:
      nodeSelector: { pool: blue }
      containers:
      - name: app
        image: mcr.microsoft.com/azuredocs/aks-helloworld:v1
        ports: [{ containerPort: 80 }]
        resources:
          requests: { cpu: 100m, memory: 64Mi }
          limits: { cpu: 200m, memory: 128Mi }
        readinessProbe:
          httpGet: { path: /, port: 80 }
          initialDelaySeconds: 3
---
apiVersion: v1
kind: Service
metadata: { name: web, namespace: migrate }
spec:
  selector: { app: web }
  ports: [{ port: 80, targetPort: 80 }]
EOF

pp() { kubectl get pods -n migrate -l app=web -o custom-columns='POD:.metadata.name,NODE:.spec.nodeName,STATUS:.status.phase'; }

kubectl rollout status deployment/web -n migrate
pp
```

Expected:
```
deployment "web" successfully rolled out

POD           NODE                             STATUS
web-xxxxx-a   aks-nppool-12345678-vmss000000   Running
web-xxxxx-b   aks-nppool-12345678-vmss000001   Running
web-xxxxx-c   aks-nppool-12345678-vmss000000   Running
```

All three pods run on `nppool` because only those nodes match `pool=blue`.

---

## Part 5: Migrate to the green pool (watch for downtime)

Start a probe pod that calls the service twice a second. It runs on any node and counts the HTTP status codes.

```bash
kubectl run probe -n migrate --image=curlimages/curl --restart=Never \
  --overrides='{"spec":{"containers":[{"name":"probe","image":"curlimages/curl","command":["sh","-c","while true; do curl -s -o /dev/null -w \"%{http_code}\\n\" -m 2 http://web; sleep 0.5; done"],"resources":{"requests":{"cpu":"20m","memory":"16Mi"},"limits":{"cpu":"50m","memory":"32Mi"}}}]}}'
sleep 10
```

Change the selector and watch the rolling update:

```bash
kubectl patch deployment web -n migrate -p '{"spec":{"template":{"spec":{"nodeSelector":{"pool":"green"}}}}}'

kubectl rollout status deployment/web -n migrate
pp
```

Expected:
```
Waiting for deployment "web" rollout to finish: 1 out of 3 new replicas have been updated...
Waiting for deployment "web" rollout to finish: 2 out of 3 new replicas have been updated...
deployment "web" successfully rolled out

POD           NODE                                STATUS
web-yyyyy-a   aks-greenpool-23456789-vmss000000   Running
web-yyyyy-b   aks-greenpool-23456789-vmss000000   Running
web-yyyyy-c   aks-greenpool-23456789-vmss000000   Running
```

Check the probe results:

```bash
kubectl logs probe -n migrate | sort | uniq -c
```

Expected:
```
  180 200
```

Only `200` responses means the move had no downtime. The rolling update starts a new pod on the green pool, waits for its readiness probe, then removes an old pod from the blue pool.

---

## Part 6: Migrate back to the blue pool

```bash
kubectl patch deployment web -n migrate -p '{"spec":{"template":{"spec":{"nodeSelector":{"pool":"blue"}}}}}'

kubectl rollout status deployment/web -n migrate
pp
kubectl logs probe -n migrate | sort | uniq -c
```

Expected: all `web-*` pods on `aks-nppool-...` nodes, and the probe totals still show only `200`.

---

## Part 7: Selector that matches nothing

```bash
kubectl patch deployment web -n migrate -p '{"spec":{"template":{"spec":{"nodeSelector":{"pool":"red"}}}}}'
sleep 15
pp
kubectl describe pod -n migrate -l app=web | grep -m1 "FailedScheduling"
kubectl rollout status deployment/web -n migrate --timeout=20s
```

Expected:
```
POD           NODE                             STATUS
web-zzzzz-a   <none>                           Pending
web-xxxxx-a   aks-nppool-12345678-vmss000000   Running
web-xxxxx-b   aks-nppool-12345678-vmss000001   Running
web-xxxxx-c   aks-nppool-12345678-vmss000000   Running

Warning  FailedScheduling  ...  0/3 nodes are available: 3 node(s) didn't match Pod's node affinity/selector.
error: timed out waiting for the condition
```

No node has `pool=red`, so the new pod stays Pending. The old pods keep serving, which is why a bad selector during a rolling update does not cause an outage. Roll back:

```bash
kubectl rollout undo deployment/web -n migrate
kubectl rollout status deployment/web -n migrate
pp
```

Expected: `successfully rolled out`, all pods Running on the blue pool.

---

## Part 8: Cleanup

```bash
kubectl delete namespace migrate
```

###Note: Remove the nodepool which you created

