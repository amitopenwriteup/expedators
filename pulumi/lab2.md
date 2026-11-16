# Lab 2: Storage Account (Step by Step)

**Cost:** A few cents (Standard LRS)

Builds on Lab 1. Same project, same stack. We build in three small parts, and after each part we run **preview → up → check**.

```
rg-lab-dev (Lab 1)
 └── Part A: Storage Account
      └── Part B: Container "data" (private)
           └── Part C: Blob "hello.txt"
```

| Part | Adds | Preview shows | Check |
|------|------|---------------|-------|
| A | Storage account | 1 to create | `az storage account list` |
| B | Private container `data` | 1 to create | `az storage container list` |
| C | Blob `hello.txt` | 1 to create | `az storage blob list` + download |

---

## Part 0: One-time setup

### Create `storage.py`

Create `storage.py` next to `__main__.py` with only this to start:

```python
import pulumi
from pulumi_azure_native import storage as az_storage

# Rules: globally unique, 3-24 characters, lowercase letters and digits only.
# Example: "stlabamit001"
STORAGE_ACCOUNT_NAME = "stlabamit001"


def create_storage(rg, account_name=STORAGE_ACCOUNT_NAME):
    env = pulumi.get_stack()

    # ---- Part A goes here ----

    # ---- Part B goes here ----

    # ---- Part C goes here ----

    return account
```

**Change `STORAGE_ACCOUNT_NAME`** to something unique, such as `st` + your name + digits.

### Call it from `__main__.py`

Add `import storage` at the top and the call at the bottom:

```python
import pulumi
import storage
from pulumi_azure_native import resources

env = pulumi.get_stack()

resource_group = resources.ResourceGroup(
    "rg",
    resource_group_name=f"rg-lab-{env}",
    tags={"environment": env, "managed-by": "pulumi"},
)

pulumi.export("resource_group", resource_group.name)
pulumi.export("location", resource_group.location)

storage.create_storage(resource_group)
```

---

## Part A: Storage Account

**Docs:** [StorageAccount](https://www.pulumi.com/registry/packages/azure-native/api-docs/storage/storageaccount/)

### A1. Code

Replace `# ---- Part A goes here ----` with:

```python
    account = az_storage.StorageAccount(
        "sa",
        resource_group_name=rg.name,
        account_name=account_name,
        location=rg.location,
        sku=az_storage.SkuArgs(name=az_storage.SkuName.STANDARD_LRS),
        kind=az_storage.Kind.STORAGE_V2,
        allow_blob_public_access=False,
        enable_https_traffic_only=True,
        minimum_tls_version=az_storage.MinimumTlsVersion.TLS1_2,
        tags={"environment": env, "managed-by": "pulumi"},
    )

    pulumi.export("storage_account", account.name)
    pulumi.export("blob_endpoint", account.primary_endpoints.apply(lambda e: e.blob))
```

Secure defaults: no public blob access, HTTPS only, TLS 1.2.

### A2. Preview

```bash
pulumi preview
```

Expect: **1 to create** (the storage account).

### A3. Up

```bash
pulumi up
```

Choose `yes`. Outputs `storage_account` and `blob_endpoint` are printed.

### A4. Check

```bash
ACCOUNT=$(pulumi stack output storage_account)

az storage account list -g rg-lab-dev -o table
az storage account show -n $ACCOUNT -g rg-lab-dev \
  --query "{sku:sku.name, tls:minimumTlsVersion, publicBlob:allowBlobPublicAccess}" -o table
```

Expected: SKU `Standard_LRS`, TLS `TLS1_2`, public blob access `False`.

---

## Part B: Container `data`

**Docs:** [BlobContainer](https://www.pulumi.com/registry/packages/azure-native/api-docs/storage/blobcontainer/)

### B1. Code

Replace `# ---- Part B goes here ----` with:

```python
    container = az_storage.BlobContainer(
        "data",
        resource_group_name=rg.name,
        account_name=account.name,
        container_name="data",
        public_access=az_storage.PublicAccess.NONE,
    )
```

`account.name` is an output of the account passed as an input here. That is how Pulumi learns the order.

### B2. Preview

```bash
pulumi preview
```

Expect: **1 to create** (the container); the account is unchanged.

### B3. Up

```bash
pulumi up
```

### B4. Check

```bash
az storage container list --account-name $ACCOUNT --auth-mode key -o table
```

Expected: container `data` listed. Check it is private:

```bash
az storage container show --account-name $ACCOUNT --name data --auth-mode key \
  --query properties.publicAccess
```

Expected: `null` (no public access).

---

## Part C: Blob `hello.txt`

**Docs:** [Blob](https://www.pulumi.com/registry/packages/azure-native/api-docs/storage/blob/)

### C1. Code

Replace `# ---- Part C goes here ----` with:

```python
    az_storage.Blob(
        "hello",
        resource_group_name=rg.name,
        account_name=account.name,
        container_name=container.name,
        blob_name="hello.txt",
        type=az_storage.BlobType.BLOCK,
        source=pulumi.StringAsset("Hello from Pulumi!"),
    )
```

### C2. Preview

```bash
pulumi preview
```

Expect: **1 to create** (the blob).

### C3. Up

```bash
pulumi up
```

### C4. Check

```bash
az storage blob list --account-name $ACCOUNT --container-name data --auth-mode key -o table

az storage blob download --account-name $ACCOUNT --container-name data \
  --name hello.txt --file hello.txt --auth-mode key
cat hello.txt
```

Expected: `Hello from Pulumi!`

Now open `<blob_endpoint>data/hello.txt` in a browser. It should **fail**, because the container is private.

---

## Exercises

1. Add a second container named `logs`.
2. Replace `StringAsset` with `pulumi.FileAsset("./myfile.txt")`.
3. Change the SKU to `STANDARD_GRS`, run `pulumi preview`, read the diff, then revert.

## Troubleshooting

| Problem | Fix |
|---------|-----|
| `AttributeError ... no attribute 'create_storage'` | `storage.py` is incomplete. Include the imports, name variable and full function |
| `NameError: name 'account' is not defined` | Part A code is missing, or `return account` runs before it |
| `StorageAccountAlreadyTaken` | Name is taken globally. Change `STORAGE_ACCOUNT_NAME` |
| `AccountNameInvalid` | Use 3-24 lowercase letters/digits only |
| `AuthorizationPermissionMismatch` | Use `--auth-mode key`, or assign *Storage Blob Data Reader* |

**Next:** `lab3.md`
