# Lab 3 (Full): VNet, Subnets, NSGs and Exercises

**Cost:** Free

This is the complete build, including all three exercises:

1. Third subnet `snet-db` (`10.0.3.0/24`) with an NSG that allows traffic only from `10.0.2.0/24`
2. VNet CIDR read from config (`vnetCidr`)
3. SSH source configurable, so you can switch from `*` to `<your-ip>/32` and change only one rule

## Architecture

```
rg-lab-dev
 └── VNet vnet-lab-dev   10.0.0.0/16  (config: vnetCidr)
      ├── snet-web  10.0.1.0/24 ── NSG nsg-web-dev (80, 443, SSH from sshSource)
      ├── snet-app  10.0.2.0/24
      └── snet-db   10.0.3.0/24 ── NSG nsg-db-dev  (allow only from 10.0.2.0/24, deny rest)
```

## Step 1: Config (optional)

Both values have defaults in code, so nothing is required. To override:

```bash
pulumi config set vnetCidr 10.0.0.0/16     # optional, default is 10.0.0.0/16
pulumi config set sshSource "*"            # optional, default is "*" (any source)
```

## Step 2: `network.py`

Create `network.py` next to `__main__.py`:

```python
import pulumi
import pulumi_azure_native as azure_native


def create_network(resource_group):
    config = pulumi.Config()

    # Exercise 2: VNet CIDR from config, with a default
    vnet_cidr = config.get("vnetCidr") or "10.0.0.0/16"

    # Exercise 3: SSH source from config. "*" = any source (lab default).
    # Set to "<your-ip>/32" later to lock SSH down.
    ssh_source = config.get("sshSource") or "*"

    web_cidr = "10.0.1.0/24"
    app_cidr = "10.0.2.0/24"
    db_cidr = "10.0.3.0/24"

    # ---------------------------------------------------------------
    # VNet
    # ---------------------------------------------------------------
    vnet = azure_native.network.VirtualNetwork(
        "vnet",
        virtual_network_name="vnet-lab-dev",
        resource_group_name=resource_group.name,
        address_space=azure_native.network.AddressSpaceArgs(
            address_prefixes=[vnet_cidr],
        ),
    )

    # ---------------------------------------------------------------
    # NSG for the web subnet: HTTP, HTTPS, SSH
    # Lower priority number = evaluated first.
    # ---------------------------------------------------------------
    def inbound_allow(name, priority, port, source):
        return azure_native.network.SecurityRuleArgs(
            name=name,
            priority=priority,
            direction="Inbound",
            access="Allow",
            protocol="Tcp",
            source_port_range="*",
            destination_port_range=port,
            source_address_prefix=source,
            destination_address_prefix="*",
        )

    nsg_web = azure_native.network.NetworkSecurityGroup(
        "nsg-web",
        network_security_group_name="nsg-web-dev",
        resource_group_name=resource_group.name,
        security_rules=[
            inbound_allow("allow-http", 100, "80", "*"),
            inbound_allow("allow-https", 110, "443", "*"),
            inbound_allow("allow-ssh", 120, "22", ssh_source),
        ],
    )

    # ---------------------------------------------------------------
    # Exercise 1: NSG for the db subnet
    # Allow only the app subnet, then explicitly deny everything else.
    # The explicit deny is needed because Azure's default AllowVnetInBound
    # rule (priority 65000) would otherwise let any VNet traffic in.
    # ---------------------------------------------------------------
    nsg_db = azure_native.network.NetworkSecurityGroup(
        "nsg-db",
        network_security_group_name="nsg-db-dev",
        resource_group_name=resource_group.name,
        security_rules=[
            azure_native.network.SecurityRuleArgs(
                name="allow-from-app",
                priority=100,
                direction="Inbound",
                access="Allow",
                protocol="*",
                source_port_range="*",
                destination_port_range="*",
                source_address_prefix=app_cidr,
                destination_address_prefix="*",
            ),
            azure_native.network.SecurityRuleArgs(
                name="deny-all-inbound",
                priority=4096,
                direction="Inbound",
                access="Deny",
                protocol="*",
                source_port_range="*",
                destination_port_range="*",
                source_address_prefix="*",
                destination_address_prefix="*",
            ),
        ],
    )

    # ---------------------------------------------------------------
    # Subnets
    # Azure rejects parallel changes to subnets in one VNet
    # (AnotherOperationInProgress), so chain them with depends_on.
    # ---------------------------------------------------------------
    web_subnet = azure_native.network.Subnet(
        "snet-web",
        subnet_name="snet-web",
        resource_group_name=resource_group.name,
        virtual_network_name=vnet.name,
        address_prefix=web_cidr,
        network_security_group=azure_native.network.NetworkSecurityGroupArgs(
            id=nsg_web.id,
        ),
    )

    app_subnet = azure_native.network.Subnet(
        "snet-app",
        subnet_name="snet-app",
        resource_group_name=resource_group.name,
        virtual_network_name=vnet.name,
        address_prefix=app_cidr,
        opts=pulumi.ResourceOptions(depends_on=[web_subnet]),
    )

    db_subnet = azure_native.network.Subnet(
        "snet-db",
        subnet_name="snet-db",
        resource_group_name=resource_group.name,
        virtual_network_name=vnet.name,
        address_prefix=db_cidr,
        network_security_group=azure_native.network.NetworkSecurityGroupArgs(
            id=nsg_db.id,
        ),
        opts=pulumi.ResourceOptions(depends_on=[app_subnet]),
    )

    return vnet, web_subnet, app_subnet, db_subnet
```

## Step 3: Enable it in `__main__.py`

`create_network` now returns four values, so update the Lab 3 block:

```python
import network
vnet, web_subnet, app_subnet, db_subnet = network.create_network(resource_group)
pulumi.export("vnet_name", vnet.name)
pulumi.export("web_subnet_id", web_subnet.id)
pulumi.export("app_subnet_id", app_subnet.id)
pulumi.export("db_subnet_id", db_subnet.id)
```

## Step 4: Preview and deploy

```bash
pulumi preview      # expect: 6 to create (2 NSGs, VNet, 3 subnets)
pulumi up
```

## Step 5: Verify

```bash
# Subnets and ranges
az network vnet subnet list -g rg-lab-dev --vnet-name vnet-lab-dev \
  --query "[].{name:name, prefix:addressPrefix}" -o table

# Web NSG rules
az network nsg rule list -g rg-lab-dev --nsg-name nsg-web-dev -o table

# DB NSG rules
az network nsg rule list -g rg-lab-dev --nsg-name nsg-db-dev -o table

# NSG attached to each subnet
az network vnet subnet show -g rg-lab-dev --vnet-name vnet-lab-dev \
  -n snet-web --query networkSecurityGroup.id -o tsv
az network vnet subnet show -g rg-lab-dev --vnet-name vnet-lab-dev \
  -n snet-db --query networkSecurityGroup.id -o tsv
```

Expected: three subnets (`10.0.1.0/24`, `10.0.2.0/24`, `10.0.3.0/24`), `allow-from-app` plus `deny-all-inbound` on the db NSG, and an NSG id for `snet-web` and `snet-db`.

## Step 6: Exercise 3, lock SSH to your IP

Find your public IP (search "what is my IP"), then:

```bash
pulumi config set sshSource 203.0.113.10/32     # replace with YOUR IP, keep /32
pulumi preview
pulumi up
```

Preview should show exactly one update: the `allow-ssh` rule inside `nsg-web-dev`. Nothing else changes.

To go back to open SSH:

```bash
pulumi config set sshSource "*"
pulumi up
```

## Step 7: Exercise 2, change the VNet CIDR

The VNet range comes from `vnetCidr`. The three subnets are fixed in code, so any new range must still contain `10.0.1.0/24`, `10.0.2.0/24` and `10.0.3.0/24`.

```bash
pulumi config set vnetCidr 10.0.0.0/8       # valid: contains all three subnets
pulumi preview                               # shows an update to the VNet only
```

An invalid range (for example `192.168.0.0/16`) makes Azure reject the subnets. Revert with:

```bash
pulumi config rm vnetCidr
```

## Troubleshooting

| Problem | Fix |
|---------|-----|
| `AnotherOperationInProgress` | Keep the `depends_on` chain: web, then app, then db |
| `InvalidCIDRNotation` | Include the prefix length, such as `/32` |
| `NetcfgSubnetRangeOutsideVnet` | The subnet is outside `vnetCidr`; widen the VNet range |
| `ValueError: too many values to unpack` | Update `__main__.py` to unpack four values |
| App cannot reach db | Check `allow-from-app` source is `10.0.2.0/24` and the VM is in `snet-app` |

## Cleanup

This lab is free, but Lab 4 adds cost. When finished with all labs:

```bash
pulumi destroy
```

**Next:** `lab4.md` (Lab 4 will place a VM in `web_subnet`; the return value now has four items, so adjust any code that unpacks it.)
