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


**Docs for this part:**

| Code | Pulumi reference |
|------|------------------|
| `pulumi.Output.secret(...)` | [Secrets handling](https://www.pulumi.com/docs/iac/concepts/secrets/) |
| `pulumi.Output.format(...)` | [Inputs & Outputs](https://www.pulumi.com/docs/iac/concepts/inputs-outputs/) |

