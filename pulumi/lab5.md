# Lab 5: Secrets and Stack References (Step by Step)

**Cost:** Free (config, outputs and one Application Security Group)

Builds on the earlier labs: the `dev` stack must already be deployed with its resource group and web subnet. We build in three small parts, and after each part we run **preview → up → check**.

```
Project pulumi-azure-labs (stack dev)          Project pulumi-azure-labs-app (stack dev)
 ├── Part A: secret config + secret output      └── Part C: StackReference to dev
 ├── Part B: Output.secret, secrets spread           reads resource_group, location
 └── outputs: resource_group, location, ...          creates ASG asg-app-dev
```

| Part | Adds | Preview shows | Check |
|------|------|---------------|-------|
| A | Secret config value and secret outputs | outputs only, no resource changes | `[secret]` in config and outputs |
| B | `Output.secret` and secret propagation | outputs only | combined output stays secret |
| C | New project reading `dev` via `StackReference` | 1 to create (ASG) | `az network asg show` |

---

## Part 0: One-time setup

- The `dev` stack deployed with its resource group and network (`pulumi stack output` shows `resource_group` and `location`).
- Passphrase exported, because secrets are encrypted with it on a local backend:

```bash
export PULUMI_CONFIG_PASSPHRASE="your-passphrase"
```

- Open `__main__.py` in the main project. Parts A and B add code at the bottom of the file, after the existing code.

---

## Part A: Secret config

**Docs for this part:**

| Code | Pulumi reference |
|------|------------------|
| `pulumi config set --secret`, `config.require_secret(...)` | [Secrets handling](https://www.pulumi.com/docs/iac/concepts/secrets/) |
| `pulumi.export(...)` | [Inputs & Outputs](https://www.pulumi.com/docs/iac/concepts/inputs-outputs/) |

### A1. Set a secret

```bash
pulumi config set --secret dbPassword 'S3cr37-Lab7'
pulumi config
```

Expected: `dbPassword` shows `[secret]`, not the value. The stored value in `Pulumi.dev.yaml` is ciphertext:

```bash
cat Pulumi.dev.yaml
```

### A2. Code

Add at the bottom of `__main__.py`:

```python
# Lab 7 Part A: secrets
cfg = pulumi.Config()
db_password = cfg.require_secret("dbPassword")
pulumi.export("db_password", db_password)
pulumi.export("db_password_length", db_password.apply(len))
```

`require_secret` returns an `Output` that is marked secret. Anything derived from it, such as `db_password_length`, stays secret.

### A3. Preview

```bash
pulumi preview
```

Expect: **no resource changes**. Only two new stack outputs.

### A4. Up

```bash
pulumi up
```

Choose `yes`. This only updates stack outputs.

### A5. Check

```bash
pulumi stack output
pulumi stack output db_password --show-secrets
```

Expected: both new outputs show `[secret]`; with `--show-secrets` you get the real value.

---

## Part B: Creating secrets in code and how they spread

**Docs for this part:**

| Code | Pulumi reference |
|------|------------------|
| `pulumi.Output.secret(...)` | [Secrets handling](https://www.pulumi.com/docs/iac/concepts/secrets/) |
| `pulumi.Output.format(...)` | [Inputs & Outputs](https://www.pulumi.com/docs/iac/concepts/inputs-outputs/) |

### B1. Code

Add below Part A (uses the existing `web_subnet` variable):

```python
# Lab 7 Part B: Output.secret and propagation
pulumi.export("web_subnet_secret", pulumi.Output.secret(web_subnet.id))
pulumi.export("web_subnet_label", pulumi.Output.format("web-{0}", web_subnet.name))
pulumi.export("mixed", pulumi.Output.format("{0}-{1}", web_subnet.name, db_password))
```

Three cases to compare:

- `web_subnet_secret`: a normal value you force to be secret.
- `web_subnet_label`: built only from non-secret values, so it stays plain.
- `mixed`: one secret input makes the whole output secret.

### B2. Preview

```bash
pulumi preview
```

Expect: **no resource changes**, three new outputs.

### B3. Up

```bash
pulumi up
```

### B4. Check

```bash
pulumi stack output
```

Expected: `web_subnet_label` is readable; `web_subnet_secret` and `mixed` show `[secret]`.

---

## Part C: Stack reference (a second project reads `dev`)

**Docs for this part:**

| Code | Pulumi reference |
|------|------------------|
| `pulumi.StackReference(...)`, `get_output`, `require_output` | [Stacks: StackReferences](https://www.pulumi.com/docs/iac/concepts/stacks/#stackreferences) |
| `ApplicationSecurityGroup(...)` | [ApplicationSecurityGroup API docs](https://www.pulumi.com/registry/packages/azure-native/api-docs/network/applicationsecuritygroup/) |

A stack reference lets one project read the outputs of another. Here a new project reads the resource group and location from `dev` and creates an Application Security Group in it.

### C1. Create the second project

```bash
cd ..
mkdir pulumi-azure-labs-app && cd pulumi-azure-labs-app
pulumi new azure-python --name pulumi-azure-labs-app --stack dev
pulumi config set azure-native:location centralindia
```

Use the same backend and the same `PULUMI_CONFIG_PASSPHRASE` as the first project.

### C2. Find the stack name

With the local backend, the fully qualified name is `organization/<project>/<stack>`:

```
organization/pulumi-azure-labs/dev
```

If the reference fails to resolve, check how your CLI prints the name: `pulumi stack ls` run in the first project.

### C3. Code

Replace `__main__.py` in `pulumi-azure-labs-app`:

```python
import pulumi
from pulumi_azure_native import network

ref = pulumi.StackReference("organization/pulumi-azure-labs/dev")

rg_name = ref.require_output("resource_group")
location = ref.require_output("location")

asg = network.ApplicationSecurityGroup(
    "asg-app",
    application_security_group_name="asg-app-dev",
    resource_group_name=rg_name,
    location=location,
)

pulumi.export("asg_id", asg.id)
pulumi.export("network_rg", rg_name)
```

`require_output` fails with a clear error if the other stack does not export that name.

### C4. Preview

```bash
pulumi preview
```

Expect: **1 to create** (the ASG).

### C5. Up

```bash
pulumi up
```

### C6. Check

```bash
az network asg show -g rg-lab-dev -n asg-app-dev --query "{name:name, location:location}" -o table
pulumi stack output
```

Expected: the ASG exists in `rg-lab-dev`, and `network_rg` shows `rg-lab-dev`.

---

> **Version check:** the links open the latest docs. The ASG link follows the same URL pattern as the Azure Native pages in earlier labs.

---

## Troubleshooting

| Problem | Fix |
|---------|-----|
| `failed to decrypt` | `PULUMI_CONFIG_PASSPHRASE` differs from the one used when the value was set |
| `Missing required configuration variable 'dbPassword'` | Set it on the **current** stack: `pulumi config set --secret dbPassword <value>` |
| `unable to find stack` in the app project | Wrong reference name. Use `organization/pulumi-azure-labs/dev` for a local backend; confirm with `pulumi stack ls` |
| `resource_group` output missing | Make sure the first project exports `resource_group`, then run `pulumi up` there |
| Secret shows in plain text | You printed it with `--show-secrets` or `pulumi config get` |

## Cleanup

The ASG is free. To remove it when finished with all labs:

```bash
# in pulumi-azure-labs-app
pulumi destroy
pulumi stack rm dev
```

## Key takeaways

- Secrets are encrypted in config and state; outputs derived from a secret stay secret.
- `Output.secret` marks any value secret, and mixing it into other outputs makes the result secret.
- A `StackReference` lets projects share outputs without copying values.

**Next:** `lab8.md`
