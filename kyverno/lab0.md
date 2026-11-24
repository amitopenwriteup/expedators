# Lab 0: Install Kyverno

Goal: install Kyverno with Helm on your kubeadm cluster (`vm-1` control plane, `vm2` worker) and confirm it is healthy.
Time: 10 minutes. Run everything on `vm-1`.



## 0.1 Pre-checks

```bash
kubectl get nodes -o wide
kubectl version
```

Both nodes must be `Ready`. The Helm chart needs Kubernetes 1.25 or newer (1.30+ recommended). Check supported versions on the Kyverno Releases page (kyverno.io/docs/installation/releases).

## 0.2 Install Helm (skip if `helm version` works)

```bash
helm version
curl -fsSL https://raw.githubusercontent.com/helm/helm/main/scripts/get-helm-3 | sudo bash
helm version
```

## 0.3 Install Kyverno

```bash
helm repo add kyverno https://kyverno.github.io/kyverno/
helm repo update
helm search repo kyverno/kyverno
helm install kyverno kyverno/kyverno -n kyverno --create-namespace
```

The default install creates four Deployments with one replica each. Note the `APP VERSION`:

```bash
helm list -n kyverno
```

## 0.4 Verify

```bash
kubectl -n kyverno get pods
kubectl -n kyverno get deploy
```

Expected Deployments, all `1/1`:

| Deployment | Role |
|---|---|
| `kyverno-admission-controller` | Webhook and Engine |
| `kyverno-background-controller` | Generate and mutate-existing policies |
| `kyverno-cleanup-controller` | Cleanup and DeletingPolicy |
| `kyverno-reports-controller` | Policy Reports |

```bash
kubectl wait --for=condition=Ready pods --all -n kyverno --timeout=180s
```

## 0.5 Check CRDs and API version

```bash
kubectl get crd | grep kyverno
kubectl api-resources --api-group=policies.kyverno.io
kubectl get crd validatingpolicies.policies.kyverno.io -o jsonpath='{.spec.versions[*].name}'; echo
```

The labs use `apiVersion: policies.kyverno.io/v1`, which needs **Kyverno 1.17 or newer**. If `v1` is not in the list, run `helm repo update` and `helm upgrade kyverno kyverno/kyverno -n kyverno`.

## 0.6 Check the webhooks

```bash
kubectl get validatingwebhookconfigurations | grep -i kyverno
kubectl get mutatingwebhookconfigurations | grep -i kyverno
```

These are the webhooks the API server calls. The Webhook Controller edits them as policies are added.



