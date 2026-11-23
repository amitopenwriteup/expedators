# Kubernetes cluster: backup, simulate a failure, restore (Rocky Linux)

Assumptions (change if they differ): single control plane node `master`, one worker `worker1`, cluster built with kubeadm, containerd, Rocky Linux 8/9. etcd runs as a static pod on master. Run everything on **master** unless stated otherwise. You are root on master, so `sudo` is optional there.

Run each command on its own, and do not paste the `<` `>` characters from examples.

Plan: **create a test app, take the backup, break the cluster on purpose, restore from the backup, verify.**

---

## Step 1: Pre-checks

```bash
kubectl get nodes -o wide
kubectl get pods -A | grep -v Running | grep -v Completed
sudo kubeadm certs check-expiration
df -h /backup /var/lib/etcd
sudo dnf install -y tar curl openssh-clients
```

Expected: all nodes `Ready`, no failing pods, and enough free disk space for the backup.

---

## Step 2: Create a test app (so you can prove the restore worked)

```bash
kubectl create namespace restore-test
kubectl -n restore-test create deployment web --image=nginx --replicas=2
kubectl -n restore-test create configmap demo-cm --from-literal=message=before-backup
kubectl -n restore-test get all
```

Wait until both `web` pods are `Running`.

---

## Step 3: Backup

### 3.1 Create the backup folder

```bash
export BK=/backup/$(date +%F-%H%M)
sudo mkdir -p $BK
ls -ld $BK
```

The folder must be listed. If not, stop and fix this first.

### 3.2 Install etcdctl and etcdutl

Skip this step if `which etcdctl etcdutl` already prints both paths.

Find the etcd version your cluster runs:

```bash
kubectl -n kube-system get pod -l component=etcd -o jsonpath='{.items[0].spec.containers[0].image}'; echo
```

It prints something like `registry.k8s.io/etcd:3.5.16-0`. Set `ETCD_VER` to that version with a leading `v` and without the `-0` (here `v3.5.16`):

```bash
export ETCD_VER=v3.5.16
```

Download and install (amd64; use `arm64` in the file name on ARM):

```bash
cd /tmp
curl -LO https://github.com/etcd-io/etcd/releases/download/${ETCD_VER}/etcd-${ETCD_VER}-linux-amd64.tar.gz
tar xzf etcd-${ETCD_VER}-linux-amd64.tar.gz
sudo mv etcd-${ETCD_VER}-linux-amd64/etcdctl etcd-${ETCD_VER}-linux-amd64/etcdutl /usr/local/bin/
etcdctl version
etcdutl version
```

### 3.3 etcd snapshot

```bash
sudo ETCDCTL_API=3 etcdctl \
  --endpoints=https://127.0.0.1:2379 \
  --cacert=/etc/kubernetes/pki/etcd/ca.crt \
  --cert=/etc/kubernetes/pki/etcd/server.crt \
  --key=/etc/kubernetes/pki/etcd/server.key \
  snapshot save $BK/etcd-snapshot.db
```

Verify:

```bash
ls -lh $BK/
sudo etcdutl snapshot status $BK/etcd-snapshot.db -w table
```

You should see a file of a few MB and a table with hash, revision, total keys and size. **Do not continue if the snapshot is missing or the table does not print.**

### 3.4 Cluster files

```bash
sudo tar czf $BK/etc-kubernetes.tgz /etc/kubernetes
sudo tar czf $BK/var-lib-kubelet.tgz /var/lib/kubelet/config.yaml /var/lib/kubelet/kubeadm-flags.env
kubectl -n kube-system get cm kubeadm-config -o yaml > $BK/kubeadm-config.yaml
kubectl -n kube-system get cm kubelet-config -o yaml > $BK/kubelet-config.yaml
kubectl -n kube-system get cm kube-proxy -o yaml > $BK/kube-proxy-cm.yaml
```

### 3.5 Workload definitions (contain Secrets, protect the files)

```bash
kubectl get ns -o yaml > $BK/namespaces.yaml
for r in deploy sts ds svc cm secret ing pvc pv sa role rolebinding netpol; do
  kubectl get $r -A -o yaml > $BK/$r.yaml 2>/dev/null
done
kubectl get clusterrole,clusterrolebinding,crd,sc -o yaml > $BK/cluster-scoped.yaml
```

### 3.6 Record versions

```bash
{ kubeadm version -o short; kubelet --version; kubectl version --client; containerd --version; } > $BK/versions.txt
rpm -qa | grep -E 'kubeadm|kubelet|kubectl|containerd|cri-tools|kubernetes-cni' >> $BK/versions.txt
```

### 3.7 Copy the backup off the node

```bash
sudo chown -R $USER $BK
scp -r $BK user@backup-server:/backups/k8s/
```

In a lab with no backup server, at least keep a copy elsewhere, for example `sudo cp -r $BK /root/backup-copy`.

Persistent volume data is not in the etcd snapshot. Back it up separately if you use PVs.

---

## Step 4: Simulate the failure

Delete the test app **after** the backup, so the backup does not contain this change.

```bash
kubectl delete namespace restore-test
kubectl get ns
```

`restore-test` is gone. This stands in for an accidental delete or a corrupted cluster state. To test a harder failure, also damage etcd itself:

```bash
sudo mkdir -p /root/manifests-stash
sudo mv /etc/kubernetes/manifests/*.yaml /root/manifests-stash/
sudo mv /var/lib/etcd /var/lib/etcd.broken
```

After about a minute `kubectl get nodes` stops working because the API server is down. That is the "failed" state. Carry on to Step 5.

---

## Step 5: Restore

### 5.1 Stop the control plane

If you did not already do this in Step 4, move the static pod manifests away so kubelet stops etcd and the API server:

```bash
sudo mkdir -p /root/manifests-stash
sudo mv /etc/kubernetes/manifests/*.yaml /root/manifests-stash/
```

Wait until this shows no apiserver or etcd container (repeat it until empty):

```bash
sudo crictl ps | grep -E 'apiserver|etcd'
```

### 5.2 Restore etcd from the snapshot

Set `BK` again if this is a new shell, and point it at your backup folder:

```bash
export BK=/backup/<your-backup-folder>
ls -lh $BK/etcd-snapshot.db
```

Move the old data aside (skip the first line if you already did it in Step 4) and restore:

```bash
sudo mv /var/lib/etcd /var/lib/etcd.broken
sudo etcdutl snapshot restore $BK/etcd-snapshot.db --data-dir /var/lib/etcd
sudo restorecon -Rv /var/lib/etcd
```

`restorecon` fixes SELinux labels on the restored data directory. It is harmless if SELinux is off.

### 5.3 Start the control plane again

```bash
sudo mv /root/manifests-stash/*.yaml /etc/kubernetes/manifests/
sudo systemctl restart kubelet
```

Wait one to two minutes, then:

```bash
kubectl get nodes
kubectl get pods -n kube-system
```

### 5.4 Restore `/etc/kubernetes` (only if files were damaged or lost)

If certificates or manifests are missing or broken, restore them before Step 5.3:

```bash
sudo tar xzf $BK/etc-kubernetes.tgz -C /
sudo restorecon -Rv /etc/kubernetes
sudo systemctl restart kubelet
```

### 5.5 If the whole master node was lost

Rebuild the node first, then:

- Install the same Kubernetes package versions listed in `$BK/versions.txt`, with the repo file and `--disableexcludes=kubernetes` as in your normal install. Install and start containerd, disable swap, and use the same hostname and IP as before.
- Copy the backup folder to the new node.
- Restore `/etc/kubernetes` (Step 5.4), restore etcd (Step 5.2), put the manifests back and restart kubelet (Step 5.3).
- Do **not** run `kubeadm init`. The restore replaces it.

---

## Step 6: Verify the restore

```bash
kubectl get nodes -o wide
kubectl get ns
kubectl -n restore-test get all
kubectl -n restore-test get cm demo-cm -o jsonpath='{.data.message}'; echo
kubectl get pods -A
kubectl get --raw='/readyz?verbose' | tail -5
```

Expected:

- `restore-test` is back, with the `web` deployment and two pods running again (they may take a minute to start).
- The configmap prints `before-backup`.
- Both nodes are `Ready` and everything in `kube-system` is `Running`.

If worker pods look stale, restart kubelet on the worker:

```bash
sudo systemctl restart kubelet
```

Clean up the test app when you are done:

```bash
kubectl delete namespace restore-test
```

Anything created after the snapshot was taken is gone after a restore. Re-create it from your own manifests or from the YAML exports in `$BK`.

---

## If the backup step itself fails

| Problem | Fix |
|---|---|
| `etcdctl: command not found` | Redo Step 3.2, or run `/usr/local/bin/etcdctl`. |
| `context deadline exceeded` or connection refused | etcd is not healthy or the endpoint is wrong. Check `sudo crictl ps -a \| grep etcd` and the `--listen-client-urls` line in `/etc/kubernetes/manifests/etcd.yaml`. |
| Certificate errors | Confirm the three cert paths exist under `/etc/kubernetes/pki/etcd/`. |
| `No space left on device` | Free space or point `BK` at a bigger disk. |
| Snapshot file is 0 bytes or `status` fails | Treat the backup as failed. Delete it and run Step 3.3 again. |

## Logs

```bash
sudo systemctl status kubelet
sudo journalctl -xeu kubelet --no-pager | tail -50
sudo crictl ps -a | grep -E 'apiserver|etcd|controller|scheduler'
sudo ausearch -m avc -ts recent
```

The last command shows recent SELinux denials.
