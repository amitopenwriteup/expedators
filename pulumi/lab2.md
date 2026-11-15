# Lab 2: Storage Account

**Cost:** A few cents (Standard LRS)

Builds on Lab 1. Same project, same stack.

## Objectives

1. Add a Storage Account to the existing Resource Group
2. Create a private Blob container and upload a blob from code
3. Verify with the Azure CLI

## Architecture

```
rg-lab-dev (Lab 1)
 └── Storage Account (name from STORAGE_ACCOUNT_NAME)
      └── Container "data" (private)
           └── Blob "hello.txt"
```

## Learner approach: build it from the docs

Try each step yourself using the docs, preview after every step, and only then compare with Step 1.

**Docs for this lab:**
- [StorageAccount](https://www.pulumi.com/registry/packages/azure-native/api-docs/storage/storageaccount/)
- [BlobContainer](https://www.pulumi.com/registry/packages/azure-native/api-docs/storage/blobcontainer/)
- [Blob](https://www.pulumi.com/registry/packages/azure-native/api-docs/storage/blob/)

> **Version check:** the links open the latest Azure Native docs. If your `requirements.txt` pins version 2, use the same page under `azure-native@2.x`, e.g. `https://www.pulumi.com/registry/packages/azure-native@2.x/api-docs/<module>/<resource>/`.

| Step | Do this | Check |
|------|---------|-------|
| 1 | On the `StorageAccount` page, list the **required** inputs | You found resource group, `sku` and `kind` |
| 2 | In `__main__.py`, write only those inputs | `pulumi preview` works |
| 3 | Add `account_name`, using a variable for the name | Preview shows your name; try a short or capitalised name to see the error |
| 4 | Add `allow_blob_public_access`, `enable_https_traffic_only`, `minimum_tls_version` | Each one appears in the preview |
| 5 | Open `BlobContainer`, pass `account.name` as an input | Preview shows 2 resources |
| 6 | Open `Blob`, add `hello.txt` with `pulumi.StringAsset(...)` | Preview shows 3 resources |
| 7 | Move the code into a function in `storage.py` and call it from `__main__.py` | `pulumi up` works, outputs print |

Notice: `account.name` is an output of one resource passed as an input to the next. That is how Pulumi learns the order.

## Step 1: Create `storage.py`

Create a new file `storage.py` in the same folder as `__main__.py`, and paste the **entire** file below (imports, name variable and function). Pasting only the variable gives `AttributeError: module 'storage' has no attribute 'create_storage'`.

```python
import pulumi
from pulumi_azure_native import storage as az_storage

# ---------------------------------------------------------------
# Storage account name (change this!)
# Rules: globally unique, 3-24 characters, lowercase letters and digits only.
# Example: "stlabamit001"
# ---------------------------------------------------------------
STORAGE_ACCOUNT_NAME = "stlabamit001"


def create_storage(rg, account_name=STORAGE_ACCOUNT_NAME):
    """Storage account + private container + one blob. Returns the account."""
    env = pulumi.get_stack()

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

    container = az_storage.BlobContainer(
        "data",
        resource_group_name=rg.name,
        account_name=account.name,
        container_name="data",
        public_access=az_storage.PublicAccess.NONE,
    )

    az_storage.Blob(
        "hello",
        resource_group_name=rg.name,
        account_name=account.name,
        container_name=container.name,
        blob_name="hello.txt",
        type=az_storage.BlobType.BLOCK,
        source=pulumi.StringAsset("Hello from Pulumi!"),
    )

    # Outputs (exported here, so __main__.py only needs one line)
    pulumi.export("storage_account", account.name)
    pulumi.export("blob_endpoint", account.primary_endpoints.apply(lambda e: e.blob))

    return account
```

**Change `STORAGE_ACCOUNT_NAME`.** Storage account names are globally unique across Azure, 3-24 characters, lowercase letters and digits only. Use something like `st` + your name + digits.

What the function does:

- Takes the Lab 1 resource group, so nothing is recreated.
- Creates the account, a private container `data`, and the blob `hello.txt`.
- Passing `account.name` into the container and blob creates the dependency order automatically.
- Exports `storage_account` and `blob_endpoint`.
- Uses secure defaults: no public blob access, HTTPS only, TLS 1.2.

## Step 1b: Call it from `__main__.py`

Add `import storage` with the other imports at the top, and the call at the bottom, so the file looks like this:

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

To use a different name without editing `storage.py`, pass it in: `storage.create_storage(resource_group, "stothername001")`.

## Step 2: Preview and deploy

```bash
pulumi preview      # expect: 3 to create (account, container, blob)
pulumi up
```

## Step 3: Verify

```bash
ACCOUNT=$(pulumi stack output storage_account)   # your chosen name

az storage account list -g rg-lab-dev -o table
az storage blob list --account-name $ACCOUNT --container-name data --auth-mode key -o table

az storage blob download --account-name $ACCOUNT --container-name data \
  --name hello.txt --file hello.txt --auth-mode key
cat hello.txt
```

Then open `<blob_endpoint>data/hello.txt` in a browser. It should **fail**: the container is private.

## Exercises

1. Add a second container named `logs` in `storage.py`.
2. Replace `StringAsset` with `pulumi.FileAsset("./myfile.txt")`.
3. Change the SKU to `STANDARD_GRS`, run `pulumi preview` and read the diff (then revert).

## Troubleshooting

| Problem | Fix |
|---------|-----|
| `AttributeError ... no attribute 'create_storage'` | `storage.py` is incomplete. Paste the full file from Step 1 |
| `StorageAccountAlreadyTaken` | The name is taken globally. Change `STORAGE_ACCOUNT_NAME` in `storage.py` |
| `AccountNameInvalid` | Use 3-24 lowercase letters/digits only (no dashes or capitals) |
| `AuthorizationPermissionMismatch` | Use `--auth-mode key`, or assign *Storage Blob Data Reader* |

**Next:** `lab3.md`
