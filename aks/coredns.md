# AKS DNS Lab: How CoreDNS Works in AKS

Runs on your **existing AKS cluster**. No new cluster needed. Everything lives in a throwaway namespace (`dns-lab`) and is cleaned up at the end. We do **not** customize CoreDNS; we observe and understand it.

> Use a dev/test cluster. The only things created are a namespace, a debug pod and a small test deployment.

---

## Lab Map

| Module | Topic | Changes cluster? |
|---|---|---|
| 0 | Setup | Namespace + test pod |
| 1 | CoreDNS anatomy in AKS | Read-only |
| 2 | Pod DNS config, search domains, ndots | Read-only |
| 3 | Record types: service, headless, SRV, external | Small test workloads |
| 4 | Cleanup | Removes everything |

**Prereqs:** `az`, `kubectl`; cluster admin access; `az aks get-credentials` already done.

---

## Module 0: Setup

```bash
export RG=<aks-resource-group>
export AKS=<aks-cluster-name>

kubectl create namespace dns-lab

# Debug pod with dig/nslookup/curl
kubectl -n dns-lab run dnsutils --image=registry.k8s.io/e2e-test-images/jessie-dnsutils:1.3 \
  --restart=Never --command -- sleep infinity

kubectl -n dns-lab wait --for=condition=Ready pod/dnsutils --timeout=120s
```

> **Every later module depends on this setup.** If you open a new terminal or come back later, re-check it before continuing.

### 0.1 Verify the setup

```bash
kubectl config current-context          # confirm you are on the right cluster
kubectl get ns dns-lab                  # namespace exists
kubectl -n dns-lab get pod dnsutils     # STATUS should be Running, READY 1/1
kubectl -n dns-lab exec dnsutils -- cat /etc/resolv.conf   # smoke test
```

---

## Module 1: CoreDNS Anatomy in AKS

### 1.1 What is running

```bash
kubectl -n kube-system get deploy coredns coredns-autoscaler
kubectl -n kube-system get pods -l k8s-app=kube-dns -o wide
kubectl -n kube-system get svc kube-dns
```

**Observe:**
- CoreDNS is a **Deployment** (`coredns`), managed by AKS, not a DaemonSet.
- `coredns-autoscaler` scales replicas based on node/core count.
- Service `kube-dns` has a fixed ClusterIP (commonly `10.0.0.10`, the 10th IP of your service CIDR). This is what pods use as their nameserver.

### 1.2 The Corefile

```bash
kubectl -n kube-system get cm coredns -o jsonpath='{.data.Corefile}'
```

Typical output, annotated:

```
.:53 {
    errors                      # log errors
    ready                       # readiness endpoint
    health                      # liveness endpoint
    kubernetes cluster.local in-addr.arpa ip6.arpa {   # answers cluster.local from the API server
      pods insecure
      fallthrough in-addr.arpa ip6.arpa
    }
    prometheus :9153            # metrics
    forward . /etc/resolv.conf  # everything else goes to the node's resolver
    cache 30                    # cache answers up to 30s
    loop                        # detect forwarding loops
    reload                      # auto reload on config change
    loadbalance
    import custom/*.override
}
import custom/*.server
```

**Key points:**
1. `kubernetes` plugin answers only `cluster.local` (and reverse zones).
2. Everything else is `forward`ed to `/etc/resolv.conf` **of the CoreDNS pod**, which is inherited from the **node**.
3. `coredns` ConfigMap is AKS-managed and gets reconciled. Customization is done through the separate `coredns-custom` ConfigMap (not part of this lab).

```bash
kubectl -n kube-system get cm coredns-custom 2>&1   # usually "not found" if nobody customized
```

### 1.3 Where CoreDNS forwards to

```bash
COREDNS_POD=$(kubectl -n kube-system get pod -l k8s-app=kube-dns -o name | head -1)
kubectl -n kube-system exec $COREDNS_POD -- cat /etc/resolv.conf 2>/dev/null \
  || echo "CoreDNS image is distroless; check the node instead (next step)"
```

CoreDNS images are distroless, so use a node debug pod:

```bash
NODE=$(kubectl get nodes -o name | head -1 | cut -d/ -f2)
kubectl debug node/$NODE -it --image=busybox -- cat /host/run/systemd/resolve/resolv.conf
```

You should see `nameserver 168.63.129.16` (Azure-provided DNS) **unless your VNet has custom DNS servers**, in which case you will see those IPs.

> **Takeaway:** the upstream for non-cluster names is controlled by the **VNet DNS setting**, not by Kubernetes.

```bash
az network vnet show -g <vnet-rg> -n <vnet-name> --query dhcpOptions.dnsServers
```

Empty list = Azure default (168.63.129.16).

---

## Module 2: Pod DNS Config, Search Domains and ndots

```bash
kubectl -n dns-lab exec dnsutils -- cat /etc/resolv.conf
```

Expected:

```
search dns-lab.svc.cluster.local svc.cluster.local cluster.local <azure-internal-suffix>
nameserver 10.0.0.10
options ndots:5
```

### 2.1 Why ndots:5 matters

A name with **fewer than 5 dots** is tried against every search suffix **first**. For `example.com` (1 dot):

1. `example.com.dns-lab.svc.cluster.local` → NXDOMAIN
2. `example.com.svc.cluster.local` → NXDOMAIN
3. `example.com.cluster.local` → NXDOMAIN
4. `example.com.<azure-suffix>` → NXDOMAIN
5. `example.com` → answer

Compare the two forms:

```bash
# Uses search list (several queries behind the scenes)
kubectl -n dns-lab exec dnsutils -- sh -c 'time nslookup example.com >/dev/null'

# Trailing dot = fully qualified, skips search list
kubectl -n dns-lab exec dnsutils -- sh -c 'time nslookup example.com. >/dev/null'
```

---

## Module 3: Record Types in Practice

### 3.1 Deploy sample workloads

```bash
kubectl -n dns-lab create deployment web --image=nginx --replicas=2
kubectl -n dns-lab expose deployment web --port=80                      # ClusterIP service "web"
kubectl -n dns-lab apply -f - <<'EOF'
apiVersion: v1
kind: Service
metadata:
  name: web-headless
spec:
  clusterIP: None
  selector:
    app: web
  ports:
  - name: http
    port: 80
EOF
kubectl -n dns-lab rollout status deploy/web
```

### 3.2 Queries

```bash
# ClusterIP service -> single virtual IP
kubectl -n dns-lab exec dnsutils -- nslookup web
kubectl -n dns-lab exec dnsutils -- nslookup web.dns-lab.svc.cluster.local

# Headless service -> one A record per pod IP
kubectl -n dns-lab exec dnsutils -- nslookup web-headless

# SRV record for named port
kubectl -n dns-lab exec dnsutils -- dig +short SRV _http._tcp.web-headless.dns-lab.svc.cluster.local

# Pod IP style name (dashes)
POD_IP=$(kubectl -n dns-lab get pod -l app=web -o jsonpath='{.items[0].status.podIP}')
kubectl -n dns-lab exec dnsutils -- nslookup ${POD_IP//./-}.dns-lab.pod.cluster.local

# Reverse lookup of the service IP
SVC_IP=$(kubectl -n dns-lab get svc web -o jsonpath='{.spec.clusterIP}')
kubectl -n dns-lab exec dnsutils -- dig +short -x $SVC_IP

# External name (forwarded to Azure DNS)
kubectl -n dns-lab exec dnsutils -- nslookup microsoft.com

# Cross-namespace
kubectl -n dns-lab exec dnsutils -- nslookup kube-dns.kube-system
```

### 3.3 Compare TTLs and caching

```bash
kubectl -n dns-lab exec dnsutils -- dig web.dns-lab.svc.cluster.local | grep -A1 "ANSWER SECTION"
kubectl -n dns-lab exec dnsutils -- dig microsoft.com | grep -A2 "ANSWER SECTION"
```

Run the external one twice a few seconds apart and watch the TTL count down (CoreDNS `cache` plugin).

### 3.4 Failure behavior

```bash
kubectl -n dns-lab exec dnsutils -- nslookup doesnotexist.dns-lab.svc.cluster.local      # NXDOMAIN
kubectl -n dns-lab exec dnsutils -- nslookup web.otherns.svc.cluster.local                # NXDOMAIN (namespace missing)
```

---

## Module 4: Cleanup

```bash
kubectl delete namespace dns-lab
```

This removes the `dnsutils` pod, the `web` deployment and both services. Nothing else was changed in the cluster.

---

## What You Learned

1. CoreDNS in AKS is a managed Deployment; the `kubernetes` plugin serves `cluster.local`, and `forward` sends everything else to the node resolver (Azure DNS 168.63.129.16 or custom VNet DNS).
2. `ndots:5` plus the search list multiplies queries; a trailing dot avoids it.
3. Services, headless services, SRV, pod and reverse records all resolve through CoreDNS, and external names are cached by CoreDNS with a counting-down TTL.
