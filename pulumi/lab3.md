# Lab 3: VNet, Subnets and NSGs (Step by Step)

**Cost:** Free

Builds on Labs 1 and 2. Same project, same stack. We build in four small parts, and after each part we run **preview → up → check**. The three exercises (db subnet with NSG, VNet CIDR from config, SSH source from config) are built into the parts.

```
rg-lab-dev
 └── Part A: VNet vnet-lab-dev   10.0.0.0/16  (config: vnetCidr)
      ├── Part B: snet-web  10.0.1.0/24 ── NSG nsg-web-dev (80, 443, SSH from sshSource)
      ├── Part C: snet-app  10.0.2.0/24
      └── Part D: snet-db   10.0.3.0/24 ── NSG nsg-db-dev  (allow only from 10.0.2.0/24, deny rest)
```

| Part | Adds | Preview shows | Check |
|------|------|---------------|-------|
| A | VNet (CIDR from config) | 1 to create | `az network vnet show` |
| B | Web NSG + `snet-web` | 2 to create | `az network nsg rule list`, subnet NSG id |
| C | `snet-app` | 1 to create | `az network vnet subnet list` |
| D | DB NSG + `snet-db` | 2 to create | db NSG rules, subnet NSG id |

Total: 6 resources (VNet, 2 NSGs, 3 subnets).

---

## Part 0: One-time setup

### Optional config

Both values have defaults in code, so nothing is required. To override:

```bash
pulumi config set vnetCidr 10.0.0.0/16     # optional, default is 10.0.0.0/16
pulumi config set sshSource "*"            # optional, default is "*" (any source)
```

### Create `network.py`

Create `network.py` next to `__main__.py` with only this to start:

```python
import pulumi
import pulumi_azure_native as azure_native


def create_network(resource_group):
    config = pulumi.Config()

    # ---- Part A goes here ----

    # ---- Part B goes here ----

    # ---- Part C goes here ----

    # ---- Part D goes here ----
```

### Call it from `__main__.py`

Add this block at the bottom (after the Lab 2 storage call). For now it only calls the function:

```python
import network
network.create_network(resource_group)
```

In Part D you will replace it with the version that unpacks the return values and exports them.

---

## Part A: Virtual Network

**Docs for this part:**

| Code | Pulumi reference |
|------|------------------|
| `VirtualNetwork(...)`, `virtual_network_name`, `address_space` | [VirtualNetwork API docs](https://www.pulumi.com/registry/packages/azure-native/api-docs/network/virtualnetwork/) (see `AddressSpaceArgs`) |
| `pulumi.Config()`, `config.get("vnetCidr")` | [Pulumi configuration](https://www.pulumi.com/docs/iac/concepts/config/) |
| `resource_group.name` passed as an input | [Inputs & Outputs](https://www.pulumi.com/docs/iac/concepts/inputs-outputs/) |

### A1. Code

Replace `# ---- Part A goes here ----` with:

```python
    # Exercise 2: VNet CIDR from config, with a default
    vnet_cidr = config.get("vnetCidr") or "10.0.0.0/16"

    web_cidr = "10.0.1.0/24"
    app_cidr = "10.0.2.0/24"
    db_cidr = "10.0.3.0/24"

    vnet = azure_native.network.VirtualNetwork(
        "vnet",
        virtual_network_name="vnet-lab-dev",
        resource_group_name=resource_group.name,
        address_space=azure_native.network.AddressSpaceArgs(
            address_prefixes=[vnet_cidr],
        ),
    )
```

### A2. Preview

```bash
pulumi preview
```

Expect: **1 to create** (the VNet).

### A3. Up

```bash
pulumi up
```

Choose `yes`.

### A4. Check

```bash
az network vnet show -g rg-lab-dev -n vnet-lab-dev \
  --query "{name:name, space:addressSpace.addressPrefixes}" -o table
```

Expected: `vnet-lab-dev` with `10.0.0.0/16`.

---

## Part B: Web NSG and `snet-web`

**Docs for this part:**

| Code | Pulumi reference |
|------|------------------|
| `NetworkSecurityGroup(...)`, `security_rules` | [NetworkSecurityGroup API docs](https://www.pulumi.com/registry/packages/azure-native/api-docs/network/networksecuritygroup/) (see `SecurityRuleArgs`) |
| `Subnet(...)`, `address_prefix`, `network_security_group` | [Subnet API docs](https://www.pulumi.com/registry/packages/azure-native/api-docs/network/subnet/) |
| `vnet.name`, `nsg_web.id` passed as inputs | [Inputs & Outputs](https://www.pulumi.com/docs/iac/concepts/inputs-outputs/) |

### B1. Code

Replace `# ---- Part B goes here ----` with:

```python
    # Exercise 3: SSH source from config. "*" = any source (lab default).
    ssh_source = config.get("sshSource") or "*"

    # Lower priority number = evaluated first.
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
```

### B2. Preview

```bash
pulumi preview
```

Expect: **2 to create** (web NSG, `snet-web`); the VNet is unchanged.

### B3. Up

```bash
pulumi up
```

### B4. Check

```bash
az network nsg rule list -g rg-lab-dev --nsg-name nsg-web-dev -o table

az network vnet subnet show -g rg-lab-dev --vnet-name vnet-lab-dev \
  -n snet-web --query networkSecurityGroup.id -o tsv
```

Expected: rules `allow-http` (80), `allow-https` (443), `allow-ssh` (22), and an NSG id for `snet-web`.

---

## Part C: `snet-app`

**Docs for this part:**

| Code | Pulumi reference |
|------|------------------|
| `Subnet(...)` | [Subnet API docs](https://www.pulumi.com/registry/packages/azure-native/api-docs/network/subnet/) |
| `opts=pulumi.ResourceOptions(depends_on=[web_subnet])` | [Resource options: dependsOn](https://www.pulumi.com/docs/iac/concepts/resources/options/dependson/) |

Azure rejects parallel changes to subnets in one VNet (`AnotherOperationInProgress`), so we chain the subnets with `depends_on`: web, then app, then db.

### C1. Code

Replace `# ---- Part C goes here ----` with:

```python
    app_subnet = azure_native.network.Subnet(
        "snet-app",
        subnet_name="snet-app",
        resource_group_name=resource_group.name,
        virtual_network_name=vnet.name,
        address_prefix=app_cidr,
        opts=pulumi.ResourceOptions(depends_on=[web_subnet]),
    )
```

### C2. Preview

```bash
pulumi preview
```

Expect: **1 to create** (`snet-app`).

### C3. Up

```bash
pulumi up
```

### C4. Check

```bash
az network vnet subnet list -g rg-lab-dev --vnet-name vnet-lab-dev \
  --query "[].{name:name, prefix:addressPrefix}" -o table
```

Expected: `snet-web` (`10.0.1.0/24`) and `snet-app` (`10.0.2.0/24`).

---

## Part D: DB NSG and `snet-db` (Exercise 1)

**Docs for this part:**

| Code | Pulumi reference |
|------|------------------|
| `NetworkSecurityGroup(...)`, `SecurityRuleArgs` (`access`, `priority`, `source_address_prefix`) | [NetworkSecurityGroup API docs](https://www.pulumi.com/registry/packages/azure-native/api-docs/network/networksecuritygroup/) |
| `Subnet(...)` with `network_security_group` | [Subnet API docs](https://www.pulumi.com/registry/packages/azure-native/api-docs/network/subnet/) |
| `depends_on=[app_subnet]` | [Resource options: dependsOn](https://www.pulumi.com/docs/iac/concepts/resources/options/dependson/) |
| `pulumi.export(...)` | [Inputs & Outputs](https://www.pulumi.com/docs/iac/concepts/inputs-outputs/) (stack outputs) |

The db NSG allows only the app subnet, then explicitly denies everything else. The explicit deny is needed because Azure's default `AllowVnetInBound` rule (priority 65000) would otherwise let any VNet traffic in.

### D1. Code

Replace `# ---- Part D goes here ----` with:

```python
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

Now update `__main__.py`. `create_network` returns four values, so replace the two-line block from Part 0 with:

```python
import network
vnet, web_subnet, app_subnet, db_subnet = network.create_network(resource_group)
pulumi.export("vnet_name", vnet.name)
pulumi.export("web_subnet_id", web_subnet.id)
pulumi.export("app_subnet_id", app_subnet.id)
pulumi.export("db_subnet_id", db_subnet.id)
```

### D2. Preview

```bash
pulumi preview
```

Expect: **2 to create** (db NSG, `snet-db`) plus the new stack outputs.

### D3. Up

```bash
pulumi up
```

### D4. Check

```bash
az network nsg rule list -g rg-lab-dev --nsg-name nsg-db-dev -o table

az network vnet subnet show -g rg-lab-dev --vnet-name vnet-lab-dev \
  -n snet-db --query networkSecurityGroup.id -o tsv

az network vnet subnet list -g rg-lab-dev --vnet-name vnet-lab-dev \
  --query "[].{name:name, prefix:addressPrefix}" -o table
```

Expected: `allow-from-app` plus `deny-all-inbound` on the db NSG, an NSG id for `snet-db`, and three subnets (`10.0.1.0/24`, `10.0.2.0/24`, `10.0.3.0/24`).

---


**Next:** `lab4.md` (Lab 4 will place a VM in `web_subnet`; the return value has four items, so adjust any code that unpacks it.)
