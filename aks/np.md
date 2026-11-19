# Lab: AKS Network Policies (frontend → API → DB)

**Goal:** Lock down a three-tier app with default-deny, then allow only the traffic each tier needs. Prove every rule with curl from debug pods, then compare Azure NPM, Calico and Cilium.

**Prereqs:** Running AKS cluster, `az`, `kubectl`.

**Traffic design:**

```
outsider (other namespace) --X--> api, db
frontend  ------------------> api   (allowed)
frontend  --------X---------> db    (blocked)
api       ------------------> db    (allowed)
db        --------X---------> anything outbound (Part 6)
```

---

## Part 1: Pre-flight (is a policy engine running?)

Without an engine, NetworkPolicy objects are accepted but silently ignored.

```bash
RG=<your-rg>
AKS=<your-cluster>

az aks get-credentials -g $RG -n $AKS --overwrite-existing

az aks show -g $RG -n $AKS \
  --query "networkProfile.{plugin:networkPlugin, mode:networkPluginMode, policy:networkPolicy, dataplane:networkDataplane}" -o json
```

Expected (Cilium example):
```json
{ "plugin": "azure", "mode": "overlay", "policy": "cilium", "dataplane": "cilium" }
```

| `policy` value | Action |
|---|---|
| `cilium`, `azure` or `calico` | Continue to Part 2 |
| `null` | No enforcement. Enable an engine first (below) |

AKS Automatic clusters normally ship with Cilium already enabled.

**If `policy` is null**, try enabling an engine on the existing cluster:

```bash
# Azure NPM
az aks update -g $RG -n $AKS --network-policy azure

# or Cilium (Azure CNI Overlay clusters)
az aks update -g $RG -n $AKS --network-dataplane cilium --network-policy cilium
```


---

## Part 2: Deploy the three tiers

```bash
kubectl create namespace shop
kubectl create namespace other

for t in frontend api db; do
cat <<EOF | kubectl apply -f -
apiVersion: apps/v1
kind: Deployment
metadata: { name: $t, namespace: shop }
spec:
  replicas: 1
  selector: { matchLabels: { app: $t } }
  template:
    metadata: { labels: { app: $t, tier: $t } }
    spec:
      containers:
      - name: app
        image: mcr.microsoft.com/azuredocs/aks-helloworld:v1
        ports: [{ containerPort: 80 }]
        env: [{ name: TITLE, value: "$t" }]
        resources:
          requests: { cpu: 50m, memory: 64Mi }
          limits: { cpu: 200m, memory: 128Mi }
        readinessProbe:
          httpGet: { path: /, port: 80 }
          initialDelaySeconds: 3
---
apiVersion: v1
kind: Service
metadata: { name: $t, namespace: shop }
spec:
  selector: { app: $t }
  ports: [{ port: 80, targetPort: 80 }]
EOF
done

kubectl get pods,svc -n shop
```

Expected:
```
NAME                            READY   STATUS    RESTARTS   AGE
pod/api-xxxxxxxxxx-aaaaa        1/1     Running   0          30s
pod/db-xxxxxxxxxx-bbbbb         1/1     Running   0          30s
pod/frontend-xxxxxxxxxx-ccccc   1/1     Running   0          30s

NAME               TYPE        CLUSTER-IP     PORT(S)   AGE
service/api        ClusterIP   10.0.x.x       80/TCP    30s
service/db         ClusterIP   10.0.x.x       80/TCP    30s
service/frontend   ClusterIP   10.0.x.x       80/TCP    30s
```

---

## Part 3: Test helper and baseline (everything open)

The helper starts a throwaway curl pod with chosen labels, so the pod is treated as that tier. It prints the HTTP code, or `000` when the connection times out (blocked).

```bash
t() {   # usage: t <namespace> <labels> <url>
  kubectl run "t-$RANDOM" -n "$1" --labels "$2" --image=curlimages/curl \
    --restart=Never --rm -i -q -- curl -s -m 3 -o /dev/null -w "%{http_code}\n" "$3" 2>/dev/null
}
```

Baseline:

```bash
t shop  tier=frontend http://api.shop
t shop  tier=frontend http://db.shop
t other app=outsider  http://db.shop
```

Expected (no policy yet):
```
200
200
200
```

The frontend can reach the DB and a pod from another namespace can too. That is the flat network we are fixing.

---

## Part 4: Default deny (ingress)

```bash
cat <<EOF | kubectl apply -f -
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata: { name: default-deny-ingress, namespace: shop }
spec:
  podSelector: {}
  policyTypes: [Ingress]
EOF

kubectl get networkpolicy -n shop
```

Expected:
```
NAME                  POD-SELECTOR   AGE
default-deny-ingress  <none>         5s
```

Re-test:

```bash
t shop  tier=frontend http://api.shop
t shop  tier=api      http://db.shop
t other app=outsider  http://frontend.shop
```

Expected:
```
000
000
000
```

Everything into the `shop` namespace is now blocked. If you still get `200`, your cluster has no policy engine (go back to Part 1).

---

## Part 5: Allow only what each tier needs

```bash
cat <<EOF | kubectl apply -f -
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata: { name: allow-frontend-from-all, namespace: shop }
spec:
  podSelector: { matchLabels: { tier: frontend } }
  policyTypes: [Ingress]
  ingress:
  - {}
---
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata: { name: allow-api-from-frontend, namespace: shop }
spec:
  podSelector: { matchLabels: { tier: api } }
  policyTypes: [Ingress]
  ingress:
  - from:
    - podSelector: { matchLabels: { tier: frontend } }
    ports: [{ protocol: TCP, port: 80 }]
---
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata: { name: allow-db-from-api, namespace: shop }
spec:
  podSelector: { matchLabels: { tier: db } }
  policyTypes: [Ingress]
  ingress:
  - from:
    - podSelector: { matchLabels: { tier: api } }
    ports: [{ protocol: TCP, port: 80 }]
EOF

kubectl get networkpolicy -n shop
```

Expected:
```
NAME                      POD-SELECTOR    AGE
allow-api-from-frontend   tier=api        5s
allow-db-from-api         tier=db         5s
allow-frontend-from-all   tier=frontend   5s
default-deny-ingress      <none>          2m
```

Test the full matrix:

```bash
echo "outsider -> frontend"; t other app=outsider  http://frontend.shop
echo "outsider -> api";      t other app=outsider  http://api.shop
echo "outsider -> db";       t other app=outsider  http://db.shop
echo "frontend -> api";      t shop  tier=frontend http://api.shop
echo "frontend -> db";       t shop  tier=frontend http://db.shop
echo "api -> db";            t shop  tier=api      http://db.shop
```

Expected:
```
outsider -> frontend
200
outsider -> api
000
outsider -> db
000
frontend -> api
200
frontend -> db
000
api -> db
200
```

The frontend is reachable by anyone (standing in for the ingress controller), the API only from the frontend, the DB only from the API. Policies are additive: allow rules open holes in the default deny.

---

## Part 6: Egress lockdown for the DB tier

First confirm the DB tier can currently reach the internet:

```bash
t shop tier=db https://example.com
```

Expected:
```
200
```

Block all outbound traffic from the DB tier except DNS:

```bash
cat <<EOF | kubectl apply -f -
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata: { name: db-egress-dns-only, namespace: shop }
spec:
  podSelector: { matchLabels: { tier: db } }
  policyTypes: [Egress]
  egress:
  - to:
    - namespaceSelector: { matchLabels: { kubernetes.io/metadata.name: kube-system } }
    ports:
    - { protocol: UDP, port: 53 }
    - { protocol: TCP, port: 53 }
EOF
```

Re-test:

```bash
t shop tier=db https://example.com
t shop tier=db http://api.shop
```

Expected:
```
000
000
```

A compromised DB pod can no longer call out, which blocks data exfiltration. DNS still works because port 53 to `kube-system` is allowed.

---

## Part 7: Compare the engines

| | Azure NPM | Calico | Cilium |
|---|---|---|---|
| Dataplane | iptables / ipsets | iptables | eBPF |
| OS support | Linux (Windows support ended 30 Sep 2026) | Linux, Windows Server 2019/2022 | Linux |
| Networking | Azure CNI | Azure CNI, kubenet (Linux) | Azure CNI (Overlay or pod subnet) |
| Standard K8s NetworkPolicy | Yes | Yes | Yes |
| Extra policy features | None | Calico-specific CRDs (AKS does not test or support these) | FQDN, L7 policies, Hubble observability |
| Scale limit | 250 nodes / 20,000 pods | None stated | None stated |
| Lifecycle | Linux support ends 30 Sep 2028 | Supported | Microsoft's recommended choice for new clusters |

Key point: the YAML you wrote in this lab is standard Kubernetes NetworkPolicy, so it works unchanged on all three. The engines differ in performance, features beyond the standard, and long-term support.

---

## Part 8: Cleanup

```bash
kubectl delete namespace shop other
kubectl get ns
```

Expected: `namespace "shop" deleted`, `namespace "other" deleted`, and neither appears in `kubectl get ns`. Deleting the namespaces removes all the policies, deployments and services.

Leave the network policy engine enabled on your cluster. If you created the throwaway cluster in Part 1, delete it with `az group delete -n rg-netpol-lab --yes --no-wait`.
