# Lab: Remote State in Azure Blob Storage (storage key)

**Cost:** a few cents a month (one Standard LRS storage account holding a small state file).

## How it works

Your state is currently a local file. You export it to `dev-state.json`, create a separate storage account for state, log in to it with the storage account key, and import the JSON.

```
rg-pulumi-state                      (created with the Azure CLI, not Pulumi)
 └── Storage Account stpulumistateamit001
      └── Container "pulumi-state"
           └── .pulumi/
                ├── meta.yaml
                └── stacks/pulumi-azure-labs/dev.json

rg-lab-dev  (Labs 1 and 2, unchanged)
```

The state storage must be separate from the Lab 2 account and created outside Pulumi. If it were part of the stack, `pulumi destroy` would delete your own state.

## Prerequisites

- Labs 1 and 2 deployed on the local backend (`pulumi login --local`)
- Run all commands from the `pulumi-azure-labs` folder with the venv active
- `az login` done
- Your account can list storage keys (Contributor on the resource group is enough)
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

`dev-state.json` is a full copy of your stack's state. Keep it until the migration is verified.

## Step 2: Create the state storage

```bash
RG=rg-pulumi-state
SA=stpulumistateamit001           # globally unique, different from your Lab 2 account
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

## Step 3: Get the storage key and create the container

```bash
export AZURE_STORAGE_ACCOUNT=$SA
export AZURE_STORAGE_KEY=$(az storage account keys list \
  -g $RG -n $SA --query '[0].value' -o tsv)

az storage container create -n $CONTAINER \
  --account-name $SA --account-key "$AZURE_STORAGE_KEY"

az storage blob list --account-name $SA --container-name $CONTAINER \
  --account-key "$AZURE_STORAGE_KEY" -o table
```

An empty list with no error means access works. Treat the key as a secret: don't put it in files, scripts, or git.

## Step 4: Log in to the Azure backend

Pulumi reads `AZURE_STORAGE_ACCOUNT` and `AZURE_STORAGE_KEY` from the environment, so both must be exported in every new terminal.

```bash
pulumi login "azblob://$CONTAINER?storage_account=$SA"
pulumi whoami -v        # shows the azblob URL
pulumi stack ls         # empty
```

## Step 5: Import the JSON into the new backend

```bash
pulumi stack init dev
pulumi stack import --file dev-state.json
pulumi preview
```

Expected: no changes. If preview wants to create or delete anything, stop and do not run `up`. Re-check that you are in the right folder, that the stack is `dev`, and that the JSON file is the one you exported.

`Pulumi.dev.yaml` stays in your project folder. It holds your config and the encryption salt, so the same passphrase keeps working.

## Step 6: Verify

```bash
az storage blob list --account-name $SA --container-name $CONTAINER \
  --account-key "$AZURE_STORAGE_KEY" --query "[].name" -o tsv
```

You should see `.pulumi/meta.yaml` and `.pulumi/stacks/pulumi-azure-labs/dev.json`.

Now make a harmless change, such as adding `"owner": "your-name"` to the tags in `__main__.py`:

```bash
pulumi up
az storage blob list --account-name $SA --container-name $CONTAINER \
  --account-key "$AZURE_STORAGE_KEY" \
  --query "[].{name:name, modified:properties.lastModified}" -o table
az storage blob list --account-name $SA --container-name $CONTAINER \
  --account-key "$AZURE_STORAGE_KEY" --include v --query "length(@)"
```

The `dev.json` timestamp should be fresh, and the version count should go up.

## Exercises

1. Export again from the new backend (`pulumi stack export --file dev-state-remote.json`) and compare it with `dev-state.json` using `diff`. Only the tag change should differ.
2. Create a 30-minute SAS token and log in with it instead of the key:
   ```bash
   az storage container generate-sas -n $CONTAINER --account-name $SA \
     --account-key "$AZURE_STORAGE_KEY" --permissions rwdl \
     --expiry $(date -u -d '+30 minutes' +%Y-%m-%dT%H:%MZ) -o tsv
   ```
   Then `unset AZURE_STORAGE_KEY`, `export AZURE_STORAGE_SAS_TOKEN="<token>"`, and run `pulumi stack ls`.

Don't disable shared key access on this account while using this lab. It would break the key-based login.

## Step 7: Clean up

**7.1 Destroy your lab resources first (Lab 5):**

```bash
pulumi destroy
```

**7.2 Remove the stack and log out (only when you no longer need the state):**

```bash
pulumi stack rm dev
pulumi logout
```

**7.3 Delete the state storage.** This deletes all state:

```bash
az group delete -n rg-pulumi-state --yes
```

**7.4 Clear the secrets from your shell and disk:**

```bash
unset AZURE_STORAGE_KEY AZURE_STORAGE_SAS_TOKEN AZURE_STORAGE_ACCOUNT
unset PULUMI_BACKEND_URL PULUMI_CONFIG_PASSPHRASE
rm -f dev-state.json dev-state-remote.json Pulumi.dev.yaml.bak
history -c        # optional: clears this session's command history
```

Only delete `dev-state.json` and the `.bak` file once you are sure you don't need them. They are your fallback copies.

**To go back to local instead of deleting everything:**

```bash
pulumi login --local
pulumi stack init dev
pulumi stack import --file dev-state.json
```

**Optional: rotate the key.** If the key was ever pasted somewhere it shouldn't be, regenerate it before deleting or reusing the account:

```bash
az storage account keys renew -g rg-pulumi-state -n stpulumistateamit001 --key key1
```

## Troubleshooting

| Problem | Fix |
|---|---|
| `AuthorizationFailed` on `keys list` | You lack `listKeys` on the account. Ask an admin for Contributor on `rg-pulumi-state`. |
| `KeyBasedAuthenticationNotPermitted` | Shared key access is disabled. Run `az storage account update -n $SA -g $RG --allow-shared-key-access true`. |
| 403 on `pulumi login` or `preview` | `AZURE_STORAGE_KEY` is not exported in this shell. Re-run the exports from Step 3. |
| `no storage account` | Add `?storage_account=<name>` to the URL, or set `AZURE_STORAGE_ACCOUNT`. |
| `container ... does not exist` | `az storage container create -n pulumi-state --account-name $SA --account-key "$AZURE_STORAGE_KEY"` |
| Passphrase error on `stack init` or import | Use the same `PULUMI_CONFIG_PASSPHRASE` as Lab 1. |
| `stack already exists` | `pulumi stack ls`, then `pulumi stack select dev`. |
| Preview shows creates after import | Wrong stack or wrong JSON file. Do not run `up`; re-check both. |
| `dev-state.json` not found | Run the commands from the folder where you did the export in Step 1. |
| Stack shows as locked | Confirm nobody else is deploying, then `pulumi cancel`. |

I removed the role-assignment step, the `--auth-mode login` flags, and the "turn off shared keys" exercise, and I added the key export and the full cleanup step. If you'd like this as a downloadable `.md` file, tell me and I'll make one.
