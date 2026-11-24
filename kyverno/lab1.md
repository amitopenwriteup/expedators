# Lab 1: Validate Resources (require a label)

Goal: block Pods that do not have a `team` label, then see the policy also cover Deployments through auto-generation.

## 1.1 Create the lab namespace

```bash
kubectl create namespace quick-start-validation
```

## 1.2 Create the policy

```bash
kubectl create -f- << EOF
apiVersion: policies.kyverno.io/v1
kind: ValidatingPolicy
metadata:
  name: require-labels
spec:
  validationActions:
    - Deny
  autogen:
    podControllers:
      controllers:
        - deployments
  matchConstraints:
    namespaceSelector:
      matchLabels:
        kubernetes.io/metadata.name: quick-start-validation
    resourceRules:
      - apiGroups: ['']
        apiVersions: ['v1']
        operations: ['CREATE', 'UPDATE']
        resources: ['pods']
  validations:
    - message: "label 'team' is required"
      expression: "has(object.metadata.labels) && has(object.metadata.labels.team) && object.metadata.labels.team != ''"
EOF
```

What it means:

| Field | Meaning |
|---|---|
| `validationActions: Deny` | Block non-compliant requests |
| `matchConstraints` | Only Pods, only in this namespace |
| `validations.expression` | CEL; must be true to pass |
| `autogen` | Also apply to Deployment Pod templates |

```bash
kubectl get validatingpolicy require-labels
```

Wait until it shows ready (about a minute).

## 1.3 Test: non-compliant Deployment (denied)

```bash
kubectl -n quick-start-validation create deployment nginx --image=nginx
```

Expected: the request is denied with `label 'team' is required`.

## 1.4 Test: compliant Pod (allowed)

```bash
kubectl -n quick-start-validation run nginx --image registry.k8s.io/pause:3.10 --labels team=backend
kubectl -n quick-start-validation get pods --show-labels
```

## 1.5 Test: other namespaces are untouched

```bash
kubectl run free-pod --image registry.k8s.io/pause:3.10
kubectl delete pod free-pod
```

Expected: allowed, because the policy only matches `quick-start-validation`.

## Cleanup

```bash
kubectl delete validatingpolicy require-labels
kubectl delete namespace quick-start-validation
```

Next: `lab2.md`.
