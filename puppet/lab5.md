# Lab 5: Hiera data and templates

Keep values in data files, pass them to the class as a parameter, and render them with a template. Needs Lab 3.

Edit on vm-1, run the agent on vm2.

## Step 1 (vm-1): Go to the code folder

```bash
cd /etc/puppet/code/environments/production
```

## Step 2 (vm-1): Create the data folders

```bash
sudo mkdir -p data/nodes
```

## Step 3 (vm-1): Hiera config

```bash
sudo vi hiera.yaml
```

File content:

```yaml
---
version: 5
defaults:
  datadir: data
  data_hash: yaml_data
hierarchy:
  - name: "Per-node data"
    path: "nodes/%{trusted.certname}.yaml"
  - name: "Common data"
    path: "common.yaml"
```

Lookup order: the node file first, then `common.yaml`. The first match wins.

## Step 4 (vm-1): Common data

```bash
sudo vi data/common.yaml
```

File content:

```yaml
---
lab::base::message: "Hello from common data"
```

## Step 5 (vm-1): Data for vm2 only

```bash
sudo vi data/nodes/vm2.yaml
```

File content:

```yaml
---
lab::base::message: "Hello from vm2 data"
```

## Step 6 (vm-1): Create the template

```bash
sudo mkdir modules/lab/templates
sudo vi modules/lab/templates/info.epp
```

File content:

```text
<%- | String $message | -%>
<%= $message %>
Host: <%= $facts['networking']['hostname'] %>
```

## Step 7 (vm-1): Update the class

```bash
sudo vi modules/lab/manifests/base.pp
```

Replace the whole file with:

```puppet
class lab::base (
  String $message = 'default message',
) {
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
    content => epp('lab/info.epp', { 'message' => $message }),
  }
}
```

`lab::base::message` in Hiera fills the `message` parameter automatically.

```bash
sudo puppet parser validate modules/lab/manifests/base.pp
```

## Step 8 (vm-1): Test the lookup

```bash
sudo puppet lookup lab::base::message --node vm2 --environment production
sudo puppet lookup lab::base::message --node vm-1 --environment production
```

Expected: vm2 returns its own message, vm-1 returns the common one. (Lookup by node needs that node's facts, which exist after its first agent run.)

## Step 9 (vm2): Run the agent

```bash
sudo puppet agent -t
cat /opt/lab/info.txt
```

Expected:

```text
Hello from vm2 data
Host: vm2
```

## Step 10 (vm-1): Run the agent here too

```bash
sudo puppet agent -t
cat /opt/lab/info.txt
```

Expected: the common message and `Host: vm-1`.

## Step 11 (vm-1): Change the data only

```bash
sudo vi data/nodes/vm2.yaml
```

Press `i` and change the message to `Hello from vm2 data, version 2`. Press `Esc`, type `:wq`, press `Enter`.

## Step 12 (vm2): Run again

```bash
sudo puppet agent -t
cat /opt/lab/info.txt
```

Expected: the file changes, although the code did not.

## Key ideas

- Code says what to manage. Hiera says which values to use.
- Node data overrides common data.
- A template turns facts and parameters into file content.

Next: `lab6.md`.
