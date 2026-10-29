# Cybersecurity Home Lab --- Linux Account and Authentication Security

**VM:** `lab-base`\
**Account:** `atlas_admin`\
**Status:** Read-only inspection completed. No SSH configuration changes
were made.

## 1. Purpose

This lesson checks Linux account records, password-file protections,
user privileges, SSH settings, VirtualBox port forwarding, and SSH
service status. These checks help establish who can log in, what
privileges an account has, and how remote access is configured.

## 2. `/etc/passwd` account records

Selected output reported during the lesson:

``` text
uuidd:x:104:105::/run/uuidd:/usr/sbin/nologin
tcpdump:x:105:107::/nonexistent:/usr/sbin/nologin
tss:x:106:108:TPM software stack,,,:/var/lib/tpm:/bin/false
landscape:x:107:109::/var/lib/landscape:/usr/sbin/nologin
fwupd-refresh:x:989:989:Firmware update daemon:/var/lib/fwupd:/usr/sbin/nologin
usbmux:x:108:46:usbmux daemon,,,:/var/lib/usbmux:/usr/sbin/nologin
atlas_admin:x:1000:1000:Constantinople:/home/atlas_admin:/bin/bash
sshd:x:109:65534::/run/sshd:/usr/sbin/nologin
```

Each line has seven colon-separated fields: username, password
placeholder, UID, primary GID, comment/GECOS, home directory, and login
shell.

For `atlas_admin`, the UID is `1000`, primary GID is `1000`, home is
`/home/atlas_admin`, and shell is `/bin/bash`. The `x` placeholder
indicates password information is held in `/etc/shadow`, rather than in
this field. Service accounts often use `/usr/sbin/nologin` or
`/bin/false` to prevent ordinary interactive shell logins.

## 3. `/etc/shadow` permissions

Command:

``` bash
ls -l /etc/shadow
```

Actual output reported:

``` text
-rw-r----- 1 root shadow 973 Oct 23 12:07 /etc/shadow
```

Interpretation: - `-`: regular file - `rw-`: owner `root` can read and
write - `r--`: group `shadow` can read - `---`: other users have no
permissions

Password-related data is sensitive, so keep access restricted. Do not
change these permissions casually.

## 4. User identity and groups

Command:

``` bash
id atlas_admin
```

Actual output reported:

``` text
uid=1000(atlas_admin) gid=1000(atlas_admin) groups=1000(atlas_admin),4(adm),24(cdrom),27(sudo),30(dip),46(plugdev),101(lxd)
```

The account belongs to `sudo`, which allows administrative commands
under the configured policy. It also belongs to `lxd`, a
security-sensitive group because container-management privileges can
sometimes enable powerful host-level actions. Do not remove groups
without first understanding their purpose and impact.

## 5. Sudo privileges

Command:

``` bash
sudo -l
```

Actual output reported:

``` text
Matching Defaults entries for atlas_admin on lab-base:
    env_reset, mail_badpass, secure_path=/usr/local/sbin\\:/usr/local/bin\\:/usr/sbin\\:/usr/bin\\:/sbin\\:/bin\\:/snap/bin, use_pty

User atlas_admin may run the following commands on lab-base:
    (ALL : ALL) ALL
```

Brief explanation: - `env_reset`: resets much of the command environment
when using sudo. - `mail_badpass`: configures mail notification behavior
for failed password attempts, depending on system setup. -
`secure_path`: specifies a controlled command search path for sudo. -
`use_pty`: uses a pseudo-terminal for sudo command execution. -
`(ALL : ALL) ALL`: allows commands as any target user and group, subject
to the sudo policy. This is broad administrative power.

`sudo -l` lists permitted sudo actions. `sudo -i` starts a root login
shell. During the lesson, `sudo -i` was entered accidentally and `exit`
returned the user to the normal account. No lasting configuration change
was reported.

## 6. Effective SSH settings

Command:

``` bash
sudo sshd -T | grep -E '^(port|permitrootlogin|passwordauthentication|pubkeyauthentication|kbdinteractiveauthentication|authenticationmethods|maxauthtries|permitemptypasswords|allowusers|denyusers) '
```

Actual output reported:

``` text
port 22
maxauthtries 6
permitrootlogin without-password
pubkeyauthentication yes
passwordauthentication yes
kbdinteractiveauthentication no
permitemptypasswords no
authenticationmethods any
```

Interpretation: - `port 22`: SSH's port inside the Ubuntu guest. -
`maxauthtries 6`: up to six authentication attempts per connection,
subject to other controls. - `permitrootlogin without-password`: root
password login is blocked, but other methods such as public-key
authentication may still be allowed. It does not mean all root SSH login
is disabled. - `pubkeyauthentication yes`: public-key authentication is
enabled. - `passwordauthentication yes`: password authentication is
enabled. - `kbdinteractiveauthentication no`: keyboard-interactive
authentication is disabled. - `permitemptypasswords no`: empty passwords
are not allowed. - `authenticationmethods any`: this setting does not
require a particular multi-method combination.

No `allowusers` or `denyusers` line appeared in this filtered output.
That alone does not prove all possible access controls have been
checked. These settings also do not prove that a public key has already
been installed for `atlas_admin`.

## 7. VirtualBox port forwarding and service state

The VirtualBox NAT forwarding rule was checked in the graphical
interface:

  Setting      Reported value
  ------------ ----------------
  Host IP      `127.0.0.1`
  Host port    `2222`
  Guest IP     blank
  Guest port   `22`

This maps Windows host port `2222` to guest port `22`. Binding the host
side to `127.0.0.1` restricts access through this rule to the Windows
host itself. Keep this loopback binding; do not expose the rule on all
host interfaces.

Commands:

``` bash
systemctl is-active ssh
systemctl is-enabled ssh
```

Reported results:

``` text
active
enabled
```

`active` means SSH was running when checked. `enabled` means it is
configured to start automatically at boot.

## 8. Current findings and gaps

Confirmed: - `atlas_admin` is UID/GID `1000`. - The account has `sudo`
and `lxd` group membership. - The account has broad sudo authorization:
`(ALL : ALL) ALL`. - `/etc/shadow` was reported as `-rw-r-----`, owned
by `root:shadow`. - SSH is running and enabled. - Both password and
public-key authentication are enabled in the reported SSH settings. -
Root password SSH login is blocked by the reported
`permitrootlogin without-password` setting, but key-based root login may
still be possible. - VirtualBox forwards `127.0.0.1:2222` to guest port
`22`.

Not yet established: - Whether a public key is already installed for
`atlas_admin`. - Whether a new key-based login works. - Whether password
authentication can safely be disabled after testing. - Whether account
password-aging settings need adjustment. - Whether `lxd` membership is
required for this lab's intended work.

This is an inspection record, not a claim that the VM is fully hardened.

## 9. Next practical exercise: SSH key authentication

The next exercise is to set up and test SSH public-key login from
Windows **before** considering disabling password authentication.

1.  On Windows, check whether an SSH key already exists. Do not
    overwrite an existing key.
2.  If needed, generate a dedicated key pair for this lab.
3.  Keep the **private key** on Windows; never share it or commit it to
    GitHub. The public key can be copied to the lab account.
4.  Install the public key for `atlas_admin` with correct ownership and
    permissions.
5.  Open a second Windows terminal and test login to `127.0.0.1` on port
    `2222`.
6.  Keep the existing working SSH session open while testing.
7.  Only after key login is verified should we review whether to disable
    password authentication. Validate any configuration change and test
    a new connection before closing the working session.

Do not disable password authentication yet. Do not change root-login
settings, firewall rules, or VirtualBox forwarding during the initial
key-generation step. Never share the private key.

## 10. Knowledge-check review

The learner correctly identified that: - `x` in `/etc/passwd` is a
placeholder and password hashes are stored in `/etc/shadow`. -
`/etc/shadow` permissions restrict access to the owner and `shadow`
group. - `sudo` membership matters because it can grant administrative
command access. - Authentication checks identity; authorization
determines what an identity may do. - Both password and public-key SSH
authentication are enabled in the reported settings, but the output does
not mean both are required. - SSH settings do not prove a public key is
already installed. - The guest SSH port is `22`, while Windows connects
through host port `2222`. - SSH was reported `active` and `enabled`.

## 11. Rebuild verification checklist

After rebuilding the VM, these read-only commands can help recheck the
account and SSH state:

``` bash
grep '^atlas_admin:' /etc/passwd
ls -l /etc/shadow
id atlas_admin
sudo -l
sudo sshd -T | grep -E '^(port|permitrootlogin|passwordauthentication|pubkeyauthentication|kbdinteractiveauthentication|authenticationmethods|maxauthtries|permitemptypasswords|allowusers|denyusers) '
systemctl is-active ssh
systemctl is-enabled ssh
```

Also inspect the VirtualBox NAT forwarding rule. Compare results with
this record and investigate differences instead of automatically forcing
them to match.

## 12. Suggested repository location

Suggested path in the existing GitHub repository:

``` text
docs/linux/account-and-authentication-security.md
```

This document is prepared as lesson notes; it has **not** been committed
or pushed to GitHub.

**Lesson status:** Read-only account and SSH configuration inspection
completed.\
**Next exercise:** SSH key authentication from Windows to `lab-base`,
with a tested recovery path before any password-authentication change.
