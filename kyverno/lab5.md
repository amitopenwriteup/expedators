# Lab 5: Audit mode, Policy Reports, and promoting to Deny

Goal: roll out a policy safely. Start with `Audit` (allow but report), read the Policy Report, then switch to `Deny`.

## 5.1 Namespace and policy in Audit mode

```bash
kubectl create namespace lab5
kubectl create -f- << EOF
apiVersion: policies.kyverno.io/v1
kind: ValidatingPolicy
metadata:
  name: audit-team-label
spec:
  validationActions:
    - Audit
  matchConstraints:
    namespaceSelector:
      matchLabels:
        kubernetes.io/metadata.name: lab5
    resourceRules:
      - apiGroups: ['']
        apiVersions: ['v1']
        operations: ['CREATE', 'UPDATE']
        resources: ['pods']
  validations:
    - message: "label 'team' is required"
      expression: "has(object.metadata.labels) && has(object.metadata.labels.team) && object.metadata.labels.team != ''"
EOF
kubectl get validatingpolicy audit-team-label
```

## 5.2 Create one bad and one good Pod

```bash
kubectl -n lab5 run no-label --image registry.k8s.io/pause:3.10
kubectl -n lab5 run with-label --image registry.k8s.io/pause:3.10 --labels team=backend
kubectl -n lab5 get pods
```

Expected: **both are allowed**. Audit never blocks (you may see a warning for the bad one).

## 5.3 Read the Policy Report

Reports are built a few seconds later by the reports controller.

```bash
kubectl api-resources | grep -i report
kubectl -n lab5 get policyreport
kubectl -n lab5 get policyreport -o yaml | grep -E 'policy:|result:|message:|name:'
```

Expected: a `fail` result for `no-label` and a `pass` for `with-label`, under policy `audit-team-label`. If empty, wait 30 seconds and retry.

## 5.4 Promote to Deny

```bash
kubectl patch validatingpolicy audit-team-label --type merge -p '{"spec":{"validationActions":["Deny"]}}'
sleep 10
kubectl -n lab5 run no-label-2 --image registry.k8s.io/pause:3.10
```

Expected: denied with `label 'team' is required`.

The existing Pod `no-label` is not removed. Deny only affects new requests; the report still lists it.

```bash
kubectl -n lab5 get pods
```

## 5.5 Lesson

Audit first, fix what the report shows, then Deny. This avoids blocking teams on day one.

## Cleanup

```bash
kubectl delete validatingpolicy audit-team-label
kubectl delete namespace lab5
```

Next: `lab6.md`.
