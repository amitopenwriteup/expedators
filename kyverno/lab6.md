# Lab 6: Policy Exceptions

Goal: keep a Deny policy enforced for everyone, but let one named Pod through using a PolicyException.

## 6.1 Check that exceptions are enabled

```bash
kubectl -n kyverno get deploy kyverno-admission-controller -o yaml | grep -i exception
```

If nothing is printed or the flag is `false`, enable it and wait for the rollout:

```bash
helm upgrade kyverno kyverno/kyverno -n kyverno --reuse-values --set features.policyExceptions.enabled=true
kubectl -n kyverno rollout status deploy/kyverno-admission-controller
```

(Value names can differ between chart versions. Check `helm show values kyverno/kyverno | grep -i -A3 policyExceptions`.)

## 6.2 Namespace and Deny policy

```bash
kubectl create namespace lab6
kubectl create -f- << EOF
apiVersion: policies.kyverno.io/v1
kind: ValidatingPolicy
metadata:
  name: require-team-label
spec:
  validationActions:
    - Deny
  matchConstraints:
    namespaceSelector:
      matchLabels:
        kubernetes.io/metadata.name: lab6
    resourceRules:
      - apiGroups: ['']
        apiVersions: ['v1']
        operations: ['CREATE', 'UPDATE']
        resources: ['pods']
  validations:
    - message: "label 'team' is required"
      expression: "has(object.metadata.labels) && has(object.metadata.labels.team) && object.metadata.labels.team != ''"
EOF
kubectl get validatingpolicy require-team-label
```

## 6.3 Before the exception: both are denied

```bash
kubectl -n lab6 run legacy-app --image registry.k8s.io/pause:3.10
kubectl -n lab6 run other-app --image registry.k8s.io/pause:3.10
```

Expected: both denied.

## 6.4 Create the exception

```bash
kubectl apply -f- << EOF
apiVersion: policies.kyverno.io/v1
kind: PolicyException
metadata:
  name: allow-legacy-app
  namespace: lab6
spec:
  policyRefs:
    - name: require-team-label
      kind: ValidatingPolicy
  matchConditions:
    - name: match-legacy-name
      expression: "object.metadata.name == 'legacy-app'"
EOF
kubectl -n lab6 get policyexception
```

## 6.5 After the exception

```bash
kubectl -n lab6 run legacy-app --image registry.k8s.io/pause:3.10
kubectl -n lab6 run other-app --image registry.k8s.io/pause:3.10
```

Expected: `legacy-app` is **allowed**, `other-app` is still **denied**.

## 6.6 Report shows the skip

```bash
kubectl -n lab6 get policyreport -o yaml | grep -E 'policy:|result:|rule:'
```

Expected: `result: skip` for `legacy-app`.

## 6.7 Remove the exception

```bash
kubectl -n lab6 delete policyexception allow-legacy-app
kubectl -n lab6 delete pod legacy-app
kubectl -n lab6 run legacy-app --image registry.k8s.io/pause:3.10
```

Expected: denied again. Exceptions are a controlled, reviewable waiver, so keep them narrow and time-boxed.

## Troubleshooting

- Exception ignored: confirm 6.1, and that the exception is in the same namespace as the Pod (or the exception namespace your install is configured for).
- Check logs: `kubectl -n kyverno logs deploy/kyverno-admission-controller --tail=30`.

## Cleanup

```bash
kubectl delete validatingpolicy require-team-label
kubectl delete namespace lab6
```

Next: `lab7.md`.
