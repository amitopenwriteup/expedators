# Lab 4: Forge module (Apache on node1, Rocky Linux)

Needs Lab 3.

## Step 1 (vm-1): Install the module

```bash
sudo puppet module install puppetlabs-apache --environment production
```

If it fails on a version requirement, add `--ignore-requirements`.

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

node 'node1' {
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

```bash
sudo puppet parser validate manifests/site.pp
```

## Step 4 (node1): Run the agent

```bash
sudo puppet agent -t
```

## Step 5 (node1): Verify

```bash
systemctl is-active httpd
curl http://localhost/
```

Expected: `active` and `Hello from Puppet`.

## Step 6 (node1): Run again

```bash
sudo puppet agent -t
```

Expected: no changes.

## Step 7 (node1): Break it and watch Puppet repair it

```bash
sudo systemctl stop httpd
echo defaced | sudo tee /var/www/html/index.html
sudo puppet agent -t
curl http://localhost/
```

Expected: httpd is running and the page says `Hello from Puppet`.

## Step 8: Cleanup (optional)

On vm-1, remove the `node 'node1'` block from `site.pp`. On node1:

```bash
sudo puppet agent -t
sudo systemctl disable --now httpd
sudo dnf remove -y httpd
sudo rm -rf /etc/httpd
```

