# Lab 3: Server-driven configuration (module and site.pp)

Write a module on vm-1 and apply it to vm2 through the agent. Each step names the VM.

Editing: `sudo vi FILE`. Quick keys: `i` to type, `Esc` then `:wq` and `Enter` to save and exit (full table in Lab 0).

## Step 1 (vm-1): Go to the code folder

```bash
cd /etc/puppetlabs/code/environments/production
ls
```

Expected: `manifests` and `modules`. Agents use the `production` environment by default.

## Step 2 (vm-1): Create the module folders

```bash
sudo mkdir -p modules/lab/manifests
```

## Step 3 (vm-1): Write the class

```bash
sudo vi modules/lab/manifests/base.pp
```

File content (the class name must match the file: `lab::base` is in `lab/manifests/base.pp`):

```puppet
class lab::base {
  package { ['tree', 'jq']:
    ensure => installed,
  }

  service { 'cron':
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

## Step 4 (vm-1): Check the syntax

```bash
sudo puppet parser validate modules/lab/manifests/base.pp
```

No output means it is fine.

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

`node default` matches every node without a more specific `node` block.

```bash
sudo puppet parser validate manifests/site.pp
```

## Step 6 (vm2): Run the agent

```bash
sudo puppet agent -t
```

Expected: `Applied catalog`, with the packages installed and `/opt/lab/info.txt` created.

## Step 7 (vm2): Check the result

```bash
cat /opt/lab/info.txt
which tree
which jq
```

## Step 8 (vm2): Run again

```bash
sudo puppet agent -t
```

Expected: no changes.

## Step 9 (vm-1, optional): Apply it to the server too

```bash
sudo puppet agent -t
cat /opt/lab/info.txt
```

## Key ideas

- A module is a folder with `manifests`, `templates`, `files` and `data`.
- `site.pp` decides which classes each node gets.
- You never log in to vm2 to change its configuration. The agent pulls it.

Next: `lab4.md`.
