# Lab 6: Drift, noop, scheduled runs, and fault injection

Prove that Puppet enforces state. Needs Labs 3 and 5. Everything is on **vm2** unless stated.

## Step 1: Baseline

```bash
sudo puppet agent -t
```

Expected: no changes.

## Step 2: Break four things

```bash
sudo systemctl stop cron
sudo apt-get remove -y tree
sudo chmod 600 /opt/lab/info.txt
sudo vi /opt/lab/info.txt
```

In the editor press `i`, delete the text and type `tampered`. Press `Esc`, type `:wq`, press `Enter`.

## Step 3: Confirm the damage

```bash
systemctl is-active cron
which tree
ls -l /opt/lab/info.txt
```

Expected: `inactive`, nothing for `tree`, and the file mode is `-rw-------`.

## Step 4: Preview with noop

```bash
sudo puppet agent -t --noop
```

Expected: lines saying the service, package, file mode and file content would be changed. Nothing is changed.

```bash
systemctl is-active cron
```

Still `inactive`: noop changed nothing.

## Step 5: Enforce

```bash
sudo puppet agent -t
```

Expected: the changes are applied (the run reports resources changed).

## Step 6: Verify the repair

```bash
systemctl is-active cron
which tree
cat /opt/lab/info.txt
```

Everything is back to the desired state.

## Step 7: Run the agent on a schedule

Show the current interval (default 30 minutes):

```bash
puppet config print runinterval
```

Shorten it for the lab and start the service:

```bash
sudo puppet config set runinterval 2m --section agent
sudo systemctl enable --now puppet
```

## Step 8: Break something and wait

```bash
sudo systemctl stop cron
```

Wait about three minutes, then:

```bash
systemctl is-active cron
sudo journalctl -u puppet -n 30 --no-pager
```

Expected: `active`. Puppet repaired it by itself.

## Step 9: Put the interval back

```bash
sudo puppet config set runinterval 30m --section agent
sudo systemctl restart puppet
```

## Step 10: Pause enforcement for maintenance

```bash
sudo puppet agent --disable "maintenance window"
sudo puppet agent -t
sudo puppet agent --enable
```

While disabled, the run is refused and the reason is shown.

## Step 11 (vm-1): Stop the server

```bash
sudo systemctl stop puppetserver
```

## Step 12 (vm2): Try a run

```bash
sudo puppet agent -t
```

Expected: a connection error to `vm-1:8140`.

## Step 13 (vm-1): Restore the server

```bash
sudo systemctl start puppetserver
curl -sk https://localhost:8140/status/v1/simple
```

Wait for `running`, then run `sudo puppet agent -t` on vm2 again. It succeeds.

## Key ideas

- `--noop` is the safety net: preview first.
- Agents repair drift on every run.
- If the server is down, agents cannot get new catalogs, but the nodes keep running.

Next: `lab7.md`.
