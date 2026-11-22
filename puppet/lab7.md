# Lab 7: Troubleshooting and cleanup

## Where to look

| Problem area | Command |
|---|---|
| Server service | `sudo systemctl status puppetserver` |
| Server log | `sudo tail -100 /var/log/puppetlabs/puppetserver/puppetserver.log` |
| Agent output | `sudo puppet agent -t --debug` |
| Agent service | `sudo journalctl -u puppet -n 50 --no-pager` |
| Config values | `puppet config print server certname` |
| Code syntax | `puppet parser validate FILE` |

## Fault 1: syntax error in code

### Step 1 (vm-1): Save a good copy

```bash
cd /etc/puppetlabs/code/environments/production
sudo cp modules/lab/manifests/base.pp /root/base.pp.good
```

### Step 2 (vm-1): Break the file

```bash
sudo vi modules/lab/manifests/base.pp
```

Press `G` to jump to the last line, press `dd` to delete the `}` line. Type `:wq` and press `Enter`.

### Step 3 (vm-1): See the error

```bash
sudo puppet parser validate modules/lab/manifests/base.pp
```

Expected: a syntax error with a line number.

### Step 4 (vm2): See what the agent sees

```bash
sudo puppet agent -t
```

Expected: `Could not retrieve catalog from remote server` with a compile error. The node keeps its old state, so nothing is half applied.

### Step 5 (vm-1): Restore

```bash
sudo cp /root/base.pp.good modules/lab/manifests/base.pp
sudo puppet parser validate modules/lab/manifests/base.pp
```

## Fault 2: lost certificate

### Step 1 (vm2): Remove the agent identity

```bash
sudo systemctl stop puppet
sudo rm -rf /etc/puppetlabs/puppet/ssl
sudo puppet agent -t
```

Expected: an error. The server still holds the old certificate for vm2.

### Step 2 (vm-1): Remove the old certificate

```bash
sudo puppetserver ca clean --certname vm2
```

### Step 3 (vm2): Request a new one

```bash
sudo puppet agent -t
```

### Step 4 (vm-1): Sign it

```bash
sudo puppetserver ca list
sudo puppetserver ca sign --certname vm2
```

### Step 5 (vm2): Run and restart the service

```bash
sudo puppet agent -t
sudo systemctl start puppet
```

## Fault 3: name or port problem

On vm2:

```bash
getent hosts vm-1
nc -zv vm-1 8140
```

Both must succeed. Typical causes: a missing line in `/etc/hosts`, a firewall blocking TCP 8140, or a wrong `server` value in `puppet.conf`.

A message that the server certificate does not match the host name means the agent uses a name that is not in the server certificate. Use `server = vm-1` exactly as set in Lab 1.

## Handy commands

```bash
sudo puppet agent -t --debug
sudo puppet agent -t --noop
sudo puppet lookup lab::base::message --node vm2 --explain
facter networking
```

## Cleanup (when all labs are done)

### On vm2

```bash
sudo systemctl disable --now puppet
sudo apt-get purge -y puppet-agent
sudo rm -rf /etc/puppetlabs /opt/puppetlabs /opt/lab
sudo rm /usr/local/bin/puppet /usr/local/bin/facter
sudo rm /etc/apt/auth.conf.d/apt-puppetcore-puppet.conf
```

### On vm-1

```bash
sudo systemctl disable --now puppetserver
sudo apt-get purge -y puppetserver puppet-agent
sudo rm -rf /etc/puppetlabs /opt/puppetlabs /var/log/puppetlabs /opt/lab
sudo rm /usr/local/bin/puppet /usr/local/bin/facter /usr/local/bin/puppetserver
sudo rm /etc/apt/auth.conf.d/apt-puppetcore-puppet.conf
```

Then remove the `vm-1 puppet` line from `/etc/hosts` on both VMs with `sudo vi /etc/hosts`.

## Lab series recap

| Lab | Topic |
|---|---|
| 0 | Server install |
| 1 | Agent and certificates |
| 2 | Resources and idempotency |
| 3 | Modules and site.pp |
| 4 | Forge module |
| 5 | Hiera and templates |
| 6 | Drift, noop, schedules |
| 7 | Troubleshooting and cleanup |
