# Lab 4: Generate Resources (copy a Secret into a namespace)

Goal: when a destination namespace is labeled ready, Kyverno copies an image pull Secret into it.

> Lab only. Use the dummy values below, never real registry credentials. Kyverno's background controller normally cannot read Secrets, so this lab grants it narrow access.

## 4.1 Source namespace and dummy Secret

```bash
kubectl create namespace quick-start-source
kubectl -n quick-start-source create secret docker-registry regcred \
  --docker-server=registry.example.invalid \
  --docker-username=demo-user \
  --docker-password=replace-with-non-sensitive-placeholder \
  --docker-email=demo@example.invalid
```

## 4.2 RBAC for the background controller

```bash
kubectl apply -f- << EOF
apiVersion: rbac.authorization.k8s.io/v1
kind: Role
metadata:
  name: kyverno-quick-start-source-secret
  namespace: quick-start-source
rules:
- apiGroups: ['']
  resources: [secrets]
  resourceNames: [regcred]
  verbs: [get]
---
apiVersion: rbac.authorization.k8s.io/v1
kind: RoleBinding
metadata:
  name: kyverno-quick-start-source-secret
  namespace: quick-start-source
subjects:
- kind: ServiceAccount
  name: kyverno-background-controller
  namespace: kyverno
roleRef:
  apiGroup: rbac.authorization.k8s.io
  kind: Role
  name: kyverno-quick-start-source-secret
---
apiVersion: v1
kind: Namespace
metadata:
  name: quick-start-generated
---
apiVersion: rbac.authorization.k8s.io/v1
kind: Role
metadata:
  name: kyverno-quick-start-generated-secret
  namespace: quick-start-generated
rules:
- apiGroups: ['']
  resources: [secrets]
  resourceNames: [regcred]
  verbs: [get]
- apiGroups: ['']
  resources: [secrets]
  verbs: [create]
---
apiVersion: rbac.authorization.k8s.io/v1
kind: RoleBinding
metadata:
  name: kyverno-quick-start-generated-secret
  namespace: quick-start-generated
subjects:
- kind: ServiceAccount
  name: kyverno-background-controller
  namespace: kyverno
roleRef:
  apiGroup: rbac.authorization.k8s.io
  kind: Role
  name: kyverno-quick-start-generated-secret
EOF
```

Create the policy only after these exist.

## 4.3 GeneratingPolicy

```bash
kubectl create -f- << EOF
apiVersion: policies.kyverno.io/v1
kind: GeneratingPolicy
metadata:
  name: sync-secrets
spec:
  matchConstraints:
    resourceRules:
      - apiGroups: ['']
        apiVersions: ['v1']
        operations: ['UPDATE']
        resources: ['namespaces']
  matchConditions:
    - name: destination-namespace-ready
      expression: "object.metadata.name == 'quick-start-generated' && object.metadata.labels.exists(k, k == 'quick-start.kyverno.io/ready' && object.metadata.labels[k] == 'true') && (!has(oldObject.metadata.labels) || !oldObject.metadata.labels.exists(k, k == 'quick-start.kyverno.io/ready'))"
  variables:
    - name: targetNs
      expression: 'object.metadata.name'
    - name: sourceSecret
      expression: 'resource.Get("v1", "secrets", "quick-start-source", "regcred")'
  generate:
    - expression: 'generator.Apply(variables.targetNs, [variables.sourceSecret])'
EOF
```

The condition fires only on the update that first sets the label `quick-start.kyverno.io/ready=true`, so later updates do not regenerate.

## 4.4 Trigger it

```bash
kubectl wait --for='jsonpath={.status.conditionStatus.ready}=true' generatingpolicy/sync-secrets --timeout=120s
kubectl label namespace quick-start-generated quick-start.kyverno.io/ready=true
kubectl wait --for=create secret/regcred -n quick-start-generated --timeout=120s
kubectl -n quick-start-generated get secret
```

Expected: `regcred` now exists in `quick-start-generated`.

## 4.5 If nothing appears

```bash
kubectl -n kyverno logs deploy/kyverno-background-controller --tail=30
kubectl describe generatingpolicy sync-secrets
```

A `forbidden` error means a Role or RoleBinding from 4.2 is missing.

## Cleanup

```bash
kubectl delete generatingpolicy sync-secrets
kubectl -n quick-start-generated delete secret regcred --ignore-not-found
kubectl -n quick-start-generated delete rolebinding kyverno-quick-start-generated-secret
kubectl -n quick-start-generated delete role kyverno-quick-start-generated-secret
kubectl -n quick-start-source delete rolebinding kyverno-quick-start-source-secret
kubectl -n quick-start-source delete role kyverno-quick-start-source-secret
kubectl -n quick-start-source delete secret regcred
kubectl delete namespace quick-start-generated quick-start-source
```

Next: `lab5.md`.
