# Lab 1: Storage Account with Pulumi (Python)

**Level:** Beginner  |  **Time:** 20 minutes  |  **Cost:** A few cents (Standard LRS)

---

## Objectives

By the end of this lab you will:

1. Create a Resource Group and a Storage Account
2. Create a private Blob container
3. Upload a file as a blob from code
4. Export the blob endpoint and verify it with the Azure CLI
5. Clean everything up

## Architecture

```
Resource Group (rg-lab-storage-dev)
 └── Storage Account (auto-named, StorageV2, Standard_LRS)
      └── Blob Container "data" (private)
           └── Blob "hello.txt"
```

## Prerequisites

- Python 3.9+, Pulumi CLI, Azure CLI
- Logged in: `az login` and `pulumi login --local` (or `pulumi login`)
- Completed the Resource Group guide (recommended)

---

## Step 1: Create the project

```bash
mkdir lab-storage && cd lab-storage
pulumi new azure-python --name lab-storage --stack dev
```

Accept the defaults. Set the region:

```bash
pulumi config set azure-native:location centralindia
```

## Step 2: Write the code

Replace `__main__.py` with:

```python
import pulumi
from pulumi_azure_native import resources, storage

env = pulumi.get_stack()

# 1. Resource Group
resource_group = resources.ResourceGroup(
    "rg",
    resource_group_name=f"rg-lab-storage-{env}",
)

# 2. Storage Account
#    No account_name given: Pulumi adds a random suffix so the name is
#    globally unique (storage names must be 3-24 lowercase letters/digits).
account = storage.StorageAccount(
    "sa",
    resource_group_name=resource_group.name,
    location=resource_group.location,
    sku=storage.SkuArgs(name=storage.SkuName.STANDARD_LRS),
    kind=storage.Kind.STORAGE_V2,
    allow_blob_public_access=False,
    enable_https_traffic_only=True,
    minimum_tls_version=storage.MinimumTlsVersion.TLS1_2,
    tags={"environment": env, "managed-by": "pulumi"},
)

# 3. Private blob container
container = storage.BlobContainer(
    "data",
    resource_group_name=resource_group.name,
    account_name=account.name,
    container_name="data",
    public_access=storage.PublicAccess.NONE,
)

# 4. Upload a blob from inline text
blob = storage.Blob(
    "hello",
    resource_group_name=resource_group.name,
    account_name=account.name,
    container_name=container.name,
    blob_name="hello.txt",
    type=storage.BlobType.BLOCK,
    source=pulumi.StringAsset("Hello from Pulumi!"),
)

# 5. Outputs
pulumi.export("resource_group", resource_group.name)
pulumi.export("storage_account", account.name)
pulumi.export("blob_endpoint", account.primary_endpoints.apply(lambda e: e.blob))
```

### What each part does

| Resource | Purpose |
|----------|---------|
| `StorageAccount` | The top-level account that holds all your data |
| `SkuName.STANDARD_LRS` | Cheapest redundancy: 3 copies in one datacenter |
| `allow_blob_public_access=False` | Blocks anonymous access, a secure default |
| `BlobContainer` | A folder-like group of blobs |
| `Blob` | The uploaded file |

## Step 3: Preview and deploy

```bash
pulumi preview
pulumi up
```

Choose **yes**. Expected outputs:

```
blob_endpoint  : "https://sa1a2b3c4.blob.core.windows.net/"
resource_group : "rg-lab-storage-dev"
storage_account: "sa1a2b3c4"
```

## Step 4: Verify

```bash
# Get the generated account name
ACCOUNT=$(pulumi stack output storage_account)

# List accounts in the group
az storage account list -g rg-lab-storage-dev -o table

# List blobs (uses the account key for simplicity in this lab)
az storage blob list --account-name $ACCOUNT --container-name data --auth-mode key -o table
```

You should see `hello.txt`. Download it:

```bash
az storage blob download --account-name $ACCOUNT --container-name data \
  --name hello.txt --file hello.txt --auth-mode key
cat hello.txt
```

Now try the public URL in a browser:
`<blob_endpoint>data/hello.txt` should return an **error**, because the container is private. That confirms the security setting works.

---

## Exercises

1. **Add a second container** named `logs`.
2. **Upload a local file**: replace `StringAsset` with `pulumi.FileAsset("./myfile.txt")`.
3. **Change redundancy** to `STANDARD_GRS`, run `pulumi preview`, and read the diff.
4. **Add a tag** and observe that Pulumi updates in place.

## Clean up

```bash
pulumi destroy
pulumi stack rm dev
```

## Troubleshooting

| Problem | Fix |
|---------|-----|
| `StorageAccountAlreadyTaken` | Names are global. Remove any fixed `account_name` and let Pulumi auto-name |
| `AuthorizationPermissionMismatch` on blob list | Use `--auth-mode key`, or assign yourself *Storage Blob Data Reader* |
| `ModuleNotFoundError` | Activate the venv and run `pip install -r requirements.txt` |

## Key takeaways

- Storage account names are global, so let Pulumi auto-name them.
- Pass one resource's **outputs** (`account.name`) into the next to create dependencies automatically.
- Secure defaults (no public access, TLS 1.2) cost nothing; set them from day one.
