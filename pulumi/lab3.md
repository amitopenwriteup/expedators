# Lab 3: Virtual Network

**Cost:** Free

Builds on Labs 1 and 2. Lab 4 will place a VM in the subnet created here.

## Objectives

1. Create a VNet with two subnets
2. Create an NSG (firewall) and attach it to the web subnet

## Architecture

```
rg-lab-dev
 └── VNet vnet-lab-dev   10.0.0.0/16
      ├── snet-web  10.0.1.0/24 ── NSG nsg-web-dev (80, 443, SSH from any source)
      └── snet-app  10.0.2.0/24
```

## Learner approach: build it from the docs

Plan first, then code. Previewing after every step catches mistakes early.

**Docs for this lab:**
- [NetworkSecurityGroup](https://www.pulumi.com/registry/packages/azure-native/api-docs/network/networksecuritygroup/)
- [VirtualNetwork](https://www.pulumi.com/registry/packages/azure-native/api-docs/network/virtualnetwork/)
- [Subnet](https://www.pulumi.com/registry/packages/azure-native/api-docs/network/subnet/)

> **Version check:** the links open the latest Azure Native docs. If your `requirements.txt` pins version 2, use the same page under `azure-native@2.x`, e.g. `https://www.pulumi.com/registry/packages/azure-native@2.x/api-docs/<module>/<resource>/`.

| Step | Do this | Check |
|------|---------|-------|
| 1 | On paper, write the VNet range (`10.0.0.0/16`) and two subnet ranges that do not overlap | Both subnets sit inside the VNet range |
| 2 | Open `VirtualNetwork`; write the minimum and use `AddressSpaceArgs` for the range | Preview shows the VNet |
| 3 | Open `Subnet`; add `snet-web` and pass `vnet.name` | Preview shows the subnet |
| 4 | Open `NetworkSecurityGroup`; add one rule using `SecurityRuleArgs` | You can explain priority and direction |
| 5 | Attach the NSG to the subnet with `NetworkSecurityGroupArgs(id=nsg.id)` | Preview shows the link |
| 6 | Add `snet-app`; read the `depends_on` note in the code | Subnets deploy without an Azure error |
| 7 | Move the code into `network.py` and return the subnet for Lab 4 | `pulumi up` works |

Notice: nested settings (like a rule or an address space) are `...Args` classes. The docs Inputs list names the type for each.

## Step 1: Enable the code

In `__main__.py`, uncomment the **Lab 3** block:

```python
import network
vnet, web_subnet, app_subnet = network.create_network(resource_group)
pulumi.export("vnet_name", vnet.name)
pulumi.export("web_subnet_id", web_subnet.id)
```

Key points in `network.py`:

- NSG rules are checked from the lowest priority number first.
- SSH (port 22) uses `source_address_prefix="*"`, so it is open to any source. This is fine for a short-lived lab, but restrict it in real environments.
- `snet-app` uses `depends_on=[web_subnet]` to avoid Azure errors when changing one VNet in parallel.
- Subnets must not overlap and must sit inside `10.0.0.0/16`.
- `create_network` returns `web_subnet`, which Lab 4 will use.

## Step 2: Preview and deploy

```bash
pulumi preview      # expect: 4 to create (NSG, VNet, 2 subnets)
pulumi up
```

## Step 3: Verify

```bash
az network vnet subnet list -g rg-lab-dev --vnet-name vnet-lab-dev \
  --query "[].{name:name, prefix:addressPrefix}" -o table

az network nsg rule list -g rg-lab-dev --nsg-name nsg-web-dev -o table

az network vnet subnet show -g rg-lab-dev --vnet-name vnet-lab-dev \
  -n snet-web --query networkSecurityGroup.id -o tsv
```

## Exercises

1. Add a third subnet `snet-db` (`10.0.3.0/24`) with an NSG allowing traffic only from `10.0.2.0/24`.
2. Move the VNet CIDR into config: `config.get("vnetCidr") or "10.0.0.0/16"`.
3. Change the SSH rule's `source_address_prefix` from `"*"` to a specific IP (`<your-ip>/32`) and run `pulumi up`. Only the SSH rule changes.

## Troubleshooting

| Problem | Fix |
|---------|-----|
| `AnotherOperationInProgress` | Keep the `depends_on` between subnets |

**Next:** `lab4.md`
