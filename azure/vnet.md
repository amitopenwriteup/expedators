# Lab: Create a Virtual Network, Subnet, and Network Security Group in the Azure Portal

## Objective
Create one virtual network with a subnet, then create a Network Security Group (NSG) and associate it with the subnet, using the Azure Portal (UI).

**Estimated time:** 15 minutes
**Prerequisites:** An active Azure subscription and a web browser.

---

## Background
- A **Virtual Network (VNet)** is a private network in Azure with its own address space (CIDR block).
- A **subnet** is a range of IP addresses inside a VNet where you place resources such as VMs.
- A **Network Security Group (NSG)** is a set of inbound and outbound rules that allow or deny traffic. When associated with a subnet, the rules apply to every resource in that subnet.

---

## Lab Design

| Item | Value |
|------|-------|
| **Resource Group** | `RG-VNet-Demo` |
| **Region** | East US |
| **VNet name** | `VNet-1` |
| **VNet address space** | `10.1.0.0/16` |
| **Subnet name** | `Subnet-1` |
| **Subnet address range** | `10.1.1.0/24` |
| **NSG name** | `NSG-VNet1` |

---

## Part 1: Create VNet-1 with Subnet-1

### Step 1: Start the wizard
1. Sign in to **https://portal.azure.com**.
2. Select **Create a resource** > search for **Virtual Network** > click **Create**.

### Step 2: Basics tab

| Field | Value |
|-------|-------|
| **Subscription** | Select your subscription |
| **Resource group** | Click **Create new** > enter `RG-VNet-Demo` > **OK** |
| **Virtual network name** | `VNet-1` |
| **Region** | East US |

Click **Next**.

### Step 3: Security tab
1. Leave the defaults.
2. Click **Next**.

### Step 4: IP Addresses tab
1. Set the **IPv4 address space** to `10.1.0.0/16`. Delete the default range if it differs.
2. Under **Subnets**, delete the **default** subnet, or click it to edit.
3. Click **+ Add a subnet** (or edit the default) and enter:

| Field | Value |
|-------|-------|
| **Subnet name** | `Subnet-1` |
| **Starting address** | `10.1.1.0` |
| **Subnet size** | `/24` (256 addresses) |

4. Click **Add** (or **Save**).

### Step 5: Review and create
1. Click **Review + create**.
2. Wait for **Validation passed**.
3. Click **Create**.
4. Wait for **Your deployment is complete**.

---

## Part 2: Create the NSG

### Step 1: Create the NSG
1. In the portal search bar, type **Network security groups** and select it.
2. Click **+ Create**.
3. On the **Basics** tab:

| Field | Value |
|-------|-------|
| **Subscription** | Your subscription |
| **Resource group** | `RG-VNet-Demo` |
| **Name** | `NSG-VNet1` |
| **Region** | East US |

4. Click **Review + create** > **Create**.
5. Click **Go to resource** when deployment completes.

### Step 2: Add inbound security rules
1. In `NSG-VNet1`, select **Inbound security rules** (left menu) > **+ Add**.
2. Create the three rules below, clicking **Add** after each.

**Rule 1: Allow SSH**

| Field | Value |
|-------|-------|
| **Source** | Any |
| **Source port ranges** | `*` |
| **Destination** | Any |
| **Service** | Custom |
| **Destination port ranges** | `22` |
| **Protocol** | TCP |
| **Action** | Allow |
| **Priority** | `100` |
| **Name** | `Allow-SSH` |

**Rule 2: Allow HTTP**

| Field | Value |
|-------|-------|
| **Source** | Any |
| **Source port ranges** | `*` |
| **Destination** | Any |
| **Service** | Custom |
| **Destination port ranges** | `80` |
| **Protocol** | TCP |
| **Action** | Allow |
| **Priority** | `110` |
| **Name** | `Allow-HTTP` |

**Rule 3: Allow ICMP (for ping testing)**

| Field | Value |
|-------|-------|
| **Source** | Any |
| **Source port ranges** | `*` |
| **Destination** | Any |
| **Service** | Custom |
| **Destination port ranges** | `*` |
| **Protocol** | ICMP |
| **Action** | Allow |
| **Priority** | `120` |
| **Name** | `Allow-ICMP` |

### Step 3: Associate the NSG with Subnet-1
1. In `NSG-VNet1`, select **Subnets** (left menu) > **+ Associate**.
2. Select **Virtual network**: `VNet-1`.
3. Select **Subnet**: `Subnet-1`.
4. Click **OK**.

---

## Part 3: Verify the Configuration

1. Go to **Resource groups** > `RG-VNet-Demo`. You should see `VNet-1` and `NSG-VNet1`.
2. Open **VNet-1** and confirm:
   - **Overview** shows address space `10.1.0.0/16`.
   - **Subnets** lists `Subnet-1` with range `10.1.1.0/24`.
   - The **Security group** column shows `NSG-VNet1`.
3. Open `NSG-VNet1` > **Inbound security rules** and confirm `Allow-SSH`, `Allow-HTTP`, and `Allow-ICMP` are listed.

> **Note:** Azure reserves 5 IP addresses in every subnet (the first 4 and the last 1), so a `/24` gives **251 usable** addresses.

---

## Verification Checklist

- [ ] Created resource group `RG-VNet-Demo`
- [ ] Created `VNet-1` (`10.1.0.0/16`) with `Subnet-1` (`10.1.1.0/24`)
- [ ] Created `NSG-VNet1` with SSH, HTTP, and ICMP rules
- [ ] Associated `NSG-VNet1` with `Subnet-1`
- [ ] Verified everything in the portal

---

## Troubleshooting

| Issue | Solution |
|-------|----------|
| **Subnet outside VNet range** | The subnet range must fall inside the VNet address space. |
| **NSG not listed when associating** | The NSG and VNet must be in the **same region** (East US). |
| **Priority already in use** | Each rule in an NSG needs a unique priority (100-4096). |
| **Name already exists** | VNet and NSG names must be unique within the resource group. |
| **Create button disabled** | You may lack the *Contributor* or *Network Contributor* role. |

---

## Key Takeaways

- A VNet defines the private IP address space; subnets divide that space into smaller segments.
- An NSG filters traffic using rules evaluated by **priority** (lower number is processed first).
- Associating an NSG with a subnet applies its rules to all resources in that subnet.
- Allowing SSH from **Any** is fine for a lab but should be restricted to specific IPs in production.
- Azure reserves 5 IPs per subnet.

---

## Clean Up (Optional)

Deleting the resource group removes the VNet and the NSG:
1. Open **Resource groups** > `RG-VNet-Demo`.
2. Click **Delete resource group**, type the name to confirm, and click **Delete**.
