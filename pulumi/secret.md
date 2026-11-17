# Lab 7: Secrets and Stack References

**Estimated cost:** Free for configuration, stack outputs, and one Azure
Application Security Group (ASG).

## Lab objective

In this lab, you will learn how to:

-   Store sensitive configuration values with Pulumi secrets.
-   Export secret values and understand how secrecy propagates through
    derived outputs.
-   Use `Output.secret()` to mark an output as secret and preserve
    secrecy when combining outputs.
-   Read outputs from another Pulumi stack with `StackReference`.
-   Create an Azure Application Security Group (ASG) using values
    exported by another stack.

The lab is divided into three parts. Each part follows the same
workflow:

**Preview → Up → Check**

## Architecture and workflow

``` text
Project: pulumi-azure-labs (stack: dev)
|
+-- Part A: Secret configuration and secret outputs
+-- Part B: Output.secret() and secret propagation
|
+-- Exports:
    +-- resource_group
    +-- location
    +-- other stack outputs

Project: pulumi-azure-labs-app (stack: dev)
|
+-- Part C: StackReference to pulumi-azure-labs/dev
    +-- Reads resource_group and location
    +-- Creates Azure Application Security Group: asg-app-dev
```

## Lab plan

  -------------------------------------------------------------------------------
  Part              What you add        Expected preview  Verification
  ----------------- ------------------- ----------------- -----------------------
  A                 Secret              No Azure resource Secret values are
                    configuration value changes; stack    masked in configuration
                    and secret outputs  outputs are       and outputs
                                        updated           

  B                 `Output.secret()`   No Azure resource Combined output remains
                    and secret          changes; stack    secret
                    propagation         outputs are       
                                        updated           

  C                 A new project that  One resource to   `az network asg show`
                    reads the `dev`     create: the ASG   displays the ASG
                    stack through                         
                    `StackReference`                      
  -------------------------------------------------------------------------------

------------------------------------------------------------------------

# Part 0: One-time setup

Complete these checks before starting Part A.

## Prerequisites

1.  The `pulumi-azure-labs` project must already have a `dev` stack.
2.  The `dev` stack must be deployed and must contain the Azure resource
    group and web subnet created in the earlier labs.
3.  The stack must export at least these outputs:
    -   `resource_group`
    -   `location`
4.  Pulumi CLI, Python, the Pulumi Azure Native provider, and Azure CLI
    must be installed and configured.
5.  You must be authenticated to Azure.

Check the current stack and its outputs:

``` bash
pulumi stack
pulumi stack output
```

Confirm that `resource_group` and `location` are present before
continuing.

## Configure the local secrets passphrase

If you are using the Pulumi local backend with passphrase-based secrets
encryption, export the passphrase in your terminal:

``` bash
export PULUMI_CONFIG_PASSPHRASE="your-passphrase"
```

Use the same passphrase that was used when the stack's secrets provider
was initialized. Do not commit the passphrase to source control or place
it in a lab script.

> **Note:** If your stack uses a different secrets provider, such as the
> Pulumi Cloud secrets provider or an external KMS, follow that
> provider's authentication and configuration requirements instead.

## Open the main project

Change to the existing `pulumi-azure-labs` project directory and open
`__main__.py`.

Parts A and B add code to this file. Add the Part A code at the bottom
of the file, after the existing resource and output definitions.

------------------------------------------------------------------------

# Part A: Secret configuration

## Objective

Store a configuration value as a Pulumi secret, read it from Python, and
export it without exposing its plaintext value in normal CLI output.

### A1. Store a configuration value as a secret

Run this command from the `pulumi-azure-labs` project directory:

``` bash
pulumi config set --secret dbPassword 'S3cr37-Lab7'
```

Display the stack configuration:

``` bash
pulumi config
```

**Expected result:** `dbPassword` is displayed as `[secret]`, not as
plaintext.

Inspect the stack configuration file:

``` bash
cat Pulumi.dev.yaml
```

The secret value is stored in encrypted form. You should not see the
original plaintext value in the file.

> **Security note:** The value above is a lab-only example. Never use a
> real production password in a training lab, terminal transcript, or
> source-control repository.

### A2. Add the Python code

Open `__main__.py` and add the following code at the bottom of the file:

``` python
# Lab 7 Part A: secrets
cfg = pulumi.Config()

db_password = cfg.require_secret("dbPassword")

pulumi.export("db_password", db_password)
pulumi.export("db_password_length", db_password.apply(len))
```

If `pulumi` is not already imported in the file, add this import at the
top:

``` python
import pulumi
```

#### Code explanation

  ---------------------------------------------------------------------------------
  Code                                          Explanation
  --------------------------------------------- -----------------------------------
  `pulumi.Config()`                             Creates a configuration object for
                                                the current Pulumi project and
                                                stack.

  `cfg.require_secret("dbPassword")`            Reads the required configuration
                                                value and returns it as a secret
                                                `Output`.

  `pulumi.export("db_password", db_password)`   Exports the secret value as a stack
                                                output.

  `db_password.apply(len)`                      Applies Python's `len` function to
                                                the resolved value without directly
                                                unwrapping the `Output`.

  `pulumi.export("db_password_length", ...)`    Exports the derived value. Pulumi
                                                propagates the secret marking, so
                                                this output is also treated as
                                                secret.
  ---------------------------------------------------------------------------------

**Important:** Do not use `db_password` as an ordinary Python string. It
is a Pulumi `Output`, and Pulumi resolves it as part of its deployment
engine.

### A3. Preview the changes

Run:

``` bash
pulumi preview
```

**Expected result:**

-   No Azure resources should be created, updated, or deleted.
-   Pulumi should report changes to the stack outputs.
-   The new outputs are `db_password` and `db_password_length`.
-   Secret values should remain masked.

If Pulumi proposes unexpected Azure resource changes, stop and review
the preview before proceeding.

### A4. Apply the changes

Run:

``` bash
pulumi up
```

Review the proposed changes. If the preview is correct, confirm the
update when prompted.

**Expected result:** The stack update completes without creating or
modifying Azure infrastructure. Pulumi records the new stack outputs.

### A5. Verify the outputs

Display the stack outputs:

``` bash
pulumi stack output
```

The secret outputs should be masked in normal output.

To inspect the password output explicitly, run:

``` bash
pulumi stack output db_password --show-secrets
```

This command reveals the plaintext value in the terminal. Use it only
with the disposable lab value and avoid capturing the output in shared
logs or screenshots.

To inspect the derived output:

``` bash
pulumi stack output db_password_length --show-secrets
```

**Expected result:**

-   `db_password` is marked as secret.
-   `db_password_length` is also marked as secret, even though it
    contains only the length of the original value.
-   The plaintext value is shown only when `--show-secrets` is
    explicitly requested.

## Part A completion checklist

-   [ ] `dbPassword` was added with `pulumi config set --secret`.
-   [ ] `pulumi config` masks the value.
-   [ ] `Pulumi.dev.yaml` stores the value encrypted.
-   [ ] The Python code uses `require_secret()`.
-   [ ] `pulumi preview` shows no Azure resource changes.
-   [ ] `pulumi up` completes successfully.
-   [ ] Both exported outputs are treated as secrets.

------------------------------------------------------------------------


  -----------------------------------------------------------------------

#
