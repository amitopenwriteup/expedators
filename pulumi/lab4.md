# Lab 4: Linux Virtual Machine (Step by Step)

**Cost:** A few cents per hour. **Destroy in Lab 5.**

Builds on Lab 3: the VM joins `snet-web`, so the NSG that allows SSH from your IP already protects it. Same project, same stack. We build in three small parts, and after each part we run **preview → up → check**.

```
rg-lab-dev
 ├── VNet / snet-web / NSG   (Lab 3)
 ├── Part A: Public IP (Standard, Static)
 ├── Part B: NIC (snet-web + public IP)
 └── Part C: VM vm-lab-dev (Ubuntu 24.04, Standard_B1s)
```

| Part | Adds | Preview shows | Check |
|------|------|---------------|-------|
| A | Public IP | 1 to create | `az network public-ip show` |
| B | Network interface | 1 to create | `az network nic show` |
| C | Virtual machine | 1 to create | `az vm get-instance-view` + SSH |

Total: 3 resources (public IP, NIC, VM). Build from the bottom up: the VM needs the NIC, and the NIC needs the public IP.

---

## Part 0: One-time setup

### Create an SSH key and set config

```bash
ssh-keygen -t ed25519 -f ~/.ssh/lab_vm -N ""
pulumi config set sshPublicKey "$(cat ~/.ssh/lab_vm.pub)"
```

> **Windows PowerShell:** `pulumi config set sshPublicKey (Get-Content $HOME\.ssh\lab_vm.pub -Raw)`

Optional VM size (default is `Standard_B1s`):

```bash
pulumi config set vmSize Standard_B1s
```

### Create `vm.py`

Create `vm.py` next to `__main__.py` with only this to start:

```python
import pulumi
import pulumi_azure_native as azure_native


def create_vm(resource_group, subnet):
    config = pulumi.Config()

    # ---- Part A goes here ----

    # ---- Part B goes here ----

    # ---- Part C goes here ----
```

### Call it from `__main__.py`

Lab 3 must already be in place. Add this block at the bottom. For now it only calls the function:

```python
import vm
vm.create_vm(resource_group, web_subnet)
```

In Part C you will replace it with the version that unpacks the return values and exports them.

---

## Part A: Public IP

**Docs for this part:**

| Code | Pulumi reference |
|------|------------------|
| `PublicIPAddress(...)`, `sku`, `public_ip_allocation_method` | [PublicIPAddress API docs](https://www.pulumi.com/registry/packages/azure-native/api-docs/network/publicipaddress/) (see `PublicIPAddressSkuArgs`) |
| `resource_group.name` passed as an input | [Inputs & Outputs](https://www.pulumi.com/docs/iac/concepts/inputs-outputs/) |

### A1. Code

Replace `# ---- Part A goes here ----` with:

```python
    public_ip = azure_native.network.PublicIPAddress(
        "pip",
        public_ip_address_name="pip-lab-dev",
        resource_group_name=resource_group.name,
        sku=azure_native.network.PublicIPAddressSkuArgs(name="Standard"),
        public_ip_allocation_method="Static",
    )
```

Standard SKU requires Static allocation.

### A2. Preview

```bash
pulumi preview
```

Expect: **1 to create** (the public IP).

### A3. Up

```bash
pulumi up
```

Choose `yes`.

### A4. Check

```bash
az network public-ip show -g rg-lab-dev -n pip-lab-dev \
  --query "{sku:sku.name, allocation:publicIPAllocationMethod, ip:ipAddress}" -o table
```

Expected: SKU `Standard`, allocation `Static`, and an IP address.

---

## Part B: Network Interface

**Docs for this part:**

| Code | Pulumi reference |
|------|------------------|
| `NetworkInterface(...)`, `ip_configurations` | [NetworkInterface API docs](https://www.pulumi.com/registry/packages/azure-native/api-docs/network/networkinterface/) (see `NetworkInterfaceIPConfigurationArgs`, `SubnetArgs`, `PublicIPAddressArgs`) |
| `subnet.id`, `public_ip.id` passed as inputs | [Inputs & Outputs](https://www.pulumi.com/docs/iac/concepts/inputs-outputs/) |

### B1. Code

Replace `# ---- Part B goes here ----` with:

```python
    nic = azure_native.network.NetworkInterface(
        "nic",
        network_interface_name="nic-lab-dev",
        resource_group_name=resource_group.name,
        ip_configurations=[
            azure_native.network.NetworkInterfaceIPConfigurationArgs(
                name="ipconfig1",
                subnet=azure_native.network.SubnetArgs(id=subnet.id),
                public_ip_address=azure_native.network.PublicIPAddressArgs(id=public_ip.id),
                private_ip_allocation_method="Dynamic",
            )
        ],
    )
```

The subnet comes from Lab 3 (`web_subnet`), so the Lab 3 NSG protects this NIC.

### B2. Preview

```bash
pulumi preview
```

Expect: **1 to create** (the NIC); the public IP is unchanged.

### B3. Up

```bash
pulumi up
```

### B4. Check

```bash
az network nic show -g rg-lab-dev -n nic-lab-dev \
  --query "ipConfigurations[0].{subnet:subnet.id, publicIp:publicIPAddress.id, private:privateIPAddress}" -o yaml
```

Expected: the `snet-web` subnet id, the public IP id, and a private IP in `10.0.1.0/24`.

---

## Part C: Virtual Machine

**Docs for this part:**

| Code | Pulumi reference |
|------|------------------|
| `VirtualMachine(...)`, `vm_name` | [VirtualMachine API docs](https://www.pulumi.com/registry/packages/azure-native/api-docs/compute/virtualmachine/) |
| `HardwareProfileArgs` | Same page: hardware profile (VM size) |
| `StorageProfileArgs`, `ImageReferenceArgs`, `OSDiskArgs` | Same page: storage profile (image and OS disk) |
| `OSProfileArgs`, `LinuxConfigurationArgs`, `SshPublicKeyArgs` | Same page: OS profile (user and SSH key) |
| `NetworkProfileArgs`, `NetworkInterfaceReferenceArgs` | Same page: network profile (NIC) |
| `config.get("vmSize")`, `config.require("sshPublicKey")` | [Pulumi configuration](https://www.pulumi.com/docs/iac/concepts/config/) |

The VirtualMachine page is long. Read only the four profile group names first (hardware, storage, OS, network), then write one profile at a time.

### C1. Code

Replace `# ---- Part C goes here ----` with:

```python
    vm_size = config.get("vmSize") or "Standard_B1s"
    ssh_public_key = config.require("sshPublicKey")

    virtual_machine = azure_native.compute.VirtualMachine(
        "vm",
        vm_name="vm-lab-dev",
        resource_group_name=resource_group.name,
        # 1. Hardware: size
        hardware_profile=azure_native.compute.HardwareProfileArgs(
            vm_size=vm_size,
        ),
        # 2. Network: attach the NIC from Part B
        network_profile=azure_native.compute.NetworkProfileArgs(
            network_interfaces=[
                azure_native.compute.NetworkInterfaceReferenceArgs(
                    id=nic.id,
                    primary=True,
                    delete_option="Delete",
                )
            ],
        ),
        # 3. Storage: Ubuntu 24.04 image and OS disk
        storage_profile=azure_native.compute.StorageProfileArgs(
            image_reference=azure_native.compute.ImageReferenceArgs(
                publisher="Canonical",
                offer="ubuntu-24_04-lts",
                sku="server",
                version="latest",
            ),
            os_disk=azure_native.compute.OSDiskArgs(
                create_option="FromImage",
                delete_option="Delete",
                managed_disk=azure_native.compute.ManagedDiskParametersArgs(
                    storage_account_type="Standard_LRS",
                ),
            ),
        ),
        # 4. OS: admin user and SSH key, no passwords
        os_profile=azure_native.compute.OSProfileArgs(
            computer_name="vm-lab-dev",
            admin_username="azureuser",
            linux_configuration=azure_native.compute.LinuxConfigurationArgs(
                disable_password_authentication=True,
                ssh=azure_native.compute.SshConfigurationArgs(
                    public_keys=[
                        azure_native.compute.SshPublicKeyArgs(
                            path="/home/azureuser/.ssh/authorized_keys",
                            key_data=ssh_public_key,
                        )
                    ],
                ),
            ),
        ),
    )

    return public_ip, virtual_machine
```

Key points:

- `disable_password_authentication=True` forces key login.
- `delete_option="Delete"` on the OS disk and NIC reference removes them with the VM.
- `nic.id` is an output passed as an input, so Pulumi creates the NIC first.

Now update `__main__.py`. Replace the two-line block from Part 0 with:

```python
import vm
public_ip, virtual_machine = vm.create_vm(resource_group, web_subnet)
pulumi.export("public_ip", public_ip.ip_address)
pulumi.export("ssh_command", public_ip.ip_address.apply(lambda ip: f"ssh -i ~/.ssh/lab_vm azureuser@{ip}"))
```

### C2. Preview

```bash
pulumi preview
```

Expect: **1 to create** (the VM) plus the new stack outputs.

### C3. Up

```bash
pulumi up
```

This takes a few minutes. `public_ip` and `ssh_command` are printed.

### C4. Check

```bash
az vm get-instance-view -g rg-lab-dev -n vm-lab-dev \
  --query "instanceView.statuses[1].displayStatus" -o tsv
```

Expected: `VM running`.

Then connect:

```bash
$(pulumi stack output ssh_command)
```

Inside the VM:

```bash
hostname
cat /etc/os-release
exit
```

Expected: hostname `vm-lab-dev` and Ubuntu 24.04.

---

## Exercises

1. **Resize:** `pulumi config set vmSize Standard_B2s`, then `pulumi up`.
3. **Pause charges without deleting:** `az vm deallocate -g rg-lab-dev -n vm-lab-dev`
> **Version check:** the registry links open the latest Azure Native docs. If your `requirements.txt` pins version 2, use the same pages under `azure-native@2.x`, e.g. `https://www.pulumi.com/registry/packages/azure-native@2.x/api-docs/compute/virtualmachine/`.

---



The VM costs a few cents per hour. Destroy it in Lab 5, or deallocate it to pause charges.

**Next:** `lab5.md`
