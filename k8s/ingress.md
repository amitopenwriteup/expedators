# Lab: NGINX Ingress Controller with Host and Path-Based Routing (Rocky Linux)

## Overview

In this lab you will install the NGINX Ingress Controller using Helm, deploy two sample applications (**apple** and **banana**), and route traffic to them through a single Ingress using the hostname `demo.localdev.me` and the paths `/apple` and `/banana`.

**Architecture**

```
curl http://demo.localdev.me/apple
            |
            v
   kubectl port-forward (80 -> 80)  or  NodePort on any node
            |
            v
   NGINX Ingress Controller
            |
     +------+--------+
     |               |
 /apple          /banana
     |               |
apple-service   banana-service
     |               |
 apple-app       banana-app
```

**Objectives**

1. Install Helm
2. Install the NGINX Ingress Controller
3. Deploy two sample apps with Services
4. Create an Ingress with path-based rules
5. Access the apps using `curl` through a port-forward
6. Clean up

**Estimated time:** 20-30 minutes

## Prerequisites

| Requirement | Details |
|---|---|
| OS | Rocky Linux 8 or 9 |
| Kubernetes | Working cluster with 1 control-plane VM and 2 worker VMs; run `kubectl` and `helm` on the control-plane VM |
| Privileges | `sudo` access on all VMs |
| Network | Internet access to pull Helm charts and container images |

---

## Part 1: Install Helm

```bash
wget https://get.helm.sh/helm-v3.10.2-linux-amd64.tar.gz
tar -zxvf helm-v3.10.2-linux-amd64.tar.gz
sudo mv linux-amd64/helm /usr/local/bin/helm
helm version
```

---

## Part 2: Install the NGINX Ingress Controller

```bash
helm install my-release oci://ghcr.io/nginx/charts/nginx-ingress --version 2.1.0
```

Verify that the controller Pod is running and find the Service name:

```bash
kubectl get pods
kubectl get svc
```

Note the controller Service name. With the release name `my-release` it is typically:

```
my-release-nginx-ingress-controller
```

You will need this name in Part 5.

---

## Part 3: Deploy the Sample Applications

### Step 3.1 - Create the manifest

```bash
vi testingress.yaml
```

```yaml
kind: Pod
apiVersion: v1
metadata:
  name: apple-app
  labels:
    app: apple
spec:
  containers:
    - name: apple-app
      image: hashicorp/http-echo
      args:
        - "-text=apple"

---

kind: Service
apiVersion: v1
metadata:
  name: apple-service
spec:
  selector:
    app: apple
  ports:
    - port: 5678 # Default port for image

---

kind: Pod
apiVersion: v1
metadata:
  name: banana-app
  labels:
    app: banana
spec:
  containers:
    - name: banana-app
      image: hashicorp/http-echo
      args:
        - "-text=banana"

---

kind: Service
apiVersion: v1
metadata:
  name: banana-service
spec:
  selector:
    app: banana
  ports:
    - port: 5678 # Default port for image
```

### Step 3.2 - Create the resources

```bash
kubectl create -f testingress.yaml
```

### Step 3.3 - Verify

```bash
kubectl get pods,svc
```

Both `apple-app` and `banana-app` should be `Running`, and both Services should exist.

---

## Part 4: Create the Ingress Resource

### Step 4.1 - Create the Ingress manifest

```bash
vi ingressrc.yaml
```

```yaml
---
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: demo-localhost
  namespace: default
spec:
  ingressClassName: nginx
  rules:
  - host: demo.localdev.me
    http:
      paths:
        - path: /apple
          backend:
            service:
              name: apple-service
              port:
                number: 5678
          pathType: Prefix
        - path: /banana
          backend:
            service:
              name: banana-service
              port:
                number: 5678
          pathType: Prefix
```

### Step 4.2 - Create and verify

```bash
kubectl create -f ingressrc.yaml
kubectl get ingress
kubectl describe ingress demo-localhost
```

The `describe` output should list both backends (`apple-service:5678` and `banana-service:5678`).

---

## Part 5: Expose the Ingress Controller

In a cluster built from plain VMs there is no cloud load balancer, so the controller Service of type `LoadBalancer` will show `EXTERNAL-IP` as `<pending>`. Pick **one** of the two options below.

| | Option A: Port-forward | Option B: NodePort |
|---|---|---|
| Best for | Quick test, reachable from any VM through the control-plane IP | Access through any node IP, closer to a real setup |
| `/etc/hosts` IP | Control-plane VM IP (`127.0.0.1` on the control-plane VM itself) | A worker node IP |
| URL | `http://demo.localdev.me/apple` | `http://demo.localdev.me:<nodeport>/apple` |

### Option A: Port-forward (run on the control-plane VM)

Find the controller Service:

```bash
kubectl get svc
```

Start the port-forward with `--address 0.0.0.0` so it listens on all interfaces of the control-plane VM, not only on localhost. Port 80 is privileged, so use `sudo` (add `-E` or pass `--kubeconfig` if `sudo` cannot find your kubeconfig).

```bash
sudo kubectl port-forward --address 0.0.0.0 svc/my-release-nginx-ingress-controller 80:80
```

Leave this terminal open and run the remaining commands in a **second terminal**.

If `firewalld` is running on the control-plane VM, open port 80 so the worker VMs can reach it:

```bash
sudo firewall-cmd --permanent --add-port=80/tcp
sudo firewall-cmd --reload
```
```bash
su -
echo "127.0.0.1  demo.localdev.me" | sudo tee -a /etc/hosts
exit
```

> **Tip:** To avoid `sudo`, forward to a high port instead: `kubectl port-forward --address 0.0.0.0 svc/my-release-nginx-ingress-controller 8080:80`, and use `http://demo.localdev.me:8080/apple` in the tests (open port 8080 in the firewall instead of 80).

### Option B: NodePort

Change the controller Service type to `NodePort`:

```bash
helm upgrade my-release oci://ghcr.io/nginx/charts/nginx-ingress \
  --version 2.1.0 \
  --set controller.service.type=NodePort
```

Get the assigned port:

```bash
kubectl get svc my-release-nginx-ingress-controller
```

Example output:

```
NAME                                  TYPE       CLUSTER-IP     PORT(S)                      AGE
my-release-nginx-ingress-controller   NodePort   10.96.12.34    80:31080/TCP,443:31443/TCP   1m
```

Here the HTTP NodePort is `31080`. Yours will differ (range 30000-32767). The port is open on **every** node, so any control-plane or worker IP works.

If `firewalld` is running on the nodes, allow the port:

```bash
sudo firewall-cmd --permanent --add-port=31080/tcp
sudo firewall-cmd --reload
```

---

## Part 6: Configure `/etc/hosts`

Add the entry on the VM where you will run `curl` (normally the control-plane VM).

### Option A (port-forward with `--address 0.0.0.0`)

The port-forward runs on the control-plane VM and listens on all its interfaces. The IP depends on where you run `curl`:

| Where you run `curl` | IP to add in `/etc/hosts` |
|---|---|
| On the **control-plane VM** (where the port-forward runs) | `127.0.0.1` |
| On a **worker VM** or any other machine | The **control-plane VM's IP** |

Find the control-plane VM's IP (run on the control-plane VM):

```bash
hostname -I
```

Example: `172.16.207.130`

On the control-plane VM:

```bash
echo "127.0.0.1  demo.localdev.me" | sudo tee -a /etc/hosts
```

On a worker VM:

```bash
echo "172.16.207.130  demo.localdev.me" | sudo tee -a /etc/hosts
```

> On the control-plane VM itself the entry is often unnecessary, because `localdev.me` publicly resolves to `127.0.0.1`. On other VMs it is **required**.

### Option B (NodePort): use a worker node IP

Find the worker IPs:

```bash
kubectl get nodes -o wide
```

Use the `INTERNAL-IP` of either worker. Example:

```bash
echo "172.16.207.131  demo.localdev.me" | sudo tee -a /etc/hosts
```

This entry is **required** for Option B, because the public DNS record for `localdev.me` points to `127.0.0.1`, not to your worker.

### Verify the entry

```bash
grep demo.localdev.me /etc/hosts
```

---

## Part 7: Test the Routing

**Option A (port-forward):**

```bash
curl -kL http://demo.localdev.me/apple
curl -kL http://demo.localdev.me/banana
```

**Option B (NodePort):** include the NodePort from Part 5 (example `31080`):

```bash
curl -kL http://demo.localdev.me:31080/apple
curl -kL http://demo.localdev.me:31080/banana
```

Expected output:

```
apple
banana
```

Each path reaches a different backend through the same hostname and the same Ingress controller.

---

## Part 8: Troubleshooting

| Symptom | Likely cause | Fix |
|---|---|---|
| `404 Not Found` from NGINX | Wrong host header or path, or Ingress not picked up | Use exactly `demo.localdev.me`; check `kubectl describe ingress demo-localhost` |
| `Could not resolve host` | Missing `/etc/hosts` entry | Add the entry from Part 6 |
| `Connection refused` | Port-forward not running, or started without `--address 0.0.0.0` | Check the port-forward terminal; restart with `--address 0.0.0.0` |
| Works on the control-plane VM but times out from a worker VM | Firewall blocking port 80 | `sudo firewall-cmd --permanent --add-port=80/tcp && sudo firewall-cmd --reload` |
| `bind: permission denied` on port 80 | Privileged port without root | Use `sudo`, or forward to port 8080 |
| `503 Service Temporarily Unavailable` | Backend Pod not ready | `kubectl get pods`; check labels match the Service selectors |
| Ingress has no ADDRESS or `EXTERNAL-IP` is `<pending>` | No cloud load balancer on plain VMs | Normal; use Option A or Option B |
| NodePort times out | Firewall on the node, or wrong port | Open the NodePort in `firewalld`; confirm with `kubectl get svc` |
| NodePort resolves to `127.0.0.1` | Missing `/etc/hosts` entry | Add the worker IP entry from Part 6 |

Useful commands:

```bash
kubectl get ingressclass
kubectl logs -l app.kubernetes.io/name=nginx-ingress
kubectl get endpoints apple-service banana-service
```

---

## Part 9: Cleanup

If you used Option A, stop the port-forward with `Ctrl+C`. Then:

```bash
kubectl delete -f ingressrc.yaml
kubectl delete -f testingress.yaml
helm uninstall my-release
```

Remove the `demo.localdev.me` line from `/etc/hosts` if you added it:

```bash
sudo sed -i '/demo.localdev.me/d' /etc/hosts
```

---

## Review Questions

1. What is the difference between an Ingress resource and an Ingress controller?
2. Why does the Ingress use `ingressClassName: nginx`?
3. What does `pathType: Prefix` mean, and how does it differ from `Exact`?
4. What does `--address 0.0.0.0` change in `kubectl port-forward`, and why is it needed for other VMs to connect?
5. Why is an `/etc/hosts` entry needed for hostname-based routing when curling an IP address directly?

## Summary

You installed Helm and the NGINX Ingress Controller, deployed two sample apps with Services, created an Ingress that routes `/apple` and `/banana` for `demo.localdev.me`, exposed the controller with `kubectl port-forward` or a NodePort, and verified routing with `curl`.
