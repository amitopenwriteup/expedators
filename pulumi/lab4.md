# Lab 4: Linux Virtual Machine

**Cost:** A few cents per hour. **Destroy in Lab 5.**

Builds on Lab 3: the VM joins `snet-web`, so the NSG that allows SSH from your IP already protects it.

## Objectives

1. Add a public IP, network interface and Ubuntu 24.04 VM
2. Log in with an SSH key (no passwords)
3. Connect using Pulumi outputs

## Architecture

```
rg-lab-dev
 ├── VNet / snet-web / NSG   (Lab 3)
 ├── Public IP (Standard, Static)
 ├── NIC (snet-web + public IP)
 └── VM vm-lab-dev (Ubuntu 24.04, Standard_B1s)
```

## Learner approach: build it from the docs

A VM is several resources. Build from the bottom up and preview after each one. Do not start with the VM page.

**Docs for this lab:**
- [PublicIPAddress](https://www.pulumi.com/registry/packages/azure-native/api-docs/network/publicipaddress/)
- [NetworkInterface](https://www.pulumi.com/registry/packages/azure-native/api-docs/network/networkinterface/)
- [VirtualMachine](https://www.pulumi.com/registry/packages/azure-native/api-docs/compute/virtualmachine/)

> **Version check:** the links open the latest Azure Native docs. If your `requirements.txt` pins version 2, use the same page under `azure-native@2.x`, e.g. `https://www.pulumi.com/registry/packages/azure-native@2.x/api-docs/<module>/<resource>/`.

| Step | Do this | Check |
|------|---------|-------|
| 1 | Open `PublicIPAddress`; write the minimum and set Standard SKU with Static allocation | Preview shows the IP |
| 2 | Open `NetworkInterface`; add one IP configuration using the Lab 3 subnet and the public IP | Preview shows the NIC |
| 3 | Open `VirtualMachine`. It is long, so read only the input group names first: hardware, storage, OS, network profile | You can name the four profiles |
| 4 | Write one profile at a time (hardware first), previewing after each | No missing-argument errors |
| 5 | Add the SSH key in the OS profile and turn off password login | Preview shows the key setting |
| 6 | Move the code into `vm.py` and call it from `__main__.py` | `pulumi up` works, `ssh_command` prints |

Notice: profile objects are nested `...Args` classes inside one resource. Reading them one group at a time makes the long page manageable.

## Step 1: Create an SSH key and set config

```bash
ssh-keygen -t ed25519 -f ~/.ssh/lab_vm -N ""
pulumi config set sshPublicKey "$(cat ~/.ssh/lab_vm.pub)"
```

> **Windows PowerShell:** `pulumi config set sshPublicKey (Get-Content $HOME\.ssh\lab_vm.pub -Raw)`

## Step 2: Enable the code

Lab 3 must already be uncommented. Now uncomment the **Lab 4** block in `__main__.py`:

```python
import vm
public_ip, virtual_machine = vm.create_vm(resource_group, web_subnet)
pulumi.export("public_ip", public_ip.ip_address)
pulumi.export("ssh_command", public_ip.ip_address.apply(lambda ip: f"ssh -i ~/.ssh/lab_vm azureuser@{ip}"))
```

Key points in `vm.py`:

- Takes the resource group and the subnet from earlier labs.
- A VM is a bundle: public IP, NIC and compute are separate resources, wired by passing outputs (`nic.id`).
- `disable_password_authentication=True` forces key login.
- `delete_option="Delete"` removes the OS disk with the VM.
- Size comes from config with a default: `config.get("vmSize") or "Standard_B1s"`.

## Step 3: Preview and deploy

```bash
pulumi preview      # expect: 3 to create (public IP, NIC, VM)
pulumi up
```

## Step 4: Connect

```bash
$(pulumi stack output ssh_command)
```

Inside the VM:

```bash
hostname
cat /etc/os-release
exit
```

Verify from the CLI (expected: `VM running`):

```bash
az vm get-instance-view -g rg-lab-dev -n vm-lab-dev \
  --query "instanceView.statuses[1].displayStatus" -o tsv
```

## Exercises

1. **Resize:** `pulumi config set vmSize Standard_B2s`, then `pulumi up`.
2. **Install nginx at boot:** in `vm.py` add `custom_data` to `OSProfileArgs`:

   ```python
   import base64
   init = "#!/bin/bash\napt-get update && apt-get install -y nginx\n"
   # inside OSProfileArgs(...):
   custom_data=base64.b64encode(init.encode()).decode(),
   ```

   Port 80 is already open in the Lab 3 NSG. Browse to the public IP.
3. **Pause charges without deleting:** `az vm deallocate -g rg-lab-dev -n vm-lab-dev`

## Troubleshooting

| Problem | Fix |
|---------|-----|
| `SkuNotAvailable` | `pulumi config set vmSize Standard_B2s` or change region. List: `az vm list-skus -l centralindia --size Standard_B -o table` |
| `Permission denied (publickey)` | Use `-i ~/.ssh/lab_vm`; the config must hold the **public** key |
| SSH times out | Your IP changed: `pulumi config set myIp <new>/32`, then `pulumi up` |
| Key format error | Key must be one line starting `ssh-ed25519` or `ssh-rsa` |
| vCPU quota error | Request a quota increase or pick a smaller size |

**Next:** `lab5.md`
