# Lab 1: Setup and Resource Group

**Time:** 20 min | **Cost:** Free

This is the only lab that sets up tools and creates the project. Labs 2 to 4 reuse it.

## Series map

| Lab | Adds | Files |
|-----|------|-------|
| 1 | Tools, project, Resource Group | `__main__.py` |
| 2 | Storage Account | `storage.py` |
| 3 | VNet, subnets, NSG | `network.py` |
| 4 | Linux VM | `vm.py` |
| 5 | Clean up | none |

## What is Pulumi?

Pulumi is Infrastructure as Code: you write Python describing Azure resources, and Pulumi creates, updates or deletes them.

| Term | Meaning |
|------|---------|
| Project | Folder with your code and `Pulumi.yaml` |
| Stack | One instance of the project (`dev`, `prod`) |
| Resource | A cloud object (Resource Group, VM) |
| State | Pulumi's record of what it created |

## Prerequisites

- Python 3.9+, Azure CLI, an Azure subscription
- Pulumi CLI:

```bash
curl -fsSL https://get.pulumi.com | sh
echo 'export PATH=$PATH:$HOME/.pulumi/bin' >> ~/.bashrc
source ~/.bashrc
pulumi version
```

## Step 1: Log in

```bash
az login
az account show                                   # confirm the subscription
az account set --subscription "<SUBSCRIPTION_ID>" # only if you need to switch

pulumi login --local        # state kept in a local file (or: pulumi login for Pulumi Cloud)
```

## Step 2: Create the project

```bash
mkdir pulumi-azure-labs && cd pulumi-azure-labs
pulumi new azure-python --name pulumi-azure-labs --stack dev
```

When asked for a passphrase, enter any value and remember it. To avoid retyping it:

```bash
export PULUMI_CONFIG_PASSPHRASE="your-passphrase"
```

Set the region:

```bash
pulumi config set azure-native:location centralindia
```

## Step 3: Add the code

Replace the **entire** contents of `__main__.py` (imports included) with:

```python
import pulumi
from pulumi_azure_native import resources

env = pulumi.get_stack()

resource_group = resources.ResourceGroup(
    "rg",
    resource_group_name=f"rg-lab-{env}",
    tags={"environment": env, "managed-by": "pulumi"},
)

pulumi.export("resource_group", resource_group.name)
pulumi.export("location", resource_group.location)
```

- `"rg"` is Pulumi's internal name (used in state).
- `resource_group_name` is the real name in Azure (`rg-lab-dev`).
- The location comes from `azure-native:location`.

> Paste the whole block, not just the `ResourceGroup(...)` part. Without the two import lines you get `NameError: name 'resources' is not defined`.

Later labs add `storage.py`, `network.py` and `vm.py`. You will create each one in its own lab, and in Labs 2 to 4 you add a few lines to this `__main__.py`.

## Step 4: Preview and deploy

```bash
source venv/bin/activate        # Windows: venv\Scripts\activate
pip install -r requirements.txt # only if pulumi new did not do it

pulumi preview
pulumi up
```

Choose **yes**. Expected: 2 resources (stack + resource group).

## Step 5: Verify

```bash
pulumi stack output
az group show --name rg-lab-dev -o table
```

## Exercise: update in place

Add `"owner": "your-name"` to `tags`, then run `pulumi up`. Pulumi updates only the tags.

## Cheat sheet (used throughout the series)

| Command | Purpose |
|---------|---------|
| `pulumi preview` | Show planned changes |
| `pulumi up` | Deploy |
| `pulumi stack output` | Show exports |
| `pulumi config` | View config |
| `pulumi refresh` | Sync state with Azure |
| `pulumi destroy` | Delete all resources (Lab 5) |

## Troubleshooting

| Problem | Fix |
|---------|-----|
| Authentication error | Run `az login` again |
| `ModuleNotFoundError: pulumi_azure_native` | Activate venv, `pip install -r requirements.txt` |
| Wrong subscription | `az account set --subscription <ID>` |
| Group already exists | `pulumi import azure-native:resources:ResourceGroup rg /subscriptions/<SUB_ID>/resourceGroups/rg-lab-dev` |

**Next:** `lab2.md`
