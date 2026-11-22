# Lab 3: Mutate Resources (add a default label)

Goal: automatically add `team=bravo` to Pods that have no `team` label, and leave Pods that already have one alone.

## 3.1 Namespace

```bash
kubectl create namespace quick-start-mutation
```

## 3.2 Policy

```bash
kubectl create -f- << EOF
apiVersion: policies.kyverno.io/v1
kind: MutatingPolicy
metadata:
  name: add-labels
spec:
  matchConstraints:
    namespaceSelector:
      matchLabels:
        kubernetes.io/metadata.name: quick-start-mutation
    resourceRules:
      - apiGroups: ['']
        apiVersions: ['v1']
        operations: ['CREATE', 'UPDATE']
        resources: ['pods']
  matchConditions:
    - name: team-label-missing
      expression: '!has(object.metadata.labels) || !has(object.metadata.labels.team)'
  mutations:
    - patchType: ApplyConfiguration
      applyConfiguration:
        expression: >
          Object{
            metadata: Object.metadata{
              labels: Object.metadata.labels{
                team: "bravo"
              }
            }
          }
EOF
kubectl get mutatingpolicy add-labels
```

| Field | Meaning |
|---|---|
| `matchConditions` | Run the mutation only when the `team` label is missing |
| `patchType: ApplyConfiguration` | CEL builds the object to merge in |

## 3.3 Test: Pod without the label

```bash
kubectl -n quick-start-mutation run redis --image registry.k8s.io/pause:3.10
kubectl -n quick-start-mutation get pod redis --show-labels
```

Expected: the Pod has `team=bravo`, which you never typed.

## 3.4 Test: Pod that already has a label

```bash
kubectl -n quick-start-mutation run newredis --image registry.k8s.io/pause:3.10 -l team=alpha
kubectl -n quick-start-mutation get pod newredis --show-labels
```

Expected: `team=alpha` is kept.

## 3.5 Try yourself

- Add a second label `managed-by: "kyverno"` in the same expression.
- Run this lab together with Lab 1's policy: the mutation runs before validation, so Pods without a label pass.

## Cleanup

```bash
kubectl delete mutatingpolicy add-labels
kubectl delete namespace quick-start-mutation
```

Next: `lab4.md`.
