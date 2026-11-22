# Lab 2: Real-world validation (image tag, privileged, registries)

Goal: one ValidatingPolicy with three rules that block `:latest` images, privileged containers, and images from unapproved registries.

## 2.1 Namespace

```bash
kubectl create namespace lab2
```

## 2.2 Policy

```bash
kubectl create -f- << EOF
apiVersion: policies.kyverno.io/v1
kind: ValidatingPolicy
metadata:
  name: pod-hygiene
spec:
  validationActions:
    - Deny
  matchConstraints:
    namespaceSelector:
      matchLabels:
        kubernetes.io/metadata.name: lab2
    resourceRules:
      - apiGroups: ['']
        apiVersions: ['v1']
        operations: ['CREATE', 'UPDATE']
        resources: ['pods']
  validations:
    - message: "images must have an explicit tag and must not use :latest"
      expression: "object.spec.containers.all(c, c.image.contains(':') && !c.image.endsWith(':latest'))"
    - message: "privileged containers are not allowed"
      expression: "object.spec.containers.all(c, !has(c.securityContext) || !has(c.securityContext.privileged) || c.securityContext.privileged == false)"
    - message: "images must come from registry.k8s.io"
      expression: "object.spec.containers.all(c, c.image.startsWith('registry.k8s.io/'))"
EOF
kubectl get validatingpolicy pod-hygiene
```

Wait for ready.

## 2.3 Test: bad image (denied)

```bash
kubectl -n lab2 run bad-latest --image=nginx:latest
```

Expected: denied; the messages for the tag rule and the registry rule are shown.

## 2.4 Test: privileged (denied)

```bash
kubectl -n lab2 apply -f- << EOF
apiVersion: v1
kind: Pod
metadata:
  name: bad-privileged
spec:
  containers:
  - name: c
    image: registry.k8s.io/pause:3.10
    securityContext:
      privileged: true
EOF
```

Expected: `privileged containers are not allowed`.

## 2.5 Test: compliant Pod (allowed)

```bash
kubectl -n lab2 run good --image=registry.k8s.io/pause:3.10
kubectl -n lab2 get pods
```

## 2.6 Notes

- This policy checks `spec.containers` only. Real policies also cover `initContainers` and `ephemeralContainers`.
- Messages come from each failed validation, so write them clearly for developers.

## Cleanup

```bash
kubectl delete validatingpolicy pod-hygiene
kubectl delete namespace lab2
```

Next: `lab3.md`.
