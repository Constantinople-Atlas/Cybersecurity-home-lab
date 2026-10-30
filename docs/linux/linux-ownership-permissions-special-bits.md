# Linux Ownership, Permissions, and Special Bits Lab

**Environment:** Ubuntu Server 24.04.5 LTS  
**User:** `atlas_admin`  
**Practice account:** `labuser`  
**Practice directory:** `/home/atlas_admin/linux-practice/ownership-permissions`

## 1. Learning objectives

This lab covered file ownership, group ownership, numeric permissions, directory permissions, setgid, the sticky bit, supplementary groups, and verification with `ls`, `stat`, and `id`.

All practical tests took place in a dedicated practice directory. No system file permissions were changed.

## 2. Inspecting ownership and permissions

Linux files and directories have an owner, a group, and permission bits for the owner, group, and others.

Inspect a file:

```bash
ls -l ownership-file.txt
```

Inspect its symbolic mode, numeric mode, owner, group, and name:

```bash
stat -c '%A %a %U %G %n' ownership-file.txt
```

The format fields are `%A` (symbolic permissions), `%a` (numeric permissions), `%U` (owner), `%G` (group), and `%n` (name).

Commands practised:

```bash
chgrp adm ownership-file.txt
sudo chown labuser ownership-file.txt
```

These temporary changes were reversed. The final owner and group were both `atlas_admin`.

## 3. Numeric permissions and chmod

Permission values are:

| Number | Permission |
|---:|---|
| 4 | Read (`r`) |
| 2 | Write (`w`) |
| 1 | Execute (`x`) |
| 0 | None |

Values are added for each class. For example, `664` means owner `rw-`, group `rw-`, others `r--`.

```bash
chmod 640 ownership-file.txt
```

Mode `640` gives the owner read/write, the group read-only, and others no access.

The file was temporarily changed to `660` to test group write access. At the end, it was restored to `664`.

Verified final output:

```text
664 -rw-rw-r-- atlas_admin atlas_admin ownership-file.txt
```

Initially, `labuser` was not in the `atlas_admin` group. With the file at `640`, this read attempt failed because `labuser` had no applicable permission:

```bash
sudo -u labuser cat ownership-file.txt
```

After temporarily adding `labuser` to the `atlas_admin` group, reading succeeded through the group permission. Writing was still denied while the file mode was `640`.

At mode `660`, the group had read/write access:

```bash
chmod 660 ownership-file.txt
sudo -u labuser sh -c 'echo "Written by labuser" >> ownership-file.txt'
```

The write succeeded, but the existing file remained owned by `atlas_admin`. Writing to an existing file does not automatically change its ownership.

The verified file content included:

```text
This file is for ownership and permissions practice.
Written by labuser
```

The content was retained, and the file was restored to mode `664`.

## 4. Directory permissions

Directory permissions differ from file permissions:

| Permission | Meaning on a directory |
|---|---|
| Read (`r`) | List directory entry names |
| Write (`w`) | Create, delete, or rename entries, normally together with execute |
| Execute (`x`) | Traverse/search the directory and access entries by name, subject to their own permissions |

The practice directory was initially mode `775`:

```bash
chmod 775 directory-permissions
```

It was temporarily changed to `760`:

```bash
chmod 760 directory-permissions
```

The test demonstrated that listing names and traversing a directory are different. In the observed test, `labuser` could list the name, but reading the file by path failed with `Permission denied` because the group lacked directory execute/traverse permission.

The directory was restored to `775`.

A file was created as `labuser`:

```bash
sudo -u labuser touch directory-permissions/created-by-labuser.txt
```

It was owned by `labuser` with group `labuser`.

## 5. Setgid on a directory

Setgid on a directory makes newly created files and subdirectories inherit the directory's group. It does not change the creating user's ownership of the new file.

Enable setgid:

```bash
chmod g+s directory-permissions
```

The directory showed `drwxrwsr-x`, numeric mode `2775`. The leading `2` represents setgid.

Create a file as `labuser`:

```bash
sudo -u labuser touch directory-permissions/setgid-test.txt
```

Verified result:

```text
-rw-rw-r-- 1 labuser atlas_admin 0 setgid-test.txt
```

The owner was `labuser`, while the group was inherited from the directory: `atlas_admin`.

The earlier `created-by-labuser.txt` remained owned by `labuser:labuser`; enabling setgid does not retroactively change existing files.

Useful checks:

```bash
ls -ld directory-permissions
stat -c '%A %a %U %G %n' directory-permissions
ls -l directory-permissions/
```

## 6. Sticky bit on a directory

The sticky bit restricts who may delete or rename entries in a shared writable directory. The permitted parties are generally the file owner, directory owner, or root, provided the necessary directory access is available.

Enable it:

```bash
chmod +t sticky-bit-practice
```

The directory showed `drwxrwxr-t`, numeric mode `1775`. The leading `1` represents the sticky bit.

Create a file as `atlas_admin`:

```bash
touch sticky-bit-practice/atlas-file.txt
```

Attempt deletion as `labuser`:

```bash
sudo -u labuser rm sticky-bit-practice/atlas-file.txt
```

The attempt failed with:

```text
rm: cannot remove 'sticky-bit-practice/atlas-file.txt': Operation not permitted
```

Create a file as `labuser`:

```bash
sudo -u labuser touch sticky-bit-practice/labuser-file.txt
ls -l sticky-bit-practice/
```

Verified ownership:

```text
-rw-rw-r-- 1 atlas_admin atlas_admin 0 atlas-file.txt
-rw-rw-r-- 1 labuser     labuser     0 labuser-file.txt
```

`labuser` could delete its own file:

```bash
sudo -u labuser rm sticky-bit-practice/labuser-file.txt
```

The directory owner could also remove entries. The practice directory was empty after the tests.

Useful checks:

```bash
ls -ld sticky-bit-practice
stat -c '%A %a %U %G %n' sticky-bit-practice
ls -l sticky-bit-practice/
```

The sticky bit does not replace ordinary directory permissions; it adds a restriction on who may remove or rename entries.

## 7. Supplementary group membership and least privilege

The practice account's groups were inspected with:

```bash
id labuser
```

For the group-access experiment, `labuser` was temporarily added to the `atlas_admin` supplementary group:

```bash
sudo usermod -aG atlas_admin labuser
```

`-a` appends supplementary group membership; `-G` specifies the supplementary groups. The `-a` matters because using `-G` without it can replace existing supplementary group memberships.

The temporary membership was verified:

```text
uid=1001(labuser) gid=1001(labuser) groups=1001(labuser),100(users),1000(atlas_admin)
```

After the tests, the additional membership was removed:

```bash
sudo gpasswd -d labuser atlas_admin
```

Final verification:

```text
uid=1001(labuser) gid=1001(labuser) groups=1001(labuser),100(users)
```

The `labuser` account was kept. Only its supplementary membership in `atlas_admin` was removed.

This demonstrates the **principle of least privilege**: give an account only the access it needs. Group membership may grant access to multiple resources, not just the file being tested.

## 8. Final verified state

Final inspection commands:

```bash
ls -ld ownership-file.txt directory-permissions sticky-bit-practice
ls -l directory-permissions/ sticky-bit-practice/
id labuser
```

Verified main items:

```text
drwxrwsr-x 2 atlas_admin atlas_admin 4096 directory-permissions
-rw-rw-r-- 1 atlas_admin atlas_admin   72 ownership-file.txt
drwxrwxr-t 2 atlas_admin atlas_admin 4096 sticky-bit-practice
```

Verified contents of `directory-permissions/`:

```text
-rw-rw-r-- 1 labuser     labuser     0 created-by-labuser.txt
-rw-rw-r-- 1 labuser     atlas_admin 0 setgid-test.txt
-rw-rw-r-- 1 atlas_admin atlas_admin 0 test.txt
```

`sticky-bit-practice/` was empty after the deletion tests.

Final account groups:

```text
uid=1001(labuser) gid=1001(labuser) groups=1001(labuser),100(users)
```

### Deliberately retained state

- `ownership-file.txt` was restored to mode `664`, owner/group `atlas_admin:atlas_admin`.
- `directory-permissions/` was left at mode `2775` as a setgid example.
- `sticky-bit-practice/` was left at mode `1775` as a sticky-bit example.
- `labuser` remains, but is no longer a member of `atlas_admin`.
- The test content in `ownership-file.txt` was retained.
- No system file permissions were changed.

## 9. Troubleshooting notes

### Permission denied when reading a file

Inspect the path components, file permissions, and account groups:

```bash
ls -ld .
ls -l filename
id username
```

A user may have permission on a file but still be unable to reach it if a parent directory does not grant execute/traverse permission.

### A user cannot write to a file

Inspect:

```bash
ls -l filename
id username
```

Check whether the user is the owner, belongs to the file's group, or is covered by the `others` permissions. For creating, deleting, or renaming directory entries, also inspect the parent directory permissions.

### A new file has an unexpected group

Inspect the parent directory:

```bash
ls -ld parent-directory
```

Setgid may cause new entries to inherit that directory's group.

### A user cannot delete another user's file

Inspect the directory:

```bash
ls -ld shared-directory
```

The sticky bit may be enabled. A user cannot delete another user's entry merely because the directory is writable.

## 10. Knowledge review

1. What does mode `640` mean?
2. What is the difference between read and execute permission on a directory?
3. What does setgid do on a directory?
4. Does setgid change the owner of a newly created file?
5. Who can delete or rename an entry in a sticky-bit directory?
6. What does the leading `2` in `2775` represent?
7. What does the leading `1` in `1775` represent?
8. Why was `labuser` removed from the `atlas_admin` supplementary group?

### Answer key

1. Owner read/write; group read; others no permissions.
2. Read lists names; execute permits traversal/search and access by name, subject to other permissions.
3. New entries inherit the directory's group.
4. No. The creating user normally remains the owner.
5. The file owner, directory owner, or root, subject to the required directory access.
6. Setgid.
7. Sticky bit.
8. To remove temporary access and follow least privilege.

## 11. Rebuild guide

Use a disposable lab VM and a directory you own. The following recreates the core directory and special-bit demonstrations; repeat account and group changes only after checking the test account and its current memberships.

```bash
mkdir -p ~/linux-practice/ownership-permissions
cd ~/linux-practice/ownership-permissions
touch ownership-file.txt
chmod 664 ownership-file.txt
mkdir directory-permissions sticky-bit-practice
chmod 775 directory-permissions sticky-bit-practice
chmod g+s directory-permissions
chmod +t sticky-bit-practice
ls -ld directory-permissions sticky-bit-practice
```

Do not copy account-management commands blindly onto a production system.

## 12. GitHub portfolio placement

Suggested repository path:

```text
docs/linux/linux-ownership-permissions-special-bits.md
```

Suggested commit message:

```text
docs(linux): add ownership permissions and special bits lab
```

Before committing, check that the document contains no passwords, private keys, password hashes, or other secrets.

Repository: https://github.com/Constantinople-Atlas/Cybersecurity-home-lab
