# Lab 2: Virtual Network with Pulumi (Python)

**Level:** Beginner to Intermediate  |  **Time:** 25 minutes  |  **Cost:** Free (VNets, subnets and NSGs have no charge)

---

## Objectives

By the end of this lab you will:

1. Create a Virtual Network (VNet) with two subnets
2. Create a Network Security Group (NSG) with rules
3. Attach the NSG to a subnet
4. Use Pulumi config for an IP address instead of hard-coding it
5. Verify the network with the Azure CLI

## Architecture

```
Resource Group (rg-lab-vnet-dev)
 └── VNet vnet-lab-dev   10.0.0.0/16
      ├── snet-web  10.0.1.0/24  ── NSG nsg-web (allow 80, 443, SSH from your IP)
      └── snet-app  10.0.2.0/24
```

## Prerequisites

- Python 3.9+, Pulumi CLI, Azure CLI, logged in to both
- Your public IP address (search "what is my IP")

---

## Step 1: Create the project

```bash
mkdir lab-vnet && cd lab-vnet
pulumi new azure-python --name lab-vnet --stack dev
pulumi config set azure-native:location centralindia
pulumi config set myIp 203.0.113.10/32     # replace with YOUR IP and /32
```

## Step 2: Write the code

Replace `__main__.py` with:

```python
import pulumi
from pulumi_azure_native import resources, network

config = pulumi.Config()
my_ip = config.require("myIp")
env = pulumi.get_stack()

# 1. Resource Group
resource_group = resources.ResourceGroup(
    "rg",
    resource_group_name=f"rg-lab-vnet-{env}",
)

# 2. Network Security Group with three inbound rules
nsg = network.NetworkSecurityGroup(
    "nsg-web",
    resource_group_name=resource_group.name,
    location=resource_group.location,
    network_security_group_name=f"nsg-web-{env}",
    security_rules=[
        network.SecurityRuleArgs(
            name="allow-http-https",
            priority=100,
            direction="Inbound",
            access="Allow",
            protocol="Tcp",
            source_port_range="*",
            destination_port_ranges=["80", "443"],
            source_address_prefix="*",
            destination_address_prefix="*",
        ),
        network.SecurityRuleArgs(
            name="allow-ssh-from-my-ip",
            priority=110,
            direction="Inbound",
            access="Allow",
            protocol="Tcp",
            source_port_range="*",
            destination_port_range="22",
            source_address_prefix=my_ip,
            destination_address_prefix="*",
        ),
    ],
)

# 3. Virtual Network
vnet = network.VirtualNetwork(
    "vnet",
    resource_group_name=resource_group.name,
    location=resource_group.location,
    virtual_network_name=f"vnet-lab-{env}",
    address_space=network.AddressSpaceArgs(address_prefixes=["10.0.0.0/16"]),
    tags={"environment": env, "managed-by": "pulumi"},
)

# 4. Subnets (web has the NSG attached)
web_subnet = network.Subnet(
    "snet-web",
    resource_group_name=resource_group.name,
    virtual_network_name=vnet.name,
    subnet_name="snet-web",
    address_prefix="10.0.1.0/24",
    network_security_group=network.NetworkSecurityGroupArgs(id=nsg.id),
)

# depends_on avoids Azure errors from changing one VNet in parallel
app_subnet = network.Subnet(
    "snet-app",
    resource_group_name=resource_group.name,
    virtual_network_name=vnet.name,
    subnet_name="snet-app",
    address_prefix="10.0.2.0/24",
    opts=pulumi.ResourceOptions(depends_on=[web_subnet]),
)

# 5. Outputs
pulumi.export("vnet_name", vnet.name)
pulumi.export("web_subnet_id", web_subnet.id)
pulumi.export("app_subnet_id", app_subnet.id)
```

### What each part does

| Resource | Purpose |
|----------|---------|
| `VirtualNetwork` | A private network. `10.0.0.0/16` gives about 65,000 addresses |
| `Subnet` | A slice of the VNet (`/24` = 256 addresses) |
| `NetworkSecurityGroup` | A firewall: rules are checked from lowest priority number first |
| `config.require("myIp")` | Keeps your IP out of the code |

## Step 3: Preview and deploy

```bash
pulumi preview
pulumi up
```

Expect about 5 resources to be created.

## Step 4: Verify

```bash
# VNet and address space
az network vnet show -g rg-lab-vnet-dev -n vnet-lab-dev \
  --query "{name:name, space:addressSpace.addressPrefixes}" -o json

# Subnets
az network vnet subnet list -g rg-lab-vnet-dev --vnet-name vnet-lab-dev \
  --query "[].{name:name, prefix:addressPrefix}" -o table

# NSG rules
az network nsg rule list -g rg-lab-vnet-dev --nsg-name nsg-web-dev -o table
```

Confirm that `snet-web` shows the NSG:

```bash
az network vnet subnet show -g rg-lab-vnet-dev --vnet-name vnet-lab-dev \
  -n snet-web --query networkSecurityGroup.id -o tsv
```

---

## Exercises

1. **Add a third subnet** `snet-db` (`10.0.3.0/24`) with its own NSG that only allows traffic from `10.0.2.0/24`.
2. **Move the CIDR values into config**: `config.get("vnetCidr") or "10.0.0.0/16"`.
3. **Change `myIp`** with `pulumi config set myIp <new>/32`, run `pulumi up`, and watch only the SSH rule change.
4. **Add a deny-all inbound rule** at priority 4000.

## Clean up

```bash
pulumi destroy
pulumi stack rm dev
```

## Troubleshooting

| Problem | Fix |
|---------|-----|
| `missing required configuration variable 'lab-vnet:myIp'` | Run `pulumi config set myIp <your-ip>/32` |
| `AnotherOperationInProgress` on subnets | Keep the `depends_on` between subnets |
| `InvalidCIDRNotation` | Include the prefix length, such as `/32` |
| Subnets overlap | Each subnet must be a separate range inside the VNet space |

## Key takeaways

- Plan CIDR ranges first: subnets cannot overlap and must sit inside the VNet range.
- NSG rules are evaluated by priority (lower number wins).
- Never open SSH to `*`; restrict it to your own IP.
