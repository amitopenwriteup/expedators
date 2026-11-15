# Lab 5: Clean Up

**Cost:** Stops all charges

## Option A: Remove only the VM (keep the rest)

The VM is the only resource that costs real money. In `__main__.py`, re-comment the **Lab 4** block, then:

```bash
pulumi up
```

Pulumi deletes the VM, NIC and public IP because they are no longer in the code. Use the same method for Labs 3 and 2 (comment out in reverse order: 4, 3, 2).

## Option B: Remove everything

```bash
pulumi destroy        # answer yes
pulumi stack rm dev   # optional: deletes the stack and its state
```

`pulumi destroy` deletes resources in the right dependency order, ending with the resource group.

## If you also did Labs 6 to 12

- **Every stack must be destroyed separately.** Run `pulumi destroy` on `dev`, then `pulumi stack select prod` and `pulumi destroy` again. Remove each with `pulumi stack rm <name>`.
- **AKS (Lab 12):** the cluster, load balancer and node group are the costly part. Destroy before anything else and confirm no `MC_` resource group remains.
- **Key Vault (Lab 7):** deleted vaults are kept for 7 days and their names stay reserved. Purge to reuse a name: `az keyvault list-deleted -o table`, then `az keyvault purge --name <vault-name>`.

## Verify nothing is left

```bash
az group exists -n rg-lab-dev        # should print: false
az resource list -o table            # no lab resources
```

## Local cleanup (optional)

```bash
deactivate                                # leave the venv
cd .. && rm -rf pulumi-azure-labs         # remove the project folder
rm ~/.ssh/lab_vm ~/.ssh/lab_vm.pub        # remove the lab SSH key
pulumi logout
```

## Troubleshooting

| Problem | Fix |
|---------|-----|
| Destroy fails or is interrupted | Run `pulumi refresh`, then `pulumi destroy` again |
| Resource deleted manually in the Portal | `pulumi refresh` syncs state, then destroy |
| Group still exists | `az group delete -n rg-lab-dev --yes` (then `pulumi refresh`) |
| Forgot the passphrase | Set `PULUMI_CONFIG_PASSPHRASE` to the value you used in Lab 1 |

## Recap

| Lab | You learned |
|-----|-------------|
| 1 | Projects, stacks, state, Resource Group |
| 2 | Passing outputs between resources, secure storage defaults |
| 3 | CIDR planning, NSG priorities, config values |
| 4 | Building a VM from parts, SSH keys, outputs |
| 5 | Safe teardown |
