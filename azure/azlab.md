# Azure Portal Lab: Resource Group, Virtual Network, NSGs and Linux VM

This lab builds the same environment as Labs 1 to 4 (the Pulumi labs), but by clicking through the Azure portal at <https://portal.azure.com>.

**Cost:** The network parts are free. The VM costs a few cents per hour while running. Delete everything at the end (Part 8).

## What you will build

```
rg-lab-dev                                  (Central India)
 ├── nsg-web-dev   80, 443, SSH
 ├── nsg-db-dev    allow only 10.0.2.0/24, deny the rest
 ├── VNet vnet-lab-dev   10.0.0.0/16
 │    ├── snet-web  10.0.1.0/24 ── nsg-web-dev   <- VM here
 │    ├── snet-app  10.0.2.0/24
 │    └── snet-db   10.0.3.0/24 ── nsg-db-dev
 ├── Public IP pip-lab-dev   (Standard, Static)
 ├── NIC (created with the VM)
 └── VM vm-lab-dev           (Ubuntu 24.04, Standard_B1s, SSH key only)
```

The NSGs are created first so you can attach them while creating the subnets.

## Portal and Pulumi side by side

| Pulumi lab | Portal equivalent |
|------------|-------------------|
| Resource group (Labs 1 and 2) | Resource groups > Create |
| `NetworkSecurityGroup` + `SecurityRuleArgs` | Network security groups > Inbound security rules |
| `VirtualNetwork` + `Subnet` | Virtual networks > IP addresses tab |
| `NetworkSecurityGroupArgs(id=nsg.id)` | Subnet > Network security group dropdown |
| `PublicIPAddress`, `NetworkInterface`, `VirtualMachine` | One wizard: Virtual machines > Create |
| `config.set sshPublicKey` | SSH public key field in the VM wizard |
| `pulumi destroy` | Delete the resource group |

---

## Part 1: Create the resource group

1. In the search bar at the top, type **Resource groups** and open it.
2. Click **Create**.
3. Fill in:
   - **Subscription:** your subscription
   - **Resource group:** `rg-lab-dev`
   - **Region:** `Central India`
4. Click **Review + create**, then **Create**.

**Check:** `rg-lab-dev` appears in the Resource groups list.

---

## Part 2: Create the web NSG (`nsg-web-dev`)

1. Search for **Network security groups** and click **Create**.
2. Fill in:
   - **Resource group:** `rg-lab-dev`
   - **Name:** `nsg-web-dev`
   - **Region:** `Central India`
3. Click **Review + create**, then **Create**, then **Go to resource**.
4. In the left menu, under **Settings**, open **Inbound security rules**. You will see three default rules (priorities 65000 to 65500). Leave them.
5. Click **Add** and create these three rules. Use the same fields each time:

| Field | Rule 1 | Rule 2 | Rule 3 |
|-------|--------|--------|--------|
| Source | Any | Any | Any |
| Source port ranges | `*` | `*` | `*` |
| Destination | Any | Any | Any |
| Service | Custom | Custom | Custom |
| Destination port ranges | `80` | `443` | `22` |
| Protocol | TCP | TCP | TCP |
| Action | Allow | Allow | Allow |
| Priority | `100` | `110` | `120` |
| Name | `allow-http` | `allow-https` | `allow-ssh` |

**Check:** The rule list shows `allow-http` (100), `allow-https` (110) and `allow-ssh` (120). The lowest priority number is checked first.

> SSH from **Any** is fine for a short-lived lab. Part 7 shows how to restrict it.

---

## Part 3: Create the database NSG (`nsg-db-dev`)

1. Create another Network security group:
   - **Resource group:** `rg-lab-dev`
   - **Name:** `nsg-db-dev`
   - **Region:** `Central India`
2. Open it and go to **Settings > Inbound security rules > Add**.
3. Create two rules:

| Field | Rule 1 | Rule 2 |
|-------|--------|--------|
| Source | IP Addresses | Any |
| Source IP addresses/CIDR ranges | `10.0.2.0/24` | (not shown) |
| Source port ranges | `*` | `*` |
| Destination | Any | Any |
| Service | Custom | Custom |
| Destination port ranges | `*` | `*` |
| Protocol | Any | Any |
| Action | Allow | **Deny** |
| Priority | `100` | `4096` |
| Name | `allow-from-app` | `deny-all-inbound` |

**Why the deny rule?** Azure has a default rule, `AllowVnetInBound` (priority 65000), that allows all traffic inside the VNet. Without an explicit deny at a lower number, the web subnet could still reach the database subnet.

---

## Part 4: Create the virtual network and subnets

1. Search for **Virtual networks** and click **Create**.
2. **Basics** tab:
   - **Resource group:** `rg-lab-dev`
   - **Virtual network name:** `vnet-lab-dev`
   - **Region:** `Central India`
3. Click **Next** twice to reach the **IP addresses** tab (skip **Security**; leave Bastion, Firewall and DDoS off).
4. Set the address space:
   - Edit the default IPv4 address space to `10.0.0.0/16`.
5. Delete the `default` subnet (select it and click **Delete**).
6. Click **Add a subnet** and create three subnets:

| Subnet name | Starting address | Size | Network security group |
|-------------|------------------|------|------------------------|
| `snet-web` | `10.0.1.0` | `/24` | `nsg-web-dev` |
| `snet-app` | `10.0.2.0` | `/24` | None |
| `snet-db` | `10.0.3.0` | `/24` | `nsg-db-dev` |

   Leave **Private subnet** (no default outbound access) unticked so the VM can reach the internet for updates.
7. Click **Review + create**, then **Create**.

**Check:** Open `vnet-lab-dev` > **Subnets**. You should see three subnets, with `nsg-web-dev` on `snet-web` and `nsg-db-dev` on `snet-db`.

If you forgot to attach an NSG: open the subnet, choose the **Network security group** dropdown, select it and click **Save**.

---

## Part 5: Create the Linux VM

The wizard creates the public IP and network interface for you, so this one step covers three Pulumi resources.

1. Search for **Virtual machines**, click **Create > Azure virtual machine**.
2. **Basics** tab:

| Field | Value |
|-------|-------|
| Resource group | `rg-lab-dev` |
| Virtual machine name | `vm-lab-dev` |
| Region | `Central India` |
| Availability options | No infrastructure redundancy required |
| Security type | Standard (if the default Trusted launch is rejected for your size) |
| Image | Ubuntu Server 24.04 LTS - x64 Gen2 |
| Size | `Standard_B1s` (click **See all sizes** if not listed) |
| Authentication type | SSH public key |
| Username | `azureuser` |
| SSH public key source | Generate new key pair |
| Key pair name | `lab_vm` |
| Public inbound ports | **None** |

   Setting inbound ports to **None** is correct: the subnet NSG you built already allows 80, 443 and SSH.

3. **Disks** tab:
   - **OS disk type:** Standard SSD or Standard HDD (cheapest is fine)
   - Keep **Delete with VM** ticked.
4. **Networking** tab:

| Field | Value |
|-------|-------|
| Virtual network | `vnet-lab-dev` |
| Subnet | `snet-web (10.0.1.0/24)` |
| Public IP | Create new: name `pip-lab-dev`, SKU **Standard**, Assignment **Static** |
| NIC network security group | **None** (the subnet NSG protects the VM) |
| Delete public IP and NIC when VM is deleted | Ticked |

5. Leave **Management**, **Monitoring** and **Tags** at their defaults. Optionally untick boot diagnostics to avoid a storage account.
6. Click **Review + create**, wait for **Validation passed**, then click **Create**.
7. A pop-up asks to **Download private key and create resource**. Click it. Save the `lab_vm.pem` file.

Deployment takes 1 to 2 minutes. Click **Go to resource** when it finishes.

> **Optional, install nginx at boot:** before creating, open the **Advanced** tab, find **Custom data**, and paste:
> ```bash
> #!/bin/bash
> apt-get update
> apt-get install -y nginx
> systemctl enable --now nginx
> ```
> Custom data only runs on first boot, so it must be set at creation time.

---

## Part 6: Connect and verify

1. On the VM **Overview** page, copy the **Public IP address**.
2. Protect the key file and connect.

   **Linux, macOS or WSL:**
   ```bash
   mkdir -p ~/.ssh
   mv ~/Downloads/lab_vm.pem ~/.ssh/
   chmod 400 ~/.ssh/lab_vm.pem
   ssh -i ~/.ssh/lab_vm.pem azureuser@<PUBLIC_IP>
   ```

   **Windows PowerShell:**
   ```powershell
   ssh -i $HOME\Downloads\lab_vm.pem azureuser@<PUBLIC_IP>
   ```

3. Type `yes` when asked to trust the host, then inside the VM:
   ```bash
   hostname
   cat /etc/os-release
   exit
   ```
4. If you used the nginx custom data, browse to `http://<PUBLIC_IP>`.

**Portal checks:**

| Where | What to confirm |
|-------|-----------------|
| VM > **Overview** | Status is **Running** |
| VM > **Networking > Network settings** | Effective rules show `allow-http`, `allow-https`, `allow-ssh` from `nsg-web-dev` |
| `vnet-lab-dev` > **Subnets** | Three subnets, correct CIDRs and NSGs |
| `nsg-db-dev` > **Inbound security rules** | `allow-from-app` (100) and `deny-all-inbound` (4096) |
| Resource group > **Overview** | All resources listed in one place |

---

## Part 7: Exercises

### Exercise 1: Restrict SSH to your IP

1. Search "what is my IP" and note your public IP.
2. Open `nsg-web-dev` > **Inbound security rules** > click `allow-ssh`.
3. Change **Source** to **IP Addresses** and enter `<your-ip>/32`.
4. Click **Save**. Only this rule changes.

### Exercise 2: Resize the VM

1. Open the VM > **Availability + scale > Size**.
2. Select `Standard_B2s` and click **Resize**.
3. Azure restarts the VM, so expect a short interruption.

### Exercise 3: Pause charges without deleting

1. On the VM **Overview**, click **Stop**. In the portal this deallocates the VM.
2. Confirm the status shows **Stopped (deallocated)**.
3. Click **Start** to resume.

Compute charges stop while deallocated. The static public IP and the OS disk still cost a small amount.

### Exercise 4: Test the database NSG

1. In `snet-db`, a VM would only accept traffic from `snet-app` (`10.0.2.0/24`).
2. Open `nsg-db-dev` > **Inbound security rules** and read the two rules aloud: what traffic is allowed, what is denied, and why priority 4096 is used.

---

## Part 8: Clean up

Deleting the resource group removes everything inside it, which is the portal equivalent of `pulumi destroy`.

1. Open **Resource groups** > `rg-lab-dev`.
2. Click **Delete resource group**.
3. Type `rg-lab-dev` to confirm and click **Delete**.
4. Also delete `lab_vm.pem` from your machine if you no longer need it.

**Check:** `rg-lab-dev` no longer appears in the list.

---

## Troubleshooting

| Problem | Fix |
|---------|-----|
| `SkuNotAvailable` or size not listed | Pick `Standard_B2s` or another size, or try another region (for example South India) |
| Trusted launch error with `Standard_B1s` | In **Basics**, set **Security type** to Standard |
| vCPU quota error | Request a quota increase at **Subscriptions > Usage + quotas**, or choose a smaller size or region |
| `Permission denied (publickey)` | Use `-i` with the downloaded `.pem`, and `chmod 400` the file on Linux or macOS |
| `UNPROTECTED PRIVATE KEY FILE` | Run `chmod 400 ~/.ssh/lab_vm.pem` |
| SSH times out | Check the VM is running; check `allow-ssh` source (your IP may have changed if you restricted it); check `nsg-web-dev` is on `snet-web` |
| Cannot attach NSG when creating the subnet | Create the NSGs first (Parts 2 and 3), or attach later from the subnet page |
| Subnet range error | Subnets must not overlap and must sit inside `10.0.0.0/16` |
| nginx page does not load | Wait 1 to 2 minutes after boot; custom data must have been set at creation time |
| Unexpected charges | Make sure the VM is stopped (deallocated) or the resource group is deleted |

## Summary

| Part | Result |
|------|--------|
| 1 | Resource group `rg-lab-dev` |
| 2 | `nsg-web-dev` with HTTP, HTTPS and SSH rules |
| 3 | `nsg-db-dev` allowing only the app subnet |
| 4 | VNet with three subnets and NSGs attached |
| 5 | Public IP, NIC and Ubuntu 24.04 VM in `snet-web` |
| 6 | SSH login and portal verification |
| 7 | Exercises: lock down SSH, resize, deallocate |
| 8 | Delete the resource group |
