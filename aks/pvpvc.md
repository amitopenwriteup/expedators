# Lab: Persistent Volumes (PV) and Persistent Volume Claims (PVC) on AKS

A hands-on lab on persistent storage in Azure Kubernetes Service. You will use dynamic and static provisioning, Azure Disk and Azure Files, reclaim policies, volume expansion and StatefulSets, then diagnose three common storage faults.

**Level:** Intermediate | **Time:** 2 to 3 hours | **Cost:** Small but real (disks, a storage account, 2 nodes). Delete the resource group at the end.

## What you will learn

- How StorageClass, PV and PVC relate to each other
- Dynamic provisioning with Azure Disk (`ReadWriteOnce`) and Azure Files (`ReadWriteMany`)
- How data survives pod deletion, and how reclaim policy (`Delete` vs `Retain`) changes what happens to it
- Static provisioning of a pre-created Azure disk
- Expanding a volume without losing data
- Per-replica storage with a StatefulSet
- Diagnosing `Pending` PVCs, Multi-Attach errors and PVCs stuck in `Terminating`

## Concepts in 60 seconds

| Object | What it is |
|---|---|
| **StorageClass** | A template that says how to create storage (driver, SKU, reclaim policy, expansion) |
| **PersistentVolume (PV)** | A piece of real storage registered in the cluster (an Azure disk or file share) |
| **PersistentVolumeClaim (PVC)** | A pod's request for storage (size, access mode, class) |
| **Binding** | Kubernetes matches a PVC to a PV (or creates one dynamically) |

**Access modes**

| Mode | Meaning | Azure option |
|---|---|---|
| `ReadWriteOnce` (RWO) | Mounted read-write by one node | Azure Disk |
| `ReadWriteMany` (RWX) | Mounted read-write by many nodes | Azure Files |
| `ReadOnlyMany` (ROX) | Mounted read-only by many nodes | Azure Files |

**Reclaim policy**

| Policy | When the PVC is deleted |
|---|---|
| `Delete` | The PV and the Azure disk or share are deleted |
| `Retain` | The PV is kept (status `Released`) and the data stays in Azure |

## Prerequisites

- An Azure subscription (Contributor on the subscription or resource group)
- Azure CLI and `kubectl` (or use **Azure Cloud Shell**)
- Enough quota for 2 small nodes (about 4 vCPUs)

> Commands use bash. Names and sizes are examples. I have not run this lab against a live subscription, and storage class names and defaults can differ by AKS version, so check `kubectl get storageclass` first.

---

## Lab setup

```bash
RG=aks-pv-lab-rg
LOCATION=eastus
CLUSTER=aks-pv-lab

az group create --name $RG --location $LOCATION

az aks create \
  --resource-group $RG \
  --name $CLUSTER \
  --node-count 2 \
  --node-vm-size Standard_D2s_v5 \
  --generate-ssh-keys

az aks get-credentials --resource-group $RG --name $CLUSTER --overwrite-existing
NODE_RG=$(az aks show -g $RG -n $CLUSTER --query nodeResourceGroup -o tsv)

kubectl create namespace pvlab
kubectl create namespace pvlab-tsg
kubectl get nodes
```

> This cluster is created without availability zones, which keeps the static provisioning part simple. Zonal clusters need zonal disks that match the node zone.

**Checkpoint:** Both nodes show `Ready`.

---

## Part 1: Explore StorageClasses

```bash
kubectl get storageclass
kubectl describe storageclass managed-csi
```

### What to observe

AKS includes built-in classes. Typical ones:

| Class | Backing storage | Access mode |
|---|---|---|
| `default` and `managed-csi` | Azure Disk (Standard SSD) | RWO |
| `managed-csi-premium` | Azure Disk (Premium SSD) | RWO |
| `azurefile-csi` | Azure Files (Standard) | RWX |
| `azurefile-csi-premium` | Azure Files (Premium) | RWX |

Note for `managed-csi`: the provisioner (`disk.csi.azure.com`), `RECLAIMPOLICY`, `VOLUMEBINDINGMODE` (usually `WaitForFirstConsumer`) and `ALLOWVOLUMEEXPANSION`.

### Questions

1. What does `WaitForFirstConsumer` mean, and why is it useful for disks?
2. Which class would you pick for a database that needs low latency?

---

## Part 2: Dynamic provisioning with Azure Disk

### 2.1 Create the PVC

Save as `data-pvc.yaml`:

```yaml
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: data-pvc
  namespace: pvlab
spec:
  accessModes:
  - ReadWriteOnce
  storageClassName: managed-csi
  resources:
    requests:
      storage: 2Gi
```

```bash
kubectl apply -f data-pvc.yaml
kubectl get pvc -n pvlab
```

Expected: status `Pending`. With `WaitForFirstConsumer`, the disk is created only when a pod uses the claim.

### 2.2 Create a pod that uses it

Save as `disk-pod.yaml`:

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: disk-pod
  namespace: pvlab
spec:
  containers:
  - name: app
    image: busybox:1.36
    command: ["sh", "-c", "echo \"written at $(date)\" >> /data/log.txt; sleep 3600"]
    volumeMounts:
    - name: data
      mountPath: /data
  volumes:
  - name: data
    persistentVolumeClaim:
      claimName: data-pvc
```

```bash
kubectl apply -f disk-pod.yaml
kubectl wait --for=condition=Ready pod/disk-pod -n pvlab --timeout=180s
kubectl get pvc,pv -n pvlab
kubectl get pv
```

Expected: the PVC is `Bound` and a PV named like `pvc-<guid>` exists.

### 2.3 Look at the Azure side

```bash
az disk list -g $NODE_RG -o table
kubectl describe pv $(kubectl get pvc data-pvc -n pvlab -o jsonpath='{.spec.volumeName}')
```

You will see a managed disk created in the node resource group. The PV description shows the disk URI in `VolumeHandle`.

### 2.4 Prove the data persists

```bash
kubectl exec disk-pod -n pvlab -- cat /data/log.txt
kubectl delete pod disk-pod -n pvlab
kubectl apply -f disk-pod.yaml
kubectl wait --for=condition=Ready pod/disk-pod -n pvlab --timeout=180s
kubectl exec disk-pod -n pvlab -- cat /data/log.txt
```

Expected: the file now has **two** lines. The new pod reused the same disk.

**Checkpoint:** Data written by the first pod is visible to the second pod.

---

## Part 3: Reclaim policy: Delete vs Retain

### 3.1 See what the policy is now

```bash
PV=$(kubectl get pvc data-pvc -n pvlab -o jsonpath='{.spec.volumeName}')
echo $PV
kubectl get pv $PV -o jsonpath='{.spec.persistentVolumeReclaimPolicy}{"\n"}'
```

Expected: `Delete`. Deleting the PVC would delete the Azure disk.

### 3.2 Change it to Retain

```bash
kubectl patch pv $PV -p '{"spec":{"persistentVolumeReclaimPolicy":"Retain"}}'
kubectl get pv $PV
```

### 3.3 Delete the pod and the PVC

```bash
kubectl delete pod disk-pod -n pvlab
kubectl delete pvc data-pvc -n pvlab
kubectl get pv $PV
az disk list -g $NODE_RG -o table
```

Expected: the PV status is `Released`, and the disk still exists in Azure with your data.

### 3.4 Reuse the retained volume

A `Released` PV cannot be claimed again until you clear its old claim reference:

```bash
kubectl patch pv $PV -p '{"spec":{"claimRef":null}}'
kubectl get pv $PV
```

Expected: status `Available`. Create a new PVC that asks for it by name:

```bash
cat <<EOF | kubectl apply -f -
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: restore-pvc
  namespace: pvlab
spec:
  accessModes:
  - ReadWriteOnce
  storageClassName: managed-csi
  volumeName: $PV
  resources:
    requests:
      storage: 2Gi
EOF

cat <<'EOF' | kubectl apply -f -
apiVersion: v1
kind: Pod
metadata:
  name: restore-pod
  namespace: pvlab
spec:
  containers:
  - name: app
    image: busybox:1.36
    command: ["sh", "-c", "sleep 3600"]
    volumeMounts:
    - name: data
      mountPath: /data
  volumes:
  - name: data
    persistentVolumeClaim:
      claimName: restore-pvc
EOF

kubectl wait --for=condition=Ready pod/restore-pod -n pvlab --timeout=180s
kubectl exec restore-pod -n pvlab -- cat /data/log.txt
```

Expected: the earlier lines are still there.

**Checkpoint:** You recovered data after deleting the PVC by using a `Retain` policy.

### Questions

1. When would you choose `Retain` over `Delete` in production?
2. What is the operational cost of `Retain`? (someone must clean up unused disks)

---

## Part 4: Expand a volume

`restore-pvc` is 2Gi. Grow it to 4Gi.

```bash
kubectl exec restore-pod -n pvlab -- df -h /data
kubectl patch pvc restore-pvc -n pvlab -p '{"spec":{"resources":{"requests":{"storage":"4Gi"}}}}'
kubectl get pvc restore-pvc -n pvlab -w
```

Wait until `CAPACITY` shows `4Gi` (press Ctrl+C). Then check inside the pod:

```bash
kubectl exec restore-pod -n pvlab -- df -h /data
kubectl exec restore-pod -n pvlab -- cat /data/log.txt
```

Expected: the filesystem is about 4G and the data is intact. If the size has not changed, wait a minute, or restart the pod:

```bash
kubectl delete pod restore-pod -n pvlab
```

(then recreate it from the same YAML if needed).

Also check the Azure disk size:

```bash
az disk list -g $NODE_RG --query "[].{name:name,sizeGb:diskSizeGb}" -o table
```

**Checkpoint:** The PVC shows `4Gi`, and the file still has its data.

### Questions

1. Can you shrink a PVC? (no, Kubernetes does not support shrinking)
2. What must the StorageClass allow for expansion to work? (`allowVolumeExpansion: true`)

---

## Part 5: Shared storage with Azure Files (RWX)

### 5.1 Create the claim and a 2-replica deployment

Save as `files.yaml`:

```yaml
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: azurefile-pvc
  namespace: pvlab
spec:
  accessModes:
  - ReadWriteMany
  storageClassName: azurefile-csi
  resources:
    requests:
      storage: 5Gi
---
apiVersion: apps/v1
kind: Deployment
metadata:
  name: files-app
  namespace: pvlab
spec:
  replicas: 2
  selector:
    matchLabels:
      app: files-app
  template:
    metadata:
      labels:
        app: files-app
    spec:
      containers:
      - name: app
        image: busybox:1.36
        command: ["sh", "-c", "sleep 3600"]
        volumeMounts:
        - name: shared
          mountPath: /shared
      volumes:
      - name: shared
        persistentVolumeClaim:
          claimName: azurefile-pvc
```

```bash
kubectl apply -f files.yaml
kubectl rollout status deployment/files-app -n pvlab
kubectl get pvc azurefile-pvc -n pvlab
kubectl get pods -n pvlab -l app=files-app -o wide
```

### 5.2 Share data between pods

```bash
POD1=$(kubectl get pods -n pvlab -l app=files-app -o jsonpath='{.items[0].metadata.name}')
POD2=$(kubectl get pods -n pvlab -l app=files-app -o jsonpath='{.items[1].metadata.name}')

kubectl exec $POD1 -n pvlab -- sh -c 'echo "hello from pod 1" > /shared/pod1.txt'
kubectl exec $POD2 -n pvlab -- sh -c 'echo "hello from pod 2" > /shared/pod2.txt'

kubectl exec $POD1 -n pvlab -- ls /shared
kubectl exec $POD2 -n pvlab -- cat /shared/pod1.txt
```

Expected: both pods see both files.

### 5.3 See the Azure side

```bash
az storage account list -g $NODE_RG -o table
```

AKS created a storage account in the node resource group, and a file share inside it for the PV.

**Checkpoint:** Two pods wrote to and read from the same volume.

### Questions

1. Why could you not do this with `managed-csi`? (Azure Disk is RWO, mounted on one node at a time)
2. What are the trade-offs of Azure Files versus Azure Disk? (shared access versus lower latency and higher IOPS)

---

## Part 6: Static provisioning of a pre-created disk

You create the Azure disk first, then register it as a PV by hand.

### 6.1 Create the disk

```bash
az disk create -g $NODE_RG -n static-disk --size-gb 4 --sku StandardSSD_LRS
DISK_ID=$(az disk show -g $NODE_RG -n static-disk --query id -o tsv)
echo $DISK_ID
```

### 6.2 Create the PV, PVC and pod

```bash
cat <<EOF | kubectl apply -f -
apiVersion: v1
kind: PersistentVolume
metadata:
  name: pv-static-disk
spec:
  capacity:
    storage: 4Gi
  accessModes:
  - ReadWriteOnce
  persistentVolumeReclaimPolicy: Retain
  storageClassName: ""
  csi:
    driver: disk.csi.azure.com
    volumeHandle: $DISK_ID
    volumeAttributes:
      fsType: ext4
---
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: static-pvc
  namespace: pvlab
spec:
  accessModes:
  - ReadWriteOnce
  storageClassName: ""
  volumeName: pv-static-disk
  resources:
    requests:
      storage: 4Gi
---
apiVersion: v1
kind: Pod
metadata:
  name: static-pod
  namespace: pvlab
spec:
  containers:
  - name: app
    image: busybox:1.36
    command: ["sh", "-c", "echo static-data > /data/info.txt; sleep 3600"]
    volumeMounts:
    - name: data
      mountPath: /data
  volumes:
  - name: data
    persistentVolumeClaim:
      claimName: static-pvc
EOF

kubectl wait --for=condition=Ready pod/static-pod -n pvlab --timeout=180s
kubectl get pv pv-static-disk
kubectl get pvc static-pvc -n pvlab
kubectl exec static-pod -n pvlab -- cat /data/info.txt
```

Expected: PV and PVC are `Bound`, and the file contains `static-data`.

**Checkpoint:** You bound a PVC to a disk that you created yourself.

### Questions

1. Why is `storageClassName: ""` set on both the PV and the PVC?
2. When is static provisioning better than dynamic? (existing disks, restored backups, strict naming or placement control)

---

## Part 7: StatefulSet with per-replica storage

Save as `statefulset.yaml`:

```yaml
apiVersion: v1
kind: Service
metadata:
  name: web
  namespace: pvlab
spec:
  clusterIP: None
  selector:
    app: web-ss
  ports:
  - port: 80
---
apiVersion: apps/v1
kind: StatefulSet
metadata:
  name: web-ss
  namespace: pvlab
spec:
  serviceName: web
  replicas: 3
  selector:
    matchLabels:
      app: web-ss
  template:
    metadata:
      labels:
        app: web-ss
    spec:
      containers:
      - name: app
        image: busybox:1.36
        command: ["sh", "-c", "hostname >> /data/id.txt; sleep 3600"]
        volumeMounts:
        - name: data
          mountPath: /data
  volumeClaimTemplates:
  - metadata:
      name: data
    spec:
      accessModes: ["ReadWriteOnce"]
      storageClassName: managed-csi
      resources:
        requests:
          storage: 1Gi
```

```bash
kubectl apply -f statefulset.yaml
kubectl rollout status statefulset/web-ss -n pvlab
kubectl get pods,pvc -n pvlab | grep web-ss
```

Expected: pods `web-ss-0`, `web-ss-1`, `web-ss-2`, and PVCs `data-web-ss-0`, `data-web-ss-1`, `data-web-ss-2`.

### 7.1 Each replica keeps its own data

```bash
kubectl exec web-ss-0 -n pvlab -- cat /data/id.txt
kubectl delete pod web-ss-0 -n pvlab
kubectl wait --for=condition=Ready pod/web-ss-0 -n pvlab --timeout=180s
kubectl exec web-ss-0 -n pvlab -- cat /data/id.txt
```

Expected: two lines (`web-ss-0` twice). The recreated pod reattached its own claim.

### 7.2 Scale down and look at the PVCs

```bash
kubectl scale statefulset web-ss -n pvlab --replicas=1
kubectl get pods,pvc -n pvlab | grep web-ss
```

Expected: only `web-ss-0` is running, but all three PVCs remain. This protects data if you scale up again.

### 7.3 Clean up StatefulSet storage

```bash
kubectl delete statefulset web-ss -n pvlab
kubectl delete pvc -n pvlab -l app=web-ss
```

**Checkpoint:** You can explain why scaling a StatefulSet does not delete its PVCs.

---

## Part 8: Troubleshooting scenarios

Use the `pvlab-tsg` namespace.

### Scenario A: PVC stays Pending

```bash
cat <<'EOF' | kubectl apply -f -
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: fast-pvc
  namespace: pvlab-tsg
spec:
  accessModes:
  - ReadWriteOnce
  storageClassName: fast-ssd
  resources:
    requests:
      storage: 1Gi
EOF

kubectl get pvc -n pvlab-tsg
```

**Find it:**

```bash
kubectl describe pvc fast-pvc -n pvlab-tsg
kubectl get storageclass
```

Events show that the storage class `fast-ssd` was not found.

**Solution:** The `storageClassName` of a PVC cannot be changed, so recreate it with a class that exists:

```bash
kubectl delete pvc fast-pvc -n pvlab-tsg
cat <<'EOF' | kubectl apply -f -
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: fast-pvc
  namespace: pvlab-tsg
spec:
  accessModes:
  - ReadWriteOnce
  storageClassName: managed-csi
  resources:
    requests:
      storage: 1Gi
EOF
kubectl get pvc -n pvlab-tsg
```

(With `WaitForFirstConsumer`, `Pending` is normal until a pod uses it. The difference is that the event no longer reports a missing class.)

### Scenario B: Multi-Attach error with a RWO disk

Two replicas on different nodes try to use the same Azure Disk.

```bash
cat <<'EOF' | kubectl apply -f -
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: shared-disk
  namespace: pvlab-tsg
spec:
  accessModes:
  - ReadWriteOnce
  storageClassName: managed-csi
  resources:
    requests:
      storage: 1Gi
---
apiVersion: apps/v1
kind: Deployment
metadata:
  name: disk-sharers
  namespace: pvlab-tsg
spec:
  replicas: 2
  selector:
    matchLabels:
      app: disk-sharers
  template:
    metadata:
      labels:
        app: disk-sharers
    spec:
      affinity:
        podAntiAffinity:
          requiredDuringSchedulingIgnoredDuringExecution:
          - labelSelector:
              matchLabels:
                app: disk-sharers
            topologyKey: kubernetes.io/hostname
      containers:
      - name: app
        image: busybox:1.36
        command: ["sh", "-c", "sleep 3600"]
        volumeMounts:
        - name: data
          mountPath: /data
      volumes:
      - name: data
        persistentVolumeClaim:
          claimName: shared-disk
EOF

kubectl get pods -n pvlab-tsg -o wide
```

Expected: one pod `Running`, the other `ContainerCreating` for a long time.

**Find it:**

```bash
kubectl describe pod -l app=disk-sharers -n pvlab-tsg | grep -A5 Events
```

Look for `Multi-Attach error ... Volume is already exclusively attached to one node and can't be attached to another`.

**Solution options:** A disk is RWO and can attach to one node. Pick one:

- Use one replica: `kubectl scale deployment disk-sharers -n pvlab-tsg --replicas=1`
- Give each replica its own disk with a StatefulSet (`volumeClaimTemplates`)
- Use a shared RWX class such as `azurefile-csi`

Apply the first option and verify:

```bash
kubectl scale deployment disk-sharers -n pvlab-tsg --replicas=1
kubectl get pods -n pvlab-tsg
```

### Scenario C: PVC stuck in Terminating

```bash
kubectl delete pvc shared-disk -n pvlab-tsg --wait=false
kubectl get pvc -n pvlab-tsg
```

Expected: `Terminating` and it stays that way.

**Find it:**

```bash
kubectl describe pvc shared-disk -n pvlab-tsg
```

The `Used By` field lists the pod, and the `kubernetes.io/pvc-protection` finalizer prevents deletion while a pod uses the claim.

**Solution:** Remove the workload that uses it:

```bash
kubectl delete deployment disk-sharers -n pvlab-tsg
kubectl get pvc -n pvlab-tsg
```

Expected: the PVC disappears once no pod uses it. Do not remove finalizers by hand except as a last resort.

**Checkpoint:** All three faults are diagnosed and fixed, with the root cause explained.

---

## Storage troubleshooting flow

1. `kubectl get pvc,pv -A`: is the PVC `Bound`, `Pending` or `Terminating`?
2. `kubectl describe pvc <name>`: read the events (missing class, provisioning errors, quota).
3. `kubectl describe pod <pod>`: look for `FailedMount`, `FailedAttachVolume`, `Multi-Attach`.
4. Check the access mode and class match your workload (RWO vs RWX).
5. Check Azure: disk or storage account exists, quota, region, zone, and permissions.
6. Check expansion: the class allows it, and the pod has reloaded the filesystem.

---

## Knowledge check

1. What is the difference between a PV and a PVC?
2. Why can a PVC be `Pending` even when everything is correct?
3. What happens to the Azure disk when a PVC with a `Delete` policy is removed? With `Retain`?
4. Why can two pods on different nodes not share one Azure Disk, and how do you share data instead?
5. Why do StatefulSet PVCs remain after scaling down?
6. A PVC will not delete. What do you check first?

**Suggested answers**

1. A PV is the storage resource; a PVC is a request that binds to one.
2. `WaitForFirstConsumer` delays provisioning until a pod uses the claim.
3. `Delete` removes the disk; `Retain` keeps the PV (`Released`) and the disk.
4. A disk attaches to one node at a time (RWO). Use Azure Files (RWX), or one disk per replica.
5. They protect data for the next scale-up; they must be deleted deliberately.
6. Whether a pod still uses it (`Used By`, pvc-protection finalizer).

---

## Cleanup

```bash
kubectl delete namespace pvlab pvlab-tsg
kubectl delete pv pv-static-disk
kubectl get pv
az disk list -g $NODE_RG -o table

az group delete --name $RG --yes --no-wait
```

Retained PVs leave their Azure disks behind. Deleting the resource group removes the node resource group and everything in it.

## Completion checklist

- [ ] StorageClasses explored and `managed-csi` described
- [ ] Dynamic Azure Disk PVC bound and data persisted across pod restarts
- [ ] Reclaim policy switched to `Retain` and a volume reused
- [ ] PVC expanded from 2Gi to 4Gi with data intact
- [ ] Azure Files RWX volume shared between two pods
- [ ] Static disk bound through a hand-made PV
- [ ] StatefulSet created with per-replica PVCs
- [ ] Three storage faults diagnosed and fixed
- [ ] Resources deleted

## Command cheat sheet

| Task | Command |
|---|---|
| List classes | `kubectl get storageclass` |
| PVCs and PVs | `kubectl get pvc,pv -A` |
| Why is it pending | `kubectl describe pvc <name> -n <ns>` |
| Mount problems | `kubectl describe pod <pod> -n <ns>` |
| Change reclaim policy | `kubectl patch pv <pv> -p '{"spec":{"persistentVolumeReclaimPolicy":"Retain"}}'` |
| Release a PV for reuse | `kubectl patch pv <pv> -p '{"spec":{"claimRef":null}}'` |
| Expand a PVC | `kubectl patch pvc <pvc> -n <ns> -p '{"spec":{"resources":{"requests":{"storage":"4Gi"}}}}'` |
| Azure disks | `az disk list -g $NODE_RG -o table` |
| Azure storage accounts | `az storage account list -g $NODE_RG -o table` |

## Notes

- Storage class names, default SKUs and behaviour can change between AKS versions. Always check `kubectl get storageclass` and the current AKS storage documentation.
- Disks, storage accounts and nodes incur charges. Always finish with the cleanup step.
