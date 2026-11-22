# Lab 8: Troubleshooting, fault injection, and uninstall

Goal: learn how to see what Kyverno is doing, break it on purpose in a safe namespace, recover, and finally remove it.

## 8.1 Health checklist

```bash
kubectl -n kyverno get pods
kubectl get validatingpolicy,mutatingpolicy,generatingpolicy -A
kubectl get validatingwebhookconfigurations | grep -i kyverno
kubectl -n kyverno logs deploy/kyverno-admission-controller --tail=30
kubectl -n kyverno logs deploy/kyverno-background-controller --tail=30
kubectl -n kyverno logs deploy/kyverno-reports-controller --tail=30
kubectl get events -n kyverno --sort-by=.lastTimestamp | tail
```

Policy status:

```bash
kubectl describe validatingpolicy <name>
```

Replace `<name>` with a real policy name. Do not paste the angle brackets.

## 8.2 Setup for the faults

```bash
kubectl create namespace lab8
kubectl create -f- << EOF
apiVersion: policies.kyverno.io/v1
kind: ValidatingPolicy
metadata:
  name: lab8-require-team
spec:
  validationActions:
    - Deny
  matchConstraints:
    namespaceSelector:
      matchLabels:
        kubernetes.io/metadata.name: lab8
    resourceRules:
      - apiGroups: ['']
        apiVersions: ['v1']
        operations: ['CREATE', 'UPDATE']
        resources: ['pods']
  validations:
    - message: "label 'team' is required"
      expression: "has(object.metadata.labels) && has(object.metadata.labels.team) && object.metadata.labels.team != ''"
EOF
kubectl get validatingpolicy lab8-require-team
kubectl -n lab8 run ok --image registry.k8s.io/pause:3.10 --labels team=a
```

Expected: the Pod is created.

## Fault A: Kyverno admission controller is down

Inject:

```bash
kubectl -n kyverno scale deploy kyverno-admission-controller --replicas=0
kubectl -n kyverno get pods
```

Observe:

```bash
kubectl -n lab8 run during-outage --image registry.k8s.io/pause:3.10 --labels team=a
```

Expected: the request fails with a webhook error (for example `no endpoints available` or `failed calling webhook`) because the policy webhook cannot be reached and fails closed. Other namespaces keep working, since the policy only matches `lab8`.

```bash
kubectl run outside-lab --image registry.k8s.io/pause:3.10
kubectl delete pod outside-lab
```

Restore:

```bash
kubectl -n kyverno scale deploy kyverno-admission-controller --replicas=1
kubectl -n kyverno rollout status deploy/kyverno-admission-controller
kubectl -n lab8 run after-outage --image registry.k8s.io/pause:3.10 --labels team=a
```

Lesson: this is why production runs the admission controller with 3 replicas and why Kyverno's own namespace is excluded from its webhooks.

## Fault B: Broken CEL expression

Inject:

```bash
kubectl create -f- << EOF
apiVersion: policies.kyverno.io/v1
kind: ValidatingPolicy
metadata:
  name: lab8-broken
spec:
  validationActions:
    - Deny
  matchConstraints:
    namespaceSelector:
      matchLabels:
        kubernetes.io/metadata.name: lab8
    resourceRules:
      - apiGroups: ['']
        apiVersions: ['v1']
        operations: ['CREATE']
        resources: ['pods']
  validations:
    - message: "broken"
      expression: "object.metadata.nope.label == 'x'"
EOF
```

Observe: either the policy is rejected at creation with a CEL error, or it is created but not ready.

```bash
kubectl get validatingpolicy lab8-broken
kubectl describe validatingpolicy lab8-broken
```

Restore:

```bash
kubectl delete validatingpolicy lab8-broken --ignore-not-found
```

## Fault C: Namespace-wide lockout and break-glass

If a policy ever blocks something you need and you cannot fix it quickly, delete the policy first:

```bash
kubectl delete validatingpolicy lab8-require-team
kubectl -n lab8 run recovered --image registry.k8s.io/pause:3.10
```

Last resort (cluster-wide webhook problem): list the Kyverno webhook configurations and delete the ones that block you, then restart Kyverno so it recreates them.

```bash
kubectl get validatingwebhookconfigurations | grep -i kyverno
kubectl get mutatingwebhookconfigurations | grep -i kyverno
```

Delete one only if truly needed: `kubectl delete validatingwebhookconfiguration <name>`. Then `kubectl -n kyverno rollout restart deploy/kyverno-admission-controller`.

## Troubleshooting cheat sheet

| Symptom | First check |
|---|---|
| Request denied, unclear why | Read the denial message; it names the policy and the failed rule |
| Policy not enforcing | `kubectl get validatingpolicy` ready? Namespace and kind in `matchConstraints`? |
| `no endpoints available` | Admission controller pods not running |
| No Policy Reports | Reports controller logs; wait 30 to 60 seconds |
| Generate does nothing | Background controller logs; RBAC (Lab 4) |
| Exception ignored | Exceptions enabled? Same namespace? (Lab 6) |

## Cleanup the lab

```bash
kubectl delete validatingpolicy lab8-require-team --ignore-not-found
kubectl delete namespace lab8
```

## Uninstall Kyverno (when you finish all labs)

```bash
helm uninstall kyverno -n kyverno
kubectl delete namespace kyverno
```

Helm keeps some Kyverno CRDs after uninstall. Leave them on a shared cluster. On an isolated lab cluster you can list them with `kubectl get crd | grep kyverno` and delete them only if nothing else depends on them.
