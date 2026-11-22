# Upgrade Kubernetes 1.30 to 1.31 (with backup)

Cluster:

| Node | Role | Version |
|---|---|---|
| `vm-1` | control plane | v1.30.14 |
| `vm2` | worker | v1.30.14 |

Target: **v1.31.x** (newest patch). Ubuntu/Debian, containerd, repo `pkgs.k8s.io`.

Order: **backup, control plane (vm-1), then worker (vm2)**. Run each command on its own, and do not paste the `<` `>` characters from examples.

---

## Step 1: Pre-checks (on vm-1)

```bash
kubectl get nodes -o wide
kubectl get pods -A | grep -v Running | grep -v Completed
sudo kubeadm certs check-expiration
swapon --show
kubectl get pdb -A
cat /etc/apt/sources.list.d/kubernetes.list
```

Expected: both nodes `Ready`, no failing pods, `swapon --show` prints nothing, repo file contains `v1.30`.

Also check that your CNI plugin and containerd version support Kubernetes 1.31.

---

## Step 2: Backup (on vm-1)

### 2.1 Create the backup folder

```bash
export BK=/backup/$(date +%F-%H%M)
sudo mkdir -p $BK
ls -ld $BK
```

The folder must be listed. If not, stop and fix this first.

### 2.2 etcd snapshot

You already have `etcdctl` and `etcdutl` in `/usr/local/bin`.

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

You should see a file of a few MB and a table with hash, revision, total keys and size.

### 2.3 Cluster files

```bash
sudo tar czf $BK/etc-kubernetes.tgz /etc/kubernetes
sudo tar czf $BK/var-lib-kubelet.tgz /var/lib/kubelet/config.yaml /var/lib/kubelet/kubeadm-flags.env
```

### 2.4 kubeadm and kubelet configuration

```bash
kubectl -n kube-system get cm kubeadm-config -o yaml > $BK/kubeadm-config.yaml
kubectl -n kube-system get cm kubelet-config -o yaml > $BK/kubelet-config.yaml
kubectl -n kube-system get cm kube-proxy -o yaml > $BK/kube-proxy-cm.yaml
```

### 2.5 Workload definitions (contains Secrets, protect the files)

```bash
kubectl get ns -o yaml > $BK/namespaces.yaml
for r in deploy sts ds svc cm secret ing pvc pv sa role rolebinding netpol; do
  kubectl get $r -A -o yaml > $BK/$r.yaml 2>/dev/null
done
kubectl get clusterrole,clusterrolebinding,crd,sc -o yaml > $BK/cluster-scoped.yaml
```

### 2.6 Record versions

```bash
{ kubeadm version -o short; kubelet --version; kubectl version --client; containerd --version; } > $BK/versions.txt
dpkg -l | grep -E 'kubeadm|kubelet|kubectl|containerd' >> $BK/versions.txt
```

### 2.7 Copy the backup off the node

```bash
sudo chown -R $USER $BK
scp -r $BK user@backup-server:/backups/k8s/
```

In a lab with no backup server, at least keep a copy elsewhere, for example `sudo cp -r $BK /root/backup-copy`.

Persistent volume data is not in the etcd snapshot. Back it up separately if you use PVs.

---

## Step 3: Upgrade the control plane (on vm-1)

### 3.1 Set the hop and switch the repository

```bash
export OLD=1.30
export NEW=1.31
sudo sed -i "s/v$OLD/v$NEW/g" /etc/apt/sources.list.d/kubernetes.list
sudo apt-get update
```

### 3.2 Pick the newest 1.31 patch

```bash
export VER=$(apt-cache madison kubeadm | grep "$NEW\." | head -1 | awk '{print $3}')
echo $VER
```

It should print something like `1.31.x-1.1`. Write it down: vm2 must use the **same** version. If it prints nothing, check the repo file.

### 3.3 Upgrade kubeadm

```bash
sudo apt-mark unhold kubeadm
sudo apt-get install -y kubeadm=$VER
sudo apt-mark hold kubeadm
kubeadm version -o short
```

### 3.4 Check the plan

```bash
sudo kubeadm upgrade plan
```

### 3.5 Apply the upgrade

```bash
sudo kubeadm upgrade apply v${VER%-*}
```

Wait for `[upgrade/successful] SUCCESS! Your cluster was upgraded`.

### 3.6 Drain vm-1

```bash
kubectl drain vm-1 --ignore-daemonsets --delete-emptydir-data
```

### 3.7 Upgrade kubelet and kubectl

```bash
sudo apt-mark unhold kubelet kubectl
sudo apt-get install -y kubelet=$VER kubectl=$VER
sudo apt-mark hold kubelet kubectl
sudo systemctl daemon-reload
sudo systemctl restart kubelet
```

### 3.8 Uncordon and check

```bash
kubectl uncordon vm-1
kubectl get nodes
kubectl get pods -n kube-system
```

Expected: `vm-1` is `Ready` and shows `v1.31.x`. `vm2` still shows `v1.30.14`, which is allowed.

---

## Step 4: Upgrade the worker (vm2)

### 4.1 Drain vm2 (run on vm-1)

```bash
kubectl drain vm2 --ignore-daemonsets --delete-emptydir-data
```

### 4.2 On vm2: switch the repository and pick the version

```bash
export OLD=1.30
export NEW=1.31
sudo sed -i "s/v$OLD/v$NEW/g" /etc/apt/sources.list.d/kubernetes.list
sudo apt-get update
export VER=$(apt-cache madison kubeadm | grep "$NEW\." | head -1 | awk '{print $3}')
echo $VER
```

`VER` must be the same as on vm-1. If it is newer, set it by hand, for example `export VER=1.31.x-1.1`.

### 4.3 On vm2: upgrade kubeadm and node config

```bash
sudo apt-mark unhold kubeadm
sudo apt-get install -y kubeadm=$VER
sudo apt-mark hold kubeadm
sudo kubeadm upgrade node
```

### 4.4 On vm2: upgrade kubelet and kubectl

```bash
sudo apt-mark unhold kubelet kubectl
sudo apt-get install -y kubelet=$VER kubectl=$VER
sudo apt-mark hold kubelet kubectl
sudo systemctl daemon-reload
sudo systemctl restart kubelet
```

### 4.5 Uncordon (run on vm-1)

```bash
kubectl uncordon vm2
kubectl get nodes
```

---

## Step 5: Verify (on vm-1)

```bash
kubectl get nodes -o wide
kubectl get pods -A
kubectl get --raw='/readyz?verbose' | tail -5
sudo kubeadm certs check-expiration
```

- Both nodes `Ready` and on `v1.31.x`.
- Everything in `kube-system` is `Running`.
- Your applications are healthy.

Next hop (1.31 to 1.32): go back to Step 2 for a **new backup**, then repeat Steps 3 and 4 with `OLD=1.31` and `NEW=1.32`.

---

## If something goes wrong

Re-run the upgrade (safe to repeat):

```bash
sudo kubeadm upgrade apply v${VER%-*}
```

Logs:

```bash
sudo systemctl status kubelet
sudo journalctl -xeu kubelet --no-pager | tail -50
sudo crictl ps -a | grep -E 'apiserver|etcd|controller|scheduler'
```

kubeadm's own backups are in `/etc/kubernetes/tmp/`.

### Restore etcd from your snapshot (vm-1)

Kubernetes does not support downgrading. Restore from the backup taken before the hop and reinstall the old package versions (`apt-get install kubeadm=<old> kubelet=<old> kubectl=<old>`).

```bash
sudo mkdir -p /root/manifests-stash
sudo mv /etc/kubernetes/manifests/*.yaml /root/manifests-stash/
```

Wait until `sudo crictl ps` shows no apiserver or etcd container, then:

```bash
sudo mv /var/lib/etcd /var/lib/etcd.broken
sudo etcdutl snapshot restore $BK/etcd-snapshot.db --data-dir /var/lib/etcd
sudo mv /root/manifests-stash/*.yaml /etc/kubernetes/manifests/
sudo systemctl restart kubelet
kubectl get nodes
```

Restore `/etc/kubernetes` only if files were damaged:

```bash
sudo tar xzf $BK/etc-kubernetes.tgz -C /
sudo systemctl restart kubelet
```
