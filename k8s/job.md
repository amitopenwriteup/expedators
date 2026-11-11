# Lab: Jobs in Kubernetes

## 1. Objective

By the end of this lab you will be able to:

- Create and run a Kubernetes **Job**
- Use `completions` to run a task multiple times
- Use `parallelism` to run several pods at the same time
- Observe the behaviour when `parallelism` is greater than `completions`

## 2. Prerequisites

- A running Kubernetes cluster (Minikube / kind / kubeadm)
- `kubectl` configured to talk to the cluster
- Basic knowledge of Pods and YAML

Verify the cluster:

```bash
kubectl cluster-info
kubectl get nodes
```

## 3. Theory

A **Job** creates one or more pods and makes sure that a specified number of them **terminate successfully**. Unlike a Deployment, a Job is meant for run-to-completion tasks (batch processing, backups, migrations), so its pods are not restarted once they finish.

### Key fields

| Field | Description | Default |
|---|---|---|
| `completions` | Number of pods that must finish successfully for the Job to be complete | 1 |
| `parallelism` | Maximum number of pods running at the same time | 1 |
| `restartPolicy` | Must be `Never` or `OnFailure` for a Job (not `Always`) | – |

### Behaviour at a glance

| completions | parallelism | Behaviour |
|---|---|---|
| 7 | not set (1) | Pods run **one after another**, 7 in total |
| 7 | 2 | Up to **2 pods at a time**, 7 in total |
| not set | 2 | **2 pods run at once**; the Job completes when they succeed |

## 4. Lab Steps

### Part A: Job with completions

#### Step 1: Create `job.yaml`

```yaml
apiVersion: batch/v1
kind: Job
metadata:
  name: node-app-job
spec:
  completions: 7
  template:
    spec:
      containers:
      - name: node-app-job
        image: alpine
        command: ["echo", "Welcome to my Node app"]
      restartPolicy: Never
```

#### Step 2: Create the Job

```bash
kubectl create -f job.yaml
```

Expected output:

```
job.batch/node-app-job created
```

#### Step 3: Watch the Job

```bash
kubectl get job node-app-job -w
```

Expected output (the pods run **one at a time**, so the count increases step by step):

```
NAME           COMPLETIONS   DURATION   AGE
node-app-job   0/7           2s         2s
node-app-job   1/7           3s         3s
node-app-job   2/7           5s         5s
...
node-app-job   7/7           20s        20s
```

Press `Ctrl + C` to stop watching.

#### Step 4: Describe the Job

```bash
kubectl describe job node-app-job
```

Observe: `Completions: 7`, `Parallelism: 1`, and `Pods Statuses: 0 Active / 7 Succeeded / 0 Failed`.

You can also view the pods and their output:

```bash
kubectl get pods
kubectl logs <pod-name>
```

#### Step 5: Delete the Job

```bash
kubectl delete -f job.yaml
```

> Deleting the Job also deletes the pods it created.

---

### Part B: Job with completions and parallelism

#### Step 1: Create `job1.yaml`

Same as before, with `parallelism: 2` added:

```yaml
apiVersion: batch/v1
kind: Job
metadata:
  name: node-app-job
spec:
  completions: 7
  parallelism: 2
  template:
    spec:
      containers:
      - name: node-app-job
        image: alpine
        command: ["echo", "Welcome to my Node app"]
      restartPolicy: Never
```

#### Step 2: Create, watch, describe and delete

```bash
kubectl create -f job1.yaml
kubectl get job node-app-job -w
kubectl describe job node-app-job
kubectl delete -f job1.yaml
```

Observe: pods now run **two at a time**, so the Job finishes faster than in Part A. The `Parallelism` field in the describe output shows `2`.

---

### Part C: Job with only parallelism (no completions)

#### Step 1: Create `job2.yaml` using `vi`

```bash
vi job2.yaml
```

Paste the following, then save and exit (`Esc`, then `:wq`):

```yaml
apiVersion: batch/v1
kind: Job
metadata:
  name: node-app-job
spec:
  parallelism: 2
  template:
    spec:
      containers:
      - name: node-app-job
        image: alpine
        command: ["echo", "Welcome to my Node app"]
      restartPolicy: Never
```

#### Step 2: Create, watch, describe and delete

```bash
kubectl create -f job2.yaml
kubectl get job node-app-job -w
kubectl describe job node-app-job
kubectl delete -f job2.yaml
```

Observe: since `completions` is not set, **two pods are started in parallel** and the Job is complete as soon as they succeed. In the describe output, `Completions` is shown as `<unset>`.

---

## 5. Exercise: parallelism greater than completions

Edit `job.yaml` so that `parallelism` is **greater than** `completions`, for example:

```yaml
spec:
  completions: 3
  parallelism: 5
```

Run the Job and check the values:

```bash
kubectl create -f job.yaml
kubectl get job node-app-job -w
kubectl describe job node-app-job
kubectl get pods
kubectl delete -f job.yaml
```

Observe:

- Only **3 pods** are created in total, not 5. Kubernetes never runs more pods than the number of remaining completions.
- The effective parallelism is `min(parallelism, remaining completions)`.
- The Job still shows `Completions: 3` and `Parallelism: 5` in the describe output.

## 6. Observations

| Job file | completions | parallelism | Pods run at a time | Total successful pods |
|---|---|---|---|---|
| `job.yaml` | 7 | 1 (default) | 1 | 7 |
| `job1.yaml` | 7 | 2 | 2 | 7 |
| `job2.yaml` | unset | 2 | 2 | at least 1 (2 started) |
| Exercise | 3 | 5 | 3 | 3 |

## 7. Cleanup

```bash
kubectl get jobs
kubectl get pods
```

If any Job or pod is left over, remove it:

```bash
kubectl delete job node-app-job
```
