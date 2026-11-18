# AKS Networking Lab

Hands-on lab for the *Networking concepts for applications in AKS* session. You run it on the AKS cluster you already deployed.

| **Source concepts** | [Networking concepts for applications in AKS](https://learn.microsoft.com/en-us/azure/aks/concepts-network) |

## What you will learn

1. Find out which network model, data plane and outbound type your cluster uses.
2. See how nodes, pods and Services get IP addresses and connectivity.
3. Expose an app with ClusterIP, public LoadBalancer and internal LoadBalancer Services, and see what Azure creates for you.
4. Check egress (outbound) behavior and where your cluster's public outbound IP comes from.
5. Inspect network security groups (NSGs) and know what you own on a custom subnet.
6. Trace DNS resolution with CoreDNS, and detect LocalDNS.
7. Lock down pod-to-pod traffic with network policies.
8. (Optional) Route HTTP traffic with an Ingress.

## Prerequisites

- An existing AKS cluster (Standard or Automatic) and permission to create namespaces, pods and Services in it.
- Azure CLI 2.80 or later, signed in with `az login`.
- `kubectl` with a working context for the cluster.
- Permission to read resources in the cluster's **node resource group** (Reader is enough for Labs 3 to 5).
- Outbound access from the cluster to pull `busybox`, `nginx` and `curlimages/curl` images.

> **Note:** If your subscription uses Azure Policy that blocks public IPs or public load balancers, the public LoadBalancer step in Lab 3 may be denied. Use the internal load balancer steps instead.

### Set your variables

Bash shown. In PowerShell, replace `export X=...` with `$X = "..."`.

```bash
export RG="<your-aks-resource-group>"
export AKS="<your-aks-cluster-name>"
export LOCATION="<your-region>"          # only needed for the optional Lab 7 fallback

az aks get-credentials --resource-group $RG --name $AKS --overwrite-existing
kubectl get nodes
```

**Checkpoint:** all nodes show `Ready`.

---

## Lab 0: Discover your cluster's network

**Goal:** know what you are working with before you test anything. Your results decide how some later steps behave.

### Step 0.1: Read the network profile

```bash
az aks show -g $RG -n $AKS \
  --query "networkProfile.{plugin:networkPlugin, mode:networkPluginMode, dataplane:networkDataplane, policy:networkPolicy, podCidr:podCidr, serviceCidr:serviceCidr, dnsServiceIp:dnsServiceIp, outboundType:outboundType}" \
  -o yaml
```

Write down your values:

| Field | Meaning | Your value |
|---|---|---|
| `plugin` / `mode` | `azure` + `overlay` means overlay model. `azure` with no mode means flat (Azure CNI Node Subnet or Pod Subnet). | |
| `dataplane` | `cilium` means Azure CNI Powered by Cilium (no `kube-proxy`). `azure` means `kube-proxy` is used. | |
| `policy` | The network policy engine (`cilium`, `azure`, `calico`), or empty if none. Lab 7 needs one. | |
| `podCidr` | Pod address range (overlay only). | |
| `serviceCidr` / `dnsServiceIp` | ClusterIP range and the CoreDNS Service IP. | |
| `outboundType` | How nodes reach the internet (`loadBalancer`, `managedNATGateway`, `userDefinedRouting`, and so on). | |

### Step 0.2: Find the node resource group

```bash
export NODE_RG=$(az aks show -g $RG -n $AKS --query nodeResourceGroup -o tsv)
echo $NODE_RG
az resource list -g $NODE_RG -o table
```

**Checkpoint:** you can see VM Scale Sets, a load balancer, an NSG, and a virtual network (if AKS manages the VNet) in the node resource group.

### Step 0.3: Create a working namespace

```bash
kubectl create namespace netlab
```

---

## Lab 1: Kubernetes networking basics

**Goal:** see how nodes and pods get IPs, and confirm pods can talk to each other.

### Step 1.1: Look at node IPs

```bash
kubectl get nodes -o wide
```

Note the `INTERNAL-IP` column. These are IPs from your VNet subnet.

### Step 1.2: Deploy a sample app

```bash
kubectl -n netlab create deployment web --image=nginx --replicas=3
kubectl -n netlab rollout status deployment/web
kubectl -n netlab get pods -o wide
```

Compare the pod `IP` column with the node `INTERNAL-IP` values.

- **Overlay cluster:** pod IPs come from the pod CIDR (often `10.244.x.x`), a range separate from the node subnet.
- **Flat cluster:** pod IPs come from a subnet in the same virtual network as the nodes.

### Step 1.3: Prove pod-to-pod connectivity

```bash
kubectl -n netlab run client --image=busybox:1.36 --restart=Never -- sleep 3600
kubectl -n netlab wait --for=condition=Ready pod/client --timeout=60s

POD_IP=$(kubectl -n netlab get pod -l app=web -o jsonpath='{.items[0].status.podIP}')
kubectl -n netlab exec client -- wget -qO- --timeout=3 http://$POD_IP | head -n 4
```

**Checkpoint:** you see the start of the nginx welcome page.

### Step 1.4: Check the service routing component

```bash
kubectl -n kube-system get daemonset
```

- If you see a `cilium` DaemonSet and **no** `kube-proxy`, your cluster uses Azure CNI Powered by Cilium.
- If you see `kube-proxy`, Service routing uses `kube-proxy`.

**Question 1:** Does your result match the `dataplane` value you wrote down in Lab 0?

---

## Lab 2: Read your network model

**Goal:** connect what you see to the two models from the session.

### Step 2.1: Compare address spaces

```bash
kubectl get nodes -o custom-columns=NAME:.metadata.name,NODE-IP:.status.addresses[0].address,POD-CIDR:.spec.podCIDR
kubectl -n netlab get pods -o custom-columns=POD:.metadata.name,POD-IP:.status.podIP,NODE:.spec.nodeName
```

### Step 2.2: Find the subnet that nodes use (AKS-managed VNet)

```bash
export VNET=$(az network vnet list -g $NODE_RG --query "[0].name" -o tsv)
az network vnet subnet list -g $NODE_RG --vnet-name $VNET \
  --query "[].{name:name, prefix:addressPrefix}" -o table
```

If this returns nothing, your cluster uses a **bring-your-own VNet**. Find it with:

```bash
az aks show -g $RG -n $AKS \
  --query "agentPoolProfiles[0].vnetSubnetId" -o tsv
```

### Step 2.3: Fill in the table

| Question | Overlay | Flat |
|---|---|---|
| Where do pod IPs come from? | Separate private pod CIDR | Subnet in the node VNet |
| Which model is your cluster? | | |
| Pod IP in the node subnet range? | No | Yes |

**Question 2:** Why does the overlay model give simpler, improved scalability compared with the flat model?

---

## Lab 3: Services and load balancing

**Goal:** expose the app three ways and see what Azure creates for you.

### Step 3.1: ClusterIP Service (internal only)

```bash
kubectl -n netlab expose deployment web --name web-clusterip --port 80
kubectl -n netlab get svc web-clusterip
kubectl -n netlab exec client -- wget -qO- --timeout=3 http://web-clusterip | head -n 4
```

The ClusterIP comes from your `serviceCidr`. It is only reachable inside the cluster.

### Step 3.2: Public LoadBalancer Service

```bash
kubectl -n netlab expose deployment web --name web-lb --type=LoadBalancer --port 80
kubectl -n netlab get svc web-lb --watch
```

Wait until `EXTERNAL-IP` changes from `<pending>` to an address, then press `Ctrl+C`.

```bash
export LB_IP=$(kubectl -n netlab get svc web-lb -o jsonpath='{.status.loadBalancer.ingress[0].ip}')
curl -I http://$LB_IP
```

**Checkpoint:** you receive `HTTP/1.1 200 OK`.

### Step 3.3: See what Azure created

Creating a Kubernetes `LoadBalancer` Service configures the matching Azure load balancer resources.

```bash
az network lb list -g $NODE_RG -o table
az network lb rule list -g $NODE_RG --lb-name kubernetes -o table
```

Look for a rule with frontend port `80`.

Azure also configures the NSG rules for Service traffic on AKS-managed resources:

```bash
export NSG=$(az network nsg list -g $NODE_RG --query "[0].name" -o tsv)
az network nsg rule list -g $NODE_RG --nsg-name $NSG \
  --query "[?destinationPortRange=='80'].{name:name, access:access, source:sourceAddressPrefix, destination:destinationAddressPrefix, port:destinationPortRange}" \
  -o table
```

**Checkpoint:** you find an `Allow` rule for port 80 pointing at your Service's public IP.

> **Custom subnet note:** If your cluster uses a custom subnet with your own NSG, AKS does not modify that NSG. Check the NSG you attached and make sure it allows the traffic.

### Step 3.4: Internal LoadBalancer Service

```bash
cat <<'EOF' | kubectl apply -f -
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
EOF

kubectl -n netlab get svc web-internal --watch
```

When `EXTERNAL-IP` shows a private address from your VNet, press `Ctrl+C` and test it from inside the cluster:

```bash
INT_IP=$(kubectl -n netlab get svc web-internal -o jsonpath='{.status.loadBalancer.ingress[0].ip}')
kubectl -n netlab exec client -- wget -qO- --timeout=3 http://$INT_IP | head -n 4
```

**Question 3:** What is the difference between the `EXTERNAL-IP` of `web-lb` and `web-internal`?

---

## Lab 4: Egress (outbound) traffic

**Goal:** see how pods reach the internet and which public IP the world sees.

### Step 4.1: See the outbound IP from a pod

```bash
kubectl -n netlab run egress-test --image=curlimages/curl --restart=Never --rm -it -- \
  curl -s https://api.ipify.org
echo
```

Note the IP address.

### Step 4.2: Match it to your cluster's outbound path

If `outboundType` is `loadBalancer`:

```bash
az aks show -g $RG -n $AKS \
  --query "networkProfile.loadBalancerProfile.effectiveOutboundIPs[].id" -o tsv

az network public-ip show \
  --ids $(az aks show -g $RG -n $AKS --query "networkProfile.loadBalancerProfile.effectiveOutboundIPs[0].id" -o tsv) \
  --query ipAddress -o tsv
```

**Checkpoint:** this IP equals the one from Step 4.1.

If your `outboundType` is `managedNATGateway` or `userDefinedRouting`, the outbound IP comes from the NAT gateway or your firewall or route setup instead. Find that resource and compare its public IP.

### Step 4.3: Check the default outbound access setting (AKS-managed VNet only)

Since March 31, 2026, AKS no longer supports default outbound access for VMs. New clusters on an AKS-managed VNet place their subnets in private subnets (`defaultOutboundAccess = false`). AKS-managed cluster traffic is not affected because it uses explicitly configured outbound paths. Clusters that bring their own VNet are not affected.

```bash
az network vnet subnet list -g $NODE_RG --vnet-name $VNET -o json \
  | grep -i defaultOutboundAccess
```

If your CLI version does not print the property, skip this step.

### Step 4.4: Discuss, do not change

By default, AKS clusters have unrestricted outbound internet access. AKS dependencies outside the VNet are almost entirely defined by FQDNs. To restrict egress, read [Control egress traffic for cluster nodes in AKS](https://learn.microsoft.com/en-us/azure/aks/limit-egress-traffic). Do not change the outbound type on a shared lab cluster.

**Question 4:** Why must you plan for FQDN-based dependencies before restricting outbound traffic?

---

## Lab 5: Network security groups

**Goal:** know what Azure manages and what you own.

### Step 5.1: List NSGs and where they attach

```bash
az network nsg list -g $NODE_RG -o table

az network vnet subnet list -g $NODE_RG --vnet-name $VNET \
  --query "[].{subnet:name, nsg:networkSecurityGroup.id}" -o table
```

If you use a custom subnet, list the NSG on that subnet instead:

```bash
az network vnet subnet show --ids <your-subnet-id> --query networkSecurityGroup.id -o tsv
```

### Step 5.2: Review all rules on the AKS-managed NSG

```bash
az network nsg rule list -g $NODE_RG --nsg-name $NSG \
  --query "sort_by([],&priority)[].{prio:priority, name:name, access:access, dir:direction, proto:protocol, port:destinationPortRange}" \
  -o table
```

### Step 5.3: Check your own NSG against the required traffic

If you restrict traffic with NSG rules on a custom VNet, make sure your rules allow this traffic.

**Node and pod traffic** (when your rules restrict these CIDR ranges):

| Source | Destination | Protocol | Port | Purpose |
|---|---|---|---|---|
| Node CIDR | Node CIDR | All | All | Node to node |
| Node CIDR | Pod CIDR | All | All | Service traffic routing |
| Pod CIDR | Pod CIDR | All | All | Pod to pod and pod to Service, including DNS |

**API Server VNet Integration** (preconfigured in AKS Automatic, optional in AKS Standard):

| Source | Destination | Protocol | Port | Purpose |
|---|---|---|---|---|
| Cluster subnet | APIServer subnet CIDR | TCP | 443 and 4443 | Nodes to API server |
| Azure Load Balancer | APIServer subnet CIDR | TCP | 9988 | Load balancer to API server |

Check whether API Server VNet Integration is on:

```bash
az aks show -g $RG -n $AKS --query "apiServerAccessProfile.enableVnetIntegration" -o tsv
```

**Question 5:** AKS created rules for your `web-lb` Service automatically. Why might the same Service stop working on a custom subnet with a restrictive NSG you attached?

---

## Lab 6: DNS resolution

**Goal:** trace how a pod resolves names and detect whether LocalDNS is active.

### Step 6.1: Inspect CoreDNS

```bash
kubectl -n kube-system get pods -l k8s-app=kube-dns -o wide
kubectl -n kube-system get svc kube-dns
az aks show -g $RG -n $AKS --query networkProfile.dnsServiceIp -o tsv
```

**Checkpoint:** the `kube-dns` Service `CLUSTER-IP` matches `dnsServiceIp`.

### Step 6.2: Look at a pod's DNS settings

```bash
kubectl -n netlab exec client -- cat /etc/resolv.conf
```

Note the `nameserver` and the `search` domains (`netlab.svc.cluster.local`, `svc.cluster.local`, `cluster.local`).

### Step 6.3: Resolve Service names

Use `busybox:1.28`, because `nslookup` is unreliable in newer BusyBox images.

```bash
# Short name (same namespace)
kubectl -n netlab run dns-test --image=busybox:1.28 --restart=Never --rm -it -- nslookup web-clusterip

# Fully qualified name
kubectl -n netlab run dns-test --image=busybox:1.28 --restart=Never --rm -it -- \
  nslookup web-clusterip.netlab.svc.cluster.local

# External name
kubectl -n netlab run dns-test --image=busybox:1.28 --restart=Never --rm -it -- nslookup learn.microsoft.com
```

**Checkpoint:** the Service names resolve to the `web-clusterip` ClusterIP.

### Step 6.4: Detect LocalDNS

LocalDNS runs a DNS proxy on each node. It is preconfigured in AKS Automatic and optional per node pool in AKS Standard.

```bash
kubectl -n netlab run dns-test --image=busybox:1.28 --restart=Never --rm -it -- nslookup kubernetes.default
```

Read the `Server:` line:

| Server value | Meaning |
|---|---|
| `169.254.10.10` or `169.254.10.11` | LocalDNS is active on that node |
| Your `dnsServiceIp` (for example `10.0.0.10`) | Pods query CoreDNS directly |

### Step 6.5: (Optional, disruptive) Enable LocalDNS on a Standard node pool

> **Warning:** Enabling LocalDNS **reimages every node** in the pool. Do this only on a lab cluster, with disruption budgets in place. Requirements include Kubernetes 1.31 or later, Azure Linux or Ubuntu 22.04 or later, and a VM SKU with at least 4 vCPUs.

Create a `localdnsconfig.json` from the template in [Configure LocalDNS in AKS](https://learn.microsoft.com/en-us/azure/aks/localdns-custom), then run:

```bash
az aks nodepool update -g $RG --cluster-name $AKS --name <nodepool-name> \
  --localdns-config ./localdnsconfig.json
```

Repeat Step 6.4 from a pod on that node pool and confirm the server is `169.254.10.10` or `169.254.10.11`.

**Question 6:** What problem does LocalDNS remove for clusters with heavy DNS traffic, and how reliable is its stale-response caching?

---

## Lab 7: Network policies

**Goal:** go from "all pods can talk to all pods" to a locked-down three-tier app.

### Step 7.0: Check that a policy engine exists

```bash
az aks show -g $RG -n $AKS --query "networkProfile.networkPolicy" -o tsv
```

- If it prints `cilium`, `azure` or `calico`, continue with Step 7.1 on your cluster.
- If it prints nothing, policies are **not enforced** on your cluster. Use the fallback cluster below, then continue.

<details>
<summary>Fallback: create a small policy-enabled lab cluster</summary>

```bash
az group create -n ${RG}-netpol -l $LOCATION

az aks create -g ${RG}-netpol -n aks-netpol \
  --node-count 2 \
  --network-plugin azure --network-plugin-mode overlay \
  --network-dataplane cilium --network-policy cilium \
  --generate-ssh-keys

az aks get-credentials -g ${RG}-netpol -n aks-netpol --context aks-netpol --overwrite-existing
kubectl config current-context
```

The final command should print `aks-netpol`. Remember to delete this cluster in the cleanup section.

</details>

### Step 7.1: Deploy a three-tier app

```bash
kubectl create namespace shop

kubectl -n shop run backend --image=busybox:1.36 --labels=app=backend --port=80 -- \
  sh -c 'echo backend > /tmp/index.html; httpd -f -p 80 -h /tmp'
kubectl -n shop run db --image=busybox:1.36 --labels=app=db --port=80 -- \
  sh -c 'echo db > /tmp/index.html; httpd -f -p 80 -h /tmp'
kubectl -n shop run frontend --image=busybox:1.36 --labels=app=frontend -- sleep 3600

kubectl -n shop expose pod backend --port 80
kubectl -n shop expose pod db --port 80

kubectl -n shop wait --for=condition=Ready pod --all --timeout=90s
```

### Step 7.2: Prove the default is wide open

```bash
kubectl -n shop exec frontend -- wget -qO- --timeout=3 http://backend
kubectl -n shop exec frontend -- wget -qO- --timeout=3 http://db
```

**Checkpoint:** both print their name (`backend`, `db`). By default, all pods can send and receive traffic. The frontend reaching the database directly is the problem we will fix.

### Step 7.3: Deny all ingress traffic in the namespace

```bash
cat <<'EOF' | kubectl apply -f -
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: default-deny-ingress
  namespace: shop
spec:
  podSelector: {}
  policyTypes:
    - Ingress
EOF

kubectl -n shop exec frontend -- wget -qO- --timeout=3 http://backend
```

**Checkpoint:** the request now fails with `download timed out`.

### Step 7.4: Allow frontend to backend only

```bash
cat <<'EOF' | kubectl apply -f -
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: allow-frontend-to-backend
  namespace: shop
spec:
  podSelector:
    matchLabels:
      app: backend
  policyTypes:
    - Ingress
  ingress:
    - from:
        - podSelector:
            matchLabels:
              app: frontend
      ports:
        - protocol: TCP
          port: 80
EOF
```

### Step 7.5: Allow backend to database only

```bash
cat <<'EOF' | kubectl apply -f -
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: allow-backend-to-db
  namespace: shop
spec:
  podSelector:
    matchLabels:
      app: db
  policyTypes:
    - Ingress
  ingress:
    - from:
        - podSelector:
            matchLabels:
              app: backend
      ports:
        - protocol: TCP
          port: 80
EOF
```

### Step 7.6: Verify the final behavior

```bash
echo "--- frontend -> backend (expect: backend)"
kubectl -n shop exec frontend -- wget -qO- --timeout=3 http://backend

echo "--- frontend -> db (expect: timeout)"
kubectl -n shop exec frontend -- wget -qO- --timeout=3 http://db

echo "--- backend -> db (expect: db)"
kubectl -n shop exec backend -- wget -qO- --timeout=3 http://db
```

| Path | Expected result |
|---|---|
| frontend to backend | Allowed (`backend`) |
| frontend to db | **Blocked** (timeout) |
| backend to db | Allowed (`db`) |

> DNS keeps working because these policies only restrict **ingress** to pods in `shop`.

### Step 7.7: Inspect the policies

```bash
kubectl -n shop get networkpolicy
kubectl -n shop describe networkpolicy allow-backend-to-db
```

**Question 7:** Your NSG rules are better for AKS nodes. Why are network policies a better fit for pods?

---

## Lab 8 (optional): Ingress for HTTP routing

**Goal:** route HTTP traffic to a Service with a Layer 7 Ingress, instead of one public IP per Service.

This lab uses the AKS application routing add-on. Check the current [application routing documentation](https://learn.microsoft.com/en-us/azure/aks/app-routing) first, because add-on options can change.

### Step 8.1: Enable the add-on

```bash
az aks approuting enable -g $RG -n $AKS
kubectl get ingressclass
```

**Checkpoint:** you see the class `webapprouting.kubernetes.azure.com`.

### Step 8.2: Create an Ingress for the `web` app

```bash
cat <<'EOF' | kubectl apply -f -
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: web-ingress
  namespace: netlab
spec:
  ingressClassName: webapprouting.kubernetes.azure.com
  rules:
    - host: demo.contoso.test
      http:
        paths:
          - path: /
            pathType: Prefix
            backend:
              service:
                name: web-clusterip
                port:
                  number: 80
EOF

kubectl -n netlab get ingress web-ingress --watch
```

### Step 8.3: Test it with a Host header

```bash
ING_IP=$(kubectl -n netlab get ingress web-ingress -o jsonpath='{.status.loadBalancer.ingress[0].ip}')
curl -I -H "Host: demo.contoso.test" http://$ING_IP
curl -I http://$ING_IP
```

**Checkpoint:** the request with the matching `Host` header returns `200 OK`. The request without it does not reach your app. That is Layer 7 routing at work.

---

## Validation checklist

Tick each item before you finish.

- [ ] I can say whether my cluster uses overlay or flat, and Cilium or `kube-proxy` (Labs 0 to 2).
- [ ] I created ClusterIP, public LoadBalancer and internal LoadBalancer Services (Lab 3).
- [ ] I found the Azure load balancer rule and NSG rule created for my Service (Lab 3).
- [ ] I matched the pod's outbound IP to the cluster's outbound path (Lab 4).
- [ ] I know which NSG applies to my nodes and what I own on a custom subnet (Lab 5).
- [ ] I resolved Service names through CoreDNS and checked for LocalDNS (Lab 6).
- [ ] I blocked frontend to db and allowed backend to db with network policies (Lab 7).

## Knowledge check

<details>
<summary>Show suggested answers</summary>

1. The DaemonSet list should agree with `dataplane`: `cilium` means no `kube-proxy`; `azure` means `kube-proxy` is present.
2. In overlay, pods get addresses from a separate private CIDR, so the node subnet does not need room for every pod. This is simpler and scales better than putting pods in the VNet subnet (flat).
3. `web-lb` gets a public IP reachable from the internet. `web-internal` gets a private IP from your VNet, reachable only from inside the VNet or connected networks.
4. Cluster dependencies outside the VNet are almost entirely defined by FQDNs. If you block them without allowing the required FQDNs, nodes cannot reach what AKS needs.
5. AKS does not apply NSGs to its subnet or modify an NSG you attach to a customer-provided subnet. Your own rules must allow the required node, pod and Service traffic.
6. LocalDNS resolves queries locally on each node, reducing latency and removing `conntrack` table pressure from DNS traffic. Stale-response caching is best effort, so a stale record is not guaranteed.
7. Pods are created and removed dynamically. Network policies select pods by labels, namespace or port, and can be applied automatically as pods appear. NSGs filter at the VM or subnet level.

</details>

## Troubleshooting

| Symptom | Likely cause | What to try |
|---|---|---|
| `EXTERNAL-IP` stays `<pending>` | Quota, policy, or permissions on the node resource group | `kubectl -n netlab describe svc web-lb` and read the Events |
| `curl` to the load balancer IP times out | NSG rule missing on a custom subnet, or pods not ready | Check pods are `Running`; review the NSG on your subnet (Lab 5) |
| Policy has no effect | No network policy engine on the cluster | Check `networkProfile.networkPolicy`; use the Lab 7 fallback cluster |
| Everything is blocked after Step 7.3 | Default deny applied, allow rules not yet applied | Apply Steps 7.4 and 7.5, then re-test |
| `nslookup` fails or looks odd | Newer BusyBox `nslookup` bug | Use `busybox:1.28` as in Lab 6 |
| Image pull errors | Cluster has restricted egress | Allow the registries, or mirror the images |
| Ingress gets no address | Add-on still starting | `kubectl -n app-routing-system get pods` and wait a few minutes |

## Cleanup

```bash
# Lab 8 (only if you enabled it)
kubectl -n netlab delete ingress web-ingress
az aks approuting disable -g $RG -n $AKS

# Labs 1 to 6
kubectl delete namespace netlab

# Lab 7
kubectl delete namespace shop
```

If you created the fallback cluster:

```bash
az group delete -n ${RG}-netpol --yes --no-wait
kubectl config delete-context aks-netpol
```

Deleting the `netlab` namespace also removes the public and internal load balancer frontends that AKS created for it. Confirm in the node resource group:

```bash
az network lb rule list -g $NODE_RG --lb-name kubernetes -o table
```

## References

- [Networking concepts for applications in AKS](https://learn.microsoft.com/en-us/azure/aks/concepts-network)
- [CNI networking in AKS](https://learn.microsoft.com/en-us/azure/aks/concepts-network-cni-overview)
- [Customize cluster egress with outbound types](https://learn.microsoft.com/en-us/azure/aks/egress-outboundtype)
- [Control egress traffic for cluster nodes](https://learn.microsoft.com/en-us/azure/aks/limit-egress-traffic)
- [Configure LocalDNS in AKS](https://learn.microsoft.com/en-us/azure/aks/localdns-custom)
- [Secure traffic between pods using network policies](https://learn.microsoft.com/en-us/azure/aks/use-network-policies)
- [Best practices for network connectivity and security in AKS](https://learn.microsoft.com/en-us/azure/aks/operator-best-practices-network)
