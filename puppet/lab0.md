# Lab 0: Install Puppet Server on vm-1 (Rocky Linux)

Based on the Puppet Core documentation (help.puppet.com/core, Puppet Core 9.x). This version is for **Rocky Linux 9** (RHEL family, uses `dnf` and `el-9` packages; this replaces the Ubuntu `apt` steps).

| VM | Role |
|---|---|
| `vm-1` | Puppet Server |
| `vm2` | Puppet agent (Lab 1) |

Rename if your hostnames differ. Everything in this lab is done on **vm-1**.

Editing files: use `sudo vi FILE` (plain `vi` for files in your home folder).

| Action | Keys |
|---|---|
| Start typing | Press `i` (INSERT appears at the bottom) |
| Stop typing | Press `Esc` |
| Save and exit | Type `:wq` and press `Enter` |
| Exit without saving | Type `:q!` and press `Enter` |
| Delete a whole line | Move to it, press `Esc`, then `dd` |
| Search | Type `/word` and press `Enter` |

Tip: if pasted text gets staircase indents, press `Esc`, type `:set paste`, press `Enter`, then press `i` before pasting.

## How Puppet works

1. The agent on a node sends its facts to the server (port 8140).
2. The server builds a catalog (desired state) from your code.
3. The agent applies it. Running it again changes nothing if the node already matches.
4. Agent and server trust each other through certificates.

## Step 1: Get a Forge API key

Puppet Core packages are in a protected repository, so you need a Forge API key. The free version for testing or development lets you use Puppet Core on up to 25 nodes, which is plenty for this lab. Do this in a web browser, not on the VMs.

### 1.1 Register on Puppet Forge

Go to `https://forge.puppet.com` and sign up or log in. If you already have a Forge account, use it.

### 1.2 Turn on multifactor authentication

In your Forge account settings, enable multifactor authentication (MFA). Puppet requires MFA before you can get an API key or download packages.

### 1.3 Accept the Puppet Core license (EULA)

1. Open your individual profile on Forge.
2. In the menu bar, click **Puppet and Puppet Core EULA** (also reachable at `https://forge.puppet.com/eula`).
3. In the **User Acceptance Role** list, choose your role:
   - **Puppet Core user** to try out the product, or
   - **Puppet Core developer** for development work.
4. Read and accept the EULA.

Both roles allow up to 25 nodes. Production use with more nodes needs a paid license (contact Puppet sales).

### 1.4 Create the API key

1. In your Forge profile, click **API Keys**.
2. If you have no active key, or it expires soon, click **Create a new key** and follow the prompts.
3. Copy the key and save it somewhere private, such as a password manager.

You use this key as the password in Step 6 (and in Lab 1). The user name is always the literal word `forge-key`.

Treat the key like a password: do not paste it into chat, scripts or Git. If it leaks, delete it on the same **API Keys** page and create a new one.

Source: the "Accessing Puppet Core for limited use for testing or development" page in the Puppet Core docs. The menu names may change slightly over time.

## Step 2: Check memory

```bash
free -h
```

Puppet Server is a Java service and needs about 1 GB free. If these VMs also run Kubernetes, memory is tight.

## Step 3: Check your Rocky version

```bash
cat /etc/os-release
```

Look at `VERSION_ID`. This lab assumes `9.x`, which uses the `el-9` package. (For Rocky 8, replace `el-9` with `el-8` everywhere below.)

## Step 4: Download the release package

The release package for Rocky/RHEL 9 is `puppet9-release-el-9.noarch.rpm`:

```bash
wget --content-disposition https://yum-puppetcore.puppet.com/public/puppet9-release-el-9.noarch.rpm
ls -l puppet9-release-el-9.noarch.rpm
```

## Step 5: Install the release package

```bash
sudo rpm -ivh puppet9-release-el-9.noarch.rpm
```

This adds a Puppet repository file under `/etc/yum.repos.d/`. Find its exact name:

```bash
ls /etc/yum.repos.d/ | grep -i puppet
```

## Step 6: Add your API key

Open the Puppet repo file you found in Step 5 (use its exact name in place of `puppet*.repo` below):

```bash
sudo vi /etc/yum.repos.d/puppet*.repo
```

Inside the Puppet Core section (the one with the `baseurl` pointing at `yum-puppetcore.puppet.com`), add these two lines. `forge-key` is the literal word, not a placeholder. Replace only `YOUR_API_KEY`:

```text
username=forge-key
password=YOUR_API_KEY
```

If the file already has `username=` and `password=` lines, edit those instead of adding duplicates. Leave the other lines (`name`, `baseurl`, `enabled`, `gpgcheck`, `gpgkey`) as they are.

Protect the file, because it now contains your key:

```bash
sudo chmod 600 /etc/yum.repos.d/puppet*.repo
```

Press `Esc`, type `:wq`, press `Enter` when you finish editing.

## Step 7: Refresh the package metadata

```bash
sudo dnf clean all
sudo dnf makecache
```

Expected: the Puppet repository appears and downloads metadata without an error. If dnf asks you to import a GPG key, check that the key comes from Puppet and answer `y`.

### If you see `401 Unauthorized` (or `403 Forbidden`)

The repository rejected the login. First test the key on its own, so you know whether the key or the repo file is wrong. Get the repository URL:

```bash
grep -i baseurl /etc/yum.repos.d/puppet*.repo
```

Then test it (replace `BASEURL` with the value after `baseurl=`; if it contains `$basearch`, use `x86_64`, or `aarch64` on ARM):

```bash
curl -I -u forge-key BASEURL/repodata/repomd.xml
```

curl asks for the password: paste your API key (nothing is shown while you paste) and press `Enter`. Read the first line of the answer:

| Answer | Meaning | What to do |
|---|---|---|
| `HTTP/2 200` | The key is valid | The repo file is wrong: do checks 1 and 2 below |
| `HTTP/2 401` | The key is rejected | Do check 3 below and create a new key |
| `HTTP/2 403` | Key valid but not allowed | EULA not accepted, or the role does not cover this package: redo Step 1.3 |

Then check these in order:

1. Show the repo file (it contains your key, so do this in a private terminal):

   ```bash
   sudo cat -A /etc/yum.repos.d/puppet*.repo
   ```

   `cat -A` shows hidden characters: each line must end with a single `$`. A `^M` before the `$` means a Windows line break was pasted in, so retype the lines. Make sure `username=forge-key` and `password=YOUR_API_KEY` have no `#` at the start, no quotes, no `<` or `>`, and no extra spaces, and that they sit inside the correct section (above any next `[section]` header).

2. Fix the file if needed, then try again:

   ```bash
   sudo vi /etc/yum.repos.d/puppet*.repo
   sudo dnf clean all
   sudo dnf makecache
   ```

3. Check your Forge account in the browser: multifactor authentication is on, the Puppet Core EULA is accepted (Step 1.3), and the key on the **API Keys** page is active and not expired. If in doubt, create a new key and paste it again, copying the whole key with no spaces.

## Step 8: Install Puppet Server

```bash
sudo dnf install -y puppetserver
```

Do not start it yet.

## Step 9: Make the commands easy to run

```bash
sudo ln -s /opt/puppetlabs/bin/puppet /usr/local/bin/puppet
sudo ln -s /opt/puppetlabs/bin/facter /usr/local/bin/facter
sudo ln -s /opt/puppetlabs/bin/puppetserver /usr/local/bin/puppetserver
```

```bash
puppet --version
puppetserver -v
```

If `sudo puppet` says command not found later, run `sudo /opt/puppetlabs/bin/puppet ...` instead, or add `/opt/puppetlabs/bin` to sudo's `secure_path`. (On Rocky, sudo's `secure_path` does not include `/usr/local/bin`.) The simplest fix is to link into `/usr/bin` instead:

```bash
sudo ln -sf /opt/puppetlabs/bin/puppet /usr/bin/puppet
sudo ln -sf /opt/puppetlabs/bin/facter /usr/bin/facter
sudo ln -sf /opt/puppetlabs/bin/puppetserver /usr/bin/puppetserver
```

## Step 10: Lower the Java memory for the lab

The Rocky settings file is in `/etc/sysconfig`:

```bash
sudo vi /etc/sysconfig/puppetserver
```

Type `/JAVA_ARGS` and press `Enter` to jump to the line. Move the cursor onto the `2g` after `-Xms`, press `x` to delete characters and `i` to type, so that `-Xms2g -Xmx2g` becomes `-Xms512m -Xmx1g`. Leave the rest of the line as it is. Press `Esc`, type `:wq`, press `Enter`.

## Step 11: Make the name vm-1 resolvable

```bash
hostname -I
```

Note the first IP address (for example `10.0.0.2`). Edit the hosts file:

```bash
sudo vi /etc/hosts
```

Press `G` then `o` to open a new line at the end, and type this line, using your IP:

```text
10.0.0.2 vm-1 puppet
```

Press `Esc`, type `:wq`, press `Enter`.

## Step 12: Set the server name (before the first start)

```bash
sudo puppet config set certname vm-1 --section main
sudo puppet config set server vm-1 --section main
cat /etc/puppetlabs/puppet/puppet.conf
```

Do this before the first start, because the certificate authority is created from this name.

## Step 13: Open port 8140 in the firewall

Rocky runs `firewalld` by default. Agents (like vm2) cannot reach the server unless port 8140 is open:

```bash
sudo firewall-cmd --permanent --add-port=8140/tcp
sudo firewall-cmd --reload
sudo firewall-cmd --list-ports
```

Expected: `8140/tcp` is in the list. (If `firewalld` is not running, skip this step.)

## Step 14: Start the server

```bash
sudo systemctl enable --now puppetserver
```

Wait one to two minutes, then:

```bash
sudo systemctl status puppetserver --no-pager
```
