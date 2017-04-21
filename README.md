# ssh-vault

These Qubes scripts allow one to keep ssh private keys in a separate VM (an
"ssh-vault"), allowing other VMs to use them only after being authorized. It
does this by using the qrexec framework to connect a local SSH Agent socket from an AppVM to the SSH Agent socket within the ssh-vault VM. The protections for the ssh private key should be similar to running `ssh -A` into the AppVM.

This was inspired by the Qubes [Split GPG](https://www.qubes-os.org/doc/split-gpg/).

Other details:
- This was developed/tested on the Fedora24 template in Qubes 3.2, though might work for other templates.
- You will be prompted to confirm each request, though like split GPG you won't see what was requested.
- Assumes ssh keys kept in an AppVM named ssh-vault.
- Assumes that the ssh-vault template automatically starts ssh-agent

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
    * qubes.SshAgent needs to run at startup in the AppVM that would like to use the key, as it makes the conduit of the ssh agent forwarding
    * This is because /etc is lost on every boot for the vault itself, so it needs to be added to the template
    * Example:
```bash
# From the VM with the git repo
qvm-copy-to-vm fedora-24 qubes.SshAgent

# On the template (in this case, fedora-24), run:
sudo mv ~user/QubesIncoming/work/qubes.SshAgent /etc/qubes-rpc/
```
- Create the ssh-vault VM (default name is "ssh-vault" in the scripts below)

- Ssh-vault: Create/copy any ssh private keys into that VM.


- Client VM: append the contents of rc.local_client to /rw/config/rc.local
    * This is what starts the client side of the ssh agent
    * Examine the contents and set $SSH_VAULT_VM appropriately
    * Be sure rc.local is executable. ie - `chmod +x /rw/config/rc.local`

- Client VM: Append bashrc_client to the client VM's ~/.bashrc
    * This sets the user's $SSH_AUTH_SOCK to the appropriate value
    * Examine the contents and set $SSH_VAULT_VM appropriately

- Every boot, you will need to run "ssh-add" on the ssh-vault VM to add your key(s) into the ssh-agent.

# Todo

- Automate adding an ssh key in the ssh-vault VM

- Convert the client rc.local into a systemd script in the client template, and allow the ssh-vault VM to be set using some well-known file (like /rw/config/ssh-vault)
	- Maybe do the above using qvm-service or qubesdb

- To address the security note above, we could use `ssh-add -c` instead of just `ssh-add`, though this would mean the user would have to click "yes" twice per access in the normal case, causing fatigue/accustomization to clicking yes too much.
