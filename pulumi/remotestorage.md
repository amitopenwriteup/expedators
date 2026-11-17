Here is the remote state lab rewritten for your setup: the Lab 1 project (`pulumi-azure-labs`), the single `dev` stack, and the migration done through the exported JSON file. I removed all the `prod` steps.

# Lab: Remote State in Azure Blob Storage

**Cost:** A few cents a month (one Standard LRS storage account holding a small state file).

## How it works

Your state is currently a local file. You will export it to `dev-state.json`, create a separate Azure storage account for state, log in to it, and import the JSON into it.

```
rg-pulumi-state                      (created with the Azure CLI, not Pulumi)
 └── Storage Account stpulumistateamit001
      └── Container "pulumi-state"
           └── .pulumi/
                ├── meta.yaml
                └── stacks/pulumi-azure-labs/dev.json

rg-lab-dev  (Labs 1 and 2: resource group + storage account, unchanged)
```

The state storage must be separate from the Lab 2 account and created outside Pulumi. If it were part of the stack, `pulumi destroy` would delete your own state.

## Prerequisites

- Labs 1 and 2 deployed on the local backend (`pulumi login --local`)
- Run all commands from the `pulumi-azure-labs` folder with the venv active
- `az login` done
- Passphrase set (the same one from Lab 1):

```bash
export PULUMI_CONFIG_PASSPHRASE="your-passphrase"
```

## Step 1: Export the state to a JSON file

```bash
pulumi whoami -v                  # must show file://... (local backend)
pulumi stack select dev
pulumi stack export --file dev-state.json
cp Pulumi.dev.yaml Pulumi.dev.yaml.bak
ls -l dev-state.json Pulumi.dev.yaml.bak
```

`dev-state.json` is a full copy of your stack's state (resource group, storage account, blob). Keep it until the migration is verified.

## Step 2: Create the state storage

```bash
RG=rg-pulumi-state
SA=stpulumistateamit001           # must differ from your Lab 2 account; globally unique
CONTAINER=pulumi-state
LOC=centralindia

az group create -n $RG -l $LOC

az storage account create -n $SA -g $RG -l $LOC \
  --sku Standard_LRS --kind StorageV2 \
  --min-tls-version TLS1_2 --allow-blob-public-access false

az storage account blob-service-properties update \
  --account-name $SA -g $RG \
  --enable-versioning true \
  --enable-delete-retention true --delete-retention-days 7
```

## Step 3: Grant access, then create the container

```bash
az role assignment create \
  --assignee-object-id $(az ad signed-in-user show --query id -o tsv) \
  --assignee-principal-type User \
  --role "Storage Blob Data Contributor" \
  --scope $(az storage account show -n $SA -g $RG --query id -o tsv)
```

Wait 1 to 2 minutes, then:

```bash
az storage container create -n $CONTAINER --account-name $SA --auth-mode login
az storage blob list --account-name $SA --container-name $CONTAINER --auth-mode login -o table
```

An empty list with no error means access works.

## Step 4: Log in to the Azure backend

```bash
pulumi login "azblob://$CONTAINER?storage_account=$SA"
pulumi whoami -v        # shows the azblob URL
pulumi stack ls         # empty
```

To avoid logging in again in every new terminal:

```bash
export PULUMI_BACKEND_URL="azblob://$CONTAINER?storage_account=$SA"
```

## Step 5: Import the JSON into the new backend

```bash
pulumi stack init dev
pulumi stack import --file dev-state.json
pulumi preview
```

Expected: **no changes**. If preview wants to create or delete anything, stop and do not run `up`. Re-check that you are in the right folder, that the stack is `dev`, and that the JSON file is the one you exported.

`Pulumi.dev.yaml` stays in your project folder. It holds your config and the encryption salt, so the same passphrase keeps working.

## Step 6: Verify

```bash
az storage blob list --account-name $SA --container-name $CONTAINER \
  --auth-mode login --query "[].name" -o tsv
```

You should see `.pulumi/meta.yaml` and `.pulumi/stacks/pulumi-azure-labs/dev.json`.

Now make a harmless change, such as adding `"owner": "your-name"` to the tags in `__main__.py`:

```bash
pulumi up
az storage blob list --account-name $SA --container-name $CONTAINER \
  --auth-mode login --query "[].{name:name, modified:properties.lastModified}" -o table
az storage blob list --account-name $SA --container-name $CONTAINER \
  --auth-mode login --include v --query "length(@)"
```

The `dev.json` timestamp should be fresh, and the version count should go up.

## Exercises

1. Export again from the new backend (`pulumi stack export --file dev-state-remote.json`) and compare it with `dev-state.json` using `diff`. Only the tag change should differ.
2. Create a 30-minute SAS token (`az storage container generate-sas --permissions rwdl`) and log in with `AZURE_STORAGE_SAS_TOKEN`.
3. Turn off shared keys (`az storage account update --allow-shared-key-access false`) and confirm the `azblob://...?storage_account=` login still works.

## Clean up

1. Destroy your lab resources first (Lab 5): `pulumi destroy`
2. Then, only when you no longer need the state:

```bash
pulumi stack rm dev
pulumi logout
az group delete -n rg-pulumi-state --yes
```

Deleting `rg-pulumi-state` deletes all state. To go back to local, run `pulumi login --local`, then `pulumi stack init dev` and `pulumi stack import --file dev-state.json`.

## Troubleshooting

| Problem | Fix |
|---------|-----|
| `403` / `AuthorizationPermissionMismatch` | The role has not propagated. Wait a few minutes and run `az login` again |
| `no storage account` | Add `?storage_account=<name>` to the URL, or set `AZURE_STORAGE_ACCOUNT` |
| `container ... does not exist` | `az storage container create -n pulumi-state --account-name $SA --auth-mode login` |
| Passphrase error on `stack init` or `import` | Use the same `PULUMI_CONFIG_PASSPHRASE` as Lab 1 |
| `stack already exists` | `pulumi stack ls`, then `pulumi stack select dev` |
| Preview shows creates after import | Wrong stack or wrong JSON file. Do not run `up`; re-check both |
| `dev-state.json` not found | Run the commands from the folder where you did the export in Step 1 |
| Stack shows as locked | Confirm nobody else is deploying, then `pulumi cancel` |

If you meant something different by "modify for json file" (for example, only the export/import steps, or a downloadable `.md` file), tell me and I'll adjust it.
