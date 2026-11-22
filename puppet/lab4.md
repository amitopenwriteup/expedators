# Lab 4: Forge module (Apache web server on vm2)

Reuse a community module from the Puppet Forge. Needs Lab 3.

## Step 1 (vm-1): Install the module

```bash
sudo puppet module install puppetlabs-apache --environment production
```

The Forge is public, so no API key is needed. Dependencies such as `puppetlabs-stdlib` come with it.

If it refuses because of a Puppet version requirement, retry for this lab with:

```bash
sudo puppet module install puppetlabs-apache --environment production --ignore-requirements
```

## Step 2 (vm-1): Check the module

```bash
sudo puppet module list --environment production
```

## Step 3 (vm-1): Edit site.pp

```bash
cd /etc/puppetlabs/code/environments/production
sudo vi manifests/site.pp
```

Replace the whole file with:

```puppet
node default {
  include lab::base
}

node 'vm2' {
  include lab::base

  class { 'apache':
  }

  file { '/var/www/html/index.html':
    ensure  => file,
    content => "Hello from Puppet\n",
    require => Class['apache'],
  }
}
```

`node 'vm2'` is more specific than `node default`, so vm2 uses it. That is why `include lab::base` appears again.

```bash
sudo puppet parser validate manifests/site.pp
```

## Step 4 (vm2): Run the agent

```bash
sudo puppet agent -t
```

## Step 5 (vm2): Verify

```bash
systemctl is-active apache2
curl http://localhost/
```

Expected: `active` and `Hello from Puppet`.

## Step 6 (vm2): Run again

```bash
sudo puppet agent -t
```

Expected: no changes.

## Step 7 (vm2): Break it and watch Puppet repair it

```bash
sudo systemctl stop apache2
sudo vi /var/www/html/index.html
```

Press `i`, change the text to `defaced`. Press `Esc`, type `:wq`, press `Enter`. Then:

```bash
sudo puppet agent -t
curl http://localhost/
```

Expected: Apache is running again and the page is restored.

## Step 8: Cleanup (optional)

On vm-1 remove the `node 'vm2'` block from `site.pp`. On vm2:

```bash
sudo puppet agent -t
sudo apt-get remove -y apache2
```

Puppet does not undo changes by itself when code is removed.

Next: `lab5.md`.
