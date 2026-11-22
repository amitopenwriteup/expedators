# Lab 0: Install Puppet Server on vm-1

Based on the Puppet Core documentation (help.puppet.com/core, Puppet Core 9.x).

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

You use this key as the password in Step 7 (and in Lab 1). The user name is always the literal word `forge-key`.

Treat the key like a password: do not paste it into chat, scripts or Git. If it leaks, delete it on the same **API Keys** page and create a new one.

Source: the "Accessing Puppet Core for limited use for testing or development" page in the Puppet Core docs. The menu names may change slightly over time.

## Step 2: Check memory

```bash
free -h
```

Puppet Server is a Java service and needs about 1 GB free. If these VMs also run Kubernetes, memory is tight.

## Step 3: Find your Ubuntu name

```bash
cat /etc/os-release
```

Look at `VERSION_CODENAME`, for example `noble` (24.04) or `jammy` (22.04).

## Step 4: Find the release package

Open this address in a web browser:

```text
https://apt-puppetcore.puppet.com/public
```

Find the file for your Ubuntu name, for example `puppet9-release-noble.deb`. Write down the exact file name.

## Step 5: Download it

Use your exact file name:

```bash
wget --content-disposition https://apt-puppetcore.puppet.com/public/puppet9-release-noble.deb
```

## Step 6: Install the release package

```bash
sudo dpkg -i puppet9-release-noble.deb
```

## Step 7: Add your API key

```bash
ls /etc/apt/auth.conf.d/
```

Open the Puppet file (normally `apt-puppetcore-puppet.conf`). If the folder has no such file, create it with the same command:

```bash
sudo vi /etc/apt/auth.conf.d/apt-puppetcore-puppet.conf
```

The file must contain these three lines. `forge-key` is the literal word, not a placeholder. Replace only `YOUR_API_KEY`:

```text
machine apt-puppetcore.puppet.com
login forge-key
password YOUR_API_KEY
```

Protect the file:

```bash
sudo chmod 600 /etc/apt/auth.conf.d/apt-puppetcore-puppet.conf
```

## Step 8: Update the package list

```bash
sudo apt-get update
```

Expected: `apt-puppetcore.puppet.com` appears without an error.

### If you see `401 Unauthorized`

The repository rejected the login. First test the key on its own, so you know whether the key or the apt file is wrong:

```bash
curl -I -u forge-key https://apt-puppetcore.puppet.com/dists/noble/InRelease
```

curl asks for the password: paste your API key (nothing is shown while you paste) and press `Enter`. Use your Ubuntu name instead of `noble` if it differs. Read the first line of the answer:

| Answer | Meaning | What to do |
|---|---|---|
| `HTTP/2 200` | The key is valid | The apt auth file is wrong: do checks 1 to 3 below |
| `HTTP/2 401` | The key is rejected | Do check 4 below and create a new key |
| `HTTP/2 403` | Key valid but not allowed | EULA not accepted, or the role does not cover this package: redo Step 1.3 |

Then check these in order:

1. Show the auth file (it contains your key, so do this in a private terminal):

   ```bash
   sudo cat -A /etc/apt/auth.conf.d/apt-puppetcore-puppet.conf
   ```

   `cat -A` shows hidden characters: each line must end with a single `$`. A `^M` before the `$` means a Windows line break was pasted in, so retype the file (check 3). It must have exactly these three lines: no `#` at the start, no quotes, no `<` or `>`, no extra spaces, and `forge-key` as the literal user name.

   ```text
   machine apt-puppetcore.puppet.com
   login forge-key
   password YOUR_API_KEY
   ```

2. Check the file name ends in `.conf`:

   ```bash
   ls -l /etc/apt/auth.conf.d/
   ```

3. Fix the file if needed. Open it, press `Esc`, type `gg` then `dG` to delete everything, press `i`, and type the three lines again:

   ```bash
   sudo vi /etc/apt/auth.conf.d/apt-puppetcore-puppet.conf
   ```

   Press `Esc`, type `:wq`, press `Enter`.

4. Check your Forge account in the browser: multifactor authentication is on, the Puppet Core EULA is accepted (Step 1.3), and the key on the **API Keys** page is active and not expired. If in doubt, create a new key and paste it again, copying the whole key with no spaces.

5. Try again:

   ```bash
   sudo apt-get update
   ```

## Step 9: Install Puppet Server

```bash
sudo apt-get install -y puppetserver
```

Do not start it yet.

## Step 10: Make the commands easy to run

```bash
sudo ln -s /opt/puppetlabs/bin/puppet /usr/local/bin/puppet
sudo ln -s /opt/puppetlabs/bin/facter /usr/local/bin/facter
sudo ln -s /opt/puppetlabs/bin/puppetserver /usr/local/bin/puppetserver
```

```bash
puppet --version
puppetserver -v
```

## Step 11: Lower the Java memory for the lab

```bash
sudo vi /etc/default/puppetserver
```

Type `/JAVA_ARGS` and press `Enter` to jump to the line. Move the cursor onto the `2g` after `-Xms`, press `x` to delete characters and `i` to type, so that `-Xms2g -Xmx2g` becomes `-Xms512m -Xmx1g`. Leave the rest of the line as it is. Press `Esc`, type `:wq`, press `Enter`.

## Step 12: Make the name vm-1 resolvable

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

## Step 13: Set the server name (before the first start)

```bash
sudo puppet config set certname vm-1 --section main
sudo puppet config set server vm-1 --section main
cat /etc/puppetlabs/puppet/puppet.conf
```

Do this before the first start, because the certificate authority is created from this name.

## Step 14: Start the server

```bash
sudo systemctl enable --now puppetserver
```

Wait one to two minutes, then:

```bash
sudo systemctl status puppetserver --no-pager
```

## Step 15: Verify

```bash
curl -sk https://localhost:8140/status/v1/simple
sudo puppetserver ca list --all
```

Expected: `running`, and a signed certificate for `vm-1`.

If it does not start:

```bash
sudo tail -50 /var/log/puppetlabs/puppetserver/puppetserver.log
```

Common causes: too little memory (lower `-Xmx` in Step 11) or port 8140 in use.

## Done when

- [ ] `puppetserver -v` prints a version
- [ ] The status address answers `running`
- [ ] `vm-1` is a signed certificate

Next: `lab1.md`.
