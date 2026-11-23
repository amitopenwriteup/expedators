# Upgrade Kubernetes 1.32 to 1.33 on Rocky Linux (with backup)

Assumptions (change if they differ): control plane node is `master`, the worker is `worker1` (use the real name from `kubectl get nodes`). Target is **v1.33.x** (newest patch). **Rocky Linux 8/9**, containerd, repo `pkgs.k8s.io` (RPM), package manager `dnf`. You are logged in as root on master, so `sudo` is optional there.

Order: **backup, control plane (master), then worker (worker1)**. Run each command on its own, and do not paste the `<` `>` characters from examples.

Kubernetes only allows upgrading one minor version at a time, so 1.32 to 1.33 is correct. Do not skip to 1.34.

---

## Step 1: Pre-checks (on master)

Check the cluster is healthy before you change anything.

```bash
kubectl get nodes -o wide
kubectl get pods -A | grep -v Running | grep -v Completed
sudo kubeadm certs check-expiration
swapon --show
getenforce
sudo firewall-cmd --state
kubectl get pdb -A
cat /etc/yum.repos.d/kubernetes.repo
kubelet --version
kubectl version
cat /etc/os-release | head -3
```

Expected: all nodes `Ready`, no failing pods, `swapon --show` prints nothing, repo file contains `v1.32` and an `exclude=` line, and kubelet shows `v1.32.x`.

`getenforce` should print `Permissive` (the usual kubeadm setup) or `Disabled`. If your cluster runs with SELinux `Enforcing` and works today, leave it as is. If `firewalld` is running, the Kubernetes ports (6443, 2379-2380, 10250, 10257, 10259 on master; 10250 and 30000-32767 on workers) must already be open. The upgrade does not change them.

Also check that your CNI plugin and containerd version support Kubernetes 1.33, and skim the 1.33 release notes for deprecations that affect your workloads.

Make sure the tools used below exist (minimal Rocky installs may lack them):

```bash
sudo dnf install -y tar curl openssh-clients
```

---

## Step 2: Backup (on master)

### 2.1 Create the backup folder

Make a folder to hold all backup files.

```bash
export BK=/backup/$(date +%F-%H%M)
sudo mkdir -p $BK
ls -ld $BK
```

The folder must be listed. If not, stop and fix this first.

### 2.2 Install etcdctl and etcdutl

The snapshot needs the `etcdctl` and `etcdutl` tools, which kubeadm does not install and Rocky does not package. Skip this step if `which etcdctl etcdutl` already prints both paths.

First find the etcd version your cluster runs:

```bash
kubectl -n kube-system get pod -l component=etcd -o jsonpath='{.items[0].spec.containers[0].image}'; echo
```

It prints something like `registry.k8s.io/etcd:3.5.16-0`. Set `ETCD_VER` to that version with a leading `v` and without the `-0` (here `v3.5.16`):

```bash
export ETCD_VER=v3.5.16
```

Download the release and copy the two tools to `/usr/local/bin` (this is for amd64; use `arm64` in the file name on ARM):

```bash
cd /tmp
curl -LO https://github.com/etcd-io/etcd/releases/download/${ETCD_VER}/etcd-${ETCD_VER}-linux-amd64.tar.gz
tar xzf etcd-${ETCD_VER}-linux-amd64.tar.gz
sudo mv etcd-${ETCD_VER}-linux-amd64/etcdctl etcd-${ETCD_VER}-linux-amd64/etcdutl /bin/
```

Verify:

```bash
etcdctl version
etcdutl version
```

Both should print the version you set. If you get `command not found` under sudo, run them as `sudo /usr/local/bin/etcdctl ...`.

### 2.3 etcd snapshot

Save a snapshot of etcd, the database that stores the whole cluster state.

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

### 2.4 Cluster files

Copy the config and certificate files so you can repair the node if needed.

```bash
sudo tar czf $BK/etc-kubernetes.tgz /etc/kubernetes
sudo tar czf $BK/var-lib-kubelet.tgz /var/lib/kubelet/config.yaml /var/lib/kubelet/kubeadm-flags.env
sudo cp /etc/yum.repos.d/kubernetes.repo $BK/kubernetes.repo.bak
```

### 2.5 kubeadm and kubelet configuration

Save the settings kubeadm and kubelet keep inside the cluster.

```bash
kubectl -n kube-system get cm kubeadm-config -o yaml > $BK/kubeadm-config.yaml
kubectl -n kube-system get cm kubelet-config -o yaml > $BK/kubelet-config.yaml
kubectl -n kube-system get cm kube-proxy -o yaml > $BK/kube-proxy-cm.yaml
```

### 2.6 Workload definitions (contains Secrets, protect the files)

Export your apps and their settings as YAML files.

```bash
kubectl get ns -o yaml > $BK/namespaces.yaml
for r in deploy sts ds svc cm secret ing pvc pv sa role rolebinding netpol; do
  kubectl get $r -A -o yaml > $BK/$r.yaml 2>/dev/null
done
kubectl get clusterrole,clusterrolebinding,crd,sc -o yaml > $BK/cluster-scoped.yaml
```

### 2.7 Record versions

Note the installed versions for reference.

```bash
{ kubeadm version -o short; kubelet --version; kubectl version --client; containerd --version; } > $BK/versions.txt
rpm -qa | grep -E 'kubeadm|kubelet|kubectl|containerd|cri-tools|kubernetes-cni' >> $BK/versions.txt
```

### 2.8 Copy the backup off the node

Keep a copy off this node so a node failure cannot lose your backup.

```bash
sudo chown -R $USER $BK
scp -r $BK user@backup-server:/backups/k8s/
```

In a lab with no backup server, at least keep a copy elsewhere, for example `sudo cp -r $BK /root/backup-copy`.

Persistent volume data is not in the etcd snapshot. Back it up separately if you use PVs.

---

## Step 3: Upgrade the control plane (on master)

### 3.1 Set the hop and switch the repository

Point dnf at the 1.33 repository. The `exclude=` line in the repo file stays in place: it keeps the Kubernetes packages from being upgraded by an ordinary `dnf update`.

```bash
export OLD=1.32
export NEW=1.33
sudo sed -i "s/v$OLD/v$NEW/g" /etc/yum.repos.d/kubernetes.repo
sudo dnf clean all
sudo dnf makecache
grep -E 'baseurl|gpgkey|exclude' /etc/yum.repos.d/kubernetes.repo
```

The `baseurl` and `gpgkey` lines must now say `v1.33`, and `exclude=kubelet kubeadm kubectl cri-tools kubernetes-cni` must still be there. If the file has no `exclude=` line, add it, otherwise a later `dnf update` could upgrade the cluster by accident.

If your repo file does not exist at all, create it:

```bash
cat <<'REPO' | sudo tee /etc/yum.repos.d/kubernetes.repo
[kubernetes]
name=Kubernetes
baseurl=https://pkgs.k8s.io/core:/stable:/v1.33/rpm/
enabled=1
gpgcheck=1
gpgkey=https://pkgs.k8s.io/core:/stable:/v1.33/rpm/repodata/repomd.xml.key
exclude=kubelet kubeadm kubectl cri-tools kubernetes-cni
REPO
```

### 3.2 Pick the newest 1.33 patch

Find the newest 1.33 patch version and store it in `VER`.

```bash
export VER=$(dnf list --showduplicates kubeadm --disableexcludes=kubernetes | grep "$NEW\." | tail -1 | awk '{print $2}')
echo $VER
```

It should print something like `1.33.x-150500.1.1`. Write it down: the worker must use the **same** version. If it prints nothing, check the repo file and run `dnf makecache` again.

### 3.3 Upgrade kubeadm

Install the new kubeadm first, since it performs the upgrade.

```bash
sudo dnf install -y kubeadm-$VER --disableexcludes=kubernetes
kubeadm version -o short
```

### 3.4 Check the plan

Preview what the upgrade will change.

```bash
sudo kubeadm upgrade plan
```

### 3.5 Apply the upgrade

Upgrade the control plane components (API server, scheduler, controller manager, etcd).

```bash
sudo kubeadm upgrade apply v${VER%-*}
```

Wait for `[upgrade/successful] SUCCESS! Your cluster was upgraded`.

### 3.6 Drain master

Move workloads off master so kubelet can be upgraded safely.

```bash
kubectl drain master --ignore-daemonsets --delete-emptydir-data
```

### 3.7 Upgrade kubelet and kubectl

Install the new kubelet and kubectl, then restart kubelet.

```bash
sudo dnf install -y kubelet-$VER kubectl-$VER --disableexcludes=kubernetes
sudo systemctl daemon-reload
sudo systemctl restart kubelet
```

### 3.8 Uncordon and check

Allow pods back on master and confirm it shows the new version.

```bash
kubectl uncordon master
kubectl get nodes
kubectl get pods -n kube-system
```

Expected: `master` is `Ready` and shows `v1.33.x`. The worker still shows `v1.32.x`, which is allowed.

---

## Step 4: Upgrade the worker (worker1)

### 4.1 Drain the worker (run on master)

Move workloads off the worker.

```bash
kubectl drain worker1 --ignore-daemonsets --delete-emptydir-data
```

### 4.2 On the worker: switch the repository and pick the version

Point the worker at the 1.33 repository and use the same `VER` as master.

```bash
export OLD=1.32
export NEW=1.33
sudo sed -i "s/v$OLD/v$NEW/g" /etc/yum.repos.d/kubernetes.repo
sudo dnf clean all
sudo dnf makecache
grep -E 'baseurl|gpgkey|exclude' /etc/yum.repos.d/kubernetes.repo
export VER=$(dnf list --showduplicates kubeadm --disableexcludes=kubernetes | grep "$NEW\." | tail -1 | awk '{print $2}')
echo $VER
```

`VER` must be the same as on master. If it is newer, set it by hand, for example `export VER=1.33.x-150500.1.1`.

### 4.3 On the worker: upgrade kubeadm and node config

Install the new kubeadm and update the worker's node configuration.

```bash
sudo dnf install -y kubeadm-$VER --disableexcludes=kubernetes
sudo kubeadm upgrade node
```

### 4.4 On the worker: upgrade kubelet and kubectl

Install the new kubelet and kubectl, then restart kubelet.

```bash
sudo dnf install -y kubelet-$VER kubectl-$VER --disableexcludes=kubernetes
sudo systemctl daemon-reload
sudo systemctl restart kubelet
```

### 4.5 Uncordon (run on master)

Let pods run on the worker again.

```bash
kubectl uncordon worker1
kubectl get nodes
```

---

## Step 5: Verify (on master)

Confirm the whole cluster is healthy on 1.33.

```bash
kubectl get nodes -o wide
kubectl get pods -A
kubectl get --raw='/readyz?verbose' | tail -5
sudo kubeadm certs check-expiration
rpm -qa | grep -E 'kubeadm|kubelet|kubectl'
```

- Both nodes `Ready` and on `v1.33.x`.
- Everything in `kube-system` is `Running`.
- Your applications are healthy.

Next hop (1.33 to 1.34): go back to Step 2 for a **new backup**, then repeat Steps 3 and 4 with `OLD=1.33` and `NEW=1.34`.

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
sudo ausearch -m avc -ts recent
```

The last command shows recent SELinux denials, in case SELinux is `Enforcing` and blocks something. kubeadm's own backups are in `/etc/kubernetes/tmp/`.

### Restore etcd from your snapshot (master)

Kubernetes does not support downgrading. Restore from the backup taken before the hop and reinstall the old package versions. Restore the old repo file first (`sudo cp $BK/kubernetes.repo.bak /etc/yum.repos.d/kubernetes.repo`, then `sudo dnf clean all`), then:

```bash
sudo dnf downgrade -y kubeadm-<old> kubelet-<old> kubectl-<old> --disableexcludes=kubernetes
```

Replace `<old>` with the version recorded in `$BK/versions.txt` (without the `<` `>` characters). Then restore etcd:

```bash
sudo mkdir -p /root/manifests-stash
sudo mv /etc/kubernetes/manifests/*.yaml /root/manifests-stash/
```

Wait until `sudo crictl ps` shows no apiserver or etcd container, then:

```bash
sudo mv /var/lib/etcd /var/lib/etcd.broken
sudo etcdutl snapshot restore $BK/etcd-snapshot.db --data-dir /var/lib/etcd
sudo restorecon -Rv /var/lib/etcd
sudo mv /root/manifests-stash/*.yaml /etc/kubernetes/manifests/
sudo systemctl restart kubelet
kubectl get nodes
```

(`restorecon` fixes SELinux labels on the restored data directory; it is harmless if SELinux is off.)

Restore `/etc/kubernetes` only if files were damaged:

```bash
sudo tar xzf $BK/etc-kubernetes.tgz -C /
sudo restorecon -Rv /etc/kubernetes
sudo systemctl restart kubelet
```
