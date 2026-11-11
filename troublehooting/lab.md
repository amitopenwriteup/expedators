# Kubernetes Lab: Troubleshooting, Maintenance, Backup & Recovery

**Environment:** 2-node kubeadm cluster (1 control plane + 1 worker)
**Runtime:** containerd
**Estimated time:** 3 to 4 hours

> Use a lab cluster only. Several exercises deliberately break things.

---

## Table of Contents

0. [Lab Setup and Baseline](#0-lab-setup-and-baseline)
1. [Pod Failure Troubleshooting](#1-pod-failure-troubleshooting)
2. [Node Failure Troubleshooting](#2-node-failure-troubleshooting)
3. [Cluster Maintenance](#3-cluster-maintenance)
4. [Backup and Recovery Concepts](#4-backup-and-recovery-concepts)
5. [Cleanup](#5-cleanup)
6. [Cheat Sheet](#6-cheat-sheet)

---

## 0. Lab Setup and Baseline

### Topology

| Hostname  | Role          | Example IP    |
|-----------|---------------|---------------|
| `cp1`     | Control plane | 192.168.56.10 |
| `worker1` | Worker        | 192.168.56.11 |

Adjust names and IPs to match your environment.

### Verify the baseline

Run on `cp1`:

```bash
kubectl get nodes -o wide
kubectl get pods -A
kubectl cluster-info
kubectl version
```

Expected: both nodes `Ready`, and all `kube-system` pods `Running`.

### Create a lab namespace

```bash
kubectl create namespace lab
kubectl config set-context --current --namespace=lab
```

### Deploy a healthy reference app

```bash
kubectl create deployment web --image=nginx:1.25 --replicas=2
kubectl expose deployment web --port=80
kubectl get pods -o wide
```

Both pods should be `Running`. Note which node each one landed on.

### Where things live on a kubeadm cluster

| Item                         | Location                                   |
|------------------------------|--------------------------------------------|
| Static pod manifests         | `/etc/kubernetes/manifests/`               |
| Component kubeconfigs        | `/etc/kubernetes/*.conf`                   |
| PKI / certificates           | `/etc/kubernetes/pki/`                     |
| etcd data                    | `/var/lib/etcd/`                           |
| kubelet config               | `/var/lib/kubelet/config.yaml`             |
| kubelet service logs         | `journalctl -u kubelet`                    |
| Container logs on node       | `/var/log/pods/`, `/var/log/containers/`   |

---

## 1. Pod Failure Troubleshooting

### 1.1 The general workflow

Always work from the outside in:

```
kubectl get pod            -> What is the STATUS?
kubectl describe pod       -> What do the Events say?
kubectl logs (--previous)  -> What did the app say?
kubectl get events         -> What did the cluster say?
node-level (crictl, journalctl) -> What did the runtime/kubelet say?
```

Core commands:

```bash
kubectl get pod <pod> -o wide
kubectl describe pod <pod>
kubectl logs <pod>
kubectl logs <pod> --previous          # logs from the last crashed container
kubectl logs <pod> -c <container>      # multi-container pods
kubectl get events --sort-by=.lastTimestamp
kubectl exec -it <pod> -- sh
kubectl get pod <pod> -o yaml
```

### 1.2 Status quick reference

| Status / Reason                | Typical cause                                         |
|--------------------------------|-------------------------------------------------------|
| `Pending`                      | Unschedulable: resources, taints, selectors, PVC      |
| `ImagePullBackOff` / `ErrImagePull` | Wrong image name/tag, registry auth, network     |
| `CrashLoopBackOff`             | App exits repeatedly: bad command, config, dependency |
| `CreateContainerConfigError`   | Missing ConfigMap/Secret or key                       |
| `OOMKilled` (in lastState)     | Memory limit too low                                  |
| `Running` but `0/1 Ready`      | Readiness probe failing                               |
| `Evicted`                      | Node pressure (disk, memory)                          |
| `Terminating` (stuck)          | Node down, finalizers, or kubelet not responding      |

---

### Lab 1.A: ImagePullBackOff

**Break it:**

```bash
kubectl run badimage --image=nginx:does-not-exist
kubectl get pod badimage -w
```

**Diagnose:**

```bash
kubectl describe pod badimage | tail -20
```

Look in Events for `Failed to pull image` and `manifest unknown`.

**Fix:**

```bash
kubectl delete pod badimage
kubectl run badimage --image=nginx:1.25
```

**Takeaway:** `ImagePullBackOff` is almost always a typo in the image or tag, a private registry without `imagePullSecrets`, or no outbound network on the node.

---

### Lab 1.B: CrashLoopBackOff

**Break it:**

```bash
kubectl run crasher --image=busybox:1.36 -- sh -c 'echo "starting"; exit 1'
kubectl get pod crasher -w
```

**Diagnose:**

```bash
kubectl describe pod crasher | grep -A8 "Last State"
kubectl logs crasher
kubectl logs crasher --previous
```

Look at `Exit Code`. Common values:

| Exit code | Meaning                                  |
|-----------|------------------------------------------|
| 0         | Container finished normally              |
| 1         | General application error                |
| 126 / 127 | Command not executable / not found       |
| 137       | SIGKILL (often OOMKilled)                |
| 143       | SIGTERM                                  |

**Fix:** Correct the command or configuration:

```bash
kubectl delete pod crasher
kubectl run crasher --image=busybox:1.36 -- sh -c 'echo "running"; sleep 3600'
```

---

### Lab 1.C: Pending due to insufficient resources

**Break it:**

```bash
cat <<'EOF' | kubectl apply -f -
apiVersion: v1
kind: Pod
metadata:
  name: hungry
spec:
  containers:
  - name: app
    image: nginx:1.25
    resources:
      requests:
        cpu: "100"
        memory: 500Gi
EOF
kubectl get pod hungry
```

**Diagnose:**

```bash
kubectl describe pod hungry | grep -A6 Events
kubectl describe nodes | grep -A8 "Allocated resources"
```

You should see `FailedScheduling ... Insufficient cpu`.

**Fix:** Lower the requests to something realistic, then re-apply.

```bash
kubectl delete pod hungry
```

---

### Lab 1.D: Pending due to nodeSelector or taints

**Break it:**

```bash
cat <<'EOF' | kubectl apply -f -
apiVersion: v1
kind: Pod
metadata:
  name: picky
spec:
  nodeSelector:
    disktype: ssd
  containers:
  - name: app
    image: nginx:1.25
EOF
kubectl describe pod picky | grep -A5 Events
```

You should see `didn't match Pod's node affinity/selector`.

**Fix (option 1):** label the node.

```bash
kubectl label node worker1 disktype=ssd
kubectl get pod picky
```

**Taint exercise:**

```bash
kubectl taint node worker1 maint=true:NoSchedule
kubectl run tainttest --image=nginx:1.25
kubectl describe pod tainttest | grep -A4 Events
```

If the pod is stuck `Pending` (the control plane is also tainted by default), remove the taint:

```bash
kubectl taint node worker1 maint=true:NoSchedule-
```

---

### Lab 1.E: OOMKilled

**Break it:**

```bash
cat <<'EOF' | kubectl apply -f -
apiVersion: v1
kind: Pod
metadata:
  name: oom
spec:
  containers:
  - name: stress
    image: polinux/stress
    command: ["stress"]
    args: ["--vm", "1", "--vm-bytes", "200M", "--vm-hang", "1"]
    resources:
      limits:
        memory: 50Mi
EOF
kubectl get pod oom -w
```

**Diagnose:**

```bash
kubectl describe pod oom | grep -B2 -A8 "Last State"
```

Expect `Reason: OOMKilled` and `Exit Code: 137`.

**Fix:** Raise the memory limit above real usage, or reduce app memory use.

```bash
kubectl delete pod oom
```

---

### Lab 1.F: Missing ConfigMap or Secret

**Break it:**

```bash
cat <<'EOF' | kubectl apply -f -
apiVersion: v1
kind: Pod
metadata:
  name: cfgerr
spec:
  containers:
  - name: app
    image: nginx:1.25
    envFrom:
    - configMapRef:
        name: app-config
EOF
kubectl get pod cfgerr
```

Status: `CreateContainerConfigError`.

**Diagnose and fix:**

```bash
kubectl describe pod cfgerr | grep -A5 Events
kubectl create configmap app-config --from-literal=MODE=lab
kubectl get pod cfgerr -w
```

The kubelet retries automatically, so the pod should start without recreation.

---

### Lab 1.G: Readiness probe failing and Service has no endpoints

**Break it:**

```bash
cat <<'EOF' | kubectl apply -f -
apiVersion: apps/v1
kind: Deployment
metadata:
  name: probe-demo
spec:
  replicas: 2
  selector:
    matchLabels: {app: probe-demo}
  template:
    metadata:
      labels: {app: probe-demo}
    spec:
      containers:
      - name: nginx
        image: nginx:1.25
        readinessProbe:
          httpGet:
            path: /does-not-exist
            port: 80
          initialDelaySeconds: 3
          periodSeconds: 5
---
apiVersion: v1
kind: Service
metadata:
  name: probe-demo
spec:
  selector: {app: probe-demo}
  ports:
  - port: 80
EOF
kubectl get pods -l app=probe-demo
kubectl get endpoints probe-demo
```

Pods are `Running` but `0/1 Ready`, and the endpoints list is empty.

**Diagnose:**

```bash
kubectl describe pod -l app=probe-demo | grep -i "readiness"
```

**Fix:** change the path to `/`:

```bash
kubectl patch deployment probe-demo --type=json -p='[{"op":"replace","path":"/spec/template/spec/containers/0/readinessProbe/httpGet/path","value":"/"}]'
kubectl get endpoints probe-demo
```

**Also check selector mismatches:**

```bash
kubectl get svc probe-demo -o jsonpath='{.spec.selector}'
kubectl get pods --show-labels
```

---

### Lab 1.H: Debugging with ephemeral containers and node tools

```bash
# Debug a running pod without modifying it
kubectl debug -it <pod> --image=busybox:1.36 --target=<container>

# Copy a pod and change its command for debugging
kubectl debug <pod> -it --copy-to=<pod>-debug --container=<container> -- sh

# On the worker node: inspect containers directly
sudo crictl ps -a
sudo crictl logs <container-id>
sudo crictl pods
```

---

## 2. Node Failure Troubleshooting

### 2.1 Node conditions

```bash
kubectl get nodes
kubectl describe node worker1
```

Review `Conditions`:

| Condition            | Healthy value | Meaning when bad                    |
|----------------------|---------------|-------------------------------------|
| `Ready`              | True          | False/Unknown: kubelet not reporting |
| `MemoryPressure`     | False         | Node low on memory                  |
| `DiskPressure`       | False         | Node low on disk                    |
| `PIDPressure`        | False         | Too many processes                  |
| `NetworkUnavailable` | False         | CNI not configured                  |

### 2.2 How the control plane reacts

1. The kubelet sends heartbeats (Lease objects in `kube-node-lease`).
2. If heartbeats stop for about 40 seconds (`node-monitor-grace-period`), the node becomes `NotReady`/`Unknown`.
3. The node lifecycle controller adds taints: `node.kubernetes.io/not-ready` and `node.kubernetes.io/unreachable`.
4. Pods without a toleration are evicted after `tolerationSeconds` (default **300 s**).
5. Controllers (Deployments, ReplicaSets) create replacement pods on healthy nodes.

> Pods with no controller (bare pods) are **not** recreated.

Check the default tolerations on a pod:

```bash
kubectl get pod <pod> -o jsonpath='{.spec.tolerations}' | jq
```

---

### Lab 2.A: Stopped kubelet

**Prep:** make sure the `web` deployment is running on `worker1`.

```bash
kubectl scale deployment web --replicas=4
kubectl get pods -o wide
```

**Break it** (on `worker1`):

```bash
sudo systemctl stop kubelet
```

**Observe** (on `cp1`):

```bash
kubectl get nodes -w
kubectl describe node worker1 | grep -A10 Conditions
kubectl describe node worker1 | grep -i taint
kubectl get pods -o wide -w
```

Expected progression: node goes `Ready` then `NotReady`, taints are added, and after about 5 minutes pods on `worker1` go `Terminating` and replacements appear on other nodes.

> With only a control plane as the other node, replacements may stay `Pending` because of the control plane taint. This is itself a useful lesson in capacity planning. For the exercise you may tolerate the taint or remove it temporarily:
> `kubectl taint nodes cp1 node-role.kubernetes.io/control-plane:NoSchedule-`
> Restore it afterward with `kubectl taint nodes cp1 node-role.kubernetes.io/control-plane:NoSchedule`

**Diagnose** (on `worker1`):

```bash
sudo systemctl status kubelet
sudo journalctl -u kubelet -n 100 --no-pager
```

**Fix:**

```bash
sudo systemctl start kubelet
```

Watch the node return to `Ready`.

---

### Lab 2.B: Container runtime down

**Break it** (on `worker1`):

```bash
sudo systemctl stop containerd
```

**Observe:**

```bash
kubectl get nodes -w
sudo systemctl status kubelet
sudo journalctl -u kubelet -n 50 --no-pager | grep -i -E "runtime|containerd|rpc"
```

**Fix:**

```bash
sudo systemctl start containerd
sudo systemctl restart kubelet
sudo crictl ps
```

**Takeaway:** the kubelet depends on the runtime. If the node is `NotReady` and kubelet logs mention CRI errors, check `containerd`.

---

### Lab 2.C: Kubelet misconfiguration

**Break it** (on `worker1`):

```bash
sudo cp /var/lib/kubelet/config.yaml /var/lib/kubelet/config.yaml.bak
echo "invalidKey: true" | sudo tee -a /var/lib/kubelet/config.yaml
sudo systemctl restart kubelet
sudo systemctl status kubelet
```

**Diagnose:**

```bash
sudo journalctl -u kubelet -n 30 --no-pager
```

**Fix:**

```bash
sudo cp /var/lib/kubelet/config.yaml.bak /var/lib/kubelet/config.yaml
sudo systemctl restart kubelet
```

Other frequent kubelet problems:

| Symptom                                    | Check                                                   |
|--------------------------------------------|---------------------------------------------------------|
| Kubelet will not start, swap errors        | `swapoff -a`, remove swap from `/etc/fstab`             |
| Cgroup driver mismatch                     | `SystemdCgroup = true` in containerd and `cgroupDriver: systemd` in kubelet |
| Cert errors / `Unauthorized`               | `/etc/kubernetes/kubelet.conf`, certificate expiry      |
| Cannot reach API server                    | Network, firewall port 6443, API server health          |

---

### Lab 2.D: Node shutdown or network loss

**Break it:** shut down `worker1` or disable its network interface (via your hypervisor), then on `cp1`:

```bash
kubectl get nodes -w
kubectl get pods -o wide
```

**Recover:** power the node back on. The kubelet should rejoin automatically.

```bash
kubectl get nodes
```

**Permanently failed node:** remove it and rejoin later.

```bash
kubectl delete node worker1
# later, on cp1:
kubeadm token create --print-join-command
# run the printed command on the rebuilt worker (after kubeadm reset)
```

---

### Lab 2.E: DiskPressure (concept and observation)

```bash
kubectl describe node worker1 | grep -i pressure
df -h /var/lib/containerd /var/lib/kubelet      # on worker1
```

Default kubelet eviction thresholds (hard): `memory.available<100Mi`, `nodefs.available<10%`, `imagefs.available<15%`. Under pressure the kubelet evicts pods (BestEffort first, then Burstable, then Guaranteed) and garbage-collects images.

Cleanup on the node:

```bash
sudo crictl rmi --prune
sudo journalctl --vacuum-size=200M
```

---

### Lab 2.F: Control plane component failure (static pods)

**Break it** (on `cp1`):

```bash
sudo mv /etc/kubernetes/manifests/kube-scheduler.yaml /tmp/
kubectl -n kube-system get pods | grep scheduler
kubectl run sched-test --image=nginx:1.25
kubectl get pod sched-test       # stays Pending, with no events
```

**Fix:**

```bash
sudo mv /tmp/kube-scheduler.yaml /etc/kubernetes/manifests/
kubectl get pod sched-test -w
```

> A pod `Pending` with **no scheduling events at all** usually means the scheduler is not running.

If the API server itself is down, `kubectl` fails. Use:

```bash
sudo crictl ps -a | grep kube-apiserver
sudo crictl logs <container-id>
sudo journalctl -u kubelet -n 50
```

---

### 2.3 Node troubleshooting checklist

```
[ ] kubectl get nodes / describe node (conditions, taints, capacity)
[ ] systemctl status kubelet, containerd
[ ] journalctl -u kubelet
[ ] swap off, time sync (chronyd), required ports open
[ ] CNI pods running (kube-system), /etc/cni/net.d present
[ ] Disk, memory, inode usage (df -h, df -i, free -m)
[ ] Certificates valid (kubeadm certs check-expiration)
[ ] API server reachable from node (curl -k https://<cp-ip>:6443/healthz)
```

---

## 3. Cluster Maintenance

### 3.1 Cordon, drain, uncordon

| Command   | Effect                                                       |
|-----------|--------------------------------------------------------------|
| `cordon`  | Marks node unschedulable. Existing pods keep running.        |
| `drain`   | Cordons, then evicts pods gracefully.                        |
| `uncordon`| Makes the node schedulable again.                            |

```bash
kubectl cordon worker1
kubectl get nodes                     # SchedulingDisabled
kubectl scale deployment web --replicas=6
kubectl get pods -o wide              # new pods avoid worker1

kubectl drain worker1 --ignore-daemonsets --delete-emptydir-data
kubectl get pods -o wide

kubectl uncordon worker1
```

Notes:

- `--ignore-daemonsets` is required because DaemonSet pods cannot be evicted.
- `--delete-emptydir-data` acknowledges `emptyDir` data will be lost.
- Bare pods block drain unless you add `--force` (they are deleted permanently).
- Drain respects PodDisruptionBudgets.

### Lab 3.A: PodDisruptionBudget

```bash
cat <<'EOF' | kubectl apply -f -
apiVersion: policy/v1
kind: PodDisruptionBudget
metadata:
  name: web-pdb
spec:
  minAvailable: 2
  selector:
    matchLabels:
      app: web
EOF
kubectl get pdb
kubectl drain worker1 --ignore-daemonsets --delete-emptydir-data
```

If the drain would drop below `minAvailable`, it will retry and print `Cannot evict pod ... disruption budget`. Observe, then:

```bash
kubectl uncordon worker1
kubectl delete pdb web-pdb
```

---

### 3.2 Upgrading a kubeadm cluster

**Rules:**

- Upgrade **one minor version at a time** (for example 1.29 to 1.30, not 1.28 to 1.30).
- Upgrade the control plane first, then workers.
- `kubelet` must not be newer than the API server.
- Back up etcd before upgrading (see section 4).

Set your versions (example values, adjust to your cluster):

```bash
export NEW_VER=1.30.2          # target version
export NEW_MINOR=v1.30         # package repo minor
```

#### Control plane (`cp1`)

```bash
# 1. Point apt at the new minor version repo (pkgs.k8s.io)
#    Edit /etc/apt/sources.list.d/kubernetes.list to use ${NEW_MINOR}

# 2. Upgrade kubeadm
sudo apt-mark unhold kubeadm
sudo apt-get update
sudo apt-get install -y kubeadm=${NEW_VER}-1.1
sudo apt-mark hold kubeadm
kubeadm version

# 3. Plan, then apply
sudo kubeadm upgrade plan
sudo kubeadm upgrade apply v${NEW_VER}

# 4. Drain, upgrade kubelet/kubectl, restart, uncordon
kubectl drain cp1 --ignore-daemonsets
sudo apt-mark unhold kubelet kubectl
sudo apt-get install -y kubelet=${NEW_VER}-1.1 kubectl=${NEW_VER}-1.1
sudo apt-mark hold kubelet kubectl
sudo systemctl daemon-reload
sudo systemctl restart kubelet
kubectl uncordon cp1
```

> Package revision suffixes (`-1.1`) vary. Run `apt-cache madison kubeadm` to see exact available versions.

#### Worker (`worker1`)

On `cp1`:

```bash
kubectl drain worker1 --ignore-daemonsets --delete-emptydir-data
```

On `worker1`:

```bash
# Update the apt repo to ${NEW_MINOR}, then:
sudo apt-mark unhold kubeadm
sudo apt-get update
sudo apt-get install -y kubeadm=${NEW_VER}-1.1
sudo apt-mark hold kubeadm

sudo kubeadm upgrade node

sudo apt-mark unhold kubelet kubectl
sudo apt-get install -y kubelet=${NEW_VER}-1.1 kubectl=${NEW_VER}-1.1
sudo apt-mark hold kubelet kubectl
sudo systemctl daemon-reload
sudo systemctl restart kubelet
```

On `cp1`:

```bash
kubectl uncordon worker1
kubectl get nodes
```

---

### 3.3 Certificate management

kubeadm certificates are valid for 1 year (CA for 10 years). Upgrading the control plane renews them automatically.

```bash
sudo kubeadm certs check-expiration
sudo kubeadm certs renew all
```

After renewing, restart the control plane static pods (move manifests out and back, or restart the kubelet) and refresh your admin kubeconfig:

```bash
sudo cp /etc/kubernetes/admin.conf ~/.kube/config
```

### 3.4 Join tokens and adding or removing nodes

```bash
# Create a new join command (tokens expire after 24 h)
kubeadm token create --print-join-command
kubeadm token list

# Remove a node cleanly
kubectl drain worker1 --ignore-daemonsets --delete-emptydir-data
kubectl delete node worker1
# On worker1:
sudo kubeadm reset
sudo rm -rf /etc/cni/net.d
```

### 3.5 Routine maintenance tasks

| Task                          | Command / Action                                         |
|-------------------------------|----------------------------------------------------------|
| Check component health        | `kubectl get --raw='/readyz?verbose'`                    |
| Review events                 | `kubectl get events -A --sort-by=.lastTimestamp`         |
| Prune unused images           | `sudo crictl rmi --prune`                                |
| Review resource usage         | `kubectl top nodes` (requires metrics-server)            |
| Check etcd health             | `etcdctl endpoint health` (see section 4)                |
| OS patching (per node)        | cordon, drain, patch, reboot, uncordon                   |

### Lab 3.B: Rolling OS maintenance simulation

```bash
kubectl drain worker1 --ignore-daemonsets --delete-emptydir-data
# on worker1: sudo apt-get update && sudo apt-get upgrade -y && sudo reboot
kubectl get nodes -w
kubectl uncordon worker1
kubectl get pods -o wide
```

Note that pods do not automatically rebalance after uncordon. Use `kubectl rollout restart deployment web` to redistribute.

---

## 4. Backup and Recovery Concepts

### 4.1 What to back up

| Component                    | Why                                   | Method                                   |
|------------------------------|---------------------------------------|------------------------------------------|
| **etcd**                     | Holds all cluster state               | `etcdctl snapshot save`                  |
| **PKI**                      | Cluster identity                      | Copy `/etc/kubernetes/pki/`              |
| **Kubeadm config & kubeconfigs** | Rebuild control plane             | Copy `/etc/kubernetes/*.conf`, manifests |
| **Application manifests**    | Redeploy workloads                    | Git (GitOps), `kubectl get -o yaml`      |
| **Persistent volumes**       | Application data                      | Storage snapshots, Velero, app-level dumps |

Key concept: **etcd backs up the cluster state, not your application data.** Persistent volume data needs its own strategy.

### 4.2 RPO and RTO

- **RPO (Recovery Point Objective):** how much data loss is acceptable. This drives backup frequency.
- **RTO (Recovery Time Objective):** how long recovery may take. This drives automation and rehearsal.

### 4.3 Install etcdctl

```bash
# Check whether it is already present
which etcdctl etcdutl

# Option: use the one in the etcd container
sudo crictl ps | grep etcd

# Option: install the client tools
sudo apt-get install -y etcd-client
```

Find the endpoints and cert paths from the manifest:

```bash
sudo grep -E "listen-client|cert-file|key-file|trusted-ca|data-dir" /etc/kubernetes/manifests/etcd.yaml
```

### Lab 4.A: Take an etcd snapshot

Create some state worth restoring:

```bash
kubectl create namespace backup-demo
kubectl -n backup-demo create deployment keepme --image=nginx:1.25 --replicas=2
kubectl -n backup-demo create configmap important --from-literal=key=value
```

Take the snapshot on `cp1`:

```bash
sudo mkdir -p /backup

sudo ETCDCTL_API=3 etcdctl \
  --endpoints=https://127.0.0.1:2379 \
  --cacert=/etc/kubernetes/pki/etcd/ca.crt \
  --cert=/etc/kubernetes/pki/etcd/server.crt \
  --key=/etc/kubernetes/pki/etcd/server.key \
  snapshot save /backup/etcd-snapshot-$(date +%F-%H%M).db
```

Verify it:

```bash
sudo etcdutl snapshot status /backup/etcd-snapshot-*.db --write-out=table
# (older versions: etcdctl snapshot status ...)
```

Also back up the PKI and configs:

```bash
sudo tar czf /backup/k8s-pki-$(date +%F).tar.gz /etc/kubernetes/pki
sudo tar czf /backup/k8s-conf-$(date +%F).tar.gz /etc/kubernetes/*.conf /etc/kubernetes/manifests
```

**Copy backups off the node** (for example with `scp`). A backup stored only on the machine it protects is not a backup.

### Lab 4.B: Simulate a disaster

```bash
kubectl delete namespace backup-demo
kubectl get ns
```

The namespace and everything inside it are gone.

### Lab 4.C: Restore etcd from snapshot

On `cp1`:

```bash
# 1. Restore into a NEW data directory
sudo ETCDCTL_API=3 etcdutl snapshot restore /backup/etcd-snapshot-<timestamp>.db \
  --data-dir=/var/lib/etcd-restore
# (older versions: etcdctl snapshot restore ... --data-dir=...)

# 2. Point the etcd static pod at the new directory
sudo vi /etc/kubernetes/manifests/etcd.yaml
```

In the manifest, change the `etcd-data` hostPath:

```yaml
  volumes:
  - hostPath:
      path: /var/lib/etcd-restore      # was /var/lib/etcd
      type: DirectoryOrCreate
    name: etcd-data
```

The kubelet detects the manifest change and recreates etcd. Wait 1 to 2 minutes (kubectl may fail briefly).

```bash
sudo crictl ps | grep etcd
kubectl get ns
kubectl -n backup-demo get all
kubectl -n backup-demo get configmap important
```

The `backup-demo` namespace and its resources should be back.

> Restoring is a point-in-time rollback of the **entire cluster state**. Anything created after the snapshot is lost.

If workloads look stale after a restore, restart the kubelet on all nodes and check that pods reconcile:

```bash
sudo systemctl restart kubelet     # on each node
kubectl get pods -A
```

### 4.4 Restore concepts for a multi-member etcd

This lab has a single stacked etcd member. In an HA cluster:

- Restore the **same snapshot** on every member.
- Supply `--name`, `--initial-cluster`, `--initial-advertise-peer-urls`, and `--initial-cluster-token` on each restore.
- Start all members together.

### 4.5 Recovering a lost control plane node

Scenario: `cp1` is gone entirely, but you have `/etc/kubernetes/pki` and an etcd snapshot.

High-level steps:

1. Build a replacement node with the same hostname/IP, container runtime, and kubeadm/kubelet versions.
2. Restore `/etc/kubernetes/pki` from backup.
3. Restore the etcd snapshot into `/var/lib/etcd`.
4. Run `kubeadm init` with the same configuration, ignoring the pre-existing etcd directory:
   `sudo kubeadm init --config kubeadm-config.yaml --ignore-preflight-errors=DirAvailable--var-lib-etcd`
5. Rejoin workers if needed (new token and CA hash).

Save your cluster configuration so you can repeat the init:

```bash
kubectl -n kube-system get cm kubeadm-config -o yaml > /backup/kubeadm-config.yaml
```

### Lab 4.D: Resource-level backups

Export manifests (useful for portability, **not** a replacement for etcd backups):

```bash
mkdir -p /backup/manifests
for ns in $(kubectl get ns -o jsonpath='{.items[*].metadata.name}'); do
  mkdir -p /backup/manifests/$ns
  for kind in deployments services configmaps statefulsets daemonsets ingresses pvc; do
    kubectl -n $ns get $kind -o yaml > /backup/manifests/$ns/$kind.yaml 2>/dev/null
  done
done
```

Caveats: exported YAML contains cluster-generated fields (`status`, `uid`, `resourceVersion`), and Secrets are only base64-encoded, so protect these files.

### 4.6 Backup tooling overview

| Tool / Approach        | Scope                                | Notes                                      |
|------------------------|--------------------------------------|--------------------------------------------|
| `etcdctl snapshot`     | Entire cluster state                 | Built in, simple, point-in-time            |
| **Velero**             | Namespaces/resources + volumes       | Supports scheduled backups, object storage |
| GitOps (Argo CD/Flux)  | Declarative app config               | Git is the source of truth                 |
| CSI VolumeSnapshots    | Persistent volume data               | Requires CSI driver snapshot support       |
| App-native dumps       | Databases                            | `pg_dump`, `mysqldump`, etc.               |

### 4.7 Backup best practices

- Automate: schedule etcd snapshots with a cron job or systemd timer.
- Store copies **off-cluster** and ideally off-site.
- Encrypt backups (etcd contains Secrets).
- Keep retention rules (for example daily for 7 days, weekly for 4 weeks).
- **Test restores regularly.** An untested backup is an assumption.
- Document the runbook and record Kubernetes and etcd versions alongside each backup.

Example cron entry for `cp1` (root):

```cron
0 */6 * * * ETCDCTL_API=3 etcdctl --endpoints=https://127.0.0.1:2379 --cacert=/etc/kubernetes/pki/etcd/ca.crt --cert=/etc/kubernetes/pki/etcd/server.crt --key=/etc/kubernetes/pki/etcd/server.key snapshot save /backup/etcd-$(date +\%F-\%H\%M).db && find /backup -name 'etcd-*.db' -mtime +7 -delete
```

---

## 5. Cleanup

```bash
kubectl config set-context --current --namespace=default
kubectl delete namespace lab backup-demo --ignore-not-found
kubectl label node worker1 disktype-
kubectl uncordon worker1 cp1
kubectl get nodes
kubectl get pods -A
```

Confirm no leftover taints:

```bash
kubectl describe nodes | grep -i taint
```

If you changed the etcd data directory in Lab 4.C, you may keep it or revert and remove the unused directory after confirming the cluster is healthy.

---

## 6. Cheat Sheet

### Pod triage

```bash
kubectl get pods -A -o wide | grep -v Running
kubectl describe pod <pod>
kubectl logs <pod> --previous
kubectl get events -A --sort-by=.lastTimestamp | tail -20
kubectl debug -it <pod> --image=busybox:1.36
```

### Node triage

```bash
kubectl get nodes -o wide
kubectl describe node <node>
sudo systemctl status kubelet containerd
sudo journalctl -u kubelet -f
sudo crictl ps -a
```

### Maintenance

```bash
kubectl cordon|drain|uncordon <node>
kubeadm upgrade plan | apply | node
kubeadm certs check-expiration
kubeadm token create --print-join-command
```

### Backup / restore

```bash
etcdctl snapshot save <file>
etcdutl snapshot restore <file> --data-dir <dir>
# update hostPath in /etc/kubernetes/manifests/etcd.yaml
```

### Common mistakes to remember

- Forgetting `--ignore-daemonsets` on drain.
- Skipping minor versions during upgrades.
- Restoring etcd into the existing data directory instead of a new one.
- Storing backups only on the control plane node.
- Assuming an etcd backup covers persistent volume data.
- Treating a `Pending` pod with no events as a resource issue instead of checking the scheduler.
