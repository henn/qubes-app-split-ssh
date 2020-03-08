# Qubes Split SSH

These Qubes scripts allow one to keep ssh private keys in a separate VM (an
"ssh-vault"), allowing other VMs to use them only after being authorized. This
is done by using Qubes's [qrexec
framework](https://www.qubes-os.org/doc/qrexec2/) to connect a local SSH Agent
socket from an AppVM to the SSH Agent socket within the ssh-vault VM. The
protections for the ssh private key should be similar to running `ssh -A` into
the client AppVM.

This was inspired by the Qubes [Split GPG](https://www.qubes-os.org/doc/split-gpg/).

Other details:
- This was developed/tested on the Fedora24 template in Qubes 3.2; it might work for other templates
    - For AppVMs or ssh-vault VMs based on Debian templates, you may need to install the nmap package in the template to get the ncat utility.
- You will be prompted to confirm each request, though like split GPG you won't see what was requested
- Assumes that the ssh-vault template automatically starts ssh-agent
- One can have an arbitrary number of ssh-vault VMs
- The scripts by default assumes ssh keys are kept in an AppVM named `ssh-vault`, though this can be changed by modifying $SSH_VAULT_VM in the client script.
- Currently, a single AppVM can only access a single ssh-vault, though this wouldn't be hard to fix


**Security note**: While in normal operation, the user will be prompted for
every use of the key, it is likely possible that a malicious VM could hold onto
an ssh-agent connection for more than one use. Therefore, if you authorize
usage once, assume that a malicious VM could then use it many more times. In
this case, though, the SSH Agent should continue to protect your private keys;
only usage of it would be available to the malicious VM until it was shut down.

# Instructions

Copy files from this repo to various destinations (VM is the first argument). You can use `qvm-copy-to-vm $DEST_VM file`

- Dom0: Copy qubes.SshAgent.policy to dom0's /etc/qubes-rpc/policy/qubes.SshAgent

- Template for Ssh Vault: Copy qubes.SshAgent to /etc/qubes-rpc/qubes.SshAgent in the template image for the Ssh Vault VM.
    * This is because /etc is lost on every boot for the vault itself, so it needs to be added to the template
    * Example:
```bash
# From the VM with the git repo
qvm-copy-to-vm fedora-24 qubes.SshAgent

# On the ssh-vault's template (in this case, fedora-24), run:
sudo mv /home/user/QubesIncoming/work/qubes.SshAgent /etc/qubes-rpc/
```
- Create the ssh-vault VM (default name is "ssh-vault" in the scripts below)
    * It's recommended to disable network access for this VM to protect it.

- Ssh-vault: Create an ssh private key or copy one in

- Ssh-vault: Copy `ssh-add.desktop_ssh_vault` to `/home/user/.config/autostart/ssh-add.desktop`
    * You may need to create the .config/autostart directory if it doesn't already exist
    * Examine the contents of this file and adjust the ssh-add command on the `Exec` line if desired (e.g you may want to pass a specific SSH key to add to the agent)

- Client VM: append the contents of rc.local_client to /rw/config/rc.local
    * This is what starts the client side of the ssh agent
    * Examine the contents and set $SSH_VAULT_VM appropriately
    * Be sure rc.local is executable. ie - `chmod +x /rw/config/rc.local`

- Client VM: Append bashrc_client to the client VM's ~/.bashrc
    * This sets the user's $SSH_AUTH_SOCK to the appropriate value
    * Examine the contents and set $SSH_VAULT_VM appropriately

# Todo

- Convert the client rc.local into a systemd script in the client template, and allow the ssh-vault VM to be set using some well-known file (like /rw/config/ssh-vault)
	- Maybe do the above using qvm-service or qubesdb

- To address the *Security Note* above, we could use `ssh-add -c` instead of just `ssh-add`, though this would mean the user would have to click "yes" twice per access in the normal case, causing fatigue/accustomization to clicking yes too much.
    * Note: `ssh-add -c` on the Fedora 24 template seems to be broken

- Another way to address the above would be to introduce an SSH Agent proxy that only allowed one request per connection. (idea from @marmarek)

- (possibly distant future) Figure out a way to display info on what is being signed
