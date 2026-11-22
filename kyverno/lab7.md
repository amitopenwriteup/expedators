# Lab 7: Kyverno CLI (test policies before deploying)

Goal: run a policy against YAML files on your laptop or VM, with no cluster, so mistakes are caught in CI.

## 7.1 Install the CLI

Match the CLI to your Kyverno version. Read `APP VERSION` from:

```bash
helm list -n kyverno
```

Option A: release binary (set the version, for example `v1.17.0`):

```bash
export KVER=v1.17.0
curl -fL https://github.com/kyverno/kyverno/releases/download/${KVER}/kyverno-cli_${KVER}_linux_x86_64.tar.gz -o /tmp/kyverno-cli.tgz
ls -lh /tmp/kyverno-cli.tgz
tar xzf /tmp/kyverno-cli.tgz -C /tmp kyverno
sudo mv /tmp/kyverno /usr/local/bin/
kyverno version
```

The file should be several MB. A tiny file means the download failed: check the version and the asset name on the GitHub releases page.

Option B: krew: `kubectl krew install kyverno`, then use `kubectl kyverno` instead of `kyverno`.

## 7.2 Work folder

```bash
mkdir -p ~/kyverno-cli-lab && cd ~/kyverno-cli-lab
```

## 7.3 Policy file (no namespace selector, so it works offline)

```bash
cat > policy.yaml << EOF
apiVersion: policies.kyverno.io/v1
kind: ValidatingPolicy
metadata:
  name: require-team-label
spec:
  validationActions:
    - Deny
  matchConstraints:
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

## 7.4 Resources to test

```bash
cat > resources.yaml << EOF
apiVersion: v1
kind: Pod
metadata:
  name: good-pod
  labels:
    team: backend
spec:
  containers:
  - name: c
    image: registry.k8s.io/pause:3.10
---
apiVersion: v1
kind: Pod
metadata:
  name: bad-pod
spec:
  containers:
  - name: c
    image: registry.k8s.io/pause:3.10
EOF
```

## 7.5 Run it

```bash
kyverno apply policy.yaml --resource resources.yaml
```

Expected: one pass (`good-pod`) and one fail (`bad-pod`) with the message `label 'team' is required`.

Fix the resource and rerun:

```bash
sed -i 's/  name: bad-pod/  name: bad-pod\n  labels:\n    team: fixed/' resources.yaml
kyverno apply policy.yaml --resource resources.yaml
```

Expected: both pass.

## 7.6 Optional: a repeatable test with `kyverno test`

```bash
cat > kyverno-test.yaml << EOF
apiVersion: cli.kyverno.io/v1alpha1
kind: Test
metadata:
  name: require-team-label-test
policies:
  - policy.yaml
resources:
  - resources.yaml
results:
  - policy: require-team-label
    isValidatingPolicy: true
    resources:
      - good-pod
    result: pass
EOF
kyverno test .
```

The test file schema can change between CLI versions. If it errors, check `kyverno test --help` and the Testing Policies guide on kyverno.io.

## 7.7 Why it matters

Run `kyverno apply` or `kyverno test` in CI on every pull request, so a policy or manifest change is checked before it reaches a cluster.

## Cleanup

```bash
cd ~ && rm -rf ~/kyverno-cli-lab
```

Next: `lab8.md`.
