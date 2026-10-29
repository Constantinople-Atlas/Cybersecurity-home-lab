# SSH Key Authentication and Password Authentication Hardening

## Lab environment

-   Host: Windows 10 using Windows PowerShell
-   VM: `lab-base`
-   Guest OS: Ubuntu Server 24.04.5 LTS
-   Linux account: `atlas_admin`
-   SSH forwarding: host `127.0.0.1:2222` to guest port `22`
-   Key type: Ed25519
-   Result: Key-based login verified; SSH password authentication
    disabled and verified.

## 1. Objective

Create a dedicated SSH key, install its public key on Ubuntu, verify key
login, then disable SSH password authentication only after key login
succeeds.

The existing SSH session was kept open, and access to the VM through the
VirtualBox console was confirmed before changing the server
configuration.

## 2. Create and verify the key pair

The key pair was created in Windows PowerShell:

``` powershell
ssh-keygen -t ed25519 -C "lab-base-atlas_admin" -f "$HOME\.ssh\lab_base_ed25519"
```

-   `-t ed25519` selects the Ed25519 algorithm.
-   `-C` adds a descriptive comment.
-   `-f` specifies the output filename.

The files were verified with:

``` powershell
Get-ChildItem "$HOME\.ssh\lab_base_ed25519*"
```

Observed files:

  File                            Size Purpose
  ------------------------ ----------- --------------------------
  `lab_base_ed25519`         464 bytes Private key; keep secret
  `lab_base_ed25519.pub`     103 bytes Public key

The private key was protected with a passphrase. Never share the private
key or passphrase.

## 3. Install the public key

From Windows PowerShell, the public key was transferred using the
existing SSH connection:

``` powershell
Get-Content "$HOME\.ssh\lab_base_ed25519.pub" | ssh -p 2222 atlas_admin@127.0.0.1 "umask 077; mkdir -p ~/.ssh && cat >> ~/.ssh/authorized_keys && chmod 700 ~/.ssh && chmod 600 ~/.ssh/authorized_keys"
```

The Ubuntu account password was requested for this initial installation.
The command returned to PowerShell without displaying an error.

Permissions were checked on Ubuntu:

``` bash
ls -ld ~/.ssh && ls -l ~/.ssh/authorized_keys
```

Observed output:

``` text
drwx------ 2 atlas_admin atlas_admin 4096 Oct 23 12:08 /home/atlas_admin/.ssh
-rw------- 1 atlas_admin atlas_admin 103 Oct 29 11:30 /home/atlas_admin/.ssh/authorized_keys
```

`700` on `~/.ssh` limits access to the owner. `600` on `authorized_keys`
allows only the owner to read and write the file.

## 4. Verify key login before changing SSH settings

The following command was run in Windows PowerShell:

``` powershell
ssh -o IdentitiesOnly=yes -i "$HOME\.ssh\lab_base_ed25519" -p 2222 atlas_admin@127.0.0.1
```

The key passphrase was requested and the connection succeeded,
displaying the Ubuntu 24.04.5 LTS welcome message and an
`atlas_admin@lab-base` shell prompt.

## 5. Inspect the existing configuration

The main SSH configuration includes additional files:

``` text
12:Include /etc/ssh/sshd_config.d/*.conf
71:KbdInteractiveAuthentication no
```

The included file `/etc/ssh/sshd_config.d/50-cloud-init.conf` contained:

``` text
PasswordAuthentication yes
```

The effective settings before the change included:

``` text
port 22
maxauthtries 6
permitrootlogin without-password
pubkeyauthentication yes
passwordauthentication yes
kbdinteractiveauthentication no
authenticationmethods any
permitemptypasswords no
```

The cloud-init configuration was left unchanged.

## 6. Back up the main SSH configuration

A timestamped backup was created with:

``` bash
sudo cp -a /etc/ssh/sshd_config "/etc/ssh/sshd_config.backup-$(date +%Y%m%d-%H%M%S)"
```

The verified backup was:

``` text
/etc/ssh/sshd_config.backup-20261029-155023
```

This backup covers the main `sshd_config` file, not all SSH
configuration files.

## 7. Add a separate hardening snippet

The filename `/etc/ssh/sshd_config.d/00-local-hardening.conf` was
checked and confirmed not to exist. It was created with:

``` bash
sudo sh -c 'printf "%s\n" "PasswordAuthentication no" > /etc/ssh/sshd_config.d/00-local-hardening.conf'
```

Its contents were verified as:

``` text
PasswordAuthentication no
```

The new file is owned by root and has permissions `-rw-r--r--`. Because
the main SSH configuration includes `sshd_config.d/*.conf`, the `00-`
filename is processed before `50-cloud-init.conf`. OpenSSH generally
uses the first value obtained for a setting, so this earlier directive
takes precedence without directly editing the cloud-init-managed file.

## 8. Validate and apply the change

Configuration syntax was validated:

``` bash
sudo sshd -t
```

It returned no output, indicating no syntax error was reported.

The effective settings were checked before reloading:

``` bash
sudo sshd -T | grep -E '^(passwordauthentication|pubkeyauthentication|kbdinteractiveauthentication) '
```

Observed output:

``` text
pubkeyauthentication yes
passwordauthentication no
kbdinteractiveauthentication no
```

SSH was then reloaded:

``` bash
sudo systemctl reload ssh
```

The service check returned:

``` text
active
```

## 9. Verify a fresh connection after reloading

A new Windows PowerShell connection was opened using the same key
command:

``` powershell
ssh -o IdentitiesOnly=yes -i "$HOME\.ssh\lab_base_ed25519" -p 2222 atlas_admin@127.0.0.1
```

The key passphrase was requested and the fresh connection succeeded. The
user confirmed that `whoami` returned `atlas_admin`.

Final effective settings:

``` text
pubkeyauthentication yes
passwordauthentication no
kbdinteractiveauthentication no
```

The SSH service remained `active`.

## 10. Final result

  Check                                          Result
  ---------------------------------------------- --------
  Dedicated Ed25519 key pair created             Passed
  Public key installed                           Passed
  SSH directory permissions `700`                Passed
  `authorized_keys` permissions `600`            Passed
  Key login succeeded before change              Passed
  Main SSH configuration backup created          Passed
  `sshd -t` validation passed                    Passed
  Effective password authentication is `no`      Passed
  Effective public-key authentication is `yes`   Passed
  Fresh key login succeeded after reload         Passed
  SSH service remains active                     Passed

**Conclusion:** SSH password authentication is disabled in the effective
configuration, and key-based SSH login succeeded after the service
reload.

## 11. Recovery and safety notes

-   Keep the VirtualBox console available when making SSH changes.
-   Keep an existing SSH session open until a fresh key-based connection
    succeeds.
-   Windows private key: `C:\Users\Contessina\.ssh\lab_base_ed25519`.
    Keep it private and maintain a secure backup.
-   Ubuntu public key: `/home/atlas_admin/.ssh/authorized_keys`.
-   Added configuration:
    `/etc/ssh/sshd_config.d/00-local-hardening.conf`.
-   Main configuration backup:
    `/etc/ssh/sshd_config.backup-20261029-155023`.
-   The backup covers only the main configuration file; it does not
    restore the added snippet automatically.
-   If a future change breaks SSH, use the VirtualBox console to inspect
    the configuration and service. Run `sudo sshd -t` before reloading.
-   Disabling SSH password authentication does not disable the Ubuntu
    account password or its use for `sudo`.
-   The previously observed `permitrootlogin without-password` setting
    was not changed.
-   VirtualBox forwarding remains host `127.0.0.1:2222` to guest port
    `22`. Do not expose SSH to the LAN or Internet as part of this
    exercise.

## 12. Useful commands

Windows PowerShell:

``` powershell
Get-ChildItem "$HOME\.ssh\lab_base_ed25519*"
ssh -o IdentitiesOnly=yes -i "$HOME\.ssh\lab_base_ed25519" -p 2222 atlas_admin@127.0.0.1
```

Ubuntu:

``` bash
ls -ld ~/.ssh && ls -l ~/.ssh/authorized_keys
sudo sshd -T | grep -E '^(passwordauthentication|pubkeyauthentication|kbdinteractiveauthentication) '
sudo sshd -t
systemctl is-active ssh
```

## 13. Suggested GitHub commit

Save as `docs/linux/ssh-key-authentication-hardening.md`.

From the local repository in Windows PowerShell:

``` powershell
git status
git add docs/linux/ssh-key-authentication-hardening.md
git commit -m "Document SSH key authentication hardening"
git push origin main
```

Review `git status` and the staged path before committing. Never commit
private keys, passphrases, or other secrets.
