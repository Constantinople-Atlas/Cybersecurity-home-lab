# Linux Permissions, umask, and Hard Links --- Lesson Notes

**Lab:** `lab-base` --- Ubuntu Server 24.04.5 LTS\
**Linux user:** `atlas_admin`\
**Purpose:** Keep the lessons on ordinary Linux permissions, default
permissions (`umask`), and hard links together as a reusable reference.

> These notes record the commands, outputs, and concepts established
> during the lesson. Where an exact command output was not preserved in
> the lesson handoff, it is not invented here.

## 1. Ordinary Linux permissions

Linux permission listings show access for three classes:

-   **Owner (user):** the user who owns the file or directory.
-   **Group:** users associated with the file's group.
-   **Others:** everyone else.

The basic permission symbols are:

  -----------------------------------------------------------------------
  Symbol                  For a regular file      For a directory
  ----------------------- ----------------------- -----------------------
  `r` --- read            Read the file's         List directory entries;
                          contents.               actual access may also
                                                  depend on execute
                                                  permission.

  `w` --- write           Modify the file's       Create, delete, or
                          contents.               rename entries, subject
                                                  to directory
                                                  permissions and other
                                                  restrictions.

  `x` --- execute         Run the file as a       Traverse/access entries
                          program/script, when    by name; often needed
                          applicable.             to enter or access
                                                  items in the directory.

  `-`                     That permission is not  That permission is not
                          granted in this         granted in this
                          permission class.       permission class.
  -----------------------------------------------------------------------

### Reading a permission string

Example:

``` text
-rwxr-xr--
```

Read the string in groups:

-   First character `-`: regular file (`d` would indicate a directory).
-   `rwx`: owner can read, write, and execute.
-   `r-x`: group can read and execute, but not write.
-   `r--`: others can read, but cannot write or execute.

The permissions alone do not describe every possible access decision.
Other controls---such as ACLs, mount options, and application
behavior---can also matter.

## 2. Numeric (octal) permissions

Each ordinary permission has a numeric value:

-   `r` = 4
-   `w` = 2
-   `x` = 1

Add the values for each class:

  Mode    Owner         Group         Others
  ------- ------------- ------------- -------------
  `755`   `7` = `rwx`   `5` = `r-x`   `5` = `r-x`
  `644`   `6` = `rw-`   `4` = `r--`   `4` = `r--`
  `775`   `7` = `rwx`   `7` = `rwx`   `5` = `r-x`
  `640`   `6` = `rw-`   `4` = `r--`   `0` = `---`
  `750`   `7` = `rwx`   `5` = `r-x`   `0` = `---`

Common examples:

-   A regular file with mode `644` is usually readable by everyone, but
    writable only by its owner.
-   A directory with mode `755` lets everyone traverse it and list its
    entries, but only the owner can create/delete/rename entries,
    assuming no other access controls.
-   A directory with mode `775` gives the owner and group write access;
    this is common for shared-work directories when group ownership is
    configured appropriately.

## 3. Commands for inspecting and changing permissions

### Inspect permissions

``` bash
ls -l
ls -ld DIRECTORY
stat -c '%a %A %U %G %n' PATH
```

-   `ls -l` shows detailed information about entries.
-   `ls -ld DIRECTORY` shows information about the directory itself, not
    its contents.
-   `stat -c` prints selected fields:
    -   `%a`: numeric permission mode
    -   `%A`: symbolic permission mode
    -   `%U`: owner username
    -   `%G`: group name
    -   `%n`: path/name

### Change ordinary permissions with `chmod`

Examples:

``` bash
chmod 644 example.txt
chmod 775 example-dir
```

-   `chmod` changes permission bits.
-   `644` and `775` are the desired numeric modes.
-   `example.txt` and `example-dir` are example paths; replace them only
    when you know exactly which practice item you intend to change.

A symbolic example:

``` bash
chmod u+x script.sh
```

This adds execute permission (`x`) for the owner (`u`) without changing
the other permission classes.

### Ownership with `chown`

`chown` changes a file's owner and, optionally, group. Ownership changes
can affect access, so inspect the target carefully before using it. Do
not experiment on system files.

**Safety rule:** Learn changes in a dedicated practice directory. Do not
casually change permissions or ownership of operating-system files, and
do not use `sudo` unless you understand why elevated privileges are
required.

## 4. Practical permissions recorded in the lab

The practice items were restored to these modes:

-   `permissions-lab/` --- directory mode `775`
-   `permissions-lab/example.txt` --- file mode `644`

The lesson also recorded these existing practice items:

-   `umask-test.txt` --- mode `664`
-   `umask-test-dir/` --- directory mode `775`

These items were intentionally left in place. Do not delete them as
cleanup unless you explicitly decide to do so.

## 5. Understanding `umask`

A **umask** is a permission mask applied when a program creates a new
file or directory. It removes permission bits from the mode the program
requests; it does not add permissions.

Typical starting modes used to explain the concept are:

-   Regular files: `666` (read and write for owner, group, and others;
    no execute bits by default).
-   Directories: `777` (read, write, and execute for all three classes).

The umask removes selected bits from these starting modes. Exact results
can depend on the program's requested mode and system behavior, but the
examples below reflect the ordinary modes used in this lesson.

### The original umask

The lab's original output was:

``` text
0002
```

The symbolic output from `umask -S` was:

``` text
u=rwx,g=rwx,o=rx
```

This means the mask normally removes the others' write bit. With the
typical starting modes:

-   New regular file: `666` becomes `664` (`rw-rw-r--`).
-   New directory: `777` becomes `775` (`rwxrwxr-x`).

### Temporary test with umask `0027`

The learner temporarily set:

``` bash
umask 0027
```

This changed the current shell's umask for the test. The mask removes
group write and all ordinary permissions for others.

Observed output for the test file:

``` text
640 -rw-r----- umask-0027-test.txt
```

Observed output for the test directory:

``` text
750 drwxr-x--- umask-0027-test-dir/
```

Why these results?

-   Typical file mode: `666`; umask `027` removes group write (`020`)
    and all others' permissions (`007`), resulting in `640`.
-   Typical directory mode: `777`; umask `027` removes group write and
    all others' permissions, resulting in `750`.

### Restoring the original umask

The original mask was restored in the shell with:

``` bash
umask 0002
```

The learner verified that the output returned to:

``` text
0002
```

This was a temporary shell-level exercise; it did not establish that the
setting persists across new login sessions.

### Important notes about umask

-   The umask is not a replacement for correct ownership and group
    design.
-   `umask` does not normally add execute permission to a regular file.
-   A program can request a mode more restrictive than the usual
    starting modes.
-   A shell's umask usually affects programs started from that shell;
    persistence depends on shell and login configuration.
-   Avoid changing system-wide defaults until you understand the impact
    and have a clear reason.

## 6. Hard links and inodes

A hard link is another directory entry that points to the same
underlying inode as an existing file. The inode stores file metadata and
references the file's data blocks.

Key ideas:

-   Two hard-link names refer to the same underlying file data.
-   If the contents are changed through one name, the changed contents
    are visible through the other name.
-   Deleting one hard-link name removes that directory entry; the data
    remains accessible through another hard link.
-   Storage can be reclaimed when the link count reaches zero and no
    process still has the file open.
-   Hard links generally cannot cross filesystem boundaries. Directories
    are normally not hard-linked by ordinary users.
-   A hard link is different from a symbolic link, which stores a path
    to another name.

In `ls -l`, the number after the permission string is the hard-link
count. A count of `1` means only one hard-link directory entry is
reported for that inode.

### Safe inspection commands

For a practice file, these commands can help compare inode numbers and
link counts:

``` bash
ls -li example.txt
stat example.txt
```

Use an existing practice file you recognize. Do not delete or modify
files just to test hard-link behavior until you have created a dedicated
practice exercise and know which names refer to the same inode.

## 7. How this connects to special permissions

The next lesson covered the special permission bits:

-   **setuid:** can make an executable run with the file owner's
    effective user ID.
-   **setgid:** can make an executable use the file's group as its
    effective group; on a directory, new entries generally inherit the
    directory's group.
-   **Sticky bit:** restricts deletion or renaming of entries in shared
    writable directories such as `/tmp`.

Those topics are documented separately in
`linux-special-permissions-lesson.md`. They should also be included in
the consolidated milestone guide.

## 8. Knowledge checks covered

Concepts checked during the lesson included:

1.  Interpreting owner, group, and others' permission bits.
2.  Recognizing that directory execute permission (`x`) allows
    traversal/access by name.
3.  Calculating new file and directory modes with umask `0007`
    (directory result: `770`).
4.  Understanding the observed umask `0027` results (`640` file and
    `750` directory).
5.  Understanding that hard links share the same inode/data, so edits
    through one name are visible through the other.
6.  Understanding that deleting one hard-link name does not remove the
    data while another hard link remains.

## 9. Quick reference

  -----------------------------------------------------------------------
  Item                                Remember
  ----------------------------------- -----------------------------------
  `r`                                 Read file contents; list directory
                                      entries (with needed directory
                                      access).

  `w`                                 Modify file contents;
                                      create/delete/rename directory
                                      entries, subject to other controls.

  `x`                                 Execute a file; traverse a
                                      directory.

  `chmod`                             Change permission bits.

  `chown`                             Change owner and/or group
                                      ownership.

  `umask`                             Mask off permissions when new
                                      files/directories are created.

  Hard link                           Another name for the same
                                      inode/data.

  `ls -l` link-count field            Number of hard-link directory
                                      entries.
  -----------------------------------------------------------------------

## 10. Suggested portfolio location

Save this file in the cybersecurity home lab repository, for example:

``` text
docs/linux/permissions-umask-hard-links.md
```

The special-permissions notes can sit beside it:

``` text
docs/linux/linux-special-permissions-lesson.md
```

## Status

These notes consolidate the ordinary permissions, umask, and hard-link
concepts covered so far. They are not, by themselves, confirmation that
a whole milestone or phase is complete. The milestone PDF should combine
the relevant lessons after the user and instructor agree the milestone
is complete.
