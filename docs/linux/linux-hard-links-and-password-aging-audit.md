# Linux Practice: Hard Links and Password-Aging Audit

## Lab environment

-   Virtual machine: `lab-base`
-   Operating system: Ubuntu Server 24.04.5 LTS
-   Linux account: `atlas_admin`
-   Practice directory: `/home/atlas_admin/linux-practice/hard-links`

This document records two completed, read-only/practice exercises. The
hard-link exercise created and modified files only inside the practice
directory. The password-aging exercise inspected account settings and
did not change them.

------------------------------------------------------------------------

# Part 1: Hard-link practical lab

## 1. Create the practice file

Command:

``` bash
printf 'This is my hard-link practice file.\\n' > original.txt
```

What it does:

-   `printf` writes the supplied text.
-   `\\n` adds a newline at the end.
-   `>` redirects the output into `original.txt`.
-   If the file did not exist, it is created. If it already existed, `>`
    would overwrite its contents.

The file was checked with:

``` bash
cat original.txt
```

Observed content:

``` text
This is my hard-link practice file.
```

## 2. Create a second hard-link name

Command:

``` bash
ln original.txt second-name.txt
```

`ln` creates a hard link by default when used this way. It creates
another directory entry for the same underlying filesystem object; it
does not make an independent copy of the file's contents.

## 3. Compare inode numbers and link counts

Command:

``` bash
ls -li original.txt second-name.txt
```

Observed output:

``` text
525370 -rw-rw-r-- 2 atlas_admin atlas_admin 36 Oct 29 17:26 original.txt
525370 -rw-rw-r-- 2 atlas_admin atlas_admin 36 Oct 29 17:26 second-name.txt
```

How to read the important columns:

-   `525370` is the inode number. Both names show the same inode.
-   `-rw-rw-r--` shows the file's permissions.
-   `2` is the hard-link count: two directory entries refer to this
    inode.
-   `atlas_admin` is the owner and group.
-   `36` is the file size in bytes.
-   The final field is the filename.

The matching inode number is the key evidence that these are hard links
to the same file object.

## 4. Read through both names

Command:

``` bash
cat original.txt second-name.txt
```

Both names displayed the same original sentence. This demonstrated that
both names access the same file contents.

## 5. Append through the second name

Command:

``` bash
printf 'This line was added through the second name. \n' >> second-name.txt
```

`>>` appends output to the end of a file. Unlike `>`, it does not
replace the existing contents.

The file was then read through the first name:

``` bash
cat original.txt
```

Observed content:

``` text
This is my hard-link practice file.
This line was added through the second name.
```

The appended line was visible through `original.txt` because both names
refer to the same inode and data.

A second inode check showed:

``` text
525370 -rw-rw-r-- 2 atlas_admin atlas_admin 82 Oct 29 17:33 original.txt
525370 -rw-rw-r-- 2 atlas_admin atlas_admin 82 Oct 29 17:33 second-name.txt
```

The inode remained `525370`, the link count remained `2`, and the size
increased from 36 to 82 bytes.

## 6. Remove one name and verify the other

Command:

``` bash
rm original.txt
```

This removed the directory entry named `original.txt`. It did not
immediately remove the file's data because another hard link still
existed.

The remaining name was checked:

``` bash
cat second-name.txt
```

It still displayed both lines.

Then:

``` bash
ls -li second-name.txt
```

Observed output:

``` text
525370 -rw-rw-r-- 1 atlas_admin atlas_admin 82 Oct 29 17:33 second-name.txt
```

The inode stayed `525370`, while the link count decreased from `2` to
`1`. The remaining filename continued to provide access to the file.

**Current lab artifact:** `second-name.txt` remains in the practice
directory. It was intentionally left in place at the end of the
exercise.

## Hard-link concepts learned

-   A hard link is another filename (directory entry) for the same
    inode.
-   Two hard-link names for the same file show the same inode number.
-   Changes made through one hard-link name are visible through the
    other.
-   Removing one hard-link name reduces the link count; it does not
    remove the data while another link remains.
-   When the link count reaches zero and no process still has the file
    open, the filesystem can reclaim the inode and its data.
-   Hard links cannot cross filesystem boundaries. Ordinary users
    generally cannot create hard links to directories.

## Knowledge-check results

1.  Why could the remaining name still read the file after
    `original.txt` was removed? **Correct answer: B.** Both names
    pointed to the same inode, and the data remained while another hard
    link existed.
2.  What does the inode number tell us in this exercise? **Correct
    answer: C.** It identifies the filesystem object; both hard-link
    names shared it.
3.  What happens when the final hard link is removed and no process has
    the file open? **Correct answer: D.** The link count reaches zero
    and Linux can reclaim the data and inode.

All three answers were correct.

------------------------------------------------------------------------

# Part 2: Read-only password-aging audit

This exercise inspected password-aging information for `atlas_admin`. No
password-aging settings were changed.

## 1. Check password status

Command:

``` bash
sudo passwd -S atlas_admin
```

Observed output:

``` text
atlas_admin P 2026-10-23 0 99999 7 -1
```

The fields mean:

1.  `atlas_admin` --- account name.
2.  `P` --- a password is set.
3.  `2026-10-23` --- date of the last password change.
4.  `0` --- minimum number of days between password changes.
5.  `99999` --- maximum number of days before password expiry; this is
    an extremely long period and is commonly treated as no practical
    expiry.
6.  `7` --- warning period, in days, before password expiry.
7.  `-1` --- no inactivity period is configured.

## 2. Read the detailed password-aging settings

Command:

``` bash
sudo chage -l atlas_admin
```

Observed output:

``` text
Last password change                                    : Oct 23, 2026
Password expires                                        : never
Password inactive                                       : never
Account expires                                         : never
Minimum number of days between password change          : 0
Maximum number of days between password change          : 99999
Number of days of warning before password expires       : 7
```

Interpretation:

-   **Last password change:** October 23, 2026.
-   **Password expires: never:** the current password-aging
    configuration does not schedule a password expiry date.
-   **Password inactive: never:** no inactivity period after password
    expiry is configured.
-   **Account expires: never:** no account-expiration date is
    configured.
-   **Minimum days: 0:** there is no minimum waiting period between
    password changes.
-   **Maximum days: 99999:** the maximum-age value is very long.
-   **Warning days: 7:** the configured warning period is seven days
    before an expiry.

These commands report account status; they do not change the settings.

## Important distinction: password expiry vs SSH authentication

The result `Password expires: never` does **not** mean that:

-   the account has no password;
-   the account cannot be compromised; or
-   SSH password authentication is enabled.

In this lab, SSH password authentication had previously been disabled
and public-key authentication verified. The Ubuntu account password is
still used for local authentication tasks such as `sudo`. Password-aging
policy and SSH authentication method are separate settings.

## Knowledge-check results

1.  What does `Password expires: never` mean? **Correct answer: B.** The
    current password-aging settings do not schedule password expiration.
2.  What does `sudo passwd -S atlas_admin` display? **Correct answer:
    C.** Password status and aging information for the specified
    account.

Both answers were correct.

------------------------------------------------------------------------

# Current checkpoint and safety notes

-   The hard-link exercise is complete.
-   `second-name.txt` remains in
    `/home/atlas_admin/linux-practice/hard-links`.
-   The password-aging audit is complete and read-only.
-   No password-aging values were changed.
-   SSH password authentication remains disabled according to the
    previously verified configuration.
-   Keep the SSH private key and its passphrase private. Do not paste
    either into notes, GitHub, or chat.
-   Do not remove the remaining practice file unless you intentionally
    want to clean up the exercise.

## Suggested next steps

1.  Review Linux permissions, umask, hard links, account status, and SSH
    authentication.
2.  Add this lesson record to the cybersecurity GitHub repository.
3.  Review the documentation and Git changes before committing.
4.  Push to GitHub only after checking the changes and explicitly
    deciding to publish them.
