# Lab: AKS Networking

A hands-on lab that builds an AKS cluster with **Azure CNI Overlay** and **Cilium network policy**, then explores IP addressing, services, DNS, network policies, egress, ingress and network troubleshooting.

**Level:** Intermediate | **Time:** 2 to 3 hours | **Cost:** Small but real. Delete the resource group at the end.

## What you will learn

- See how node, pod and service IP ranges differ in Azure CNI Overlay
- Expose an app with ClusterIP, public LoadBalancer and internal LoadBalancer services
- Resolve names with cluster DNS, including across namespaces
- Restrict traffic with Kubernetes network policies (ingress and egress)
- Publish an app through the managed application routing (NGINX) ingress
- Diagnose three common network faults

## Prerequisites

- An Azure subscription (Contributor on the subscription or resource group)
- Azure CLI and `kubectl` (or use **Azure Cloud Shell**, which has both)
- Enough vCPU quota for 2 small nodes (about 4 vCPUs)

> Commands use bash syntax. Names and ranges are examples. This lab has not been run against a live subscription, and flags can change between Azure CLI versions, so check `az aks create --help` if a command fails.

---

## Lab setup

### Variables and resource group

```bash
RG=aks-net-lab-rg
LOCATION=eastus
CLUSTER=aks-net-lab

az group create --name $RG --location $LOCATION
```

### Create the cluster

```bash
az aks create \
  --resource-group $RG \
  --name $CLUSTER \
  --node-count 2 \
  --node-vm-size Standard_D2s_v5 \
  --network-plugin azure \
  --network-plugin-mode overlay \
  --pod-cidr 192.168.0.0/16 \
  --service-cidr 172.16.0.0/16 \
  --dns-service-ip 172.16.0.10 \
  --network-dataplane cilium \
  --network-policy cilium \
  --generate-ssh-keys
```

This takes 5 to 10 minutes.

### Connect and create namespaces

```bash
az aks get-credentials --resource-group $RG --name $CLUSTER --overwrite-existing
kubectl get nodes

kubectl create namespace netlab
kubectl create namespace netlab2
kubectl create namespace netlab-tsg
```

**Checkpoint:** Both nodes show `Ready`.

---

## Part 1: Explore IP addressing

### Tasks

1. Look at node IPs:

```bash
kubectl get nodes -o wide
```

2. Look at pod IPs and services across the cluster:

```bash
kubectl get pods -A -o wide
kubectl get svc -A
```

3. View the cluster's network settings:

```bash
az aks show -g $RG -n $CLUSTER --query networkProfile -o jsonc
```

4. Find the virtual network AKS created in the node resource group:

```bash
NODE_RG=$(az aks show -g $RG -n $CLUSTER --query nodeResourceGroup -o tsv)
az network vnet list -g $NODE_RG -o table
```

### What to observe

| Item | Range | Where you see it |
|---|---|---|
| Node IPs | VNet range (for example `10.224.x.x`) | `kubectl get nodes -o wide` |
| Pod IPs | Overlay range `192.168.0.0/16` | `kubectl get pods -A -o wide` |
| Service IPs | Service CIDR `172.16.0.0/16` | `kubectl get svc -A` |
| DNS service IP | `172.16.0.10` | `kube-dns` service in `kube-system` |

Some `kube-system` pods use the node IP because they run with host networking.

### Questions

1. Why do pod IPs not consume addresses from the VNet?
2. Which of the three ranges must not overlap with your other networks?

**Checkpoint:** You can name the node, pod and service ranges and where each is used.

---

## Part 2: Services and load balancers

### Create a test app and a client pod

```bash
kubectl create deployment web --image=nginx:1.27 --replicas=2 -n netlab
kubectl run client --image=busybox:1.36 --restart=Never --labels=role=client -n netlab -- sleep 3600
kubectl wait --for=condition=Ready pod/client -n netlab --timeout=90s
kubectl get pods -n netlab -o wide
```

### 2.1 ClusterIP (internal only)

```bash
kubectl expose deployment web --port=80 --name=web-clusterip -n netlab
kubectl get svc web-clusterip -n netlab
kubectl exec client -n netlab -- wget -qO- -T 5 http://web-clusterip
```

Expected: the nginx welcome page HTML. The service IP is in `172.16.x.x`.

### 2.2 Public LoadBalancer

```bash
kubectl expose deployment web --port=80 --type=LoadBalancer --name=web-public -n netlab
kubectl get svc web-public -n netlab -w
```

When `EXTERNAL-IP` appears (press Ctrl+C), test it:

```bash
PUBLIC_IP=$(kubectl get svc web-public -n netlab -o jsonpath='{.status.loadBalancer.ingress[0].ip}')
curl -s http://$PUBLIC_IP | head -5
```

### 2.3 Internal LoadBalancer

Save as `web-internal.yaml`:

```yaml
apiVersion: v1
kind: Service
metadata:
  name: web-internal
  namespace: netlab
  annotations:
    service.beta.kubernetes.io/azure-load-balancer-internal: "true"
spec:
  type: LoadBalancer
  selector:
    app: web
  ports:
  - port: 80
    targetPort: 80
```

```bash
kubectl apply -f web-internal.yaml
kubectl get svc web-internal -n netlab -w
INTERNAL_IP=$(kubectl get svc web-internal -n netlab -o jsonpath='{.status.loadBalancer.ingress[0].ip}')
echo $INTERNAL_IP
kubectl exec client -n netlab -- wget -qO- -T 5 http://$INTERNAL_IP
```

The internal IP is a private address from the VNet. It works from inside the network but not from the internet.

### 2.4 See the Azure side

```bash
az network lb list -g $NODE_RG -o table
az network lb frontend-ip list -g $NODE_RG --lb-name kubernetes -o table
az network lb frontend-ip list -g $NODE_RG --lb-name kubernetes-internal -o table
```

**Checkpoint:** You reached the app through ClusterIP, the public IP and the internal IP, and you found each frontend IP in Azure.

### Questions

1. Which service type would you use for a private API reachable only from your corporate network?
2. What does the Azure load balancer named `kubernetes` hold?

---

## Part 3: DNS

### Tasks

```bash
kubectl exec client -n netlab -- cat /etc/resolv.conf
kubectl exec client -n netlab -- nslookup web-clusterip
kubectl exec client -n netlab -- nslookup web-clusterip.netlab.svc.cluster.local
```

Create a service in another namespace and reach it by name:

```bash
kubectl create deployment web2 --image=nginx:1.27 -n netlab2
kubectl expose deployment web2 --port=80 -n netlab2
kubectl exec client -n netlab -- nslookup web2.netlab2
kubectl exec client -n netlab -- wget -qO- -T 5 http://web2.netlab2 | head -3
```

Check CoreDNS:

```bash
kubectl get pods -n kube-system -l k8s-app=kube-dns
kubectl get svc kube-dns -n kube-system
```

### What to observe

- `resolv.conf` uses nameserver `172.16.0.10` and a search path starting with `netlab.svc.cluster.local`.
- A short name (`web-clusterip`) works inside the same namespace.
- Another namespace needs `<service>.<namespace>`.

**Checkpoint:** You resolved a service in the same namespace and one in another namespace.

---

## Part 4: Network policies (ingress)

By default every pod can talk to every other pod. You will lock this down.

### 4.1 Confirm everything is open

```bash
kubectl run other --image=busybox:1.36 --restart=Never -n netlab2 -- sleep 3600
kubectl wait --for=condition=Ready pod/other -n netlab2 --timeout=90s
kubectl exec other -n netlab2 -- wget -qO- -T 5 http://web-clusterip.netlab | head -3
```

It works.

### 4.2 Default deny ingress

Save as `default-deny-ingress.yaml`:

```yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: default-deny-ingress
  namespace: netlab
spec:
  podSelector: {}
  policyTypes:
  - Ingress
```

```bash
kubectl apply -f default-deny-ingress.yaml
kubectl exec client -n netlab -- wget -qO- -T 3 http://web-clusterip
kubectl exec other -n netlab2 -- wget -qO- -T 3 http://web-clusterip.netlab
```

Expected: both requests time out (`download timed out`).

### 4.3 Allow only the client pod

Save as `allow-client-to-web.yaml`:

```yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: allow-client-to-web
  namespace: netlab
spec:
  podSelector:
    matchLabels:
      app: web
  policyTypes:
  - Ingress
  ingress:
  - from:
    - podSelector:
        matchLabels:
          role: client
    ports:
    - protocol: TCP
      port: 80
```

```bash
kubectl apply -f allow-client-to-web.yaml
kubectl exec client -n netlab -- wget -qO- -T 5 http://web-clusterip | head -3
kubectl exec other -n netlab2 -- wget -qO- -T 3 http://web-clusterip.netlab
```

Expected: the client works; the pod in `netlab2` still times out.

### 4.4 Effect on the public load balancer

```bash
curl -s -m 5 http://$PUBLIC_IP | head -3
```

The public IP may stop responding because ingress to the `web` pods is now restricted. In real designs, allow external traffic explicitly, for example:

```yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: allow-internet-to-web
  namespace: netlab
spec:
  podSelector:
    matchLabels:
      app: web
  policyTypes:
  - Ingress
  ingress:
  - from:
    - ipBlock:
        cidr: 0.0.0.0/0
    ports:
    - protocol: TCP
      port: 80
```

Apply it only if you want to restore public access. Behaviour can depend on the traffic path, so test it and record what you see.

**Checkpoint:** You demonstrated allowed and blocked traffic using labels.

### Questions

1. Why does the policy select pods with `app: web` and the source with `role: client`?
2. What happens when a pod is selected by no policy?

---

## Part 5: Egress and outbound traffic

### 5.1 Find the cluster outbound IP

```bash
OUT_ID=$(az aks show -g $RG -n $CLUSTER --query "networkProfile.loadBalancerProfile.effectiveOutboundIPs[0].id" -o tsv)
az network public-ip show --ids $OUT_ID --query ipAddress -o tsv

kubectl run egress-check --rm -it --image=curlimages/curl --restart=Never -n netlab -- curl -s https://api.ipify.org
```

The two IP addresses should match. All pods share this outbound address by default.

### 5.2 Restrict egress to DNS only

Save as `client-egress-dns-only.yaml`:

```yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: client-egress-dns-only
  namespace: netlab
spec:
  podSelector:
    matchLabels:
      role: client
  policyTypes:
  - Egress
  egress:
  - to:
    - namespaceSelector:
        matchLabels:
          kubernetes.io/metadata.name: kube-system
      podSelector:
        matchLabels:
          k8s-app: kube-dns
    ports:
    - protocol: UDP
      port: 53
    - protocol: TCP
      port: 53
```

```bash
kubectl apply -f client-egress-dns-only.yaml
kubectl exec client -n netlab -- nslookup web-clusterip
kubectl exec client -n netlab -- wget -qO- -T 3 http://web-clusterip
kubectl exec client -n netlab -- wget -qO- -T 3 http://example.com
```

Expected: the lookup works, but both `wget` requests time out.

### 5.3 Allow egress to the web pods only

Save as `client-egress-web.yaml`:

```yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: client-egress-web
  namespace: netlab
spec:
  podSelector:
    matchLabels:
      role: client
  policyTypes:
  - Egress
  egress:
  - to:
    - podSelector:
        matchLabels:
          app: web
    ports:
    - protocol: TCP
      port: 80
```

```bash
kubectl apply -f client-egress-web.yaml
kubectl exec client -n netlab -- wget -qO- -T 5 http://web-clusterip | head -3
kubectl exec client -n netlab -- wget -qO- -T 3 http://example.com
```

Expected: the internal request works; the internet request is still blocked.

**Checkpoint:** You can allow DNS and one internal destination while blocking everything else.

### Think about it

- How would you give your cluster a fixed outbound IP? (a NAT Gateway or Azure Firewall with user defined routes)
- Which tool would you use to restrict outbound traffic by domain name? (Azure Firewall, or an egress gateway)

---

## Part 6: Ingress with application routing

### 6.1 Enable the managed ingress

```bash
az aks approuting enable --resource-group $RG --name $CLUSTER
kubectl get pods -n app-routing-system
kubectl get svc -n app-routing-system
```

Wait until the `nginx` service has an `EXTERNAL-IP`.

### 6.2 Create an Ingress

Save as `web-ingress.yaml`:

```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: web-ingress
  namespace: netlab
spec:
  ingressClassName: webapprouting.kubernetes.azure.com
  rules:
  - host: web.lab.local
    http:
      paths:
      - path: /
        pathType: Prefix
        backend:
          service:
            name: web-clusterip
            port:
              number: 80
```

```bash
kubectl apply -f web-ingress.yaml
ING_IP=$(kubectl get svc nginx -n app-routing-system -o jsonpath='{.status.loadBalancer.ingress[0].ip}')
echo $ING_IP
curl -s -m 10 -H "Host: web.lab.local" http://$ING_IP | head -5
```

### 6.3 Why does it fail?

Your default-deny policy from Part 4 still applies, so the ingress controller pods in `app-routing-system` cannot reach the `web` pods. You will probably see a `504` or a timeout.

Confirm:

```bash
kubectl describe ingress web-ingress -n netlab
kubectl get networkpolicy -n netlab
```

### 6.4 Fix it with a policy

Save as `allow-ingress-controller.yaml`:

```yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: allow-ingress-controller
  namespace: netlab
spec:
  podSelector:
    matchLabels:
      app: web
  policyTypes:
  - Ingress
  ingress:
  - from:
    - namespaceSelector:
        matchLabels:
          kubernetes.io/metadata.name: app-routing-system
    ports:
    - protocol: TCP
      port: 80
```

```bash
kubectl apply -f allow-ingress-controller.yaml
curl -s -m 10 -H "Host: web.lab.local" http://$ING_IP | head -5
```

**Checkpoint:** The nginx page is returned through the ingress using the `Host` header.

### Questions

1. Why use one ingress public IP for many apps instead of one load balancer per service?
2. How would you add HTTPS? (a TLS secret or certificate from Key Vault, and a `tls` section in the Ingress)

---

## Part 7: Troubleshooting scenarios

Use the `netlab-tsg` namespace, which has no network policies. For each scenario, set it up, find the cause, fix it, and verify.

First, a test pod:

```bash
kubectl run tester --image=busybox:1.36 --restart=Never -n netlab-tsg -- sleep 3600
kubectl wait --for=condition=Ready pod/tester -n netlab-tsg --timeout=90s
```

### Scenario A: Service has no endpoints

```bash
kubectl create deployment shop --image=nginx:1.27 -n netlab-tsg
kubectl create service clusterip shop --tcp=80:80 -n netlab-tsg
kubectl patch svc shop -n netlab-tsg -p '{"spec":{"selector":{"app":"shop-typo"}}}'
kubectl exec tester -n netlab-tsg -- wget -qO- -T 3 http://shop
```

**Find it:**

```bash
kubectl get endpoints shop -n netlab-tsg
kubectl get svc shop -n netlab-tsg -o wide
kubectl get pods -n netlab-tsg --show-labels
```

**Solution:**

```bash
kubectl patch svc shop -n netlab-tsg -p '{"spec":{"selector":{"app":"shop"}}}'
kubectl exec tester -n netlab-tsg -- wget -qO- -T 3 http://shop | head -3
```

### Scenario B: Wrong target port

```bash
kubectl create deployment api --image=nginx:1.27 -n netlab-tsg
kubectl create service clusterip api --tcp=80:8080 -n netlab-tsg
kubectl exec tester -n netlab-tsg -- wget -qO- -T 3 http://api
```

**Find it:**

```bash
kubectl get endpoints api -n netlab-tsg
kubectl describe svc api -n netlab-tsg
```

Endpoints exist, but they use port `8080`, where nginx does not listen. The request is refused or times out.

**Solution:**

```bash
kubectl patch svc api -n netlab-tsg --type=json \
  -p '[{"op":"replace","path":"/spec/ports/0/targetPort","value":80}]'
kubectl exec tester -n netlab-tsg -- wget -qO- -T 3 http://api | head -3
```

### Scenario C: Egress policy blocks DNS

```bash
kubectl run locked --image=busybox:1.36 --restart=Never --labels=role=locked -n netlab-tsg -- sleep 3600
kubectl wait --for=condition=Ready pod/locked -n netlab-tsg --timeout=90s

cat <<'EOF' | kubectl apply -f -
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: locked-deny-egress
  namespace: netlab-tsg
spec:
  podSelector:
    matchLabels:
      role: locked
  policyTypes:
  - Egress
EOF

kubectl exec locked -n netlab-tsg -- nslookup shop
```

Expected: the lookup times out.

**Find it:**

```bash
kubectl get networkpolicy -n netlab-tsg
kubectl describe networkpolicy locked-deny-egress -n netlab-tsg
kubectl exec tester -n netlab-tsg -- nslookup shop
```

A normal pod resolves the name, so DNS works. The `locked` pod is blocked because the egress policy has no rule that allows port 53.

**Solution:**

```bash
cat <<'EOF' | kubectl apply -f -
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: locked-deny-egress
  namespace: netlab-tsg
spec:
  podSelector:
    matchLabels:
      role: locked
  policyTypes:
  - Egress
  egress:
  - to:
    - namespaceSelector:
        matchLabels:
          kubernetes.io/metadata.name: kube-system
      podSelector:
        matchLabels:
          k8s-app: kube-dns
    ports:
    - protocol: UDP
      port: 53
    - protocol: TCP
      port: 53
EOF
kubectl exec locked -n netlab-tsg -- nslookup shop
```

**Checkpoint:** All three scenarios are fixed and you can explain each root cause.

---

## Network troubleshooting flow

1. **Does the name resolve?** `nslookup <service>` from a pod. If not: CoreDNS, egress policy on DNS.
2. **Does the service have endpoints?** `kubectl get endpoints <svc>`. If none: selector or pod readiness.
3. **Are the ports right?** Compare `port`, `targetPort` and the container port.
4. **Is a network policy in the way?** `kubectl get networkpolicy -A`, and test from a pod in a namespace with no policy.
5. **Is the Azure layer healthy?** Load balancer rules, NSGs, routes, firewall rules.

---

## Knowledge check

1. In Azure CNI Overlay, which addresses come from the VNet and which from the overlay range?
2. What is the difference between `ClusterIP`, `LoadBalancer` and `Ingress`?
3. Why does a default-deny ingress policy also break your ingress controller?
4. A pod resolves names in its own namespace but not in another. What would you check first?
5. Why must egress policies usually include a rule for DNS?
6. A Service has endpoints but connections fail. Name two causes.

**Suggested answers**

1. Nodes (and internal load balancer frontends) use the VNet; pods use the overlay range.
2. ClusterIP is internal only; LoadBalancer creates an Azure load balancer frontend; Ingress routes HTTP or HTTPS by host and path through one entry point.
3. The controller pods are in another namespace and are treated like any other source, so they must be allowed explicitly.
4. Use the full form `<service>.<namespace>`; then check the search path and any network policy.
5. Without it, the pod cannot resolve service names.
6. Wrong `targetPort`, application not listening, network policy blocking, readiness issues.

---

## Cleanup

```bash
az group delete --name $RG --yes --no-wait
```

Also remove local files if you wish: `rm -f *.yaml` in your working folder.

## Completion checklist

- [ ] Cluster created with Azure CNI Overlay and Cilium
- [ ] Node, pod and service ranges identified
- [ ] ClusterIP, public and internal LoadBalancer tested
- [ ] DNS resolved within and across namespaces
- [ ] Ingress policy allowed only the client pod
- [ ] Egress limited to DNS and one internal target
- [ ] Ingress reached through the application routing add-on
- [ ] Three fault scenarios diagnosed and fixed
- [ ] Resource group deleted

## Command cheat sheet

| Task | Command |
|---|---|
| Node and pod IPs | `kubectl get nodes -o wide`, `kubectl get pods -A -o wide` |
| Service endpoints | `kubectl get endpoints <svc> -n <ns>` |
| Network settings | `az aks show -g $RG -n $CLUSTER --query networkProfile` |
| DNS test | `kubectl exec <pod> -- nslookup <name>` |
| List policies | `kubectl get networkpolicy -A` |
| Azure LB frontends | `az network lb frontend-ip list -g $NODE_RG --lb-name kubernetes -o table` |
| Ingress IP | `kubectl get svc nginx -n app-routing-system` |

## Notes

- Behaviour of load balancer traffic under network policies can vary by traffic path and version. Record what you see and compare it with the current AKS networking documentation.
- Running clusters, node VMs, load balancers and public IPs incur charges. Always finish with the cleanup step.
