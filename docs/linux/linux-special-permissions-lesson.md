# Linux Special Permissions --- Lesson Notes

**Lab:** `lab-base` --- Ubuntu Server 24.04.5 LTS\
**Linux user:** `atlas_admin`\
**Topic:** setuid, setgid, and the sticky bit\
**Safety:** This lesson used read-only inspection commands. No system
permissions were changed.

## Learning goals

By the end of this lesson, you should be able to:

-   Recognize setuid, setgid, and the sticky bit in symbolic
    permissions.
-   Explain the difference between lowercase `s`/`t` and uppercase
    `S`/`T`.
-   Read numeric permission modes that include special permission bits.
-   Explain why these permissions are useful and why they should be
    handled carefully.

## 1. setuid --- set user ID

Command used:

``` bash
ls -l /usr/bin/passwd
```

Observed output:

``` text
-rwsr-xr-x 1 root root 64152 May 30 2024 /usr/bin/passwd
```

Then we ran:

``` bash
stat -c '%a %A %U %G %n' /usr/bin/passwd
```

Observed output:

``` text
4755 -rwsr-xr-x root root /usr/bin/passwd
```

### How to read this

-   `-` means this is a regular file.
-   `rws` is the owner's permission group. The `s` means setuid is
    enabled and the owner's execute permission is enabled.
-   `r-x` for the group means read and execute, but not write.
-   `r-x` for others means read and execute, but not write.
-   The owner and group are both `root`.
-   `4755` is the numeric mode: leading `4` indicates setuid; `755`
    represents ordinary permissions.

### What setuid does

For an executable, setuid can cause the program to run with the file
owner's effective user ID, subject to operating-system rules and the
program's own controls. In this example, the owner is `root`. The
`passwd` program needs controlled privileges for certain
password-management operations. Setuid does **not** mean that every user
gets unrestricted root access.

## 2. setgid --- set group ID

Command used:

``` bash
ls -ld /var/mail
```

Observed output:

``` text
drwxrwsr-x 2 root mail 4096 Sep 9 06:29 /var/mail
```

### How to read this

-   `d` means this is a directory.
-   `rwx` for the owner means the owner has read, write, and execute
    permissions.
-   `rws` for the group indicates that setgid is enabled and group
    execute permission is enabled.
-   `r-x` for others means read and execute, but not write.
-   The owner is `root`; the group is `mail`.

### What setgid does

-   **On an executable:** setgid can make the program run with the
    file's group as its effective group, subject to system behavior.
-   **On a directory:** new files and subdirectories generally inherit
    the directory's group. This is useful for shared directories where
    users need consistent group ownership.

The setgid bit does not automatically grant write permission to
everyone.

## 3. Sticky bit

Command used:

``` bash
ls -ld /tmp
```

Observed output:

``` text
drwxrwxrwt 14 root root 4096 Oct 27 15:27 /tmp
```

### How to read this

-   `d` means this is a directory.
-   The owner, group, and others all have `rwx` permissions in the
    displayed mode.
-   The final `t` means the sticky bit is enabled and execute permission
    for others is enabled.
-   The directory owner and group are both `root`.

### What the sticky bit does

In a shared writable directory such as `/tmp`, the sticky bit generally
prevents an ordinary user from deleting or renaming another user's
entries. The entry owner, directory owner, and privileged users have
different rights, so this is a restriction rather than an absolute
guarantee.

## 4. Recognizing the symbols

  Example        Meaning
  -------------- ---------------------------------------------------
  `-rwsr-xr-x`   setuid is enabled; owner execute is enabled
  `-rwxr-sr-x`   setgid is enabled; group execute is enabled
  `drwxrwsr-x`   setgid is enabled on a directory
  `drwxrwxrwt`   sticky bit is enabled; others' execute is enabled

### Lowercase and uppercase

-   Lowercase `s`: the corresponding special bit is set **and** execute
    permission is enabled.
-   Uppercase `S`: the corresponding special bit is set, but execute
    permission is not enabled.
-   Lowercase `t`: the sticky bit is set and others' execute permission
    is enabled.
-   Uppercase `T`: the sticky bit is set, but others' execute permission
    is not enabled.

## 5. Numeric special-permission modes

The numeric mode reported for `/usr/bin/passwd` was `4755`.

-   Leading `4`: setuid.
-   Leading `2`: setgid.
-   Leading `1`: sticky bit.
-   The final three digits describe ordinary owner, group, and others
    permissions.

For example, `4755` means setuid plus ordinary permissions `755`. Do not
change system permission modes just to experiment; use a dedicated
practice directory and understand the command before running it.

## 6. Quick comparison

  -----------------------------------------------------------------------
  Special permission                  Main purpose
  ----------------------------------- -----------------------------------
  **setuid**                          An executable can use the file
                                      owner's effective user ID.

  **setgid**                          An executable can use the file's
                                      effective group; on directories,
                                      new entries generally inherit the
                                      directory's group.

  **Sticky bit**                      Restricts deletion or renaming of
                                      entries in shared writable
                                      directories.
  -----------------------------------------------------------------------

## 7. Commands practised

These commands inspect metadata; they do not change permissions:

``` bash
ls -l /usr/bin/passwd
stat -c '%a %A %U %G %n' /usr/bin/passwd
ls -ld /var/mail
ls -ld /tmp
```

In `stat -c`:

-   `%a` --- numeric permissions
-   `%A` --- symbolic permissions
-   `%U` --- owner username
-   `%G` --- group name
-   `%n` --- file or directory name

## 8. Knowledge-check results

The learner correctly answered:

1.  The setuid marker in `/usr/bin/passwd` indicates setuid is enabled
    and owner execute is enabled.
2.  The setgid bit on a directory generally makes new entries inherit
    the directory's group.
3.  The sticky bit on `/tmp` generally restricts ordinary users from
    deleting or renaming entries owned by other users.
4.  In `-rwxr-sr-x`, the group-position `s` indicates setgid is enabled
    and group execute is enabled.

## 9. Safety notes

-   Prefer read-only inspection commands when learning from real system
    paths.
-   Do not casually add or remove setuid, setgid, or sticky bits on
    system files.
-   Do not run unfamiliar commands with `sudo` merely to see what
    happens.
-   Use an isolated practice directory for future permission
    experiments.
-   Remember that special permissions interact with operating-system
    rules, program behavior, filesystem behavior, and other access
    controls.

## 10. Interview-style summary

**Question:** What are Linux special permissions?

**Answer:** Linux has special permission bits called setuid, setgid, and
the sticky bit. Setuid can let an executable use its owner's effective
user ID. Setgid can let an executable use its file group and causes new
entries in a setgid directory generally to inherit that directory's
group. The sticky bit restricts deletion or renaming of other users'
entries in shared writable directories such as `/tmp`.

## Status

This lesson's read-only inspection and knowledge checks are complete.
The overall learning phase should only be marked complete after the
learner confirms the milestone and any required repository documentation
is saved.
