# Lab 9: Remote State in Azure Blob Storage

**Cost:** A few cents a month (one Standard LRS storage account holding small state files).

Until now your state lived on your own machine (`pulumi login --local`). In this lab you move it to an Azure Blob container, so it is backed up, versioned and shareable. This is Pulumi's "DIY backend": no Pulumi Cloud account needed.

## Objectives

1. Create a storage account and container for state, outside of Pulumi
2. Secure it: private, versioned, soft delete
3. Log in to the Azure Blob backend
4. Migrate your existing `dev` and `prod` stacks into it
5. Verify, share, and clean up

## Architecture

```
rg-pulumi-state                      (created with the Azure CLI, not Pulumi)
 └── Storage Account stpulumistate001
      └── Container "pulumi-state"
           └── .pulumi/              (written by Pulumi)
                ├── meta.yaml
                └── stacks/pulumi-azure-labs/dev.json, prod.json, ...

Your lab resources stay in rg-lab-dev / rg-lab-prod (unchanged).
```

Why create the state storage with the CLI? Pulumi needs the state **before** it can manage anything, so it cannot create its own state store.

## Prerequisites

- Labs 6 or later finished with `dev` (and optionally `prod`) deployed on the **local** backend. Fresh start? Skip Step 5 and just create stacks after login.
- Azure CLI logged in (`az login`)
- Passphrase exported: `export PULUMI_CONFIG_PASSPHRASE="your-passphrase"`. State holds encrypted secrets, so keep this passphrase safe. Without it the secrets cannot be read.

## Learner approach

Docs for this lab:
- [Using a DIY backend](https://www.pulumi.com/docs/iac/guides/basics/using-a-diy-backend/) (read the Azure Blob Storage section)
- [Azure built-in role: Storage Blob Data Contributor](https://learn.microsoft.com/en-us/azure/role-based-access-control/built-in-roles#storage-blob-data-contributor)

| Step | Do this | Check |
|------|---------|-------|
| 1 | Read the Azure Blob section of the DIY docs: URL format, account setting, authentication, required role | You can name the three things Pulumi needs |
| 2 | Create the storage with `az` | Container exists |
| 3 | Grant yourself the data role | `az storage blob list --auth-mode login` works |
| 4 | Log in with `azblob://` | `pulumi whoami -v` shows the new backend |
| 5 | Export, import, preview | Preview shows **no changes** |

## Step 1: Back up first

Before touching state, save what you have. You are still on the local backend:

```bash
pulumi whoami -v                     # shows the current backend (file://...)
pulumi stack select dev
pulumi stack export --file dev-state.json
pulumi stack select prod             # skip if you have no prod
pulumi stack export --file prod-state.json
cp Pulumi.dev.yaml Pulumi.dev.yaml.bak
cp Pulumi.prod.yaml Pulumi.prod.yaml.bak
```

Keep these files until the migration is verified.

## Step 2: Create the state storage

Pick a globally unique account name (3 to 24 lowercase letters and digits):

```bash
RG=rg-pulumi-state
SA=stpulumistate001
CONTAINER=pulumi-state
LOC=centralindia

az group create -n $RG -l $LOC

az storage account create -n $SA -g $RG -l $LOC \
  --sku Standard_LRS --kind StorageV2 \
  --min-tls-version TLS1_2 --allow-blob-public-access false

# Keep old versions of state and recover deleted blobs for 7 days
az storage account blob-service-properties update \
  --account-name $SA -g $RG \
  --enable-versioning true \
  --enable-delete-retention true --delete-retention-days 7

az storage container create -n $CONTAINER --account-name $SA --auth-mode login
```

Versioning matters: state is small but critical, and a bad edit or delete is recoverable.

If the last command says you are not authorized, do Step 3 first, wait a minute, then repeat it.

## Step 3: Grant access

Pulumi needs permission to read, write and delete blobs. Give your user the **Storage Blob Data Contributor** role on the account:

```bash
az role assignment create \
  --assignee-object-id $(az ad signed-in-user show --query id -o tsv) \
  --assignee-principal-type User \
  --role "Storage Blob Data Contributor" \
  --scope $(az storage account show -n $SA -g $RG --query id -o tsv)
```

Role changes can take a few minutes to apply. Test:

```bash
az storage blob list --account-name $SA --container-name $CONTAINER --auth-mode login -o table
```

An empty list with no error means access works.

## Step 4: Log in to the Azure backend

Option A (recommended, no keys): use your `az login` identity and put the account in the URL.

```bash
pulumi login "azblob://$CONTAINER?storage_account=$SA"
```

Option B: environment variables.

```bash
export AZURE_STORAGE_ACCOUNT=$SA
export AZURE_STORAGE_KEY=$(az storage account keys list -n $SA -g $RG --query '[0].value' -o tsv)
pulumi login azblob://$CONTAINER
```

Option B uses an account key, which grants full access to the whole account. Prefer Option A, or a SAS token (`AZURE_STORAGE_SAS_TOKEN`) with a short expiry.

Check where you are logged in:

```bash
pulumi whoami -v
pulumi stack ls         # empty: the new backend has no stacks yet
```

To skip `pulumi login` in new terminals, set the URL once:

```bash
export PULUMI_BACKEND_URL="azblob://$CONTAINER?storage_account=$SA"
```

## Step 5: Move your stacks into the new backend

Run this for each stack. Example for `dev`:

```bash
pulumi stack init dev
pulumi stack import --file dev-state.json
pulumi preview
```

Expected: **no changes**. Pulumi now believes the resources in Azure are exactly what the state says, so only the storage location changed.

Repeat for `prod`:

```bash
pulumi stack init prod
pulumi stack import --file prod-state.json
pulumi preview
```

Notes:

- `Pulumi.dev.yaml` and `Pulumi.prod.yaml` stay in your project folder. They hold config and the encryption salt, so the stack keeps decrypting your secrets with the same passphrase.
- If `pulumi stack init` asks for a passphrase, enter the **same** one as before.
- If preview wants to create or delete resources, stop. Do not run `up`. Check that you imported the right file into the right stack.

## Step 6: Verify the state is in Azure

```bash
az storage blob list --account-name $SA --container-name $CONTAINER \
  --auth-mode login --query "[].name" -o tsv
```

You should see paths under `.pulumi/`, including `meta.yaml` and a file per stack (for example `.pulumi/stacks/pulumi-azure-labs/dev.json`) plus backups and history files.

Now prove the loop works end to end. Make a harmless change, such as adding a tag, then:

```bash
pulumi up
az storage blob list --account-name $SA --container-name $CONTAINER \
  --auth-mode login --query "[].{name:name, modified:properties.lastModified}" -o table
```

The stack's state file has a fresh timestamp. Check the older copy too:

```bash
az storage blob list --account-name $SA --container-name $CONTAINER \
  --auth-mode login --include v --query "length(@)"
```

More blobs than before means versioning is keeping old copies.

## Step 7: Share it with a teammate

Anyone who needs the state needs two things:

1. The **Storage Blob Data Contributor** role on the account (Step 3, with their object ID)
2. The **passphrase** (shared through a password manager, never chat)

Then they run the same `pulumi login "azblob://..."` and `pulumi stack select dev`.

During an update Pulumi writes a lock file under `.pulumi/locks`, so two people running `pulumi up` on the same stack at the same time are blocked. If a run was killed and left a lock behind, run `pulumi cancel`, but only after confirming nobody else is deploying.

## Exercises

1. Delete `dev-state.json` from the container's `.pulumi/stacks/...` path in a test copy of the project and recover it using versions or soft delete. (Never do this on a stack you need.)
2. Create a SAS token valid for 30 minutes (`az storage container generate-sas --permissions rwdl`) and log in with `AZURE_STORAGE_SAS_TOKEN` instead of Option A.
3. Turn off shared keys on the account (`az storage account update --allow-shared-key-access false`) and confirm Option A still works while Option B fails.
4. Use Azure Key Vault instead of a passphrase to encrypt secrets: create a key, then `pulumi stack init test --secrets-provider="azurekeyvault://<vault>.vault.azure.net/keys/<key>"`. The DIY backend docs and the Pulumi secrets docs describe the URL format.

## Clean up

Destroy your lab stacks first (Lab 5). Then, only when you no longer need the state:

```bash
pulumi stack rm dev
pulumi stack rm prod
pulumi logout
az group delete -n rg-pulumi-state --yes
```

Deleting `rg-pulumi-state` deletes all state. If you want to go back to local state, run `pulumi login --local` and import the exported JSON files the same way as in Step 5.

## Troubleshooting

| Problem | Fix |
|---------|-----|
| `read ".pulumi/meta.yaml" ... 403` / `AuthorizationPermissionMismatch` | Missing **Storage Blob Data Contributor** role, or it has not propagated. Wait a few minutes, `az login` again |
| `no storage account` / account not found | Add `?storage_account=<name>` to the URL, or set `AZURE_STORAGE_ACCOUNT` |
| `container ... does not exist` | Create it: `az storage container create -n pulumi-state --account-name $SA --auth-mode login` |
| Passphrase error on `stack init` or `import` | Use the passphrase from Lab 1 (`PULUMI_CONFIG_PASSPHRASE`) |
| `stack already exists` | Run `pulumi stack ls`, then `pulumi stack select <name>` |
| Preview shows creates after import | Wrong stack or wrong file. Stop, do not `up`. Re-check the stack name and the exported JSON |
| Stack shows as locked | Confirm nobody else is deploying, then `pulumi cancel` |
| Account name rejected | 3 to 24 characters, lowercase letters and digits only |

## Key takeaways

- A DIY backend stores state in your own storage. For Azure it is `azblob://<container>`.
- Create the state storage outside Pulumi, make it private, and turn on versioning.
- Use Azure identity and the Blob Data Contributor role instead of account keys.
- Config and passphrase stay with the stack; only the state file moves. Always back up with `pulumi stack export` before migrating.
