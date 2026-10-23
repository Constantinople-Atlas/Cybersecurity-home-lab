# Cybersecurity Home Lab — Phase 0 Notes

## Purpose

This document records the verified initial setup of the cybersecurity home lab and provides a reproducible reference for the next learning phase.

## Lab inventory

### Host PC
- Host OS: Windows 10, version 22H2 (standard support ended on 2025-10-14; plan migration to a supported, patched OS).
- Virtualization: Oracle VirtualBox 7.2.14.
- Default VM folder: `C:\Users\Contessina\VirtualBox VMs`.

### New lab VM
- VM name / hostname: `lab-base`
- OS: Ubuntu Server 24.04.5 LTS (`noble`)
- Linux account: `atlas_admin`
- RAM: 2 GB
- vCPUs: 2
- Virtual disk: 25 GB dynamically allocated
- Root filesystem: `/dev/sda2`, about 25 GB total
- Guest network: VirtualBox NAT; guest IPv4 observed as `10.0.2.15` at time of setup (may change).

### Existing VMs — preserve
- Existing Ubuntu VM: configured as Ubuntu 24.10, which is out of support. Preserve it; do not use it for security-sensitive tasks.
- Existing Kali Linux VM: preserve for later controlled, authorized lab practice.
- Do not delete or overwrite pre-existing VMs.

## Phase 0 verification

The following checks were reported as successful during setup:
- Ubuntu Server 24.04.5 LTS installed and booted.
- `whoami` returned `atlas_admin`.
- `hostname` returned `lab-base`.
- `ping -c 4 ubuntu.com` succeeded with 4 replies and 0% packet loss.
- SSH server installed, enabled, and active after reboot.
- SSH listened on guest port 22 over IPv4 and IPv6.
- Windows PowerShell connected to the guest using SSH through VirtualBox NAT port forwarding.
- The client host-key fingerprint was compared with the fingerprint shown by the Ubuntu console before accepting it.
- `df -h`, `free -h`, and `systemctl status ssh --no-pager` were inspected.

## SSH access configuration

VirtualBox Adapter 1 is set to NAT. The port-forwarding rule is:

| Setting | Value |
|---|---|
| Name | `Ubuntu-SSH` |
| Protocol | TCP |
| Host IP | `127.0.0.1` |
| Host port | `2222` |
| Guest IP | blank |
| Guest port | `22` |

From Windows PowerShell, the connection commands are:

```powershell
Test-NetConnection 127.0.0.1 -Port 2222
ssh -p 2222 atlas_admin@127.0.0.1
```

The reported TCP test succeeded. The SSH login succeeded.

Keep the host IP bound to `127.0.0.1`. Do not change it to `0.0.0.0`, expose SSH through the router, or publish it to the public Internet.

On the Ubuntu VM, the SSH service was verified with:

```bash
sudo systemctl enable --now ssh
systemctl is-active ssh
systemctl is-enabled ssh
sudo ss -lntp | grep ssh
systemctl status ssh --no-pager
```

Reported results included `active`, `enabled`, and listeners on guest port 22.

To inspect the ED25519 host-key fingerprint on the Ubuntu console:

```bash
sudo ssh-keygen -lf /etc/ssh/ssh_host_ed25519_key.pub
```

Compare the fingerprint with the SSH client's prompt before accepting a host key. If a previously trusted key unexpectedly changes, stop and investigate instead of blindly accepting it.

## Recorded resource checks

### Disk space (`df -h`)
At the time of inspection, `/dev/sda2` showed approximately:
- Size: 25 GB
- Used: 4.8 GB
- Available: 19 GB
- Usage: 21%

### Memory (`free -h`)
At the time of inspection, the VM showed approximately:
- RAM total: 1.9 GiB
- Used: 333 MiB
- Free: 1.5 GiB
- Available: 1.6 GiB
- Swap total: 2.0 GiB
- Swap used: 0 B

These values are point-in-time observations and will vary.

## Repository workflow

Local project folder:

`C:\Users\Contessina\cybersecurity-home-lab`

Remote repository:

`https://github.com/Constantinople-Atlas/Cybersecurity-home-lab.git`

The local repository is initialized on `main`, with `origin/main` configured as its upstream branch. The first README commit was published successfully. Git's project-specific commit identity is:
- Name: `Cybersecurity Home Lab`
- Email: GitHub-provided noreply address

The global Git identity remains separate and unchanged.

A normal update workflow is:
1. Edit or create documentation.
2. Review changes with `git status` and, for tracked files, `git diff`.
3. Stage only the intended file(s), for example `git add docs/phase-0/setup-and-inventory.md`.
4. Review staged changes with `git diff --cached`.
5. Commit with a descriptive message, for example `git commit -m "Document Phase 0 lab setup"`.
6. Push with `git push`.
7. Verify with `git status` and inspect the GitHub repository.

## Repository safety

The root `.gitignore` should exclude common local secrets and large VM media:
- `.env` files (except `.env.example`)
- private key extensions such as `*.pem` and `*.key`
- common private key filenames such as `id_rsa` and `id_ed25519`
- editor and OS metadata
- VirtualBox disk formats and ISO files (`*.vdi`, `*.vmdk`, `*.vhd`, `*.vhdx`, `*.iso`)

A `.gitignore` is not a complete secret scanner. It does not untrack files that are already tracked. Always inspect `git status` and staged changes before committing or pushing. Never publish passwords, private keys, access tokens, VM disks, or ISO images.

## Security notes and limitations

- Windows 10 standard support ended on 2025-10-14. Unless covered by a supported Extended Security Updates or equivalent arrangement, plan to move to a supported, patched host OS.
- Ubuntu 24.10 is out of support; preserve the existing VM but avoid security-sensitive use.
- Keep all testing limited to systems you own or are explicitly authorized to test.
- Back up important files before major changes. VirtualBox snapshots are helpful but do not replace independent backups.
- Do not record passwords, private keys, or tokens in this repository.

## Phase 0 acceptance checklist

- [x] Ubuntu Server 24.04.5 LTS installed and booted.
- [x] Network connectivity and DNS verified.
- [x] SSH installed, active, and enabled.
- [x] VirtualBox NAT port forwarding configured with host loopback binding.
- [x] SSH remote login from Windows verified.
- [x] Disk, memory, and SSH status inspected.
- [x] User passed the Phase 0 learning check.
- [ ] Keep documentation current and confirm milestone completion with the learner.

## Next lesson

Phase 1, Lesson 1: Linux users, groups, file permissions, and sudo.

Start with safe read-only commands in the Ubuntu SSH session:

```bash
whoami
id
groups
ls -ld ~
```

Explain each command and each output field before asking the learner to interpret the results. Do not create or modify users, groups, or permissions until the inspection output is understood and any proposed change has been explained.

## Rebuild outline

1. Install a supported, patched host operating system.
2. Install a supported version of VirtualBox.
3. Download Ubuntu Server 24.04 LTS from the official Ubuntu site and verify its published checksum.
4. Create a new VM without deleting or overwriting existing VMs.
5. Install Ubuntu Server, create a dedicated admin account, apply updates, and reboot.
6. Install and verify SSH.
7. Configure VirtualBox NAT port forwarding with host IP 127.0.0.1, host port 2222, and guest port 22.
8. Verify host-to-guest SSH access and compare the host-key fingerprint.
9. Re-run disk, memory, and service checks.
10. Record any differences from this document; do not assume guest IPs or resource values will remain identical.
