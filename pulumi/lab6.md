# Lab 6: Stacks and Config (Step by Step, Preview Only)

**Cost:** Free. This version never runs `pulumi up`, so nothing new is deployed.

Builds on Labs 1 to 4. One project, two environments: `dev` and `prod`, from the same code with different config. We build in four small parts, and after each part we run **preview → check**. There is **no `pulumi up`** anywhere in this lab: you read the preview to see what *would* happen.

```
Project pulumi-azure-labs (same code)
 ├── Stack dev   ->  rg-lab-dev   storage, VNet 10.0.x, VM
 └── Stack prod  ->  rg-lab-prod  storage, VNet 10.1.x, no VM
```

Each stack has its own state and its own config file (`Pulumi.dev.yaml`, `Pulumi.prod.yaml`).

| Part | Does | Preview shows | Check |
|------|------|---------------|-------|
| A | Look at the stack and its config | (none) | `pulumi stack ls`, `Pulumi.dev.yaml` |
| B | Move fixed values into config | `dev`: no changes | Nothing to create, update or delete |
| C | Create `prod` and set its config | missing-config error, then 10 to create | Preview lists resources, no VM |
| D | Compare the two stacks | (none) | `diff` of the two config files |

---

## Part 0: Prerequisites

- Labs 1 to 4 finished and `dev` deployed. If you already ran Lab 5 (destroy), the `dev` preview will show everything to create; that is expected, but Part B's "no changes" check will not apply until you redeploy `dev`.
- Passphrase exported so you are not asked every time:

```bash
export PULUMI_CONFIG_PASSPHRASE="your-passphrase"
```

- Azure CLI logged in (`az login`), because `pulumi preview` still reads from Azure.

---

## Part A: Understand the stack and its config

**Docs for this part:**

| Topic | Pulumi reference |
|-------|------------------|
| What a stack is | [Stacks](https://www.pulumi.com/docs/iac/concepts/stacks/) |
| Where config lives | [Configuration](https://www.pulumi.com/docs/iac/concepts/config/) |

### A1. Look

```bash
pulumi stack ls
cat Pulumi.dev.yaml
```

Config you set in earlier labs (`sshPublicKey`, `sshSource`, `vnetCidr`, `azure-native:location`) is stored here, per stack.

### A2. List what should differ per environment

Write a short list of fixed values in your code that must differ between `dev` and `prod`:

- Storage account name (must be globally unique)
- VNet address range
- VM on or off

### A3. Check

You can name which values are config (in `Pulumi.dev.yaml`) and which are hard-coded in Python. No preview needed yet.

---

## Part B: Move fixed values into config

**Docs for this part:**

| Code | Pulumi reference |
|------|------------------|
| `cfg.get(...)`, `cfg.require(...)`, `cfg.get_bool(...)` | [Configuration](https://www.pulumi.com/docs/iac/concepts/config/) |
| `pulumi.get_stack()` | [Stacks](https://www.pulumi.com/docs/iac/concepts/stacks/) |

### B1. Code: `network.py`

Subnet ranges now come from a prefix in config. Three small edits. Assuming the full Lab 3 `network.py`:

1. Add this line right after `config = pulumi.Config()`:

   ```python
   prefix = config.get("vnetPrefix") or "10.0"
   ```

2. Change the four address values:

   | Before | After |
   |--------|-------|
   | `vnet_cidr = config.get("vnetCidr") or "10.0.0.0/16"` | `vnet_cidr = config.get("vnetCidr") or f"{prefix}.0.0/16"` |
   | `web_cidr = "10.0.1.0/24"` | `web_cidr = f"{prefix}.1.0/24"` |
   | `app_cidr = "10.0.2.0/24"` | `app_cidr = f"{prefix}.2.0/24"` |
   | `db_cidr = "10.0.3.0/24"` | `db_cidr = f"{prefix}.3.0/24"` |

`vm.py` and `storage.py` stay as they are.

### B2. Code: `__main__.py`

Replace the whole file:

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
vnet, web_subnet, app_subnet, db_subnet = network.create_network(resource_group)
pulumi.export("vnet_name", vnet.name)
pulumi.export("web_subnet_id", web_subnet.id)
pulumi.export("app_subnet_id", app_subnet.id)
pulumi.export("db_subnet_id", db_subnet.id)

# Lab 4: VM, only in stacks where deployVm is true
if cfg.get_bool("deployVm"):
    public_ip, virtual_machine = vm.create_vm(resource_group, web_subnet)
    pulumi.export("public_ip", public_ip.ip_address)
    pulumi.export(
        "ssh_command",
        public_ip.ip_address.apply(lambda ip: f"ssh -i ~/.ssh/lab_vm azureuser@{ip}"),
    )
```

What changed: the storage name and the VM switch now come from config. Everything else is the same as Labs 1 to 4.

### B3. Set config on `dev`

```bash
pulumi stack select dev
pulumi config set storageAccountName $(pulumi stack output storage_account)
pulumi config set deployVm true
```

> **Why that first command?** It reuses the name already deployed. A different name would make the preview show a **replace** of the storage account.

### B4. Preview

```bash
pulumi preview
```

Expected: **no changes**. You moved values into config without touching real resources. If you see a replace or update, stop and compare your config with what is deployed. Since we never run `up` in this lab, nothing is changed either way.

### B5. Check

```bash
pulumi config
```

Expected: `storageAccountName` and `deployVm` listed for `dev`.

---

## Part C: Create `prod` and preview it

### C1. Create the stack

```bash
pulumi stack init prod
pulumi preview
```

The preview **fails** with a missing-configuration error. A new stack starts with empty config.

### C2. Set the config

```bash
pulumi config set azure-native:location centralindia
pulumi config set storageAccountName amit23owprod     # must be globally unique
pulumi config set sshPublicKey "$(cat ~/.ssh/lab_vm.pub)"
pulumi config set sshSource 203.0.113.10/32           # your IP, keep /32
pulumi config set vnetPrefix 10.1
pulumi config set deployVm false
```

### C3. Preview

```bash
pulumi preview
```

Expected: **10 to create**:

- 1 resource group
- 3 storage resources (account, container, blob)
- 6 network resources (VNet, 2 NSGs, 3 subnets)

No VM, public IP or NIC, because `deployVm` is false. Ranges should start with `10.1.`.

> `pulumi preview` does not check whether the storage account name is globally unique. A `StorageAccountAlreadyTaken` error would only appear on `pulumi up`, which this lab skips.

### C4. Check

```bash
pulumi stack ls
pulumi config
pulumi stack select dev
pulumi stack ls
pulumi stack select prod
pulumi stack ls
```

Expected: `dev` and `prod` listed, and `prod` config shows `vnetPrefix 10.1` and `deployVm false`.

Nothing is deployed to `prod`, so `az group list` will **not** show `rg-lab-prod`.

---

## Part D: Compare the two stacks (config only)

### D1. Compare

```bash
diff Pulumi.dev.yaml Pulumi.prod.yaml
pulumi config --stack dev
pulumi config --stack prod
```

### D2. Check

Expected differences: storage name, `vnetPrefix`, `deployVm`, and possibly `sshSource`. The code did not change between environments. The resource group name differs because it comes from `pulumi.get_stack()`.

Preview the other stack without switching:

```bash
pulumi preview --stack dev
pulumi stack ls
pulumi stack select dev
pulumi stack ls
```

---

## Exercises (preview only)

1. Create a third stack `staging` with `vnetPrefix 10.2` and its own storage name; run `pulumi preview` only.
2. Add a `costCenter` config value, use it in the resource group tags, and preview `dev`. What kind of change is it?
3. Run `pulumi config set vmSize Standard_B2s` on `dev` and read the preview. What kind of change is it? (Do not run `up`.)
4. Run `pulumi stack select prod` then `pulumi config`. Why can the same command show different values?

> **Version check:** the links open the latest Pulumi docs. Registry links from Labs 2 to 4 still apply to the resources in this lab.

---

## Troubleshooting

| Problem | Fix |
|---------|-----|
| `missing required configuration variable` | Set it on the **current** stack: `pulumi config set <key> <value>` |
| `dev` preview wants to replace the storage account | The name in config differs from the deployed one. Set it back |
| `ValueError: too many values to unpack` | `create_network` returns four values; unpack `vnet, web_subnet, app_subnet, db_subnet` |
| Passphrase prompt on every command | `export PULUMI_CONFIG_PASSPHRASE` with the passphrase you chose in Lab 1 |
| Previewing the wrong environment | Run `pulumi stack` to see the selected stack before every preview |

## Cleanup

Nothing was deployed to `prod`, so removing it only deletes its empty state and config:

```bash
pulumi stack rm prod
```

## Key takeaways

- A stack is one deployed copy of the project; its config is separate from the code.
- Put anything that differs between environments into config, and derive names from `pulumi.get_stack()`.
- New stacks start empty. Always set config, then preview. Preview shows what *would* change and deploys nothing.

**Next:** `lab7.md`
