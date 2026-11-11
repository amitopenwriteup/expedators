# Lab 6: StatefulSets in Kubernetes

## 1. Objective

By the end of this lab you will be able to:

- Explain how a **StatefulSet** differs from a Deployment
- Create a **headless Service** and a **StatefulSet** that uses it
- Verify the **stable network identity** (DNS name) of StatefulSet pods
- Observe how a pod is recreated with the **same name** after deletion

## 2. Prerequisites

- A running Kubernetes cluster (Minikube / kind / kubeadm)
- `kubectl` configured to talk to the cluster
- Basic knowledge of Pods, Services and Deployments

Verify the cluster:

```bash
kubectl cluster-info
kubectl get nodes
```

## 3. Theory

A **StatefulSet** manages stateful applications (databases, message queues, etc.). Unlike a Deployment, it gives every pod:

| Feature | Deployment | StatefulSet |
|---|---|---|
| Pod names | Random (`app-7d9f8-xk2l`) | Ordered and stable (`web-0`, `web-1`, ...) |
| Startup / termination | All at once | Ordered (0 → N-1, reverse on scale down) |
| Network identity | Changes on restart | Stable DNS name per pod |
| Storage | Shared / ephemeral | Own PersistentVolumeClaim per pod (optional) |

### Key fields

- **`clusterIP: None`** – The IP address assigned to the Service. Setting it to `None` makes it a **headless Service** (no virtual IP, no load balancing; DNS returns the individual pod IPs).
- **`serviceName`** – The name of the headless Service that governs the StatefulSet. It provides the DNS domain for each pod:

  ```
  <pod-name>.<service-name>.<namespace>.svc.cluster.local
  e.g. web-0.nginx.default.svc.cluster.local
  ```

This lab creates a headless Service named `nginx` and a StatefulSet named `web` running Nginx pods. The headless Service is used as the `serviceName` of the StatefulSet, giving DNS-based discovery of each pod.

## 4. Lab Steps

### Step 1: Create the manifest `sf.yaml`

```yaml
apiVersion: v1
kind: Service
metadata:
  name: nginx
  labels:
    app: nginx
spec:
  ports:
  - port: 80
    name: web
  clusterIP: None          # headless service
  selector:
    app: nginx
---
apiVersion: apps/v1
kind: StatefulSet
metadata:
  name: web
spec:
  serviceName: "nginx"     # must match the headless service name
  replicas: 2
  selector:
    matchLabels:
      app: nginx
  template:
    metadata:
      labels:
        app: nginx
    spec:
      containers:
      - name: nginx
        image: nginx:1.25
        ports:
        - containerPort: 80
          name: web
```

### Step 2: Create the resources

```bash
kubectl create -f sf.yaml
```

Expected output:

```
service/nginx created
statefulset.apps/web created
```

### Step 3: Describe the StatefulSet

```bash
kubectl describe statefulset web
```

Observe: replicas, selector, pod template, and the events showing `web-0` and `web-1` being created.

### Step 4: List the pods

```bash
kubectl get pods
```

Expected output (note the ordered names, created one after the other):

```
NAME    READY   STATUS    RESTARTS   AGE
web-0   1/1     Running   0          30s
web-1   1/1     Running   0          20s
```

Also check the headless service:

```bash
kubectl get svc nginx
```

`CLUSTER-IP` should show `None`.

### Step 5: Check DNS

Get the pod IPs:

```bash
kubectl get pods -o wide
```

Start a temporary DNS-utility pod:

```bash
kubectl run -it --rm --restart=Never dnsutils2 --image=tutum/dnsutils --command -- bash
```

Inside the pod, do a lookup using the IP of a pod (replace with your own pod IP):

```bash
nslookup 10.244.0.5
```

Expected output (example):

```
5.0.244.10.in-addr.arpa   name = web-0.nginx.default.svc.cluster.local.
```

You can also resolve the service and the pod name directly:

```bash
nslookup nginx
nslookup web-0.nginx
```

Leave the pod:

```bash
exit
```

> The `--rm` flag deletes the `dnsutils2` pod automatically on exit.

### Step 6: Delete one of the pods

```bash
kubectl delete pod web-0
```

### Step 7: Check again

```bash
kubectl get pods -o wide
```

Observe:

- A new pod with the **same name `web-0`** is created automatically.
- The **IP address may change**, but the **DNS name** `web-0.nginx.default.svc.cluster.local` stays the same.
- `web-1` is not affected.

Re-run the `nslookup` from a new dnsutils pod to confirm the DNS name now points to the new IP.

## 5. Optional: Scale the StatefulSet

```bash
kubectl scale statefulset web --replicas=4
kubectl get pods -w          # pods are created in order: web-2, then web-3

kubectl scale statefulset web --replicas=2
kubectl get pods -w          # pods are removed in reverse order: web-3, then web-2
```

## 6. Observations

| Check | Result |
|---|---|
| Pod names | `web-0`, `web-1` (ordered, predictable) |
| Service ClusterIP | `None` (headless) |
| DNS record format | `<pod>.<service>.<namespace>.svc.cluster.local` |
| After deleting `web-0` | Recreated with same name, possibly new IP |

## 7. Cleanup

```bash
kubectl delete -f sf.yaml
kubectl get pods
```

If you later add `volumeClaimTemplates`, PVCs are **not** deleted with the StatefulSet. Remove them manually:

```bash
kubectl get pvc
kubectl delete pvc <pvc-name>
```
