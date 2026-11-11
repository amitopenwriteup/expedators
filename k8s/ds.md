# Lab: Updating DaemonSets in Kubernetes

## 1. Objective

By the end of this lab you will be able to:

- Create a DaemonSet and verify that it runs one pod on every node
- Update a DaemonSet using a **configuration file** (`kubectl apply`)
- Update a DaemonSet using an **imperative command** (`kubectl edit`)
- Update only the **container image** (`kubectl set image`)
- Monitor a rolling update and roll it back

## 2. Prerequisites

- A running Kubernetes cluster (Minikube / kind / kubeadm)
- `kubectl` configured to talk to the cluster
- Internet access to pull the example manifests and the fluentd image

Verify the cluster:

```bash
kubectl cluster-info
kubectl get nodes
```

## 3. Theory

A **DaemonSet** ensures that a copy of a pod runs on **all (or selected) nodes**. Typical uses are log collectors (fluentd), monitoring agents (node-exporter) and network plugins.

When you change the pod template of a DaemonSet, its **update strategy** decides how the pods are replaced:

| Strategy | Behaviour |
|---|---|
| `RollingUpdate` (default) | Old pods are deleted and new ones created automatically, node by node |
| `OnDelete` | New pods are created only after you delete the old pods manually |

### Ways to update a DaemonSet

| Method | Command | When to use |
|---|---|---|
| Configuration file | `kubectl apply -f <file>` | Declarative; manifests are kept in version control |
| Imperative edit | `kubectl edit ds/<name>` | Quick change to any field in an editor |
| Image only | `kubectl set image ds/<name> <container>=<image>` | Only the container image changes |

## 4. Lab Steps

### Step 1: Create the DaemonSet

Create the `fluentd-elasticsearch` DaemonSet in the `kube-system` namespace from the Kubernetes example manifest:

```bash
kubectl apply -f https://k8s.io/examples/controllers/daemonset.yaml
```

Expected output:

```
daemonset.apps/fluentd-elasticsearch created
```

### Step 2: Verify the DaemonSet and its pods

```bash
kubectl get ds -n kube-system
kubectl get pods -n kube-system -l name=fluentd-elasticsearch -o wide
```

Observe: `DESIRED`, `CURRENT` and `READY` equal the number of nodes, and there is **one pod per node**.

Check the image currently in use:

```bash
kubectl get ds fluentd-elasticsearch -n kube-system \
  -o jsonpath='{.spec.template.spec.containers[0].image}'
```

### Step 3: Check the update strategy

```bash
kubectl get ds fluentd-elasticsearch -n kube-system \
  -o jsonpath='{.spec.updateStrategy}'
```

The default is `RollingUpdate` with `maxUnavailable: 1`.

---

### Method 1: Update using a configuration file (`kubectl apply`)

If you update DaemonSets using configuration files, use `kubectl apply`:

```bash
kubectl apply -f https://k8s.io/examples/controllers/fluentd-daemonset-update.yaml
```

Watch the rollout:

```bash
kubectl rollout status ds/fluentd-elasticsearch -n kube-system
kubectl get pods -n kube-system -l name=fluentd-elasticsearch
```

Observe: the old pods are terminated and replaced by new ones. Check the manifest you applied (download it or open the URL in a browser) to see which fields were changed.

---

### Method 2: Update using an imperative command (`kubectl edit`)

If you update DaemonSets using imperative commands, use `kubectl edit`:

```bash
kubectl edit ds/fluentd-elasticsearch -n kube-system
```

This opens the live object in your editor (`vi` by default). For example, change the container image tag, then save and exit (`Esc`, then `:wq`).

Expected output:

```
daemonset.apps/fluentd-elasticsearch edited
```

Watch the rollout:

```bash
kubectl rollout status ds/fluentd-elasticsearch -n kube-system
```

---

### Method 3: Update only the container image (`kubectl set image`)

If you only need to update the container image in the DaemonSet template, i.e. `.spec.template.spec.containers[*].image`, use `kubectl set image`:

```bash
kubectl set image ds/fluentd-elasticsearch \
  fluentd-elasticsearch=quay.io/fluentd_elasticsearch/fluentd:v2.6.0 \
  -n kube-system
```

Expected output:

```
daemonset.apps/fluentd-elasticsearch image updated
```

Here `fluentd-elasticsearch` on the left of `=` is the **container name** and the value on the right is the **new image**.

Watch the rollout:

```bash
kubectl rollout status ds/fluentd-elasticsearch -n kube-system
```

---

### Step 4: Verify the update

```bash
kubectl get ds fluentd-elasticsearch -n kube-system \
  -o jsonpath='{.spec.template.spec.containers[0].image}'

kubectl get pods -n kube-system -l name=fluentd-elasticsearch -o wide
kubectl describe ds fluentd-elasticsearch -n kube-system
```

Observe: the image is `quay.io/fluentd_elasticsearch/fluentd:v2.6.0`, the pod AGE is recent, and the Events section shows the old pods being deleted and new pods created.

### Step 5: View the rollout history

```bash
kubectl rollout history ds/fluentd-elasticsearch -n kube-system
```

### Step 6: Roll back the update

```bash
kubectl rollout undo ds/fluentd-elasticsearch -n kube-system
kubectl rollout status ds/fluentd-elasticsearch -n kube-system
```

Check the image again to confirm it has returned to the previous version.

## 5. Observations

| Method | Command | What changes |
|---|---|---|
| Configuration file | `kubectl apply -f <file>` | Whatever differs between the file and the live object |
| Imperative edit | `kubectl edit ds/<name>` | Any field you edit in the editor |
| Image only | `kubectl set image ds/<name> <container>=<image>` | Only the container image |

All three methods change the pod template, so with the `RollingUpdate` strategy the pods are replaced automatically.

## 6. Cleanup

```bash
kubectl delete ds fluentd-elasticsearch -n kube-system
kubectl get pods -n kube-system
```
