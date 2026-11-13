# Puppet Lab: Manifests, Resources, Desired State, Configuration Drift & Troubleshooting

**Environment:** 3 Linux VMs (Ubuntu 22.04 LTS) | **Puppet:** 8.x (Open Source) | **Duration:** ~3 hours | **Level:** Beginner to Intermediate

---

## 1. Lab Objectives

By the end of this lab you will be able to:

1. Build a Puppet server and connect two agents.
2. Write **manifests** that declare **resources** (package, file, service, user).
3. Explain and demonstrate **desired state** and **idempotency**.
4. Detect and remediate **configuration drift** (`--noop`, scheduled runs).
5. **Troubleshoot** common Puppet problems (certificates, syntax, dependencies, connectivity).

---

## 2. Lab Topology

| VM | Hostname | Role | Example IP | Suggested RAM |
|----|----------|------|-----------|---------------|
| VM1 | `puppet.lab.local` | Puppet Server (primary) | 192.168.56.10 | 3 GB (min 2 GB) |
| VM2 | `web01.lab.local` | Agent - web node | 192.168.56.11 | 1 GB |
| VM3 | `app01.lab.local` | Agent - app node | 192.168.56.12 | 1 GB |

```
            +-----------------------+
            |  puppet.lab.local     |
            |  puppetserver :8140   |
            +-----------+-----------+
                        |
          +-------------+-------------+
          |                           |
 +--------+---------+       +---------+--------+
 | web01.lab.local  |       | app01.lab.local  |
 | puppet-agent     |       | puppet-agent     |
 +------------------+       +------------------+
```

> Adjust the IP addresses to match your hypervisor network (host-only or bridged). All three VMs must be able to reach each other.

---

## 3. Prerequisites

- 3 Ubuntu 22.04 VMs with a sudo user and internet access.
- Basic Linux command-line knowledge.
- Time is synchronized (checked in Part 0).

---

## Part 0: Prepare the VMs

### 0.1 Set hostnames (run on each VM with its own name)

```bash
sudo hostnamectl set-hostname puppet.lab.local     # VM1
sudo hostnamectl set-hostname web01.lab.local      # VM2
sudo hostnamectl set-hostname app01.lab.local      # VM3
```

### 0.2 Add name resolution (on **all three** VMs)

```bash
sudo tee -a /etc/hosts <<'EOF'
192.168.56.10 puppet.lab.local puppet
192.168.56.11 web01.lab.local web01
192.168.56.12 app01.lab.local app01
EOF
```

### 0.3 Verify connectivity and time

```bash
ping -c 2 puppet.lab.local
ping -c 2 web01.lab.local
ping -c 2 app01.lab.local
timedatectl        # "System clock synchronized: yes"
```

**Checkpoint:** All names resolve and clocks are in sync. (Certificates fail if clocks differ significantly.)

---

## Part 1: Install Puppet

### 1.1 Puppet Server (VM1)

```bash
wget https://apt.puppet.com/puppet8-release-jammy.deb
sudo dpkg -i puppet8-release-jammy.deb
sudo apt update
sudo apt install -y puppetserver
```

Reduce JVM memory for a lab VM:

```bash
sudo sed -i 's/^JAVA_ARGS=.*/JAVA_ARGS="-Xms512m -Xmx512m"/' /etc/default/puppetserver
```

Start and enable the service, open port 8140 if a firewall is active:

```bash
sudo systemctl enable --now puppetserver
sudo systemctl status puppetserver --no-pager
sudo ufw allow 8140/tcp        # only if ufw is enabled
```

Add Puppet binaries to your PATH:

```bash
echo 'export PATH=$PATH:/opt/puppetlabs/bin' | sudo tee /etc/profile.d/puppet.sh
source /etc/profile.d/puppet.sh
puppetserver --version
```

### 1.2 Puppet Agents (VM2 and VM3)

```bash
wget https://apt.puppet.com/puppet8-release-jammy.deb
sudo dpkg -i puppet8-release-jammy.deb
sudo apt update
sudo apt install -y puppet-agent
echo 'export PATH=$PATH:/opt/puppetlabs/bin' | sudo tee /etc/profile.d/puppet.sh
source /etc/profile.d/puppet.sh
```

Point the agent at the server:

```bash
sudo puppet config set server puppet.lab.local --section main
sudo systemctl enable --now puppet
```

### 1.3 Sign agent certificates (VM1)

Trigger a certificate request from each agent:

```bash
sudo /opt/puppetlabs/bin/puppet agent -t       # run on web01 and app01
```

On the server, list and sign:

```bash
sudo puppetserver ca list
sudo puppetserver ca sign --certname web01.lab.local
sudo puppetserver ca sign --certname app01.lab.local
```

Re-run on each agent:

```bash
sudo puppet agent -t
```

**Checkpoint:** Both agents complete a run with no errors (an empty catalog is fine at this stage).

---

## Part 2: Resources

A **resource** is the basic unit of Puppet: a single thing you manage (a package, file, service, user). Each has a **type**, a **title**, and **attributes**.

```puppet
type { 'title':
  attribute => value,
}
```

### 2.1 Inspect resources with the Resource Abstraction Layer (RAL)

On any agent:

```bash
puppet resource user root
puppet resource service ssh
puppet resource package curl
puppet resource file /etc/hosts
```

**Questions:**
1. What attributes does `puppet resource user root` show?
2. How is `ensure` represented for the `ssh` service?

### 2.2 Use `puppet resource` to change state (on web01)

```bash
sudo puppet resource package tree ensure=present
sudo puppet resource package tree ensure=absent
```

### 2.3 Core resource types used in this lab

| Type | Purpose | Key attributes |
|------|---------|----------------|
| `package` | Install/remove software | `ensure`, `name` |
| `file` | Files, dirs, content, perms | `ensure`, `content`, `source`, `mode`, `owner`, `group` |
| `service` | Manage daemons | `ensure`, `enable` |
| `user` | Local accounts | `ensure`, `shell`, `home`, `managehome` |
| `exec` | Run commands (last resort) | `command`, `creates`, `unless`, `onlyif` |

---

## Part 3: Manifests

A **manifest** is a `.pp` file containing Puppet code. Manifests can be applied locally (`puppet apply`) or compiled into catalogs by the server.

### 3.1 Your first manifest: local apply (on web01)

```bash
mkdir -p ~/manifests && cd ~/manifests
cat > hello.pp <<'EOF'
file { '/tmp/hello.txt':
  ensure  => file,
  content => "Hello from Puppet on ${facts['networking']['fqdn']}\n",
  mode    => '0644',
}
EOF

puppet parser validate hello.pp
sudo puppet apply hello.pp
cat /tmp/hello.txt
```

Run `sudo puppet apply hello.pp` again and observe: **no changes** are reported. This is idempotency.

### 3.2 Create a module with classes (on the Puppet Server, VM1)

Classes group resources into reusable units. Create a module called `lab`:

```bash
cd /etc/puppetlabs/code/environments/production/modules
sudo mkdir -p lab/manifests
```

**`lab/manifests/baseline.pp`** (applied to all nodes):

```bash
sudo tee lab/manifests/baseline.pp <<'EOF'
class lab::baseline {
  package { ['curl', 'vim', 'tree']:
    ensure => installed,
  }

  user { 'labuser':
    ensure     => present,
    shell      => '/bin/bash',
    home       => '/home/labuser',
    managehome => true,
  }

  file { '/etc/motd':
    ensure  => file,
    owner   => 'root',
    group   => 'root',
    mode    => '0644',
    content => "Managed by Puppet - ${facts['networking']['hostname']}\nDo not edit manually.\n",
  }
}
EOF
```

**`lab/manifests/webserver.pp`** (for web01):

```bash
sudo tee lab/manifests/webserver.pp <<'EOF'
class lab::webserver {
  package { 'nginx':
    ensure => installed,
  }

  file { '/var/www/html/index.html':
    ensure  => file,
    mode    => '0644',
    content => "<h1>Hello from ${facts['networking']['hostname']} - managed by Puppet</h1>\n",
    require => Package['nginx'],
  }

  service { 'nginx':
    ensure    => running,
    enable    => true,
    subscribe => File['/var/www/html/index.html'],
  }
}
EOF
```

**`lab/manifests/appserver.pp`** (for app01):

```bash
sudo tee lab/manifests/appserver.pp <<'EOF'
class lab::appserver {
  package { 'chrony':
    ensure => installed,
  }

  service { 'chrony':
    ensure  => running,
    enable  => true,
    require => Package['chrony'],
  }

  file { '/opt/app':
    ensure => directory,
    mode   => '0755',
  }

  file { '/opt/app/app.conf':
    ensure  => file,
    mode    => '0640',
    content => "environment=lab\nlog_level=info\n",
    require => File['/opt/app'],
  }
}
EOF
```

### 3.3 Assign classes to nodes with `site.pp`

```bash
sudo tee /etc/puppetlabs/code/environments/production/manifests/site.pp <<'EOF'
node default {
  include lab::baseline
}

node 'web01.lab.local' {
  include lab::baseline
  include lab::webserver
}

node 'app01.lab.local' {
  include lab::baseline
  include lab::appserver
}
EOF
```

Validate all code:

```bash
sudo /opt/puppetlabs/bin/puppet parser validate \
  /etc/puppetlabs/code/environments/production/manifests/site.pp \
  /etc/puppetlabs/code/environments/production/modules/lab/manifests/*.pp
```

### 3.4 Apply on the agents

On **web01** and **app01**:

```bash
sudo puppet agent -t
```

### 3.5 Verify

```bash
# web01
curl http://localhost
cat /etc/motd
systemctl is-active nginx

# app01
cat /opt/app/app.conf
systemctl is-active chrony
id labuser
```

**Checkpoint:** web01 serves the Puppet-managed page; app01 has `/opt/app/app.conf`; both have `labuser` and a managed `/etc/motd`.

### 3.6 Resource relationships (concept)

| Mechanism | Meaning |
|-----------|---------|
| `require` / `before` | Ordering only |
| `notify` / `subscribe` | Ordering **plus** refresh (e.g., restart a service when a file changes) |
| Chaining arrows `->` / `~>` | Same, written as arrows |

**Question:** In `lab::webserver`, what happens to nginx if the `index.html` content changes?

---

## Part 4: Desired State & Idempotency

Puppet is **declarative**: you describe *what* the system should look like, not *how* to get there. Puppet compares **current state** to **desired state** and only changes what differs.

### 4.1 Observe idempotency

Run twice on web01:

```bash
sudo puppet agent -t
sudo puppet agent -t
```

The second run should report no resource changes. Check the summary:

```bash
sudo cat /opt/puppetlabs/puppet/cache/state/last_run_summary.yaml
```

### 4.2 Change the desired state on the server

Edit the app config on VM1:

```bash
sudo sed -i 's/log_level=info/log_level=debug/' \
  /etc/puppetlabs/code/environments/production/modules/lab/manifests/appserver.pp
```

On app01:

```bash
sudo puppet agent -t
cat /opt/app/app.conf       # log_level=debug now
```

### 4.3 Understand the output

Look at lines like:

```
Notice: /Stage[main]/Lab::Appserver/File[/opt/app/app.conf]/content: content changed ...
Notice: Applied catalog in 0.45 seconds
```

**Questions:**
1. What does `Stage[main]/Lab::Appserver/File[...]` tell you?
2. Why does Puppet change the file only when content differs?

### 4.4 Ensure absent

Add the following to `lab::baseline`, run an agent, then confirm `tree` is removed:

```puppet
package { 'tree':
  ensure => absent,
}
```

> You must remove `tree` from the earlier `['curl','vim','tree']` array first, or you will get a **Duplicate declaration** error (a useful troubleshooting preview!).

---

## Part 5: Configuration Drift

**Configuration drift** occurs when a system's actual state diverges from its declared desired state (manual edits, accidental changes, failed updates).

### 5.1 Create drift on web01

```bash
# 1. Edit a managed file
echo "HACKED" | sudo tee /var/www/html/index.html

# 2. Stop a managed service
sudo systemctl stop nginx

# 3. Change file permissions
sudo chmod 777 /etc/motd

# 4. Remove a managed package
sudo apt remove -y tree
```

Confirm the drift:

```bash
curl http://localhost       # fails: service stopped
ls -l /etc/motd
```

### 5.2 Detect drift without changing anything (`--noop`)

```bash
sudo puppet agent -t --noop
```

Look for lines such as:

```
Notice: /Stage[main]/Lab::Webserver/Service[nginx]/ensure: current_value 'stopped', should be 'running' (noop)
Notice: /Stage[main]/Lab::Baseline/File[/etc/motd]/mode: current_value '0777', should be '0644' (noop)
```

**Record** every drifted resource in the table below:

| Resource | Current value | Desired value |
|----------|---------------|---------------|
| | | |
| | | |
| | | |
| | | |

### 5.3 Remediate drift

```bash
sudo puppet agent -t
curl http://localhost       # back to the managed page
ls -l /etc/motd
systemctl is-active nginx
dpkg -l tree | tail -1
```

### 5.4 Automatic remediation on a schedule

By default the agent runs every 30 minutes. Shorten it for the lab (on web01):

```bash
sudo puppet config set runinterval 120 --section agent
sudo systemctl restart puppet
```

Cause drift again and wait up to two minutes:

```bash
sudo systemctl stop nginx
sleep 150
systemctl is-active nginx       # should be "active" again
```

### 5.5 Temporarily pause enforcement (maintenance window)

```bash
sudo puppet agent --disable "Maintenance - ticket 1234"
sudo systemctl stop nginx
sudo puppet agent -t            # refuses to run; shows the disable message
sudo puppet agent --enable
sudo puppet agent -t            # drift corrected
```

### 5.6 Audit-only mode for a resource (optional)

To monitor a file without enforcing it, use `audit` or `noop` metaparameters:

```puppet
file { '/etc/hosts':
  noop => true,       # report only; never change
}
```

**Discussion:**
1. Why is drift dangerous in production?
2. When might you intentionally use `--noop` or `--disable`?

---

## Part 6: Puppet Troubleshooting

### 6.1 Troubleshooting toolbox

| Task | Command |
|------|---------|
| Validate syntax | `puppet parser validate file.pp` |
| Lint style (if installed) | `puppet-lint file.pp` |
| Run with detail | `puppet agent -t --debug` |
| Stack traces | `puppet agent -t --trace` |
| Simulate changes | `puppet agent -t --noop` |
| Last run result | `cat /opt/puppetlabs/puppet/cache/state/last_run_summary.yaml` |
| Show facts | `puppet facts` or `facter -p` |
| Show config values | `puppet config print server certname runinterval` |
| Server logs | `sudo journalctl -u puppetserver -f` or `/var/log/puppetlabs/puppetserver/puppetserver.log` |
| Agent logs | `sudo journalctl -u puppet` |
| Cert management | `puppetserver ca list --all` |
| Test connectivity | `nc -zv puppet.lab.local 8140` |

### 6.2 Exercise A: Syntax error

On VM1, break `webserver.pp` by removing a closing brace or a comma:

```bash
sudo cp /etc/puppetlabs/code/environments/production/modules/lab/manifests/webserver.pp /tmp/webserver.pp.bak
sudo sed -i '0,/ensure => installed,/s//ensure => installed/' \
  /etc/puppetlabs/code/environments/production/modules/lab/manifests/webserver.pp
```

Then:

```bash
sudo /opt/puppetlabs/bin/puppet parser validate \
  /etc/puppetlabs/code/environments/production/modules/lab/manifests/webserver.pp
sudo puppet agent -t     # on web01 - observe the compile error
```

**Task:** Read the error, identify the file/line, fix the issue (or restore from `/tmp/webserver.pp.bak`), and re-run.

### 6.3 Exercise B: Duplicate declaration

Add this to `lab::webserver` (which already includes `lab::baseline` on the same node, declaring `tree`):

```puppet
package { 'tree':
  ensure => installed,
}
```

Run `puppet agent -t` on web01.

**Expected error:** `Duplicate declaration: Package[tree] is already declared ...`

**Fix:** Declare each resource only once (remove the duplicate or move it to a shared class).

### 6.4 Exercise C: Dependency cycle

Temporarily add to `lab::webserver`:

```puppet
Package['nginx'] -> Service['nginx'] -> Package['nginx']
```

Run the agent and observe `Found 1 dependency cycle`.

**Fix:** Remove the circular relationship. Optional: render a graph with `puppet agent -t --graph` and inspect `.dot` files under `/opt/puppetlabs/puppet/cache/state/graphs/`.

### 6.5 Exercise D: Certificate problems

Simulate a broken certificate on app01:

```bash
sudo systemctl stop puppet
sudo rm -rf /etc/puppetlabs/puppet/ssl
sudo puppet agent -t
```

The agent creates a new request, but the server still holds the old signed certificate, so you will see a certificate mismatch error.

**Fix (on VM1):**

```bash
sudo puppetserver ca clean --certname app01.lab.local
```

**Then on app01:**

```bash
sudo puppet agent -t
```

**Back on VM1:**

```bash
sudo puppetserver ca sign --certname app01.lab.local
```

**On app01:**

```bash
sudo puppet agent -t
sudo systemctl start puppet
```

### 6.6 Exercise E: Connectivity / DNS / firewall

On web01:

```bash
# Break name resolution
sudo sed -i 's/^192.168.56.10 .*/#&/' /etc/hosts
sudo puppet agent -t        # could not resolve host

# Restore
sudo sed -i 's/^#\(192.168.56.10 .*\)/\1/' /etc/hosts
```

Simulate a firewall block on VM1:

```bash
sudo ufw enable
sudo ufw deny 8140/tcp
```

On web01:

```bash
nc -zv puppet.lab.local 8140      # connection refused/timeout
sudo puppet agent -t
```

Restore:

```bash
sudo ufw allow 8140/tcp
```

### 6.7 Exercise F: Clock skew (discussion)

Certificates are time-sensitive. If an agent's clock is far ahead or behind, you get "certificate is not yet valid" or "expired" style errors. Fix with `chrony`/NTP, then retry.

### 6.8 Troubleshooting decision flow

```
Agent run fails
 |
 +-- "Could not resolve / connect"   -> DNS, /etc/hosts, firewall port 8140, puppetserver running?
 +-- "certificate verify failed"     -> clock sync, ssl dir, ca clean + re-sign
 +-- "Evaluation Error / Syntax"     -> puppet parser validate, check line number
 +-- "Duplicate declaration"         -> find where the resource is declared twice
 +-- "Dependency cycle"              -> review require/before/notify/subscribe
 +-- Resource "failed"               -> --debug, check package repo/permissions/service unit
 +-- Nothing changes                 -> is the node classified in site.pp? noop enabled? agent disabled?
```

---

## Part 7: Final Challenge

Complete these tasks without step-by-step instructions:

1. Add a class `lab::ssh_hardening` that manages `/etc/ssh/sshd_config.d/99-lab.conf` with `PermitRootLogin no`, and restarts `ssh` when the file changes. Apply it to **all** nodes.
2. Create drift by editing that file on app01, detect it with `--noop`, and remediate it.
3. Introduce one deliberate error, find it using `puppet parser validate` or `--debug`, and document the fix.

---

## Part 8: Cleanup (optional)

On each agent:

```bash
sudo puppet resource package nginx ensure=absent       # web01
sudo puppet resource user labuser ensure=absent
sudo systemctl disable --now puppet
```

On the server, remove the lab module if desired:

```bash
sudo rm -rf /etc/puppetlabs/code/environments/production/modules/lab
```

---

## Review Questions

1. What is the difference between a **resource**, a **class**, and a **manifest**?
2. Explain **desired state** and **idempotency** in your own words.
3. What does `puppet agent -t --noop` do, and why is it useful for detecting drift?
4. What is the difference between `require` and `subscribe`?
5. Name three causes of agent-to-server connection failures and how to diagnose each.
6. Why should `exec` be a last resort?

---

## Quick Reference Cheat Sheet

```bash
# Agent
sudo puppet agent -t                 # run now
sudo puppet agent -t --noop          # dry run
sudo puppet agent -t --debug         # verbose
sudo puppet agent --disable "msg"    # pause
sudo puppet agent --enable           # resume

# Local
puppet parser validate file.pp
sudo puppet apply file.pp
puppet resource <type> [title] [attr=value]
puppet config print <setting>
facter -p

# Server
sudo puppetserver ca list [--all]
sudo puppetserver ca sign --certname <name>
sudo puppetserver ca clean --certname <name>
sudo systemctl restart puppetserver
```

---

## Submission / Grading Checklist

| # | Item | Points |
|---|------|--------|
| 1 | Puppet server and 2 agents installed, certificates signed (screenshot of `puppetserver ca list --all`) | 15 |
| 2 | Classes `baseline`, `webserver`, `appserver` working (screenshots of verification) | 20 |
| 3 | Idempotency demonstrated (two consecutive runs) | 10 |
| 4 | Drift created, detected with `--noop`, and remediated (completed table in 5.2) | 20 |
| 5 | Troubleshooting exercises A-E documented (error text + fix) | 25 |
| 6 | Final challenge completed | 10 |
| | **Total** | **100** |
