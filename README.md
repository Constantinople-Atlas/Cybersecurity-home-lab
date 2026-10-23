Cybersecurity Home Lab

About This Project

This repository documents my journey building and learning through a beginner-friendly cybersecurity home lab.

The goal is to develop practical skills in Linux system administration, networking, secure remote access, security hardening, and authorized cybersecurity testing.

Lab Environment

- Host operating system: Windows 10
- Virtualization platform: Oracle VirtualBox 7.2.14
- Primary lab VM: Ubuntu Server 24.04.5 LTS
- Additional existing VMs: Ubuntu and Kali Linux, reserved for later learning and controlled practice
- Primary lab hostname: lab-base
- Lab Linux account: atlas_admin

Progress

Phase 0 — Lab Setup and Verification

- [x] Installed Ubuntu Server 24.04.5 LTS in a new virtual machine.
- [x] Verified the operating system and Linux account.
- [x] Updated the system and rebooted successfully.
- [x] Installed and verified the SSH service.
- [x] Configured VirtualBox NAT port forwarding for SSH using host port 2222 and guest port 22.
- [x] Connected from Windows PowerShell to the Ubuntu VM over SSH.
- [x] Inspected disk space, memory, and SSH service status.

Phase 1 — Linux Administration

- [ ] Learn Linux users, groups, file permissions, and sudo.
- [ ] Practice inspecting permissions safely.
- [ ] Document commands, concepts, and troubleshooting.

Repository Structure

Documentation will be organized by phase, with supporting command references and troubleshooting notes.

Safety Principles

- Practice only on systems I own or have explicit permission to test.
- Keep SSH access restricted to the local host during initial setup.
- Never publish passwords, private keys, authentication tokens, or other secrets.
- Preserve existing virtual machines and make backups before significant changes.
- Document procedures so the lab can be rebuilt and verified.

Current Status

Phase 0 core setup and SSH connectivity have been verified. Phase 1 begins with Linux users, groups, permissions, and sudo.

This repository will be updated as lessons are completed and the lab evolves.