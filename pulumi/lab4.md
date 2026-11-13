# Lab 3: Linux Virtual Machine with Pulumi (Python)

**Level:** Intermediate  |  **Time:** 30 minutes  |  **Cost:** A few cents per hour (B-series VM + Standard public IP). **Destroy when done.**

---

## Objectives

By the end of this lab you will:

1. Build the full set of resources a VM needs: VNet, subnet, NSG, public IP, NIC
2. Deploy an Ubuntu VM that uses SSH key login (no passwords)
3. Read the public IP from Pulumi outputs and connect over SSH
4. Clean up to stop charges

## Architecture

```
Resource Group (rg-lab-vm-dev)
 ├── VNet 10.0.0.0/16
 │    └── Subnet 10.0.1.0/24 ── NSG (SSH from your IP only)
 ├── Public IP (Standard, Static)
 ├── Network Interface (subnet + public IP)
 └── VM vm-lab-dev (Ubuntu 24.04, Standard_B1s)
```

## Prerequisites

- Python 3.9+, Pulumi CLI, Azure CLI, logged in to both
- An SSH key pair. If you have none:

```bash
ssh-keygen -t ed25519 -f ~/.ssh/lab_vm -N ""
```

- Your public IP address

---

## Step 1: Create the project

```bash
mkdir lab-vm && cd lab-vm
pulumi new azure-python --name lab-vm --stack dev
pulumi config set azure-native:location centralindia
pulumi config set myIp 203.0.113.10/32              # replace with YOUR IP
pulumi config set sshPublicKey "$(cat ~/.ssh/lab_vm.pub)"
```

> **Windows PowerShell:** `pulumi config set sshPublicKey (Get-Content $HOME\.ssh\lab_vm.pub -Raw)`

## Step 2: Write the code

Replace `__main__.py` with:

```python
import pulumi
from pulumi_azure_native import resources, network, compute

config = pulumi.Config()
my_ip = config.require("myIp")
ssh_key = config.require("sshPublicKey")
env = pulumi.get_stack()
admin_user = "azureuser"

# 1. Resource Group
resource_group = resources.ResourceGroup(
    "rg", resource_group_name=f"rg-lab-vm-{env}"
)

# 2. NSG: SSH only from your IP
nsg = network.NetworkSecurityGroup(
    "nsg",
    resource_group_name=resource_group.name,
    location=resource_group.location,
    security_rules=[
        network.SecurityRuleArgs(
            name="allow-ssh-from-my-ip",
            priority=100,
            direction="Inbound",
            access="Allow",
            protocol="Tcp",
            source_port_range="*",
            destination_port_range="22",
            source_address_prefix=my_ip,
            destination_address_prefix="*",
        )
    ],
)

# 3. VNet and subnet
vnet = network.VirtualNetwork(
    "vnet",
    resource_group_name=resource_group.name,
    location=resource_group.location,
    address_space=network.AddressSpaceArgs(address_prefixes=["10.0.0.0/16"]),
)

subnet = network.Subnet(
    "subnet",
    resource_group_name=resource_group.name,
    virtual_network_name=vnet.name,
    address_prefix="10.0.1.0/24",
    network_security_group=network.NetworkSecurityGroupArgs(id=nsg.id),
)

# 4. Public IP (Standard SKU requires Static)
public_ip = network.PublicIPAddress(
    "pip",
    resource_group_name=resource_group.name,
    location=resource_group.location,
    sku=network.PublicIPAddressSkuArgs(name="Standard"),
    public_ip_allocation_method="Static",
)

# 5. Network interface
nic = network.NetworkInterface(
    "nic",
    resource_group_name=resource_group.name,
    location=resource_group.location,
    ip_configurations=[
        network.NetworkInterfaceIPConfigurationArgs(
            name="ipconfig1",
            subnet=network.SubnetArgs(id=subnet.id),
            private_ip_allocation_method="Dynamic",
            public_ip_address=network.PublicIPAddressArgs(id=public_ip.id),
        )
    ],
)

# 6. Virtual machine
vm = compute.VirtualMachine(
    "vm",
    resource_group_name=resource_group.name,
    location=resource_group.location,
    vm_name=f"vm-lab-{env}",
    hardware_profile=compute.HardwareProfileArgs(vm_size="Standard_B1s"),
    storage_profile=compute.StorageProfileArgs(
        image_reference=compute.ImageReferenceArgs(
            publisher="Canonical",
            offer="ubuntu-24_04-lts",
            sku="server",
            version="latest",
        ),
        os_disk=compute.OSDiskArgs(
            create_option="FromImage",
            delete_option="Delete",
            managed_disk=compute.ManagedDiskParametersArgs(
                storage_account_type="Standard_LRS"
            ),
        ),
    ),
    os_profile=compute.OSProfileArgs(
        computer_name=f"vm-lab-{env}",
        admin_username=admin_user,
        linux_configuration=compute.LinuxConfigurationArgs(
            disable_password_authentication=True,
            ssh=compute.SshConfigurationArgs(
                public_keys=[
                    compute.SshPublicKeyArgs(
                        path=f"/home/{admin_user}/.ssh/authorized_keys",
                        key_data=ssh_key,
                    )
                ]
            ),
        ),
    ),
    network_profile=compute.NetworkProfileArgs(
        network_interfaces=[
            compute.NetworkInterfaceReferenceArgs(id=nic.id, primary=True)
        ]
    ),
    tags={"environment": env, "managed-by": "pulumi"},
)

# 7. Outputs
pulumi.export("vm_name", vm.name)
pulumi.export("public_ip", public_ip.ip_address)
pulumi.export("ssh_command", public_ip.ip_address.apply(lambda ip: f"ssh {admin_user}@{ip}"))
```

### What each part does

| Resource | Purpose |
|----------|---------|
| `NetworkSecurityGroup` | Firewall: only your IP can reach port 22 |
| `PublicIPAddress` | A fixed address so you can reach the VM from the internet |
| `NetworkInterface` | The VM's network card; joins subnet and public IP |
| `VirtualMachine` | The compute resource, referencing the NIC |
| `delete_option="Delete"` | Removes the OS disk with the VM |
| `disable_password_authentication` | Forces SSH key login |

## Step 3: Preview and deploy

```bash
pulumi preview
pulumi up
```

Expect 7 resources (plus the stack). The VM takes 1 to 3 minutes.

## Step 4: Connect

```bash
pulumi stack output ssh_command
ssh -i ~/.ssh/lab_vm azureuser@$(pulumi stack output public_ip)
```

Inside the VM, try:

```bash
hostname
cat /etc/os-release
exit
```

## Step 5: Verify from the Azure CLI

```bash
az vm list -g rg-lab-vm-dev -o table
az vm get-instance-view -g rg-lab-vm-dev -n vm-lab-dev \
  --query "instanceView.statuses[1].displayStatus" -o tsv
```

Expected: `VM running`.

---

## Exercises

1. **Install nginx at boot**: add `custom_data` to `OSProfileArgs`:

   ```python
   import base64
   init = "#!/bin/bash\napt-get update && apt-get install -y nginx\n"
   # inside OSProfileArgs(...):
   custom_data=base64.b64encode(init.encode()).decode(),
   ```

   Then add an NSG rule for port 80 and browse to the public IP.
2. **Resize the VM**: change `vm_size` to `Standard_B2s` and run `pulumi up`.
3. **Make the VM size a config value**: `config.get("vmSize") or "Standard_B1s"`.
4. **Stop charges without deleting**: `az vm deallocate -g rg-lab-vm-dev -n vm-lab-dev`.

## Clean up (important)

```bash
pulumi destroy
pulumi stack rm dev
```

Confirm nothing is left: `az group exists -n rg-lab-vm-dev` should print `false`.

## Troubleshooting

| Problem | Fix |
|---------|-----|
| `SkuNotAvailable` for `Standard_B1s` | Try `Standard_B2s`, or another region. List sizes: `az vm list-skus -l centralindia --size Standard_B -o table` |
| `Permission denied (publickey)` | Use `-i` with the matching private key; confirm `sshPublicKey` holds the **public** key |
| SSH times out | Your IP changed. Update `myIp` and run `pulumi up` |
| `InvalidParameter ... SSH key` | The key must be one line starting with `ssh-ed25519` or `ssh-rsa` |
| Quota error for vCPUs | Request a quota increase or pick a smaller size |

## Key takeaways

- A VM is a bundle: network, IP, NIC and compute are separate resources.
- Pulumi wires dependencies automatically from the outputs you pass (`nic.id`, `subnet.id`).
- Use SSH keys and a restricted NSG. Never expose SSH to the whole internet.
- Always `pulumi destroy` after a lab to avoid surprise charges.
