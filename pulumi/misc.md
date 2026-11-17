I read this as two labs: **Lab 7** covers StackReference, and **Lab 8** covers drift, import and the resource options (`ignore_changes`, `protect`, `retain_on_delete`, `depends_on`, `aliases`). Both are free and build on Labs 1 to 3.

---

# Lab 7: StackReference (Step by Step)

**Cost:** Free (one network interface card, which Azure doesn't charge for)

Builds on Labs 1 to 3 and Lab 6. A second project, `pulumi-azure-app`, reads outputs from the first one and uses them. We build in three parts, with **preview → up → check** after each.

```
Project pulumi-azure-labs / stack dev     (the "network" stack, unchanged)
   outputs: resource_group, location, web_subnet_id, primary_key ...
        │  StackReference("organization/pulumi-azure-labs/dev")
        ▼
Project pulumi-azure-app / stack dev      (the "app" stack, new)
   Part A: read outputs and re-export them
   Part B: create a NIC inside snet-web
   Part C: config for the reference name, secrets, missing outputs
```

| Part | Adds | Preview shows | Check |
|------|------|---------------|-------|
| 0 | New project and stack | none | `pulumi stack ls` |
| A | Read-only reference | outputs only | `pulumi stack output` |
| B | NIC in `snet-web` | 1 to create | `az network nic show` |
| C | Config, secret, error case | outputs only | `pulumi stack output` |

## Part 0: One-time setup

Same shell variables as Lab 6 if you use the Azure blob backend (`PULUMI_CONFIG_PASSPHRASE`, `AZURE_STORAGE_ACCOUNT`, `AZURE_STORAGE_KEY`, `pulumi login azblob://...`). Both projects must use the **same backend and the same passphrase**.

First confirm the network stack's outputs exist:

```bash
cd ~/pulumi-azure-labs
pulumi stack output resource_group
pulumi stack output web_subnet_id
pulumi stack ls
```

Create the app project in a **sibling folder**:

```bash
cd ~
mkdir pulumi-azure-app && cd pulumi-azure-app
pulumi new azure-python --name pulumi-azure-app --stack dev --yes
pulumi config set azure-native:location centralindia
source venv/bin/activate
pulumi stack ls --all        # now shows both projects
```

On a self-managed backend the organization is always `organization`, so the full name of the network stack is:

```
organization/pulumi-azure-labs/dev
```

---

## Part A: Read outputs from the other stack

**Docs for this part:**

| Code | Pulumi reference |
|------|------------------|
| `pulumi.StackReference(...)`, `require_output`, `get_output` | [Stack references](https://www.pulumi.com/docs/iac/concepts/stacks/#stackreferences) |

### A1. Code

Replace the **entire** `__main__.py` of `pulumi-azure-app` with:

```python
import pulumi

env = pulumi.get_stack()

net = pulumi.StackReference("organization/pulumi-azure-labs/dev")

rg_name = net.require_output("resource_group")
location = net.require_output("location")
subnet_id = net.require_output("web_subnet_id")

pulumi.export("seen_resource_group", rg_name)
pulumi.export("seen_location", location)
pulumi.export("seen_web_subnet_id", subnet_id)
```

`require_output` fails if the output doesn't exist. `get_output` returns an empty value instead.

### A2. Preview and up

```bash
pulumi preview
pulumi up
```

Expect: only the stack itself is created, plus three outputs. No Azure resources.

### A3. Check

```bash
pulumi stack output
pulumi stack output seen_web_subnet_id
```

Expected: `rg-lab-dev`, `centralindia`, and the full ID of `snet-web`, read from the other project.

---

## Part B: Use the values: a NIC in `snet-web`

**Docs for this part:**

| Code | Pulumi reference |
|------|------------------|
| `NetworkInterface(...)`, `ip_configurations`, `SubnetArgs` | [NetworkInterface API docs](https://www.pulumi.com/registry/packages/azure-native/api-docs/network/networkinterface/) |

### B1. Code

Add `from pulumi_azure_native import network as az_network` at the top, and add below the exports:

```python
nic = az_network.NetworkInterface(
    "nic-app",
    network_interface_name=f"nic-app-{env}",
    resource_group_name=rg_name,
    location=location,
    ip_configurations=[
        az_network.NetworkInterfaceIPConfigurationArgs(
            name="ipconfig1",
            subnet=az_network.SubnetArgs(id=subnet_id),
            private_ip_allocation_method="Dynamic",
        )
    ],
)

pulumi.export("nic_name", nic.name)
pulumi.export("nic_private_ip", nic.ip_configurations.apply(lambda c: c[0].private_ip_address))
```

### B2. Preview and up

```bash
pulumi preview      # 1 to create (the NIC)
pulumi up
```

### B3. Check

```bash
az network nic show -g rg-lab-dev -n nic-app-dev \
  --query "{ip:ipConfigurations[0].privateIPAddress, subnet:ipConfigurations[0].subnet.id}" -o json
pulumi stack output nic_private_ip
```

Expected: an IP in `10.0.1.0/24` and the `snet-web` subnet ID.

---



## Clean up

Order matters, because the NIC lives in the network stack's subnet:

```bash
cd ~/pulumi-azure-app
pulumi destroy
pulumi stack rm dev
cd ~ && rm -rf pulumi-azure-app
```

Do this **before** destroying the network stack in Lab 5, or the subnet delete will fail because the NIC is still attached.

## Troubleshooting

| Problem | Fix |
|---------|-----|
| `stack ... not found` | Check the name with `pulumi stack ls --all`. Format is `organization/<project>/<stack>` |
| Output is missing | Run `pulumi up` in the network project first, and check `pulumi stack output` there |
| Secret decrypt error | Different `PULUMI_CONFIG_PASSPHRASE` in the two projects |
| Reference works in one shell, not another | The other shell is on a different backend. Check `pulumi whoami -v` |
| Subnet delete fails in Lab 5 | The app stack's NIC still exists. Destroy the app stack first |
| `AttributeError ... private_ip_address` | Use `.apply(lambda c: c[0].private_ip_address)` as shown |

---

# Lab 8: Drift, Import and Resource Options (Step by Step)

**Cost:** Free (empty NSGs only)

Builds on Labs 1 to 3 in the `pulumi-azure-labs` project. We build in six parts, with **preview → up → check** after each.

```
Part A: drift       change a tag outside Pulumi, then refresh / revert
Part B: ignore_changes   tell Pulumi to ignore that tag
Part C: import      adopt an NSG created with az
Part D: protect     block deletion of the imported NSG
Part E: retain_on_delete + depends_on   scratch NSG
Part F: aliases     rename a resource without replacing it
```

| Part | Adds | Check |
|------|------|-------|
| A | Detect drift with refresh | `az group show` tags |
| B | `ignore_changes` | tag survives `pulumi up` |
| C | Imported NSG | `preview` shows no changes |
| D | `protect` | delete is blocked |
| E | `retain_on_delete`, `depends_on` | NSG still in Azure after removal |
| F | `aliases` | URN changes, no replace |

## Part 0: One-time setup

```bash
cd ~/pulumi-azure-labs
source venv/bin/activate
pulumi stack select dev
pulumi preview          # expect: no changes
SUB=$(az account show --query id -o tsv)
```

In `__main__.py`, add this import at the top (the name `network` is already your Lab 3 module):

```python
from pulumi_azure_native import network as az_network
```

and this marker at the bottom:

```python
# ---- Lab 8 parts go below this line ----
```

---

## Part A: Drift

**Docs for this part:**

| Code | Pulumi reference |
|------|------------------|
| `pulumi refresh` | [pulumi refresh](https://www.pulumi.com/docs/iac/cli/commands/pulumi_refresh/) |

Drift means the real Azure resource no longer matches Pulumi's state, for example after a portal edit.

### A1. Create drift

```bash
az group update -n rg-lab-dev --set tags.drift=manual
pulumi preview          # still "no changes": Pulumi has not looked yet
```

### A2. Detect it

```bash
pulumi refresh --diff
```

Expect: the `drift` tag shown as added. Choose `yes` to update the state.

```bash
pulumi preview --diff
```

Expect: Pulumi now wants to **remove** the tag, because your code doesn't have it.

### A3. Fix it

```bash
pulumi up
az group show -n rg-lab-dev --query tags
```

Expected: `drift` is gone. Shortcut for the future: `pulumi up --refresh` does both steps.

---

## Part B: `ignore_changes`

**Docs:** [ignoreChanges](https://www.pulumi.com/docs/iac/concepts/resources/options/ignorechanges/)

### B1. Code

In `__main__.py`, add `opts` to the Lab 1 resource group:

```python
resource_group = resources.ResourceGroup(
    "rg",
    resource_group_name=f"rg-lab-{env}",
    tags={"environment": env, "managed-by": "pulumi"},
    opts=pulumi.ResourceOptions(ignore_changes=["tags"]),
)
```

```bash
pulumi preview     # no changes
pulumi up
```

### B2. Check

```bash
az group update -n rg-lab-dev --set tags.drift=manual
pulumi up --refresh
az group show -n rg-lab-dev --query tags
```

Expected: refresh notices the tag, but `up` does **not** remove it.

### B3. Restore

Remove the `opts=...` line from the resource group, then:

```bash
pulumi up --refresh
az group show -n rg-lab-dev --query tags      # drift tag gone again
```

Use `ignore_changes` sparingly, for values another system legitimately manages (autoscalers, policy-added tags).

---

## Part C: Import an existing resource

**Docs:** [Importing resources](https://www.pulumi.com/docs/iac/adopting-pulumi/import/)

### C1. Create something outside Pulumi

```bash
az network nsg create -g rg-lab-dev -n nsg-imported-dev -l centralindia
export SUB=$(az account show --query id -o tsv)
echo $SUB
```

### C2. Import it

```bash
pulumi import azure-native:network:NetworkSecurityGroup nsg-imported \
  /subscriptions/$SUB/resourceGroups/rg-lab-dev/providers/Microsoft.Network/networkSecurityGroups/nsg-imported-dev \
  --out imported.py
cat imported.py
```

Choose `yes`. The resource is now in state, and `imported.py` holds generated code.

### C3. Put the code in your program

Add below the marker, adapting what was generated:

```python
nsg_imported = az_network.NetworkSecurityGroup(
    "nsg-imported",
    network_security_group_name="nsg-imported-dev",
    resource_group_name=resource_group.name,
    location="centralindia",
    opts=pulumi.ResourceOptions(protect=True),
)
```

### C4. Check

```bash
pulumi preview
```

Expected: **no changes**. If it shows a diff, change your code to match what Azure has (not the other way round) until preview is clean. Then `rm imported.py`.

Note that `pulumi import` marks resources `protect=True` by default, which leads into Part D.

Exercise variant: instead of the CLI, write the resource with `opts=pulumi.ResourceOptions(import_="<resource id>")`, and preview shows `= import`.

---

## Part D: `protect`

**Docs:** [protect](https://www.pulumi.com/docs/iac/concepts/resources/options/protect/)

### D1. Try to delete a protected resource

Comment out the whole `nsg_imported` block, then:

```bash
pulumi up
```

Expected: an **error** saying the resource can't be deleted because it is protected. Nothing is changed in Azure. Uncomment the block again.

### D2. Check

```bash
pulumi stack export | jq '.deployment.resources[]
  | select(.urn|test("nsg-imported")) | {urn, protect}'
```

Expected: `"protect": true`.

`pulumi destroy` is blocked by protected resources too, so remember this for Lab 5. To release them:

```bash
pulumi state unprotect --all
```

---

## Part E: `retain_on_delete` and `depends_on`

**Docs:** [retainOnDelete](https://www.pulumi.com/docs/iac/concepts/resources/options/retainondelete/), [dependsOn](https://www.pulumi.com/docs/iac/concepts/resources/options/dependson/)

### E1. Code

```python
scratch = az_network.NetworkSecurityGroup(
    "nsg-scratch",
    network_security_group_name="nsg-scratch-dev",
    resource_group_name=resource_group.name,
    location=resource_group.location,
    opts=pulumi.ResourceOptions(retain_on_delete=True, depends_on=[vnet]),
)
```

```bash
pulumi preview      # 1 to create
pulumi up
```

### E2. Check the options are recorded

```bash
pulumi stack export | jq '.deployment.resources[]
  | select(.urn|test("nsg-scratch")) | {retainOnDelete, dependencies}'
```

Expected: `retainOnDelete: true` and the VNet's URN in `dependencies`. (`depends_on` matters when there is no property link between resources, as in Lab 3 where subnets were chained.)

### E3. Remove it from code, keep it in Azure

Delete the `scratch` block, then:

```bash
pulumi up
az network nsg show -g rg-lab-dev -n nsg-scratch-dev --query name
pulumi stack export | jq '[.deployment.resources[].urn | select(test("nsg-scratch"))]'
```

Expected: gone from state (empty list) but **still in Azure**. Clean it by hand:

```bash
az network nsg delete -g rg-lab-dev -n nsg-scratch-dev
```

---

## Part F: `aliases`

**Docs:** [aliases](https://www.pulumi.com/docs/iac/concepts/resources/options/aliases/)

Renaming the Pulumi name of a resource changes its URN, which Pulumi treats as delete + create. With a fixed Azure name that is dangerous, because the create and the delete hit the **same** Azure resource.

### F1. Create a resource

```python
alias_demo = az_network.NetworkSecurityGroup(
    "nsg-alias",
    network_security_group_name="nsg-alias-dev",
    resource_group_name=resource_group.name,
    location=resource_group.location,
)
```

```bash
pulumi up
```

### F2. Rename without an alias: PREVIEW ONLY

Change `"nsg-alias"` to `"nsg-alias-v2"` and run:

```bash
pulumi preview
```

Expect: a **replace** (delete and create). **Do not run `pulumi up`.**

### F3. Rename with an alias

```python
alias_demo = az_network.NetworkSecurityGroup(
    "nsg-alias-v2",
    network_security_group_name="nsg-alias-dev",
    resource_group_name=resource_group.name,
    location=resource_group.location,
    opts=pulumi.ResourceOptions(aliases=[pulumi.Alias(name="nsg-alias")]),
)
```

```bash
pulumi preview      # no replace
pulumi up
pulumi stack --show-urns | grep nsg-alias
```

Expected: the URN now ends in `nsg-alias-v2`, and `az network nsg show -g rg-lab-dev -n nsg-alias-dev` still works, so nothing was recreated. Keep the alias in code for one deploy cycle, then you can remove it.

---

## Exercises

1. Put `protect=True` on the resource group, then try `pulumi destroy`. Unprotect and retry later.
2. Use `ignore_changes=["tags.owner"]` to ignore only one tag.
3. Move `nsg_alias` into a component or another parent using `Alias(parent=...)`.
4. Run `pulumi refresh --preview-only` hmm to see drift without writing state; compare with `--diff`.

## Clean up

1. Remove the `alias_demo` and `nsg_imported` blocks from `__main__.py`.
2. Release protection, then apply:

```bash
pulumi state unprotect --all
pulumi up
```

3. Make sure nothing is left in Azure:

```bash
az network nsg list -g rg-lab-dev --query "[].name" -o tsv
```

Delete leftovers with `az network nsg delete -g rg-lab-dev -n <name>`.

4. Confirm `pulumi preview` is clean, and remove the drift tag if it is still there:

```bash
az group update -n rg-lab-dev --remove tags.drift
```

## Troubleshooting

| Problem | Fix |
|---------|-----|
| Import says resource not found | Check the ID: `az network nsg show -g rg-lab-dev -n nsg-imported-dev --query id -o tsv` |
| Preview shows diffs after import | Edit the code to match Azure's real properties (location, rules) |
| `cannot be deleted because it is protected` | Expected. `pulumi state unprotect <urn>` or `--all` |
| `NameError: az_network` | Add the import in Part 0 |
| Refresh seems to do nothing | Check you changed the resource in the right group (`rg-lab-dev`) |
| Alias rename still shows replace | The alias `name=` must be the **old** Pulumi name |
| Retained NSG blocks later deletes | Delete it with `az network nsg delete` |

---

I wrote a couple of exercise lines too loosely (Lab 7 exercise 3 and Lab 8 exercise 4). Ignore the "hmm skip" wording and use `pulumi stack export` in Lab 7 and `pulumi refresh --preview-only` in Lab 8 as written. Tell me if you'd like these as downloadable `.md` files like your other labs.
