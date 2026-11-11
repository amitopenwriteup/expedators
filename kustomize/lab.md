# Lab: Kustomize - Bases, Overlays and Environment-Specific Configuration (Rocky Linux)

## Overview

**Kustomize** lets you customize Kubernetes YAML without templates. You keep one common set of manifests (a **base**) and describe per-environment differences in **overlays**. Kustomize merges them into final manifests that you apply to the cluster.

In this lab you will build one web application and deploy it to two environments from the same base:

| Setting | `dev` | `prod` |
|---|---|---|
| Namespace | `dev` | `prod` |
| Name prefix | `dev-` | `prod-` |
| Replicas | 1 | 3 |
| Image tag | `nginx:1.25` (base default) | `nginx:1.27` |
| Web page content | "Hello from DEV" | "Hello from PROD" |
| Resource limits | none | CPU and memory limits |
| Service | NodePort `30081` | NodePort `30082` |

**Directory layout you will create**

```
kustomize-lab/
├── base/
│   ├── kustomization.yaml
│   ├── deployment.yaml
│   ├── service.yaml
│   └── index.html
└── overlays/
    ├── dev/
    │   ├── kustomization.yaml
    │   ├── namespace.yaml
    │   └── index.html
    └── prod/
        ├── kustomization.yaml
        ├── namespace.yaml
        ├── resources-patch.yaml
        └── index.html
```

**Objectives**

1. Install Kustomize (and use the built-in `kubectl kustomize`)
2. Create a base with a Deployment, Service and generated ConfigMap
3. Create `dev` and `prod` overlays
4. Use `namespace`, `namePrefix`, `labels`, `replicas`, `images`, `configMapGenerator` and `patches`
5. Preview, compare and apply each environment
6. Verify and clean up

**Estimated time:** 40-50 minutes

## Prerequisites

| Requirement | Details |
|---|---|
| Kubernetes | Working cluster (1 control-plane VM and 2 worker VMs), `kubectl` configured |
| OS | Rocky Linux 8 or 9 (run commands on the control-plane VM) |
| Network | Internet access to download Kustomize and pull the `nginx` image |
| Firewall | If `firewalld` is running on the nodes, NodePorts must be reachable |

---

## Part 1: Install Kustomize

`kubectl` already includes Kustomize (`kubectl kustomize` and `kubectl apply -k`). The bundled version can be older than the standalone binary, so installing the standalone tool is recommended.

### Step 1.1 - Check the built-in version

```bash
kubectl version --client
kubectl kustomize --help | head -5
```

### Step 1.2 - Install the standalone binary

```bash
sudo dnf install -y curl tar
curl -s "https://raw.githubusercontent.com/kubernetes-sigs/kustomize/master/hack/install_kustomize.sh" | bash
sudo mv kustomize /usr/local/bin/kustomize
kustomize version
```

> This lab works with both `kustomize build` and `kubectl kustomize`. Examples use `kustomize build`.

---

## Part 2: Understand Bases and Overlays

| Concept | Meaning |
|---|---|
| **Base** | A directory with a `kustomization.yaml` and the common manifests. It works on its own. |
| **Overlay** | A directory with its own `kustomization.yaml` that points to a base (under `resources:`) and changes it. |
| **Patch** | A small piece of YAML (or JSON operations) that modifies fields of a resource. |
| **Transformer** | A built-in field, such as `namePrefix`, `namespace`, `replicas` or `images`, that changes many resources at once. |
| **Generator** | A field, such as `configMapGenerator`, that creates resources (with a content hash in the name). |

The base is never edited per environment. Each environment only stores what is **different**.

---

## Part 3: Create the Base

### Step 3.1 - Create the directories

```bash
mkdir -p ~/kustomize-lab/base ~/kustomize-lab/overlays/dev ~/kustomize-lab/overlays/prod
cd ~/kustomize-lab
```

### Step 3.2 - Deployment

```bash
vi base/deployment.yaml
```

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: webapp
  labels:
    app: webapp
spec:
  replicas: 2
  selector:
    matchLabels:
      app: webapp
  template:
    metadata:
      labels:
        app: webapp
    spec:
      containers:
        - name: nginx
          image: nginx:1.25
          ports:
            - containerPort: 80
          volumeMounts:
            - name: web-content
              mountPath: /usr/share/nginx/html
      volumes:
        - name: web-content
          configMap:
            name: web-content
```

The Deployment mounts a ConfigMap named `web-content` as the web root. Kustomize creates that ConfigMap in Step 3.5.

### Step 3.3 - Service

```bash
vi base/service.yaml
```

```yaml
apiVersion: v1
kind: Service
metadata:
  name: webapp
  labels:
    app: webapp
spec:
  type: ClusterIP
  selector:
    app: webapp
  ports:
    - port: 80
      targetPort: 80
```

### Step 3.4 - Default web page

```bash
vi base/index.html
```

```html
<h1>Hello from BASE</h1>
```

### Step 3.5 - Base kustomization

```bash
vi base/kustomization.yaml
```

```yaml
apiVersion: kustomize.config.k8s.io/v1beta1
kind: Kustomization

resources:
  - deployment.yaml
  - service.yaml

configMapGenerator:
  - name: web-content
    files:
      - index.html
```

### Step 3.6 - Preview the base

```bash
kustomize build base
```

Notice:

- The ConfigMap name is `web-content-<hash>` (for example `web-content-7d2h8k9f5m`).
- The Deployment's volume reference was **automatically updated** to the hashed name.

The hash changes whenever the content changes, which triggers a rolling update of the Pods.

---

## Part 4: Create the `dev` Overlay

### Step 4.1 - Namespace

```bash
vi overlays/dev/namespace.yaml
```

```yaml
apiVersion: v1
kind: Namespace
metadata:
  name: dev
```

> The `namespace:` field in `kustomization.yaml` sets the namespace on resources but does **not** create the Namespace object. That is why you add it as a resource.

### Step 4.2 - Environment-specific web page

```bash
vi overlays/dev/index.html
```

```html
<h1>Hello from DEV</h1>
```

### Step 4.3 - Dev kustomization

```bash
vi overlays/dev/kustomization.yaml
```

```yaml
apiVersion: kustomize.config.k8s.io/v1beta1
kind: Kustomization

namespace: dev
namePrefix: dev-

resources:
  - ../../base
  - namespace.yaml

labels:
  - pairs:
      environment: dev
    includeTemplates: true

replicas:
  - name: webapp
    count: 1

configMapGenerator:
  - name: web-content
    behavior: replace
    files:
      - index.html

patches:
  - target:
      kind: Service
      name: webapp
    patch: |-
      - op: replace
        path: /spec/type
        value: NodePort
      - op: add
        path: /spec/ports/0/nodePort
        value: 30081
```

What each field does:

| Field | Effect |
|---|---|
| `namespace: dev` | Puts all namespaced resources in the `dev` namespace |
| `namePrefix: dev-` | Names become `dev-webapp`, `dev-web-content-<hash>` |
| `resources` | Includes the base and the Namespace manifest |
| `labels` | Adds `environment: dev` to all resources and Pod templates (not to selectors) |
| `replicas` | Sets the Deployment named `webapp` to 1 replica |
| `configMapGenerator` with `behavior: replace` | Replaces the base ConfigMap content with the dev page |
| `patches` (JSON 6902) | Changes the Service to NodePort 30081 |

### Step 4.4 - Preview the dev output

```bash
kustomize build overlays/dev
```

Check for:

- `namespace: dev` on the Deployment, Service and ConfigMap
- Names starting with `dev-`
- `replicas: 1`
- Service `type: NodePort` with `nodePort: 30081`
- The ConfigMap containing `Hello from DEV`

---

## Part 5: Create the `prod` Overlay

### Step 5.1 - Namespace

```bash
vi overlays/prod/namespace.yaml
```

```yaml
apiVersion: v1
kind: Namespace
metadata:
  name: prod
```

### Step 5.2 - Environment-specific web page

```bash
vi overlays/prod/index.html
```

```html
<h1>Hello from PROD</h1>
```

### Step 5.3 - Strategic merge patch for resource limits

```bash
vi overlays/prod/resources-patch.yaml
```

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: webapp
spec:
  template:
    spec:
      containers:
        - name: nginx
          resources:
            requests:
              cpu: 100m
              memory: 64Mi
            limits:
              cpu: 250m
              memory: 128Mi
```

A strategic merge patch only lists the fields you want to add or change. Kustomize finds the Deployment by `kind` and `name` and merges the container (matched by its `name: nginx`).

### Step 5.4 - Prod kustomization

```bash
vi overlays/prod/kustomization.yaml
```

```yaml
apiVersion: kustomize.config.k8s.io/v1beta1
kind: Kustomization

namespace: prod
namePrefix: prod-

resources:
  - ../../base
  - namespace.yaml

labels:
  - pairs:
      environment: prod
    includeTemplates: true

replicas:
  - name: webapp
    count: 3

images:
  - name: nginx
    newTag: "1.27"

configMapGenerator:
  - name: web-content
    behavior: replace
    files:
      - index.html

patches:
  - path: resources-patch.yaml
  - target:
      kind: Service
      name: webapp
    patch: |-
      - op: replace
        path: /spec/type
        value: NodePort
      - op: add
        path: /spec/ports/0/nodePort
        value: 30082
```

New fields compared with `dev`:

| Field | Effect |
|---|---|
| `images` | Changes the `nginx` image tag to `1.27` without editing the Deployment |
| `patches` with `path` | Applies the strategic merge patch file (resource limits) |

### Step 5.5 - Preview the prod output

```bash
kustomize build overlays/prod
```

Check for `replicas: 3`, `image: nginx:1.27`, the `resources:` block, and `Hello from PROD`.

---

## Part 6: Compare the Environments

Show exactly how `dev` and `prod` differ:

```bash
diff <(kustomize build overlays/dev) <(kustomize build overlays/prod)
```

Compare the base with an overlay:

```bash
diff <(kustomize build base) <(kustomize build overlays/dev)
```

---

## Part 7: Deploy to the Cluster

### Step 7.1 - Apply the dev environment

```bash
kubectl apply -k overlays/dev
```

Expected output (hash will differ):

```
namespace/dev created
configmap/dev-web-content-xxxxxxxxxx created
service/dev-webapp created
deployment.apps/dev-webapp created
```

### Step 7.2 - Apply the prod environment

```bash
kubectl apply -k overlays/prod
```

### Step 7.3 - Verify

```bash
kubectl get all,cm -n dev
kubectl get all,cm -n prod
kubectl get deploy -A -l environment
```

Expected:

- `dev`: 1 Pod, Deployment `dev-webapp`, Service `dev-webapp` (NodePort)
- `prod`: 3 Pods, Deployment `prod-webapp`, Service `prod-webapp` (NodePort)

Check the image tags and resources:

```bash
kubectl get deploy dev-webapp -n dev -o jsonpath='{.spec.template.spec.containers[0].image}{"\n"}'
kubectl get deploy prod-webapp -n prod -o jsonpath='{.spec.template.spec.containers[0].image}{"\n"}'
kubectl get deploy prod-webapp -n prod -o jsonpath='{.spec.template.spec.containers[0].resources}{"\n"}'
```

Expected: `nginx:1.25`, `nginx:1.27`, and the CPU/memory values.

### Step 7.4 - Test the applications

Get a node IP:

```bash
kubectl get nodes -o wide
```

Then test (replace `<NODE-IP>` with any node's `INTERNAL-IP`):

```bash
curl http://<NODE-IP>:30081
curl http://<NODE-IP>:30082
```

Expected output:

```
<h1>Hello from DEV</h1>
<h1>Hello from PROD</h1>
```

> If `firewalld` is running on the nodes, open the NodePorts: `sudo firewall-cmd --permanent --add-port=30081-30082/tcp && sudo firewall-cmd --reload`

---

## Part 8: Make a Change and See the Rolling Update

### Step 8.1 - Preview the difference before applying

Edit the dev page:

```bash
echo "<h1>Hello from DEV - version 2</h1>" > overlays/dev/index.html
kubectl diff -k overlays/dev
```

`kubectl diff` shows the new ConfigMap (with a new hash) and the Deployment pointing to it.

### Step 8.2 - Apply and watch the rollout

```bash
kubectl apply -k overlays/dev
kubectl rollout status deployment/dev-webapp -n dev
curl http://<NODE-IP>:30081
```

The ConfigMap name changed with its content, so Kubernetes rolled out new Pods automatically.

### Step 8.3 - Change an image tag without touching the base

Edit `overlays/prod/kustomization.yaml` and change `newTag: "1.27"` to `newTag: "1.27.1"` (or another valid tag), then:

```bash
kubectl apply -k overlays/prod
kubectl rollout status deployment/prod-webapp -n prod
```

> Alternatively, use the CLI helper: `cd overlays/prod && kustomize edit set image nginx=nginx:1.27`

---

## Part 9: Troubleshooting

| Symptom | Likely cause | Fix |
|---|---|---|
| `kustomization.yaml not found` | Wrong directory | Run the command from `~/kustomize-lab` and pass the overlay path |
| `no matches for Id Deployment.v1.apps/webapp` in a patch | Patch name does not match the base resource name | Use the **base** name (`webapp`), not the prefixed name |
| `namespaces "dev" not found` | Namespace manifest missing from `resources` | Add `namespace.yaml` to the overlay |
| ConfigMap not replaced | Missing `behavior: replace` | Add `behavior: replace` (or `merge`) in the overlay generator |
| `Service ... nodePort ... already allocated` | Same NodePort used twice | Use a different port for each environment |
| Pods `ImagePullBackOff` | Wrong image tag or no internet | Check `kubectl describe pod`; use a valid tag |
| NodePort not reachable | Firewall or wrong node IP | Open the ports in `firewalld`; check `kubectl get svc -n dev` |
| Deprecation warning about `commonLabels` | Old field | Use `labels:` as in this lab |

Useful commands:

```bash
kustomize build overlays/dev | less
kubectl kustomize overlays/prod
kubectl describe deploy dev-webapp -n dev
kubectl get events -n dev --sort-by=.lastTimestamp
```

---

## Part 10: Cleanup

```bash
kubectl delete -k overlays/dev
kubectl delete -k overlays/prod
```

This removes the resources **and** the `dev` and `prod` namespaces (because the Namespace manifests are part of each overlay).

---

## Challenge (Optional)

Create a third overlay named `staging`:

- Namespace `staging`, prefix `stg-`
- 2 replicas
- Image `nginx:1.26`
- NodePort `30083`
- Page text "Hello from STAGING"

Hint: copy `overlays/dev`, rename the values, and run `kustomize build overlays/staging` before applying.

---

## Review Questions

1. What is the difference between a base and an overlay?
2. Why does Kustomize not require templates or variables?
3. Does `namespace: dev` in a `kustomization.yaml` create the namespace? Why or why not?
4. What is the difference between a strategic merge patch and a JSON 6902 patch?
5. Why does `configMapGenerator` add a hash to the ConfigMap name, and how does that help with updates?
6. What is the difference between `kubectl apply -k` and `kubectl apply -f`?
7. How does the `images` transformer let you change the image tag per environment without editing the base?

## Summary

You created a Kustomize base (Deployment, Service, generated ConfigMap) and two overlays, `dev` and `prod`. Using `namespace`, `namePrefix`, `labels`, `replicas`, `images`, `configMapGenerator` and `patches`, you deployed the same application with different settings per environment, compared the rendered output, applied it with `kubectl apply -k`, and rolled out a change without modifying the base.
