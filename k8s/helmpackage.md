# Lab: Create Your First Helm Chart (Nginx)

## Objective

In this lab you will build a Helm chart from scratch that deploys an Nginx application on Kubernetes. You will:

- Scaffold a chart with `helm create` and clean it down to a blank slate
- Generate Kubernetes manifests with `kubectl --dry-run` and use them as chart templates
- Define chart metadata in `Chart.yaml` and add a `NOTES.txt`
- Lint, dry-run, install, package, and uninstall the chart

**Estimated time:** 20–30 minutes

---

## Prerequisites

| Requirement | Check with |
|---|---|
| A running Kubernetes cluster (minikube, kind, k3s, EKS, etc.) | `kubectl get nodes` |
| `kubectl` configured to talk to the cluster | `kubectl version --client` |
| Helm 3.x installed | `helm version` |

---

## Step 1: Create the chart scaffold

```bash
helm create testchart
```

This creates a directory named `testchart` with a default chart structure.

## Step 2: Move into the chart directory

```bash
cd testchart
```

## Step 3: Explore the generated structure

```bash
ls
```

Expected output:

```
Chart.yaml  charts  templates  values.yaml
```

| File / Directory | Purpose |
|---|---|
| `Chart.yaml` | Chart metadata (name, version, description) |
| `values.yaml` | Default configuration values |
| `charts/` | Dependency (sub-chart) directory |
| `templates/` | Kubernetes manifest templates |

## Step 4: Clean up the templates directory

The default templates are feature-rich but complex. Remove them so we can start from scratch.

```bash
rm -rf templates/*
```

Verify it is empty:

```bash
ls templates/
```

## Step 5: Create the Deployment template

Generate a Deployment manifest using `kubectl` without actually creating it in the cluster:

```bash
kubectl create deployment nginx --image=nginx --dry-run=client -o yaml > templates/deployment.yaml
```

Review the file:

```bash
cat templates/deployment.yaml
```

> **Note:** Using `>` (overwrite) instead of `>>` (append) avoids duplicated content if you run the command more than once.

## Step 6: Create the Service template

`kubectl expose` needs an existing deployment to read from, so first create a temporary deployment in the cluster, then generate the Service manifest from it.

```bash
kubectl create deployment nginx --image=nginx

kubectl expose deploy nginx --port 80 --type NodePort --dry-run=client -o yaml > templates/service.yaml
```

Review the file:

```bash
cat templates/service.yaml
```

## Step 7: Define the required values in Chart.yaml

View the current file:

```bash
cat Chart.yaml
```

Only the essential fields are needed. Edit `Chart.yaml` so it looks like this (remove the extra comments):

```yaml
apiVersion: v2
name: testchart
description: My first Helm chart that deploys nginx
type: application
version: 0.1.0
appVersion: "1.0"
```

## Step 8: Add a NOTES.txt

`NOTES.txt` is printed to the terminal after a successful `helm install`.

```bash
echo "This is first helm chart and it will deploy nginx application" >> templates/NOTES.txt
```

## Step 9: Go back to the parent directory

```bash
cd ..
```

Your layout should now be:

```
testchart/
├── Chart.yaml
├── charts/
├── templates/
│   ├── NOTES.txt
│   ├── deployment.yaml
│   └── service.yaml
└── values.yaml
```

> `values.yaml` is not used by the templates yet. You can leave it or empty it; Helm does not require it to have content.

## Step 10: Lint the chart

Always lint before deploying to catch syntax errors and best-practice violations.

```bash
helm lint ./testchart
```

Expected output:

```
==> Linting ./testchart
[INFO] Chart.yaml: icon is recommended

1 chart(s) linted, 0 chart(s) failed
```

The `icon is recommended` message is informational and can be ignored.

## Step 11: Delete the temporary resources

The Deployment and Service created manually in Step 6 would conflict with the Helm release (same names), so remove them first.

```bash
kubectl delete deployment nginx
kubectl delete svc nginx
```

> If `kubectl delete svc nginx` returns "not found", that's fine. In Step 6 only the Deployment was created in the cluster; the Service was just written to a file.

## Step 12: Perform a dry run

Render the templates and validate against the cluster without installing anything.

```bash
helm install testchart ./testchart --dry-run
```

Check that the output shows the rendered Deployment, Service, and the contents of `NOTES.txt`.

## Step 13: Install the chart

```bash
helm install testchart ./testchart
```

Verify the resources created by the release:

```bash
kubectl get deployment
kubectl get svc
```

Expected output (values will differ):

```
NAME    READY   UP-TO-DATE   AVAILABLE   AGE
nginx   1/1     1            1           10s

NAME         TYPE        CLUSTER-IP      EXTERNAL-IP   PORT(S)        AGE
nginx        NodePort    10.96.123.45    <none>        80:3xxxx/TCP   10s
```

Optionally test access using the NodePort:

```bash
kubectl get svc nginx -o jsonpath='{.spec.ports[0].nodePort}'
curl http://<node-ip>:<nodeport>
```

## Step 14: List Helm releases

```bash
helm list
```

Expected output:

```
NAME       NAMESPACE  REVISION  STATUS    CHART            APP VERSION
testchart  default    1         deployed  testchart-0.1.0  1.0
```

## Step 15: Package the chart

```bash
helm package testchart/
```

Expected output:

```
Successfully packaged chart and saved it to: /root/testchart-0.1.0.tgz
```

The `.tgz` archive can be shared or pushed to a chart repository.

## Step 16: Uninstall the release

```bash
helm uninstall testchart
```

Confirm everything was removed:

```bash
kubectl get svc
kubectl get deployment
```

The `nginx` Deployment and Service should no longer be listed.

---

## Verification Checklist

- [ ] `helm lint` passes with 0 failures
- [ ] `helm install --dry-run` renders both manifests and the notes
- [ ] `helm list` shows `testchart` as `deployed`
- [ ] The `nginx` Deployment and Service exist after install
- [ ] A `testchart-0.1.0.tgz` package was created
- [ ] All resources are gone after `helm uninstall`

---

## Troubleshooting

| Problem | Likely cause | Fix |
|---|---|---|
| `Error: INSTALLATION FAILED: ... already exists` | Manually created `nginx` Deployment/Service still present | Run `kubectl delete deployment nginx` and `kubectl delete svc nginx` |
| `Error: cannot re-use a name that is still in use` | A release named `testchart` already exists | `helm uninstall testchart` or use a different release name |
| `helm lint` reports YAML parse errors | Broken indentation or duplicated content in templates | Re-generate the file with `>` instead of `>>` |
| Pods stuck in `ImagePullBackOff` | Cluster cannot reach Docker Hub | Check network/registry access |
| `kubectl expose` says deployment not found | Deployment was not created first | Run `kubectl create deployment nginx --image=nginx` |

---

## Cleanup

```bash
helm uninstall testchart 2>/dev/null
kubectl delete deployment nginx --ignore-not-found
kubectl delete svc nginx --ignore-not-found
rm -rf testchart testchart-0.1.0.tgz
```

---

## Challenge (Optional)

Make the chart configurable by moving hard-coded values into `values.yaml`:

1. Add to `values.yaml`:
   ```yaml
   replicaCount: 2
   image:
     repository: nginx
     tag: latest
   service:
     type: NodePort
     port: 80
   ```
2. Update `templates/deployment.yaml` to use `{{ .Values.replicaCount }}` and `{{ .Values.image.repository }}:{{ .Values.image.tag }}`.
3. Update `templates/service.yaml` to use `{{ .Values.service.type }}` and `{{ .Values.service.port }}`.
4. Re-run `helm lint`, then `helm install testchart ./testchart --set replicaCount=3`.
5. Try `helm upgrade` and `helm rollback` to see revision history with `helm history testchart`.
