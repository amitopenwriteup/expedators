# k8s Pod Security Standards Lab (Pod Security Admission)

| | |
|---|---|
| **Format** | Hands-on lab, Azure portal (Run command and Apply YAML) |
| **Duration** | 60 to 75 minutes |
| **Level** | Intermediate |
| **Starting point** | An existing, healthy k8 cluster |
| **Cost** | No extra cost beyond the running cluster |

> **PSP vs PSS:** The old **PodSecurityPolicy (PSP)** was removed from Kubernetes in v1.25 and is not available on current AKS versions. Its replacement is **Pod Security Standards (PSS)**, enforced by the built-in **Pod Security Admission (PSA)** controller. This lab covers PSS and PSA.

---

## 1. Lab Objectives

1. Explain the three Pod Security Standards: `privileged`, `baseline`, `restricted`.
2. Use the three PSA modes: `warn`, `audit`, `enforce`.
3. Enforce `baseline` and `restricted` on namespaces and see what each blocks.
4. Fix a rejected pod by correcting its `securityContext`.
5. Roll out PSS safely to an existing namespace without breaking workloads.

## 2. Concepts

### 2.1 The three standards

| Standard | Meaning | Typical use |
|---|---|---|
| `privileged` | No restrictions | System components such as `kube-system` |
| `baseline` | Blocks known privilege escalations; allows default pod settings | Most general workloads |
| `restricted` | Follows pod hardening best practice | Sensitive or internet-facing workloads |

### 2.2 The three modes

| Mode | Effect when a pod violates the standard |
|---|---|
| `enforce` | Pod is **rejected** |
| `audit` | Violation is recorded in the audit log; pod is allowed |
| `warn` | User sees a warning; pod is allowed |

### 2.3 Namespace labels

Format: `pod-security.kubernetes.io/<mode>=<standard>`, and optionally `pod-security.kubernetes.io/<mode>-version=<version>` to pin a Kubernetes version of the standard (default is `latest`).

### 2.4 What each standard checks (selected controls)

| Control | baseline | restricted |
|---|---|---|
| Privileged containers | Blocked | Blocked |
| Host namespaces (hostNetwork, hostPID, hostIPC) | Blocked | Blocked |
| hostPath volumes | Blocked | Blocked |
| Added capabilities beyond a safe list | Blocked | Blocked |
| `allowPrivilegeEscalation: false` required | No | **Yes** |
| `runAsNonRoot: true` required | No | **Yes** |
| `seccompProfile` (RuntimeDefault or Localhost) required | No | **Yes** |
| Capabilities must `drop: ["ALL"]` | No | **Yes** |



### 3.1 Confirm the cluster

Cluster > **Run command**:

```
kubectl get nodes
kubectl version
```

Record the Kubernetes version: ______

---

## 4. Exercise 1: Explore the Defaults

1. Run command:

```
kubectl get namespaces --show-labels
kubectl describe namespace kube-system
```

 Never apply `restricted` or `baseline` enforcement to `kube-system`; system components need elevated settings.

---

## 5. Exercise 2: Baseline Enforcement

### 5.1 Create a baseline namespace

```
kubectl create namespace pss-baseline
kubectl label namespace pss-baseline pod-security.kubernetes.io/enforce=baseline
kubectl get namespace pss-baseline --show-labels
```

### 5.2 Try a privileged pod (should fail)

Apply via **Apply a YAML**:

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: privileged-pod
  namespace: pss-baseline
spec:
  containers:
  - name: nginx
    image: nginx:stable
    securityContext:
      privileged: true
```

Expected: rejected with `violates PodSecurity "baseline:latest": privileged`. 

### 5.3 Try a host-access pod (should fail)

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: hostnet-pod
  namespace: pss-baseline
spec:
  hostNetwork: true
  containers:
  - name: nginx
    image: nginx:stable
```

Expected: rejected for `hostNamespaces`. **Record:** ______

### 5.4 Run a plain pod (should pass)

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: plain-pod
  namespace: pss-baseline
spec:
  containers:
  - name: nginx
    image: nginx:stable
```

Expected: **Running**. A pod running as root is allowed under `baseline`.

```
kubectl get pod plain-pod -n pss-baseline
```

---

## 6. Exercise 3: Restricted Enforcement

### 6.1 Create a restricted namespace

```
kubectl create namespace pss-restricted
kubectl label namespace pss-restricted pod-security.kubernetes.io/enforce=restricted
```

### 6.2 Apply the same plain pod (should fail)

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: plain-pod
  namespace: pss-restricted
spec:
  containers:
  - name: nginx
    image: nginx:stable
```

Expected: rejected. The error lists several controls, for example:

- `allowPrivilegeEscalation != false`
- `unrestricted capabilities`
- `runAsNonRoot != true`
- `seccompProfile`


### 6.3 Fix it step by step

Apply the compliant pod below. It uses an unprivileged nginx image listening on 8080:

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: compliant-pod
  namespace: pss-restricted
spec:
  securityContext:
    runAsNonRoot: true
    runAsUser: 101
    seccompProfile:
      type: RuntimeDefault
  containers:
  - name: nginx
    image: nginxinc/nginx-unprivileged:stable
    ports:
    - containerPort: 8080
    securityContext:
      allowPrivilegeEscalation: false
      capabilities:
        drop: ["ALL"]
```

```
kubectl get pod compliant-pod -n pss-restricted
```

Expected: **Running**.

### 6.4 Why the image matters

The standard `nginx` image starts as root and binds to port 80, which conflicts with `runAsNonRoot`. If you set `runAsNonRoot: true` on it, the pod is admitted but fails to start with a `CreateContainerConfigError` (container would run as root). Use images built for non-root users.

**Test it:** change `compliant-pod` to use `nginx:stable` with `runAsNonRoot: true` and no `runAsUser`. Apply it, then run:

```
kubectl describe pod <pod-name> -n pss-restricted
```

---

## 7. Exercise 4: Safe Rollout to an Existing Namespace

Scenario: a namespace already runs workloads and you want to adopt `restricted` without an outage.

### 7.1 Set up an "existing" workload

```
kubectl create namespace pss-existing
```

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: legacy-app
  namespace: pss-existing
spec:
  replicas: 2
  selector:
    matchLabels:
      app: legacy-app
  template:
    metadata:
      labels:
        app: legacy-app
    spec:
      containers:
      - name: nginx
        image: nginx:stable
```

### 7.2 Dry-run the enforcement

This shows what **would** be violated by existing pods without changing anything:

```
kubectl label --dry-run=server --overwrite namespace pss-existing pod-security.kubernetes.io/enforce=restricted
```

The output includes warnings for each existing pod that violates the standard. **Record:** ______

### 7.3 Use warn and audit first

```
kubectl label namespace pss-existing pod-security.kubernetes.io/warn=restricted pod-security.kubernetes.io/audit=restricted
```

Restart the Deployment and observe warnings (existing pods keep running):

```
kubectl rollout restart deploy/legacy-app -n pss-existing
```

### 7.4 Fix, then enforce

1. Update the Deployment's pod template to be compliant (same as 6.3).
2. Confirm pods are Running.
3. Enforce:

```
kubectl label namespace pss-existing pod-security.kubernetes.io/enforce=restricted --overwrite
```
