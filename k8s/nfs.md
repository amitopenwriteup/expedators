# Lab: Dynamic NFS Storage Provisioner for Kubernetes (Rocky Linux)

## Overview

In this lab you will turn a Rocky Linux machine into an **NFS server** and use the **NFS Subdir External Provisioner** (installed with Helm) so that Kubernetes can dynamically create PersistentVolumes (PVs) whenever a PersistentVolumeClaim (PVC) is created.

**Architecture**

```
+----------------------+        NFS (2049)        +---------------------------+
|  Rocky Linux NFS     | <----------------------> |  Kubernetes nodes         |
|  /srv/nfs/kubedata   |                          |  (nfs-utils installed)    |
|  172.16.207.130      |                          |  nfs-subdir-provisioner   |
+----------------------+                          +---------------------------+
```

**Objectives**

1. Install and configure an NFS server on Rocky Linux
2. Install NFS client utilities on all Kubernetes nodes
3. Install Helm
4. Deploy the NFS subdir external provisioner
5. Create a PVC and verify dynamic provisioning
6. Mount the volume in a Pod and verify data lands on the NFS server
7. Clean up

**Estimated time:** 30-40 minutes

## Prerequisites

| Requirement | Details |
|---|---|
| OS | Rocky Linux 8 or 9 (NFS server and Kubernetes nodes) |
| Kubernetes | Working cluster with `kubectl` access |
| Privileges | `sudo`/root on the NFS server and every node |
| Network | Nodes can reach the NFS server on TCP/UDP 2049 and rpcbind (111) |

> **Note:** Replace `172.16.207.130` throughout this lab with the IP address of **your** NFS server.

---

## Part 1: Set Up the NFS Server

Run these steps on the machine that will host the storage.

### Step 1.1 - Install NFS packages

```bash
sudo dnf install -y nfs-utils
```

> On Ubuntu/Debian the equivalent is `nfs-kernel-server` and `nfs-common`. Rocky Linux uses the single `nfs-utils` package for both server and client.

### Step 1.2 - Create the export directory

This directory will hold all PVC data created by the provisioner.

```bash
sudo mkdir -p /srv/nfs/kubedata
sudo chown nobody:nobody /srv/nfs/kubedata
sudo chmod 777 /srv/nfs/kubedata
```

### Step 1.3 - Configure the export

Edit `/etc/exports`:

```bash
sudo vi /etc/exports
```

Add the following line:

```
/srv/nfs/kubedata    *(rw,sync,no_subtree_check,no_root_squash,no_all_squash,insecure)
```

| Option | Meaning |
|---|---|
| `rw` | Read/write access |
| `sync` | Write changes to disk before replying (safer) |
| `no_subtree_check` | Disables subtree checking, improves reliability |
| `no_root_squash` | Remote root keeps root privileges (needed by the provisioner) |
| `no_all_squash` | Do not map all users to anonymous |
| `insecure` | Allow client ports above 1024 |

> **Security tip:** `*` allows any host. In real environments, restrict it to your node subnet, e.g. `172.16.207.0/24(rw,...)`.

### Step 1.4 - Start and enable the service

```bash
sudo systemctl enable --now nfs-server
sudo systemctl status nfs-server
```

### Step 1.5 - Export the directory

```bash
sudo exportfs -rav
```

Expected output:

```
exporting *:/srv/nfs/kubedata
```

### Step 1.6 - Open the firewall (Rocky Linux specific)

```bash
sudo firewall-cmd --permanent --add-service=nfs
sudo firewall-cmd --permanent --add-service=mountd
sudo firewall-cmd --permanent --add-service=rpc-bind
sudo firewall-cmd --reload
sudo firewall-cmd --list-services
```

### Step 1.7 - Verify the export

```bash
showmount -e localhost
```

Expected output:

```
Export list for localhost:
/srv/nfs/kubedata *
```

---

## Part 2: Install NFS Client Packages on All Kubernetes Nodes

Run on **every** control-plane and worker node. Without this, Pods will fail to mount NFS volumes.

```bash
sudo dnf install -y nfs-utils
```

Test connectivity from a node:

```bash
showmount -e 172.16.207.130
```

You should see `/srv/nfs/kubedata` listed. If not, re-check the firewall, exports, and network.

---

## Part 3: Install Helm

Run on the machine where you use `kubectl`.

```bash
wget https://get.helm.sh/helm-v3.13.2-linux-amd64.tar.gz
tar -zxvf helm-v3.13.2-linux-amd64.tar.gz
sudo mv linux-amd64/helm /usr/local/bin/helm
helm version
```

---

## Part 4: Deploy the NFS Subdir External Provisioner

### Step 4.1 - Add the Helm repository

```bash
helm repo add nfs-subdir-external-provisioner https://kubernetes-sigs.github.io/nfs-subdir-external-provisioner/
helm repo update
```

### Step 4.2 - Install the chart

Provide the IP of your NFS server and the exported path.

```bash
helm install nfs-subdir-external-provisioner \
  nfs-subdir-external-provisioner/nfs-subdir-external-provisioner \
  --set nfs.server=172.16.207.130 \
  --set nfs.path=/srv/nfs/kubedata
```

> **Optional:** To make this StorageClass the cluster default, add `--set storageClass.defaultClass=true`.

### Step 4.3 - Verify the provisioner Pod

```bash
kubectl get pods
```

The `nfs-subdir-external-provisioner-...` Pod should be in `Running` state.

### Step 4.4 - Verify the StorageClass

```bash
kubectl get storageclass
```

Example output:

```
NAME                   PROVISIONER                                     RECLAIMPOLICY   VOLUMEBINDINGMODE      ALLOWVOLUMEEXPANSION   AGE
nfs-client (default)   cluster.local/nfs-client-provisioner            Delete          Immediate              true                   83m
standard               rancher.io/local-path                           Delete          WaitForFirstConsumer   false                  110m
```

Inspect the details:

```bash
kubectl get storageclass nfs-client -o yaml
```

---

## Part 5: Test Dynamic Provisioning with a PVC

### Step 5.1 - Create the PVC manifest

```bash
vi nfsclaim.yaml
```

```yaml
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: pvctest
spec:
  storageClassName: nfs-client
  accessModes:
    - ReadWriteMany
  resources:
    requests:
      storage: 100Mi
```

### Step 5.2 - Create the PVC

```bash
kubectl create -f nfsclaim.yaml
```

Expected output:

```
persistentvolumeclaim/pvctest created
```

### Step 5.3 - Verify PVC and PV

```bash
kubectl get pvc,pv
```

Expected output:

```
NAME                            STATUS   VOLUME                                     CAPACITY   ACCESS MODES   STORAGECLASS   AGE
persistentvolumeclaim/pvctest   Bound    pvc-2e5dab38-6ec0-4236-8c2b-54717e63108e   100Mi      RWX            nfs-client     2m57s

NAME                                                        CAPACITY   ACCESS MODES   RECLAIM POLICY   STATUS   CLAIM             STORAGECLASS   AGE
persistentvolume/pvc-2e5dab38-6ec0-4236-8c2b-54717e63108e   100Mi      RWX            Delete           Bound    default/pvctest   nfs-client     2m57s
```

The PVC is `Bound`, and a PV was created automatically; no manual PV was needed.

### Step 5.4 - Check the directory on the NFS server

```bash
ls -l /srv/nfs/kubedata
```

You will see a sub-directory named like `default-pvctest-pvc-<uid>`.

---

## Part 6: Mount the Volume in a Pod

### Step 6.1 - Create the Pod manifest

```bash
vi podnfs.yaml
```

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: busybox
spec:
  volumes:
  - name: myvol
    persistentVolumeClaim:
      claimName: pvctest
  containers:
  - image: busybox
    name: busybox
    command: ["/bin/sh"]
    args: ["-c", "sleep 600"]
    volumeMounts:
    - name: myvol
      mountPath: /data
```

### Step 6.2 - Create the Pod

```bash
kubectl create -f podnfs.yaml
kubectl get pod busybox
```

Wait until the status is `Running`.

### Step 6.3 - Write data from the Pod

```bash
kubectl exec busybox -- sh -c 'echo "Hello from Kubernetes on NFS" > /data/hello.txt'
kubectl exec busybox -- ls -l /data
```

### Step 6.4 - Verify on the NFS server

```bash
cat /srv/nfs/kubedata/default-pvctest-pvc-*/hello.txt
```

Expected output:

```
Hello from Kubernetes on NFS
```

The file written inside the Pod is stored on the NFS server.

---

## Part 7: Troubleshooting

| Symptom | Likely cause | Fix |
|---|---|---|
| PVC stuck in `Pending` | Provisioner Pod not running, or wrong `nfs.server`/`nfs.path` | `kubectl logs deploy/nfs-subdir-external-provisioner`; re-install with correct values |
| Pod stuck in `ContainerCreating` | `nfs-utils` missing on the node, or firewall blocking | Install `nfs-utils`; check `firewall-cmd --list-services`; run `showmount -e <server>` from the node |
| `mount.nfs: access denied` | Export doesn't allow the client | Review `/etc/exports`, then `sudo exportfs -rav` |
| `showmount` times out | rpcbind/mountd blocked | Open `rpc-bind` and `mountd` in firewalld |

Useful commands:

```bash
kubectl describe pvc pvctest
kubectl describe pod busybox
kubectl logs -l app=nfs-subdir-external-provisioner
sudo journalctl -u nfs-server -f
```

---

## Part 8: Cleanup

```bash
kubectl delete pod busybox
kubectl delete pvc pvctest
helm uninstall nfs-subdir-external-provisioner
```

Because the StorageClass reclaim policy is `Delete`, deleting the PVC removes the PV. By default the provisioner may also archive or remove the backing directory on the NFS server; check `/srv/nfs/kubedata`.

---

## Review Questions

1. Why must `nfs-utils` be installed on the Kubernetes worker nodes?
2. What does the `no_root_squash` export option do, and why is it a security concern?
3. What is the difference between static and dynamic PV provisioning?
4. Why does the PVC use `ReadWriteMany`, and which access modes does NFS support?
5. What happens to the data on the NFS server when the PVC is deleted with reclaim policy `Delete`?

## Summary

You configured an NFS server on Rocky Linux (including firewalld), installed NFS client tools on the cluster nodes, deployed the NFS subdir external provisioner using Helm, and confirmed that a PVC dynamically creates a PV backed by a directory on the NFS server and can be mounted in a Pod.
