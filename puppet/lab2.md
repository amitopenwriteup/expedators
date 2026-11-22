# Lab 2: Resources and idempotency (puppet apply)

No server is involved here. Run everything on **vm2**.

A resource describes desired state: what should exist and how.

## Step 1: Inspect the system

```bash
sudo puppet resource service cron
sudo puppet resource package tree
sudo puppet resource user root
```

## Step 2: Create a work folder

```bash
mkdir ~/puppet-lab
cd ~/puppet-lab
```

## Step 3: Write a manifest with one file resource

```bash
vi hello.pp
```

File content:

```puppet
file { '/tmp/hello.txt':
  ensure  => file,
  content => "Hello from Puppet\n",
  mode    => '0644',
}
```

## Step 4: Check the syntax

```bash
puppet parser validate hello.pp
```

No output means the syntax is fine.

## Step 5: Apply it

```bash
sudo puppet apply hello.pp
cat /tmp/hello.txt
```

Expected: a notice that the file content was defined, and the text in the file.

## Step 6: Apply again

```bash
sudo puppet apply hello.pp
```

Expected: **no changes**. Puppet acts only when reality differs from the code. This is idempotency.

## Step 7: Break the file by hand

```bash
sudo vi /tmp/hello.txt
```

Press `i`, delete the text and type `tampered`. Press `Esc`, type `:wq`, press `Enter`.

## Step 8: Preview the repair

```bash
sudo puppet apply hello.pp --noop
```

Expected: it says it would change the file. Nothing is changed yet. Check:

```bash
cat /tmp/hello.txt
```

## Step 9: Repair

```bash
sudo puppet apply hello.pp
cat /tmp/hello.txt
```

The original text is back.

## Step 10: Package, file, service

```bash
vi web.pp
```

File content:

```puppet
package { 'nginx':
  ensure => installed,
}

file { '/var/www/html/index.nginx-debian.html':
  ensure  => file,
  content => "Managed by Puppet\n",
  require => Package['nginx'],
}

service { 'nginx':
  ensure    => running,
  enable    => true,
  subscribe => File['/var/www/html/index.nginx-debian.html'],
}
```

```bash
puppet parser validate web.pp
sudo puppet apply web.pp
curl http://localhost/
```

Expected: the page says `Managed by Puppet`.

| Keyword | Meaning |
|---|---|
| `require` | Apply the other resource first |
| `subscribe` | Refresh or restart when the other resource changes |

## Step 11: Use a fact

```bash
facter networking.hostname
vi fact.pp
```

File content:

```puppet
notice("This node is ${facts['networking']['hostname']}")
```

```bash
sudo puppet apply fact.pp
```

## Step 12: Clean up

```bash
sudo systemctl stop nginx
sudo apt-get remove -y nginx
sudo rm /tmp/hello.txt
```

## Key ideas

- A manifest describes the end state, not the steps.
- Running it again is safe.
- Always run `puppet parser validate` first and `--noop` to preview.

Next: `lab3.md`.
