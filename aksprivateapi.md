# Lab: Private AKS Cluster (portal create, private DNS, command invoke and jumpbox access)

**Goal:** Create a private AKS cluster from the portal, see how the private API server and private DNS zone work, then reach the cluster in two ways: `az aks command invoke` and a jumpbox VM.

**Prereqs:** Azure subscription with Contributor access, Azure Cloud Shell (Bash).

```
Cloud Shell / laptop --X--> API server (private IP only)

Private cluster:
  API server  <-- private endpoint --> AKS VNet
  Private DNS zone (privatelink.<region>.azmk8s.io) linked to the AKS VNet
  Jumpbox VM in the AKS VNet --> resolves private FQDN --> API server  (works)
```

---

## Part 1: Create the private cluster (Azure portal)

1. Portal → **Kubernetes services** → **Create** → **Kubernetes cluster**.
2. **Basics tab**
   - Cluster preset: **Dev/Test**
   - Resource group: create new `rg-private-aks-lab`
   - Cluster name: `aks-private-lab`
   - Region: **Central India** (or your usual region)
3. **Node pools tab:** keep the defaults, set the node count to 1 or 2.
4. **Networking tab**
   - Network configuration: **Azure CNI Overlay**
   - Tick **Enable private cluster**
   - Private DNS zone: **System assigned** (the default)
   - Leave the rest as default. A new VNet and subnet are created automatically.
5. **Review + create** → **Create**. It takes about 5 to 10 minutes.

Labels can differ slightly between portal versions.

Expected: "Your deployment is complete". On the cluster's **Overview** page, **API server address** shows a name ending in `.privatelink.<region>.azmk8s.io`.

---

## Part 2: Verify it is private

Open Cloud Shell (Bash):

```bash
RG=rg-private-aks-lab
AKS=aks-private-lab

az aks show -g $RG -n $AKS \
  --query "{private:apiServerAccessProfile.enablePrivateCluster, dnsZone:apiServerAccessProfile.privateDnsZone, fqdn:fqdn, privateFqdn:privateFqdn}" -o json
```

Expected:
```json
{
  "dnsZone": "system",
  "fqdn": "aks-private-lab-xxxxxxxx.hcp.centralindia.azmk8s.io",
  "private": true,
  "privateFqdn": "aks-private-lab-xxxxxxxx.<guid>.privatelink.centralindia.azmk8s.io"
}
```

`dnsZone` may show the zone's resource ID instead of `system`.

Now prove that Cloud Shell cannot reach it:

```bash
az aks get-credentials -g $RG -n $AKS --overwrite-existing
kubectl get nodes
```

Expected (one of these):
```
Unable to connect to the server: dial tcp 10.224.x.x:443: i/o timeout
Unable to connect to the server: dial tcp: lookup ... no such host
```

The API server has no public path. Cloud Shell sits outside the VNet.

---

## Part 3: Inspect the private DNS zone and private endpoint

The system-managed zone and endpoint live in the node resource group (the `MC_...` group).

```bash
NRG=$(az aks show -g $RG -n $AKS --query nodeResourceGroup -o tsv)
echo $NRG

ZONE=$(az network private-dns zone list -g $NRG --query "[0].name" -o tsv)
echo $ZONE

az network private-dns record-set a list -g $NRG -z $ZONE -o table
az network private-dns link vnet list -g $NRG -z $ZONE -o table
az network private-endpoint list -g $NRG -o table
```

Expected:
```
MC_rg-private-aks-lab_aks-private-lab_centralindia
<guid>.privatelink.centralindia.azmk8s.io

Name                      ResourceGroup   Ttl   Type   Fqdn / IPs
aks-private-lab-xxxxxxxx  MC_...          30    A      10.224.x.x

Name              ResourceGroup   VirtualNetwork          RegistrationEnabled   State
<vnet>-link       MC_...          aks-vnet-xxxxxxxx       False                 Completed

Name            ResourceGroup   Location       ProvisioningState
kube-apiserver  MC_...          centralindia   Succeeded
```

What this shows:
- The **A record** maps the API server name to a private IP in the AKS subnet.
- The **VNet link** is why anything inside that VNet can resolve the name.
- The **private endpoint** is the actual entry to the control plane.

---

## Part 4: Access option 1, `az aks command invoke`

No network setup needed. Azure runs the command in a temporary pod inside the cluster and returns the output.

```bash
az aks command invoke -g $RG -n $AKS --command "kubectl get nodes"
```

Expected:
```
command started at 2026-11-19 10:00:00+00:00, finished at 2026-11-19 10:00:05+00:00 with exitcode=0
NAME                                STATUS   ROLES    AGE   VERSION
aks-nodepool1-12345678-vmss000000   Ready    <none>   20m   v1.xx.x
```

Deploy something:

```bash
az aks command invoke -g $RG -n $AKS \
  --command "kubectl create deployment hello --image=mcr.microsoft.com/azuredocs/aks-helloworld:v1 --replicas=2"

az aks command invoke -g $RG -n $AKS --command "kubectl get pods -o wide"
```

Expected:
```
deployment.apps/hello created

NAME                     READY   STATUS    RESTARTS   AGE   IP
hello-xxxxxxxxxx-aaaaa   1/1     Running   0          15s   10.244.x.x
hello-xxxxxxxxxx-bbbbb   1/1     Running   0          15s   10.244.x.x
```

Run a manifest file from Cloud Shell:

```bash
cat > hello-svc.yaml <<EOF
apiVersion: v1
kind: Service
metadata: { name: hello }
spec:
  selector: { app: hello }
  ports: [{ port: 80, targetPort: 80 }]
EOF

az aks command invoke -g $RG -n $AKS --command "kubectl apply -f hello-svc.yaml" --file hello-svc.yaml
```

Expected: `service/hello created`.

Notes:
- Your role needs the `runcommand/action` permission (Owner and Contributor have it).
- `kubectl`, `helm` and `kustomize` are available inside the command pod.
- Each call takes a few seconds, and it is not interactive. Good for occasional commands, not for a long working session.

---

## Part 5: Access option 2, jumpbox VM in the AKS VNet

Put a VM in a new subnet of the AKS VNet. The private DNS zone is already linked to this VNet, so the VM resolves the API server with no extra DNS work. Do **Option A (portal)** or **Option B (CLI)**, then run the shared test.

### Option A: Azure portal (UI)

**A1. Find the AKS VNet**

1. Portal → **Resource groups** → open `MC_rg-private-aks-lab_aks-private-lab_centralindia` (the node resource group).
2. Open the **Virtual network** named `aks-vnet-xxxxxxxx`.
3. On **Overview**, note the address space.

Expected: address space `10.224.0.0/12`.

**A2. Add a subnet for the jumpbox**

1. In the VNet → **Settings** → **Subnets** → **+ Subnet**.
2. Name: `jumpbox-subnet`
3. Starting address: `10.225.0.0`, subnet size: `/24`
4. Leave the rest as default → **Save**.

Expected: the Subnets list shows `aks-subnet` (10.224.0.0/16) and `jumpbox-subnet` (10.225.0.0/24).

**A3. Create the VM**

1. Portal → **Virtual machines** → **Create** → **Azure virtual machine**.
2. **Basics tab**
   - Resource group: create new `rg-private-jumpbox`
   - Virtual machine name: `jumpbox`
   - Region: same as the cluster (Central India)
   - Image: **Ubuntu Server 22.04 LTS - x64 Gen2**
   - Size: **Standard_B2s**
   - Authentication type: **SSH public key**
   - Username: `azureuser`
   - SSH public key source: **Generate new key pair**, key pair name `jumpbox_key`
   - Public inbound ports: **Allow selected ports** → **SSH (22)**
3. **Networking tab**
   - Virtual network: select `aks-vnet-xxxxxxxx` (it is listed even though it sits in the `MC_` group, as long as region and subscription match)
   - Subnet: **jumpbox-subnet**
   - Public IP: create new (lab only)
   - NIC network security group: **Basic**
4. **Review + create** → **Create** → **Download private key and create resource**. The browser saves `jumpbox_key.pem`.

Expected: "Your deployment is complete". The VM **Overview** shows a public IP and a private IP such as `10.225.0.4`.

**A4. Verify the placement in the portal**

- AKS VNet → **Connected devices**: shows the jumpbox network interface in `jumpbox-subnet`.
- `MC_...` group → the **Private DNS zone** (`<guid>.privatelink.<region>.azmk8s.io`) → **Virtual network links**: shows a link to `aks-vnet-xxxxxxxx`.

Expected: the VM's NIC is listed, and the link status is **Completed**. This link is why the VM can resolve the API server.

**A5. Connect**

From a terminal on your laptop (replace the path and IP):

```bash
chmod 400 ~/Downloads/jumpbox_key.pem
ssh -i ~/Downloads/jumpbox_key.pem azureuser@<public-ip>
```

Alternative: VM → **Connect** → **Bastion** if you have Azure Bastion. Use it instead of a public IP in real environments.

Continue with "Test from the jumpbox" below.

### Option B: Azure CLI

```bash
VNET=$(az network vnet list -g $NRG --query "[0].name" -o tsv)
az network vnet show -g $NRG -n $VNET --query "{space:addressSpace.addressPrefixes, subnets:subnets[].{name:name, prefix:addressPrefix}}" -o json
```

Expected (Azure CNI Overlay default):
```json
{ "space": ["10.224.0.0/12"], "subnets": [{ "name": "aks-subnet", "prefix": "10.224.0.0/16" }] }
```

Create the subnet and the VM (pick a free range inside `space`; `10.225.0.0/24` fits the default):

```bash
az network vnet subnet create -g $NRG --vnet-name $VNET -n jumpbox-subnet --address-prefixes 10.225.0.0/24

SUBNET_ID=$(az network vnet subnet show -g $NRG --vnet-name $VNET -n jumpbox-subnet --query id -o tsv)

az group create -n rg-private-jumpbox -l centralindia

az vm create -g rg-private-jumpbox -n jumpbox \
  --image Ubuntu2204 --size Standard_B2s \
  --subnet $SUBNET_ID \
  --admin-username azureuser --generate-ssh-keys \
  --public-ip-sku Standard
```

Expected:
```json
{ "powerState": "VM running", "privateIpAddress": "10.225.0.4", "publicIpAddress": "20.x.x.x", ... }
```

Connect:

```bash
ssh azureuser@<publicIpAddress>
```

The public IP is for the lab only. In real environments use Azure Bastion or a VPN and no public IP.

### Test from the jumpbox (both options)

Install the tools on the VM:

```bash
curl -sL https://aka.ms/InstallAzureCLIDeb | sudo bash
sudo az aks install-cli
az login --use-device-code
```

Then run:

```bash
RG=rg-private-aks-lab
AKS=aks-private-lab

PRIVATE_FQDN=$(az aks show -g $RG -n $AKS --query privateFqdn -o tsv)
nslookup $PRIVATE_FQDN

az aks get-credentials -g $RG -n $AKS
kubectl get nodes
kubectl get pods
```

Expected:
```
Name:   aks-private-lab-xxxxxxxx.<guid>.privatelink.centralindia.azmk8s.io
Address: 10.224.x.x

NAME                                STATUS   ROLES    AGE   VERSION
aks-nodepool1-12345678-vmss000000   Ready    <none>   30m   v1.xx.x

NAME                     READY   STATUS    RESTARTS   AGE
hello-xxxxxxxxxx-aaaaa   1/1     Running   0          10m
hello-xxxxxxxxxx-bbbbb   1/1     Running   0          10m
```

The name resolves to a private IP and `kubectl` works from inside the VNet. The same cluster that timed out from Cloud Shell in Part 2 now responds.

---

## Part 6 (reference): Reaching the cluster from a different VNet

If the jumpbox lives in another VNet, two things are needed. Missing the DNS link is the most common failure.

```bash
# 1. Peer both directions (VNET_A = AKS VNet, VNET_B = jumpbox VNet)
az network vnet peering create -g <rg-a> -n a-to-b --vnet-name <vnet-a> --remote-vnet <vnet-b-id> --allow-vnet-access
az network vnet peering create -g <rg-b> -n b-to-a --vnet-name <vnet-b> --remote-vnet <vnet-a-id> --allow-vnet-access

# 2. Link the private DNS zone to the jumpbox VNet
az network private-dns link vnet create -g $NRG -z $ZONE -n jumpbox-link -v <vnet-b-id> -e false
```

| Missing piece | Symptom on the jumpbox |
|---|---|
| No peering | Name resolves, but `kubectl` times out |
| No DNS link | `nslookup` of the private FQDN returns NXDOMAIN |

---

## Part 7: Cleanup

```bash
# Remove the jumpbox and its subnet
az group delete -n rg-private-jumpbox --yes --no-wait
az network vnet subnet delete -g $NRG --vnet-name $VNET -n jumpbox-subnet

# Remove the cluster and everything else from this lab
az group delete -n $RG --yes --no-wait
```

Expected: both commands return immediately. After a few minutes `az group list -o table` no longer shows `rg-private-aks-lab`, `rg-private-jumpbox` or the `MC_...` group. Delete the subnet before the resource group if you keep the cluster, otherwise the group deletion covers it.

### Cleanup from the portal

1. **Resource groups** → `rg-private-jumpbox` → **Delete resource group** → type the name → **Delete**.
2. `MC_...` group → VNet `aks-vnet-xxxxxxxx` → **Subnets** → select `jumpbox-subnet` → **Delete**.
3. **Resource groups** → `rg-private-aks-lab` → **Delete resource group** → type the name → **Delete**. This also removes the `MC_...` group.

Expected: the resource group list no longer shows the three groups after a few minutes.
