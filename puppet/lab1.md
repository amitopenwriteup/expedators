# Lab 1 (Rocky Linux): Install the agent on vm2 and sign its certificate

Needs: Lab 0 complete. Each step says which VM to use.

## Step 1 (vm2): Make vm-1 resolvable

```bash
sudo vi /etc/hosts
```

Press `G` then `o`, type this line (use the IP of vm-1):

```text
10.0.0.2 vm-1 puppet
```

Press `Esc`, type `:wq`, press `Enter`.

## Step 2 (vm2): Test the network

```bash
ping -c 2 vm-1
nc -zv vm-1 8140
```

If `nc` is missing:

```bash
sudo dnf install -y nmap-ncat
```

If the port test fails, open TCP 8140 on **vm-1**:

```bash
sudo firewall-cmd --permanent --add-port=8140/tcp
sudo firewall-cmd --reload
```

## Step 3 (vm2): Download the release package

```bash
wget --content-disposition https://yum-puppetcore.puppet.com/public/puppet9-release-el-9.noarch.rpm
```

## Step 4 (vm2): Install the release package

```bash
sudo rpm -ivh puppet9-release-el-9.noarch.rpm
```

## Step 5 (vm2): Add your API key

Use the same key from Lab 0, Step 1.

```bash
sudo vi /etc/yum.repos.d/puppetcore-puppet9.repo
```

Find the `[puppetcore9]` section and set these two lines (replace only `YOUR_API_KEY`):

```text
username=forge-key
password=YOUR_API_KEY
```

Then:

```bash
sudo chmod 600 /etc/yum.repos.d/puppetcore-puppet9.repo
sudo dnf clean all
sudo dnf makecache
```

If the repo file name differs, check with `ls /etc/yum.repos.d/`.

## Step 6 (vm2): Install the agent

```bash
sudo dnf install -y puppet-agent
sudo ln -s /opt/puppetlabs/bin/puppet /usr/local/bin/puppet
sudo ln -s /opt/puppetlabs/bin/facter /usr/local/bin/facter
puppet --version
```

Check you are using the new agent:

```bash
which -a puppet
```

Expected: `/opt/puppetlabs/bin/puppet` and version 9.x. If an older version shows:

```bash
sudo dnf remove -y puppet
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

Note the certificate fingerprint printed here. Leave this terminal open and go to `vm-1`.

## Step 9 (vm-1): Sign the certificate

Run on **vm-1**, not vm2:

```bash
sudo puppetserver ca list
sudo puppetserver ca sign --certname vm2
sudo puppetserver ca list --all
```

Check that the fingerprint from `ca list` matches the one on vm2.

If `vm2` is not listed, on vm2 run:

```bash
sudo puppet agent --configprint server
getent hosts vm-1
nc -zv vm-1 8140
```

## Step 10 (vm2): Second run

```bash
sudo puppet agent -t
```

Expected: `Caching catalog for vm2` and `Applied catalog`.

## Step 11 (vm2): Look at facts

```bash
facter os
facter networking.hostname
facter memory.system.total
```

## Done when

- [ ] `vm2` is signed on the server
- [ ] `sudo puppet agent -t` on vm2 ends with `Applied catalog`

Next: `lab2.md`.
