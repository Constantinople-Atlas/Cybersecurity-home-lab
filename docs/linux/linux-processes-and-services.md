# Phase 1 — Linux Processes and Services

**Cybersecurity Home Lab**  
**System:** Ubuntu Server 24.04.5 LTS  
**Hostname:** `lab-base`  
**Account:** `atlas_admin`  
**Status:** Completed  
**GitHub repository:** https://github.com/Constantinople-Atlas/Cybersecurity-home-lab

---

## 1. Learning objectives

This phase covered:

- Linux processes and programs.
- Process IDs (PIDs) and parent process IDs (PPIDs).
- Process inspection using `ps` and `pgrep`.
- Resource monitoring using `top`.
- Process ownership and states.
- Signals: SIGTERM, SIGKILL, SIGSTOP and SIGCONT.
- Services, daemons and systemd.
- Service inspection using `systemctl`.
- Log analysis using `journalctl`.
- SSH service and socket activation.
- Memory, CPU load and disk monitoring.
- Security investigation of root-owned processes.

## 2. Lab safety and scope

All process-management practice was performed using harmless test processes such as `sleep 300`. System services, SSH processes, and kernel threads were inspected but not terminated.

No service configuration was changed during this phase. The SSH hardening work completed in the earlier SSH lesson remained in place.

**Important:** A process running as root is not automatically malicious. A process name or a single resource snapshot is not enough to establish compromise. Investigate ownership, parent process, executable path, service configuration, logs, and behaviour together.

---

## 3. Process identification and relationships

### 3.1 Commands practised

```bash
ps
ps -e
ps -o pid,ppid,cmd
ps -p 6045 -f
pgrep -a bash
```

Key concepts:

- **PID:** Process ID; identifies a process while it exists.
- **PPID:** Parent process ID; identifies the process that started another process.
- `ps` displays process information.
- `ps -e` selects all processes.
- `ps -o` selects output columns.
- `ps -p PID` selects a specific PID.
- `pgrep -a bash` searches for processes matching `bash` and displays their command lines.
- A `?` in the TTY column means the process has no controlling terminal.

### 3.2 Actual observed output — process relationship

```text
    PID    PPID CMD
   6045    6044 -bash
   7135    6045 ps -o pid,ppid,cmd
```

This showed the Bash shell (PID 6045) as the parent of the `ps` command (PID 7135).

Another actual observation:

```text
UID          PID    PPID  C STIME TTY          TIME CMD
atlas_a+    6045    6044  0 11:40 pts/0    00:00:00 -bash
```

The displayed username `atlas_a+` was truncated by the output column; the account is `atlas_admin`.

The `pgrep` command returned:

```text
911 -bash
6045 -bash
```

This showed two Bash processes at that point in time. PIDs are temporary and can change after a process exits or the system restarts.

---

## 4. Monitoring processes and resource consumption

### 4.1 `top`

`top` provides a live, updating view of processes and system resources. It can show CPU use, memory use, process states, and load averages.

Actual snapshot observed during the lesson:

```text
top - 17:25:45 up 18:32,  2 users,  load average: 0.00, 0.01, 0.00
Tasks: 114 total,   1 running, 113 sleeping,   0 stopped,   0 zombie
%Cpu(s):  0.2 us,  0.8 sy,  0.0 ni, 98.9 id,  0.0 wa,  0.0 hi,  0.2 si,  0.0 st
MiB Mem :   1967.7 total,   1085.3 free,    356.1 used,    684.6 buff/cache
MiB Swap:   2048.0 total,   2048.0 free,      0.0 used.   1611.6 avail Mem
```

Interpretation:

- 114 tasks were listed; 1 was running and 113 were sleeping.
- There were no stopped or zombie processes in this snapshot.
- CPU idle (`id`) was 98.9%, indicating very low CPU activity at that moment.
- The VM had approximately 1.9 GiB RAM and 2 GiB swap.
- No swap was being used in this snapshot.

A snapshot describes one moment. It does not establish long-term system health.

### 4.2 Sorting process information

Command practised:

```bash
ps -eo user,pid,ppid,%cpu,%mem,comm --sort=-%cpu
```

This displays selected process fields and sorts by CPU use, highest first.

During one observation, the `ps` command itself briefly showed 100% CPU. This is a short-lived measurement effect: the command can consume CPU while collecting and printing its own snapshot. A single high reading should be investigated and rechecked rather than treated as proof of malicious activity.

---

## 5. Safe process management and signals

Signals are notifications sent to processes. They can request a process to change state or terminate.

| Signal | Number | Purpose |
|---|---:|---|
| `SIGHUP` | 1 | Hangup; some services interpret it as a reload request |
| `SIGTERM` | 15 | Requests graceful termination |
| `SIGKILL` | 9 | Forces termination; cannot be caught or ignored |
| `SIGSTOP` | 19 | Stops/suspends a process |
| `SIGCONT` | 18 | Resumes a stopped process |

Signal numbers can be platform-dependent; the numbers above are those used in this Ubuntu lab.

### 5.1 Harmless test process

A test process was started with:

```bash
sleep 300 &
```

The `&` runs the command in the background. The shell reported PID `9171` in the first exercise. The process was inspected and then sent `SIGTERM`:

```bash
ps -p 9171 -f
kill -15 9171
```

The shell reported that the test process had terminated.

### 5.2 Stop and resume exercise

A second harmless process was started:

```bash
sleep 300 &
```

The shell reported PID `9185`.

It was inspected, stopped, and checked:

```bash
kill -STOP 9185
ps -p 9185 -o pid,stat,cmd
```

Actual observed stopped state:

```text
PID STAT CMD
9185 T    sleep 300
```

`T` indicates a stopped process.

It was resumed:

```bash
kill -CONT 9185
ps -p 9185 -o pid,stat,cmd
```

Actual observed resumed state:

```text
PID STAT CMD
9185 S    sleep 300
```

`S` indicates interruptible sleep, a normal waiting state for this test process.

When cleanup was later attempted, the process had already completed its 300-second duration:

```text
-bash: kill: (9185) - No such process
[1]+  Done                    sleep 300
```

A subsequent `ps -p 9185 -f` showed only the header, confirming that the process no longer existed.

**Lesson:** A PID can disappear between checking it and sending a signal. Always verify the current process state. Do not use `SIGKILL` as the default response; investigate first and prefer graceful termination when appropriate.

---

## 6. Services, daemons, and systemd

A **process** is a running instance of a program. A **service** is a managed system function, commonly provided by one or more processes. A **daemon** is a background process that provides a service.

On this Ubuntu system, `systemd` is PID 1 and manages system services.

Actual observed output from:

```bash
ps -p 1 -f
```

```text
UID          PID    PPID  C STIME TTY          TIME CMD
root           1       0  0 Nov04 ?        00:00:17 /usr/lib/systemd/systemd --system --deserialize=69
```

This confirmed that PID 1 was `systemd`, running as root and with PPID 0.

### 6.1 Inspecting SSH service status

Command:

```bash
systemctl status ssh
```

Actual observed status included:

```text
● ssh.service - OpenBSD Secure Shell server
     Loaded: loaded (/usr/lib/systemd/system/ssh.service; enabled; preset: enabled)
     Active: active (running) since Thu 2026-11-05 08:50:05 UTC; 4h 13min ago
TriggeredBy: ● ssh.socket
       Docs: man:sshd(8)
             man:sshd_config(5)
    Process: 8726 ExecStartPre=/usr/sbin/sshd -t (code=exited, status=0/SUCCESS)
   Main PID: 8753 (sshd)
      Tasks: 1 (limit: 2263)
     Memory: 1.2M (peak: 1.4M)
     CPU: 100ms
     CGroup: /system.slice/ssh.service
             └─8753 "sshd: /usr/sbin/sshd -D [listener] 0 of 10-100 startups"
```

Important fields:

- `Loaded`: unit file is loaded; `enabled` means configured to start automatically according to its enablement configuration.
- `Active: active (running)`: the service was running at the time of inspection.
- `ExecStartPre`: a command run before service startup. Here `/usr/sbin/sshd -t` checks SSH configuration syntax.
- `status=0/SUCCESS`: the pre-start check completed successfully.
- `Main PID`: the main process associated with the service.
- `CGroup`: the service's control group and associated process information.
- `TriggeredBy`: identifies the socket unit associated with this service.

Short status checks:

```bash
systemctl is-active ssh
systemctl is-enabled ssh
systemctl is-active cron
```

Actual results observed:

```text
active
enabled
active
```

Remember the distinction:

- **Active** describes whether the service is currently running.
- **Enabled** describes whether it is configured to start automatically through the relevant boot-time enablement mechanism.

A service can be active but disabled, or inactive but enabled.

### 6.2 Inspecting SSH socket activation

Command:

```bash
systemctl status ssh.socket
```

Actual observed output:

```text
● ssh.socket - OpenBSD Secure Shell server socket
     Loaded: loaded (/usr/lib/systemd/system/ssh.socket; enabled; preset: enabled)
     Active: active (running) since Fri 2026-10-23 13:15:22 UTC; 1 week 6 days ago
   Triggers: ● ssh.service
     Listen: 0.0.0.0:22 (Stream)
             [::]:22 (Stream)
      Tasks: 0 (limit: 2263)
     Memory: 8.0K (peak: 260.0K)
        CPU: 3ms
     CGroup: /system.slice/ssh.socket
```

Interpretation:

- The socket unit was active.
- It was configured to listen on TCP port 22 on IPv4 (`0.0.0.0`) and IPv6 (`[::]`).
- `Triggers` identified `ssh.service` as the associated service.
- The output showed the socket/service relationship; it should not be interpreted as proof that every incoming connection starts a brand-new service process.

The dependency inspection command was:

```bash
systemctl list-dependencies ssh.service
```

It showed dependencies and relationships including `ssh.socket`, `system.slice`, and `sysinit.target`. A dependency tree is not a simple list of every condition required for successful operation; interpret the unit relationships carefully.

---

## 7. Logs and event investigation with journalctl

The systemd journal stores system and service messages.

### 7.1 SSH logs

Command:

```bash
sudo journalctl -u ssh --no-pager -n 10
```

Selected actual lines observed:

```text
Nov 05 08:47:32 lab-base sshd[7572]: Connection reset by invalid user admin 10.0.2.2 port 56139 [preauth]
Nov 05 08:47:37 lab-base sshd[7599]: Invalid user admin from 10.0.2.2 port 56140
Nov 05 08:47:37 lab-base sshd[7599]: Connection reset by invalid user admin 10.0.2.2 port 56140 [preauth]
Nov 05 08:48:08 lab-base sshd[7731]: Invalid user admin from 10.0.2.2 port 56143
Nov 05 08:48:08 lab-base sshd[7731]: Connection reset by invalid user admin 10.0.2.2 port 56143 [preauth]
Nov 05 08:49:36 lab-base sshd[8340]: Accepted publickey for atlas_admin from 10.0.2.2 port 56153 ssh2: ED25519 SHA256:VZXkkZwcvxYIbP0VkRvZpU/j/22pfLBRICytAwj2cQE
Nov 05 08:49:36 lab-base sshd[8340]: pam_unix(sshd:session): session opened for user atlas_admin(uid=1000) by atlas_admin(uid=0)
Nov 05 08:50:05 lab-base sshd[8753]: Server listening on 0.0.0.0 port 22.
Nov 05 08:50:05 lab-base sshd[8753]: Server listening on :: port 22.
Nov 05 08:50:05 lab-base systemd[1]: Started ssh.service - OpenBSD Secure Shell server.
```

Security interpretation:

- `Invalid user admin` means an authentication attempt used a username that does not exist on the system.
- `[preauth]` indicates the connection was reset before authentication completed.
- `Accepted publickey for atlas_admin` records successful public-key authentication for that account.
- The PAM message records opening the authenticated session.
- The server reported listening on IPv4 and IPv6 port 22.

The source address `10.0.2.2` was observed in the log. In this VirtualBox NAT lab, it is consistent with the NAT gateway address. The log alone does not identify a real-world person or prove the original source beyond what the environment records.

An invalid username attempt is suspicious activity worth reviewing, but it is not proof of successful compromise. Correlate timestamps, source addresses, authentication outcomes, and other logs.

### 7.2 Cron logs

Command:

```bash
sudo journalctl -u cron --no-pager -n 10
```

Actual observed entries included:

```text
Nov 05 13:05:01 lab-base CRON[9274]: pam_unix(cron:session): session opened for user root(uid=0) by root(uid=0)
Nov 05 13:05:01 lab-base CRON[9275]: (root) CMD (command -v debian-sa1 > /dev/null && debian-sa1 1 1)
Nov 05 13:05:01 lab-base CRON[9274]: pam_unix(cron:session): session closed for user root
Nov 05 13:17:01 lab-base CRON[9299]: pam_unix(cron:session): session opened for user root(uid=0) by root(uid=0)
Nov 05 13:17:01 lab-base CRON[9300]: (root) CMD (cd / && run-parts --report /etc/cron.hourly)
Nov 05 13:17:01 lab-base CRON[9299]: pam_unix(cron:session): session closed for user root
```

These entries showed scheduled jobs running as root. One command invoked `debian-sa1`, associated with system activity data collection. Another ran scripts in `/etc/cron.hourly` using `run-parts`.

Root-run scheduled jobs have high privileges. Analysts should investigate unexpected scheduled tasks, unfamiliar scripts, unusual execution times, and changes to cron configuration. Do not assume every root cron job is malicious.

---

## 8. Resource monitoring

### 8.1 Memory and swap

Command:

```bash
free -h
```

Actual observed output:

```text
               total        used        free      shared  buff/cache   available
Mem:           1.9Gi       347Mi       1.0Gi       1.1Mi       751Mi       1.6Gi
Swap:          2.0Gi          0B       2.0Gi
```

Interpretation:

- Total RAM: 1.9 GiB.
- Used memory: 347 MiB.
- Completely free memory: 1.0 GiB.
- Buffers/cache: 751 MiB.
- Estimated available memory: 1.6 GiB.
- Swap used: 0 B.

For assessing memory pressure, `available` is generally more informative than `free`, because Linux uses otherwise-unused RAM for caches that can be reclaimed.

### 8.2 Uptime and load average

Command:

```bash
uptime
```

Actual observed output:

```text
13:53:32 up 21:47,  2 users,  load average: 0.00, 0.00, 0.00
```

Interpretation:

- The system had been running for 21 hours and 47 minutes.
- Two user sessions were reported.
- Load averages over 1, 5, and 15 minutes were all 0.00.

The VM has two virtual CPUs. Load average is not a CPU utilisation percentage. It represents the average number of tasks running or waiting for CPU time, plus tasks in certain uninterruptible states. Compare load average with the number of CPUs and inspect CPU utilisation separately.

### 8.3 Filesystem capacity

Command:

```bash
df -h /
```

Actual observed output:

```text
Filesystem      Size  Used Avail Use% Mounted on
/dev/sda2        25G  4.8G   19G  21% /
```

Interpretation:

- Root filesystem: `/dev/sda2`.
- Total size: 25 GB.
- Used: 4.8 GB.
- Available: 19 GB.
- Usage: 21%.

High filesystem usage can prevent applications and services from writing files or logs. Investigate what is consuming space before deleting anything. Preserve logs that may be needed for security investigations.

---

## 9. Root-owned process review

Command used:

```bash
ps -U root -u root -o pid,ppid,%cpu,%mem,stat,comm --sort=-%cpu
```

This selected processes running under root's real and effective user IDs, displayed process and resource fields, and sorted by CPU usage.

Selected actual observations included:

```text
    PID    PPID %CPU %MEM STAT COMMAND
   9240       2  0.4  0.0 I    kworker/0:1-events
   9175       2  0.1  0.0 I    kworker/1:1-events
      1       0  0.0  0.6 Ss   systemd
   8719       1  0.0  1.3 SLsl multipathd
   8714       1  0.0  2.2 Ssl  fwupd
   8744       1  0.0  0.4 Ssl  upowerd
   8753       1  0.0  0.4 Ss   sshd
   8699       1  0.0  0.1 Ss   cron
```

The output also contained many kernel threads such as `kworker`, `ksoftirqd`, and `migration`.

A root-owned process is not inherently suspicious. Kernel threads commonly have PPID 2, while system services commonly have systemd (PID 1) as their parent. Process names are clues, not proof of legitimacy.

### 9.1 Inspecting the SSH listener

Command:

```bash
ps -p 8753 -f
```

Actual observed output:

```text
UID          PID    PPID  C STIME TTY          TIME CMD
root        8753       1  0 10:45 ?        00:00:00 sshd: /usr/sbin/sshd -D [listener] 0 of 10-100 startups
```

This confirmed at the time of inspection:

- The SSH listener ran as root.
- Its PID was 8753.
- Its parent PID was 1 (`systemd`).
- It had no controlling terminal.
- The command identified it as the SSH listener.

The `-D` option keeps the SSH daemon in the foreground rather than having it detach. The listener is a critical system process; terminating it without a plan could disrupt remote administration.

---

## 10. Security investigation workflow

When investigating an unfamiliar or suspicious process:

1. **Identify it:** Record its PID and command.
2. **Identify ownership:** Determine which user runs it.
3. **Inspect ancestry:** Check PPID and parent process.
4. **Inspect resource use:** Look for unusual CPU or memory consumption over time.
5. **Inspect executable and arguments:** A process name alone can be misleading.
6. **Map it to a service:** Check whether systemd manages it and inspect the unit.
7. **Review logs:** Correlate relevant service and system events by timestamp.
8. **Assess impact:** Determine whether it is affecting availability, confidentiality, or integrity.
9. **Preserve evidence:** Avoid deleting files or logs before understanding the situation.
10. **Act carefully:** Prefer documented, reversible actions. Do not terminate critical services or use `SIGKILL` without understanding the consequences.

Useful commands from this phase:

```bash
ps -ef
ps -p PID -f
ps -o pid,ppid,cmd
pgrep -a PROCESS_NAME
top
free -h
uptime
df -h /
systemctl status SERVICE
systemctl is-active SERVICE
systemctl is-enabled SERVICE
systemctl list-dependencies SERVICE
journalctl -u SERVICE --no-pager -n 20
```

Replace placeholders such as `PID`, `PROCESS_NAME`, and `SERVICE` with the actual values being investigated. Commands that inspect logs may require `sudo` depending on permissions.

---

## 11. Final knowledge assessment

The learner completed a five-question security assessment and answered all five correctly.

| Topic | Correct understanding |
|---|---|
| Invalid SSH username | Indicates an attempted authentication using an invalid username; does not by itself prove compromise |
| Accepted public key | Records successful public-key authentication for the named account |
| High CPU process | Investigate PID, owner, parent, command, and behaviour before taking action |
| `journalctl` | Provides system and service logs useful for reconstructing events |
| Active but disabled service | Running now, but not configured for automatic startup through the relevant enablement configuration |

**Assessment result: 5/5.**

---

## 12. Cleanup and safety notes

- The harmless `sleep 300` test processes completed and were verified absent.
- No SSH or other system service was terminated.
- No service configuration was modified during this phase.
- No package installation was required.
- The existing SSH key-based administration setup was preserved.

If repeating the signal exercise, use only a newly started harmless test process. Capture its PID from the shell output, inspect it, and verify its state after each signal. Do not reuse old PIDs.

---

## 13. Phase 1 completion summary

**Phase 1 — Linux Processes and Services: Completed.**

The learner can inspect Linux processes, understand process relationships, interpret service status, review logs, monitor basic system resources, and apply these observations to an introductory security investigation.

The lab observations documented here are snapshots from the dates and times shown in the terminal output. PIDs, resource values, and service state will naturally change over time.

**Next curriculum step:** Proceed to the next planned Linux/security topic without repeating the completed process and service lessons.

---

## 14. GitHub publication checklist

From the Windows PowerShell terminal, open the existing repository:

```powershell
cd C:\Users\Contessina\cybersecurity-home-lab
```

Check the working tree:

```powershell
git status
```

Save this document as:

```text
docs/phase1/linux-processes-and-services.md
```

Then stage, commit, and push:

```powershell
git add docs/phase1/linux-processes-and-services.md
git commit -m "Document Linux processes and services lab"
git push origin main
```

Run each command separately and inspect its output before continuing. Do not change the global Git identity; the repository already has its own project-specific identity configured.
