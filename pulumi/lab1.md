# Pulumi IaC for Azure: Create a Resource Group (Python)

A beginner-friendly guide, starting from the basics.

---

## 1. What is Pulumi?

Pulumi is an **Infrastructure as Code (IaC)** tool. Instead of clicking in the Azure Portal, you write real Python code that describes the infrastructure you want. Pulumi then creates, updates, or deletes it for you.

**Key terms:**

| Term | Meaning |
|------|---------|
| **Project** | A folder with your Pulumi code and `Pulumi.yaml` |
| **Stack** | An instance of your project (e.g., `dev`, `prod`) |
| **Resource** | A cloud object (e.g., Resource Group, VM) |
| **State** | Pulumi's record of what it has created |

---

## 2. Prerequisites

Install these first:

1. **Python 3.9+**: https://www.python.org/downloads/
2. **Pulumi CLI**: https://www.pulumi.com/docs/install/
3. **Azure CLI**: https://learn.microsoft.com/cli/azure/install-azure-cli
4. An **Azure subscription**

Verify the installs:

```bash
python --version
pulumi version
az --version
```

---

## 3. Log in to Azure

```bash
az login
```

Check which subscription is active:

```bash
az account show
```

To switch subscription:

```bash
az account set --subscription "<SUBSCRIPTION_ID>"
```

---

## 4. Log in to Pulumi

Pulumi needs a place to store state.

**Option A: Pulumi Cloud (free for individuals)**
```bash
pulumi login
```

**Option B: Local file (no account needed)**
```bash
pulumi login --local
```

---

## 5. Create a New Project

```bash
mkdir pulumi-azure-rg
cd pulumi-azure-rg
pulumi new azure-python
```

You will be asked for:

- **Project name**: `pulumi-azure-rg`
- **Stack name**: `dev`
- **Azure location**: e.g., `centralindia` or `eastus`

This generates:

```
pulumi-azure-rg/
├── __main__.py          # Your infrastructure code
├── Pulumi.yaml          # Project definition
├── Pulumi.dev.yaml      # Stack configuration
├── requirements.txt     # Python dependencies
└── venv/                # Virtual environment
```

---

## 6. Write the Resource Group Code

Open `__main__.py` and replace the contents with:

```python
import pulumi
from pulumi_azure_native import resources

# Create an Azure Resource Group
resource_group = resources.ResourceGroup(
    "my-resource-group",              # Pulumi logical name
    resource_group_name="rg-demo-dev",  # Actual name in Azure
    location="centralindia",
    tags={
        "environment": "dev",
        "managed-by": "pulumi",
    },
)

# Export outputs so you can see them after deployment
pulumi.export("resource_group_name", resource_group.name)
pulumi.export("resource_group_location", resource_group.location)
pulumi.export("resource_group_id", resource_group.id)
```

### Code Explained

| Line | Purpose |
|------|---------|
| `import pulumi` | Core Pulumi library |
| `pulumi_azure_native` | Azure Native provider (maps directly to Azure APIs) |
| `ResourceGroup(...)` | Declares the resource group |
| `"my-resource-group"` | Pulumi's internal name (used in state) |
| `resource_group_name` | The real name that appears in Azure |
| `pulumi.export` | Prints values after deployment |

---

## 7. Install Dependencies

If not already installed by `pulumi new`:

```bash
# Activate the virtual environment
# Windows
venv\Scripts\activate

# macOS / Linux
source venv/bin/activate

pip install -r requirements.txt
```

Your `requirements.txt` should contain:

```
pulumi>=3.0.0,<4.0.0
pulumi-azure-native>=2.0.0,<3.0.0
```

---

## 8. Preview the Changes

See what Pulumi will do without actually creating anything:

```bash
pulumi preview
```

Expected output:

```
Previewing update (dev):
     Type                                   Name                    Plan
 +   pulumi:pulumi:Stack                    pulumi-azure-rg-dev     create
 +   └─ azure-native:resources:ResourceGroup  my-resource-group     create

Resources:
    + 2 to create
```

---

## 9. Deploy

```bash
pulumi up
```

Select **yes** when prompted. You'll see:

```
Outputs:
    resource_group_id      : "/subscriptions/xxxx/resourceGroups/rg-demo-dev"
    resource_group_location: "centralindia"
    resource_group_name    : "rg-demo-dev"
```

---

## 10. Verify in Azure

```bash
az group show --name rg-demo-dev
```

Or check the Azure Portal under **Resource groups**.

---

## 11. Make a Change (Update)

Edit the tags in `__main__.py`:

```python
tags={
    "environment": "dev",
    "managed-by": "pulumi",
    "owner": "your-name",
},
```

Then run:

```bash
pulumi up
```

Pulumi detects the difference and updates only the tags.

---

## 12. Clean Up (Destroy)

Delete everything Pulumi created:

```bash
pulumi destroy
```

Optionally remove the stack itself:

```bash
pulumi stack rm dev
```

---

## 13. Using Config Instead of Hard-Coding

Better practice: keep values in stack config.

**Set config:**
```bash
pulumi config set azure-native:location centralindia
pulumi config set rgName rg-demo-dev
```

**Updated `__main__.py`:**

```python
import pulumi
from pulumi_azure_native import resources

config = pulumi.Config()
rg_name = config.require("rgName")
env = pulumi.get_stack()   # "dev", "prod", etc.

resource_group = resources.ResourceGroup(
    "my-resource-group",
    resource_group_name=rg_name,
    tags={
        "environment": env,
        "managed-by": "pulumi",
    },
)

pulumi.export("resource_group_name", resource_group.name)
```

The location is picked up automatically from `azure-native:location`.

---

## 14. Common Commands Cheat Sheet

| Command | What it does |
|---------|--------------|
| `pulumi new azure-python` | Create a new project |
| `pulumi preview` | Show planned changes |
| `pulumi up` | Deploy |
| `pulumi stack output` | Show exported outputs |
| `pulumi config` | View config |
| `pulumi stack ls` | List stacks |
| `pulumi refresh` | Sync state with real Azure |
| `pulumi destroy` | Delete all resources |

---

## 15. Troubleshooting

| Problem | Fix |
|---------|-----|
| `az login` not recognized | Install Azure CLI and restart the terminal |
| Authentication error | Run `az login` again |
| `ModuleNotFoundError: pulumi_azure_native` | Activate venv and run `pip install -r requirements.txt` |
| Wrong subscription | `az account set --subscription <ID>` |
| Resource group already exists | Import it: `pulumi import azure-native:resources:ResourceGroup my-resource-group /subscriptions/<SUB_ID>/resourceGroups/<NAME>` |

---

## 16. Next Steps

Once comfortable with resource groups, try adding:

1. **Storage Account**
2. **Virtual Network**
3. **App Service / VM**
4. Multiple stacks (`dev`, `staging`, `prod`)

Would you like me to extend this with a storage account or virtual network example next?
