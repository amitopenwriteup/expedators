# Lab 6: Stacks and Config

**Cost:** Same as Lab 4 for `dev`. The `prod` stack here has no VM, so it adds almost nothing.

Builds on Labs 1 to 4. One project, two environments: `dev` and `prod`, from the same code with different config.

## Objectives

1. Understand what a stack is and where its config lives
2. Create a second stack (`prod`) and give it its own config
3. Make the code read values from config instead of fixed values
4. Use a config flag to turn a resource on or off per stack
5. Deploy and compare both stacks

## Architecture

```
Project pulumi-azure-labs (same code)
 ├── Stack dev   ->  rg-lab-dev   storage, VNet 10.0.x, VM
 └── Stack prod  ->  rg-lab-prod  storage, VNet 10.1.x, no VM
```

Each stack has its own state and its own config file (`Pulumi.dev.yaml`, `Pulumi.prod.yaml`).

## Prerequisites

- Labs 1 to 4 finished and `dev` deployed. If you already ran Lab 5, run `pulumi up` on `dev` first to bring everything back.
- Passphrase exported so you are not asked every time: `export PULUMI_CONFIG_PASSPHRASE="your-passphrase"`

## Learner approach

Concept docs for this lab:
- [Stacks](https://www.pulumi.com/docs/iac/concepts/stacks/)
- [Configuration](https://www.pulumi.com/docs/iac/concepts/config/)

| Step | Do this | Check |
|------|---------|-------|
| 1 | Run `pulumi stack ls` and read `Pulumi.dev.yaml` | You can see which values are config |
| 2 | List every fixed value in your code that should differ between environments (names, ranges, VM on/off) | You have a short list |
| 3 | Replace each with `cfg.get(...)`, `cfg.require(...)` or `cfg.get_bool(...)` | Code still previews with no changes on `dev` |
| 4 | Create `prod`, run `pulumi preview` **before** setting config | You see the missing-config error and fix it |
| 5 | Set the config, preview, deploy | Both resource groups exist |

## Step 1: Look at the stack you have

```bash
pulumi stack ls
cat Pulumi.dev.yaml
```

Config values you set in earlier labs (`myIp`, `sshPublicKey`, `azure-native:location`) are stored here, per stack.

## Step 2: Update the code

Two files change. `vm.py` and `storage.py` stay as they are.

**`network.py`: four small edits.** Subnet ranges now come from a prefix in config.

1. Add this line right after `env = pulumi.get_stack()`:

   ```python
   prefix = config.get("vnetPrefix") or "10.0"
   ```

2. Change the three address values:

   | Before | After |
   |--------|-------|
   | `address_prefixes=["10.0.0.0/16"]` | `address_prefixes=[f"{prefix}.0.0/16"]` |
   | `address_prefix="10.0.1.0/24"` | `address_prefix=f"{prefix}.1.0/24"` |
   | `address_prefix="10.0.2.0/24"` | `address_prefix=f"{prefix}.2.0/24"` |

**`__main__.py`: replace the whole file.**

```python
import pulumi
from pulumi_azure_native import resources

import storage
import network
import vm

cfg = pulumi.Config()
env = pulumi.get_stack()

# Lab 1: Resource Group
resource_group = resources.ResourceGroup(
    "rg",
    resource_group_name=f"rg-lab-{env}",
    tags={"environment": env, "managed-by": "pulumi"},
)
pulumi.export("resource_group", resource_group.name)
pulumi.export("location", resource_group.location)

# Lab 2: Storage (the name now comes from stack config)
storage.create_storage(resource_group, cfg.require("storageAccountName"))

# Lab 3: Network
vnet, web_subnet, app_subnet = network.create_network(resource_group)
pulumi.export("vnet_name", vnet.name)
pulumi.export("web_subnet_id", web_subnet.id)

# Lab 4: VM, only in stacks where deployVm is true
if cfg.get_bool("deployVm"):
    public_ip, virtual_machine = vm.create_vm(resource_group, web_subnet)
    pulumi.export("public_ip", public_ip.ip_address)
    pulumi.export(
        "ssh_command",
        public_ip.ip_address.apply(lambda ip: f"ssh -i ~/.ssh/lab_vm azureuser@{ip}"),
    )
```

What changed: the storage name and the VM switch now come from config, and everything else is the same as Labs 1 to 4.

## Step 3: Set config on `dev` and check nothing changes

```bash
pulumi stack select dev
pulumi config set storageAccountName $(pulumi stack output storage_account)
pulumi config set deployVm true
pulumi preview
```

Expected: no changes. You moved values into config without touching real resources.

> **Why that first command?** It reuses the name already deployed. If you set a different name, preview would show a **replace** of the storage account. Do not run `up` if you see that.

## Step 4: Create `prod`

```bash
pulumi stack init prod
pulumi preview
```

The preview **fails** with a missing-configuration error. A new stack starts with empty config. Fix it:

```bash
pulumi config set azure-native:location centralindia
pulumi config set storageAccountName amit23owprod     # must be globally unique
pulumi config set myIp 203.0.113.10/32                # your IP, keep /32
pulumi config set vnetPrefix 10.1
pulumi config set deployVm false
pulumi preview
```

Expected: resource group, storage account, container, blob, NSG, VNet and two subnets. No VM, because `deployVm` is false.

```bash
pulumi up
```

## Step 5: Compare and verify

```bash
pulumi stack ls
diff Pulumi.dev.yaml Pulumi.prod.yaml
pulumi stack output --stack dev
pulumi stack output --stack prod
az group list --query "[?starts_with(name,'rg-lab')].name" -o tsv
```

You should see `rg-lab-dev` and `rg-lab-prod`. The resource group name came from `pulumi.get_stack()`, so the code did not need to change.

## Exercises

1. Create a third stack `staging` with its own `vnetPrefix` (`10.2`) and storage name.
2. Add a `costCenter` config value and use it in the tags of the resource group.
3. Run `pulumi config set vmSize Standard_B2s` on `dev` and read the preview. What kind of change is it?
4. Run `pulumi stack select prod` and `pulumi config`. Why can the same command show different values?

## Troubleshooting

| Problem | Fix |
|---------|-----|
| `missing required configuration variable` | Set it on the **current** stack: `pulumi config set <key> <value>` |
| Preview wants to replace the storage account | The name in config differs from the deployed one. Set it back |
| `StorageAccountAlreadyTaken` on prod | Pick a different `storageAccountName` |
| Passphrase prompt on every command | `export PULUMI_CONFIG_PASSPHRASE` with the passphrase you chose in Lab 1 |
| Changes went to the wrong environment | Run `pulumi stack` to see the selected stack before every `up` |

## Key takeaways

- A stack is one deployed copy of the project; its config is separate from the code.
- Put anything that differs between environments into config, and derive names from `pulumi.get_stack()`.
- New stacks start empty. Always set config, then preview, then `up`.

**Next:** `lab7.md`
