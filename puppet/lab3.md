# Lab 3: Server-driven configuration (module and site.pp)

Write a module on **vm-1** and apply it to **vm2** through the agent. Each step names the VM.

**Editing:** `sudo vi FILE`. Quick keys: `i` to type, `Esc` then `:wq` and `Enter` to save and exit (full table in Lab 0).

---

## Step 1 (vm-1): Go to the code folder

Find where this Puppet install keeps its environments:

```bash
puppet config print environmentpath
```

On these VMs it prints `/etc/puppet/code/environments`. Create the `production` environment if it does not exist yet, then go into it:

```bash
sudo mkdir -p /etc/puppet/code/environments/production/{manifests,modules}
cd /etc/puppetlabs/code/environments/production
ls
```

**Expected:** `manifests` and `modules`. Agents use the `production` environment by default.

> If `environmentpath` printed a different path on your VM, use that path instead of `/etc/puppet/code/environments`. All later steps are relative to the `production` folder.

---

## Step 2 (vm-1): Create the module folders

```bash
sudo mkdir -p modules/lab/manifests
```

---

## Step 3 (vm-1): Write the class

```bash
sudo vi modules/lab/manifests/base.pp
```

File content. The class name must match the file path: `lab::base` lives in `lab/manifests/base.pp`.

```puppet
class lab::base {
  package { ['tree', 'jq']:
    ensure => installed,
  }

  service { 'crond':
    ensure => running,
    enable => true,
  }

  file { '/opt/lab':
    ensure => directory,
  }

  file { '/opt/lab/info.txt':
    ensure  => file,
    content => "Managed by Puppet\n",
  }
}
```

---

## Step 4 (vm-1): Check the syntax

```bash
sudo puppet parser validate modules/lab/manifests/base.pp
```

**Expected:** no output. No output means the syntax is fine.

---

## Step 5 (vm-1): Tell nodes to use the class

```bash
sudo vi manifests/site.pp
```

File content:

```puppet
node default {
  include lab::base
}
```

`node default` matches every node that has no more specific `node` block.

Validate it:

```bash
sudo puppet parser validate manifests/site.pp
```

---

## Step 6 (vm-1): Make sure the Puppet server is running

The agent on vm2 pulls its catalog from the Puppet server on vm-1.

```bash
sudo systemctl status puppetserver --no-pager | head -5
```

**Expected:** `active (running)`.

> If the service is not found, the server package is not installed. Check with `dpkg -l | grep -i puppet` before continuing.

---

## Step 7 (vm2): Run the agent

```bash
sudo puppet agent -t
```

**Expected:** `Applied catalog`, with the packages installed and `/opt/lab/info.txt` created.

---

## Step 8 (vm2): Check the result

```bash
cat /opt/lab/info.txt
which tree
which jq
```

**Expected:** `Managed by Puppet`, then a path for each of `tree` and `jq`.

---

## Step 9 (vm2): Run again

```bash
sudo puppet agent -t
```

**Expected:** no changes. The system already matches the code, so Puppet does nothing.

---

## Step 10 (vm-1, optional): Apply it to the server too

```bash
sudo puppet agent -t
cat /opt/lab/info.txt
```

---

## Troubleshooting

| Symptom on vm2 | Likely cause | Fix |
|---|---|---|
| `Could not find class lab::base` | The server reads code from a different folder than the one you edited | Run `sudo grep -r codedir /etc/puppetlabs/puppet/ /etc/puppet/ 2>/dev/null`, then move the `production` folder to the path the server uses |
| Agent applies nothing new after you edit code | Server is caching the environment | `sudo systemctl restart puppetserver`, then run the agent again |
| Syntax error from `puppet parser validate` | Typo in the `.pp` file | Fix the line named in the error and validate again |
| Certificate or connection error | Agent is not signed or cannot reach vm-1 | Revisit the setup from the earlier lab |

---

## Key ideas

- A module is a folder with `manifests`, `templates`, `files` and `data`.
- `site.pp` decides which classes each node gets.
- You never log in to vm2 to change its configuration. The agent pulls it.

**Next:** `lab4.md`
