# Lab 1: Install the agent on vm2 and sign its certificate

Needs: Lab 0 complete. Each step says which VM to use.

## Step 1 (vm2): Make vm-1 resolvable

On `vm-1` you noted its IP. On `vm2`:

```bash
sudo vi /etc/hosts
```

Press `G` then `o` to open a new line at the end, and type this line, using the IP of vm-1:

```text
10.0.0.2 vm-1 puppet
```

Press `Esc`, type `:wq`, press `Enter`.

## Step 2 (vm2): Test the network

```bash
ping -c 2 vm-1
nc -zv vm-1 8140
```

Expected: the port test succeeds. If it fails, open TCP 8140 in your VPC or host firewall. If `nc` is missing: `sudo apt-get install -y netcat-openbsd`.

## Step 3 (vm2): Download the release package

Use the same file name you used on vm-1:

```bash
wget --content-disposition https://apt-puppetcore.puppet.com/public/puppet9-release-noble.deb
```

## Step 4 (vm2): Install the release package

```bash
sudo dpkg -i puppet9-release-noble.deb
```

## Step 5 (vm2): Add your API key

Use the same key you created in Lab 0, Step 1 (Forge profile, **API Keys**).

```bash
sudo vi /etc/apt/auth.conf.d/apt-puppetcore-puppet.conf
```

Contents (replace only `YOUR_API_KEY`):

```text
machine apt-puppetcore.puppet.com
login forge-key
password YOUR_API_KEY
```

```bash
sudo chmod 600 /etc/apt/auth.conf.d/apt-puppetcore-puppet.conf
sudo apt-get update
```

## Step 6 (vm2): Install the agent

```bash
sudo apt-get install -y puppet-agent
sudo ln -s /opt/puppetlabs/bin/puppet /usr/local/bin/puppet
sudo ln -s /opt/puppetlabs/bin/facter /usr/local/bin/facter
puppet --version
```

## Step 7 (vm2): Point the agent at the server

```bash
sudo puppet config set server vm-1 --section main
sudo puppet config set certname vm2 --section main
cat /etc/puppetlabs/puppet/puppet.conf
```

## Step 8 (vm2): First run

```bash
sudo puppet agent -t
```

Expected: the agent says its certificate is not signed yet. This is normal.

## Step 9 (vm-1): Sign the certificate

```bash
sudo puppetserver ca list
sudo puppetserver ca sign --certname vm2
sudo puppetserver ca list --all
```

`vm2` is now listed as signed.

## Step 10 (vm2): Second run

```bash
sudo puppet agent -t
```

Expected: `Caching catalog for vm2` and `Applied catalog`. There is no code yet, so nothing changes.

## Step 11 (vm2): Look at facts

```bash
facter os
facter networking.hostname
facter memory.system.total
```

Facts are the data the server uses to build the catalog for this node.

## Done when

- [ ] `vm2` is signed on the server
- [ ] `sudo puppet agent -t` on vm2 ends with `Applied catalog`

Next: `lab2.md`.
