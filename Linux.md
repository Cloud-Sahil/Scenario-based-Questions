# Linux Scenario-Based Interview Questions 

---

### 1. A server's disk is 100% full. How do you troubleshoot and fix it?
**Answer:**
1. `df -h` — check which partition/mount is full.
2. `du -sh /* 2>/dev/null | sort -rh | head -10` — find the biggest directories.
3. Check `/var/log` — log files often grow huge (`journalctl` logs, application logs).
4. Check for large core dumps or old kernel images in `/boot`.
5. Delete/rotate old logs (`logrotate`), clear package cache (`yum clean all` / `apt clean`).
6. If a process is holding a deleted file open (disk shows full but `du` doesn't match), check with `lsof | grep deleted` and restart that process.
7. Long-term fix: set up **log rotation**, disk usage **monitoring/alerts** (CloudWatch/Nagios), and increase volume size if genuinely needed.

---

### 2. Application is running but not accessible from outside the server. How do you debug?
**Answer:**
1. Check if the process is actually listening: `netstat -tulnp` or `ss -tulnp` — confirm port and bind address (`0.0.0.0` vs `127.0.0.1` — if bound to localhost only, external access won't work).
2. Check local firewall: `iptables -L` / `firewalld` / `ufw status`.
3. Check cloud-level Security Group/NACL (if on AWS).
4. Test locally first: `curl localhost:PORT` on the server itself.
5. Check DNS resolution from client side.
6. Check if a reverse proxy (nginx/Apache) is misconfigured in front of the app.

---

### 3. How do you find which process is using the most CPU/Memory?
**Answer:**
- `top` or `htop` — real-time view, sort by CPU (`Shift+P`) or Memory (`Shift+M`).
- `ps aux --sort=-%cpu | head` / `ps aux --sort=-%mem | head`.
- For a specific culprit process: `pidstat`, `strace -p <pid>` to see what it's doing.
- If it's a memory leak, `pmap -x <pid>` shows memory map details.

---

### 4. A user cannot SSH into the server. How do you troubleshoot?
**Answer:**
1. Check network connectivity: `ping`, `telnet <ip> 22`, or `nc -zv <ip> 22`.
2. Check SSH service status: `systemctl status sshd`.
3. Check firewall/Security Group allows port 22 from the client IP.
4. Check `/etc/ssh/sshd_config` — is `PasswordAuthentication`/`PermitRootLogin` set as expected?
5. Check user's key permissions: `.ssh` folder should be `700`, `authorized_keys` should be `600`.
6. Check `/var/log/secure` or `/var/log/auth.log` for the actual rejection reason.
7. Check if account is locked (`passwd -S username`) or disk is full (SSH daemon can fail to write session logs if disk full).

---

### 5. What is the difference between hard link and soft link (symlink)?
**Answer:**
- **Hard link** – points directly to the same inode as the original file. Deleting original doesn't affect hard link. Cannot link across filesystems/partitions, cannot link directories.
- **Soft link (symlink)** – a pointer/shortcut to the file path. Breaks if original is deleted/moved (dangling link). Can link across filesystems and to directories.
Command: `ln file1 file2` (hard), `ln -s file1 file2` (soft).

---

### 6. How do you check and fix file permission issues?
**Answer:**
- `ls -l` to see permissions (rwx for owner/group/others).
- `chmod` to change permissions (`chmod 755 file`), `chown` to change owner (`chown user:group file`).
- For recursive: `chmod -R`, `chown -R`.
- Common scenario: "Web server can't read app files" → check if files are owned by `root` but Apache/Nginx runs as `www-data`/`nginx` user — fix ownership or add group access.
- Check **SELinux context** too (`ls -Z`), sometimes permissions look fine but SELinux blocks access — use `getenforce`, `audit2allow`, or `semanage`/`restorecon`.

---

### 7. Server load average is very high. How do you investigate?
**Answer:**
- `uptime` / `top` — check load average (1, 5, 15 min).
- Load average > number of CPU cores = system is overloaded.
- Check if it's **CPU-bound** (high %us in `top`) or **I/O-bound** (high %wa – waiting on disk).
- `iostat -x 1` — check disk I/O wait.
- `vmstat 1` — check run queue, blocked processes.
- Identify the top process with `top`/`htop` and decide: kill, optimize, or scale.

---

### 8. How do you schedule a job to run daily at 2 AM? What if it needs to run only on weekdays?
**Answer:**
- Use **cron**: `crontab -e`
  - Daily 2 AM: `0 2 * * *`
  - Weekdays only (Mon-Fri) 2 AM: `0 2 * * 1-5`
- Always redirect output to a log for debugging: `0 2 * * 1-5 /path/script.sh >> /var/log/script.log 2>&1`
- For more complex scheduling/monitoring at scale, use **systemd timers** or a job scheduler (Jenkins/Airflow/cron + monitoring).

---

### 9. What's the difference between `kill`, `kill -9`, and `kill -15`?
**Answer:**
- `kill` (default) = `kill -15` (SIGTERM) – graceful shutdown, lets process clean up (close files, save state) before exiting.
- `kill -9` (SIGKILL) – force kill immediately, no cleanup, used only when process is unresponsive (zombie/hung).
- Best practice: always try `-15` first, use `-9` only as last resort since it can cause data corruption for things like databases.

---

### 10. A process becomes a "zombie". What does that mean and how do you handle it?
**Answer:**
- Zombie = process has finished execution but its exit status hasn't been read by the parent process yet — it stays in process table as `<defunct>`.
- Usually harmless in small numbers (cleared once parent reads exit status), but many zombies = parent process bug (not calling `wait()`).
- You **can't kill a zombie directly** (it's already dead) — you fix/restart the **parent process**; once parent dies, `init`/`systemd` (PID 1) adopts and reaps the zombie.

---

### 11. How do you check what changed on a server that broke the application?
**Answer:**
- Check `/var/log/messages`, `/var/log/syslog`, application-specific logs, `journalctl -xe` for recent errors.
- Check package changes: `rpm -qa --last | head` (RHEL) or `/var/log/apt/history.log` (Debian).
- Check config file changes with version control (if configs are in Git) or `stat filename` to see last modified time.
- Check recent cron jobs, deployments (CI/CD pipeline history), or user login history (`last`, `lastlog`).
- Use monitoring/alerting history (CloudWatch/Grafana) to correlate the exact time it broke with a metric change.

---

### 12. How do you increase the size of a mounted disk on Linux without downtime (e.g., on AWS EC2)?
**Answer:**
1. Increase EBS volume size from AWS console/CLI (`aws ec2 modify-volume`).
2. On the instance, check the disk sees new size: `lsblk` (may need `sudo growpart /dev/xvda 1` if using partitions).
3. Resize the partition: `growpart /dev/xvda 1`.
4. Resize the filesystem:
   - ext4: `resize2fs /dev/xvda1`
   - xfs: `xfs_growfs /`
5. Verify with `df -h`. This can be done live without unmounting/rebooting on most modern setups.

---

### 13. What is the difference between `/etc/passwd` and `/etc/shadow`?
**Answer:**
- `/etc/passwd` – user account info (username, UID, GID, home dir, shell) — readable by all users.
- `/etc/shadow` – actual **encrypted passwords** and password aging info — readable only by root, for security.

---

### 14. How do you troubleshoot high memory usage / OOM killer killing your process?
**Answer:**
- Check `dmesg | grep -i "out of memory"` or `/var/log/messages` for OOM killer logs — it tells which process was killed and why.
- `free -h` to check current memory/swap usage.
- Identify memory-hungry process with `top`/`ps aux --sort=-%mem`.
- Fixes: optimize app memory usage, add swap space (temporary fix), increase instance memory, or set proper resource limits (cgroups/systemd `MemoryMax`, or container memory limits in Docker/K8s).

---

### 15. What's the difference between `yum`/`dnf` and `rpm`? Between `apt` and `dpkg`?
**Answer:**
- `rpm`/`dpkg` – low-level package managers, install a single package file, **don't resolve dependencies automatically**.
- `yum`/`dnf`/`apt` – high-level package managers, connect to repositories, **auto-resolve dependencies**, handle updates easily.

---

### 16. How do you check open ports and which process owns them?
**Answer:**
- `ss -tulnp` (modern, preferred) or `netstat -tulnp` (legacy).
- `lsof -i :PORT` to see which process uses a specific port.
- `nmap localhost` for external-view style port scan.

---

### 17. Your application logs are growing huge and filling disk. How do you manage this properly (production scenario)?
**Answer:**
- Configure **logrotate** (`/etc/logrotate.d/`) — rotate daily/weekly, compress old logs, keep limited history (e.g., 7-14 days), auto-delete old ones.
- For centralized management, ship logs to **CloudWatch Logs / ELK stack / Splunk** and delete local copies after shipping.
- Set disk usage **alerts** (e.g., alert at 80% usage) proactively instead of reacting after it's full.

---

### 18. What is the boot process of a Linux system (high-level)?
**Answer:**
BIOS/UEFI → **Bootloader** (GRUB) → **Kernel** loads → **initramfs** (initial RAM disk) → **systemd (PID 1)** starts → systemd starts targets/services in order (based on dependencies) → login prompt/services ready.
Troubleshooting boot issues: use rescue mode/single-user mode, check `journalctl -b` for boot logs.

---

### 19. How do you set up passwordless SSH between two servers?
**Answer:**
1. On source server: `ssh-keygen -t rsa -b 4096` (generates public/private key pair).
2. Copy public key to target: `ssh-copy-id user@target-server` (or manually append to `~/.ssh/authorized_keys` on target).
3. Ensure correct permissions: `~/.ssh` = 700, `authorized_keys` = 600.
4. Test: `ssh user@target-server` — should log in without password prompt.

---

### 20. A cron job that worked when run manually doesn't work when scheduled via cron. Why?
**Answer:** Very common real scenario:
- **Cron has a minimal environment (`PATH`)** — script may use commands/binaries not found by cron's limited PATH. Fix: use absolute paths in script, or set `PATH` explicitly in crontab.
- **Environment variables** set in `.bashrc`/`.bash_profile` aren't loaded by cron — source them explicitly in the script if needed.
- Script assumes a certain **working directory** — cron runs from home dir by default, so use `cd /path && ./script.sh` or absolute paths.
- Always redirect cron output to a log file to catch silent errors: `* * * * * /path/script.sh >> /var/log/cron.log 2>&1`.

---

### 21. How do you extend an LVM logical volume without downtime?
**Answer:**
1. Add/extend the underlying disk (e.g., resize EBS volume).
2. `pvresize /dev/xvdf` (extend physical volume to use new space).
3. `lvextend -L +10G /dev/vgname/lvname` (extend logical volume).
4. Resize the filesystem: `resize2fs /dev/vgname/lvname` (ext4) or `xfs_growfs /mountpoint` (xfs).
5. Verify: `df -h`, `lvs`, `vgs`. All done live, no unmount/reboot needed.

---

### 22. What are the common RAID levels and when do you use each?
**Answer:**
- **RAID 0** – striping, high performance, **no redundancy** (one disk fails = all data lost). Use for temp/cache data.
- **RAID 1** – mirroring, full redundancy, 50% storage efficiency. Use for OS/critical small data.
- **RAID 5** – striping + distributed parity, tolerates 1 disk failure, good balance of performance/redundancy/cost. Use for general file servers.
- **RAID 10 (1+0)** – mirrored + striped, high performance + redundancy, needs minimum 4 disks. Use for databases needing both speed and safety.

---

### 23. What is the difference between `iptables` and `firewalld`?
**Answer:**
- **iptables** – older, rule-based directly manipulating netfilter tables, rules applied immediately but not persistent by default (need `iptables-save`).
- **firewalld** – newer, dynamic firewall manager with **zones** (concepts like "public," "trusted") and **services**, supports runtime changes without dropping existing connections, config is persistent by default.
- Most modern RHEL/CentOS 7+ systems use firewalld; Ubuntu commonly uses `ufw` (a simpler wrapper) or raw iptables/nftables.

---

### 24. How do you create and troubleshoot a custom systemd service?
**Answer:**
1. Create unit file: `/etc/systemd/system/myapp.service` with `[Unit]`, `[Service]` (ExecStart, Restart, User), and `[Install]` sections.
2. `systemctl daemon-reload` (after any unit file change).
3. `systemctl enable --now myapp` (enable at boot + start now).
**Troubleshooting:**
- `systemctl status myapp` – quick status + last few log lines.
- `journalctl -u myapp -f` – live logs.
- Check `ExecStart` path is correct and executable, check `User`/`WorkingDirectory` permissions, and verify environment variables are set (`Environment=` or `EnvironmentFile=` in unit file) since systemd doesn't inherit your shell's environment.

---

### 25. Application works fine, but SELinux is blocking it (e.g., "Permission denied" despite correct file permissions). How do you fix it?
**Answer:**
1. Check `getenforce` — confirm SELinux is in `Enforcing` mode.
2. Check denials: `ausearch -m avc -ts recent` or `/var/log/audit/audit.log`.
3. Generate a suggested policy: `audit2allow -a -M mypolicy` then `semodule -i mypolicy.pp` (creates a custom allow rule) — better than disabling SELinux entirely.
4. For file context issues (e.g., moved a file to wrong SELinux context), use `restorecon -Rv /path` to reset to default context, or `semanage fcontext` to set a custom context permanently.
5. Avoid `setenforce 0` (disabling) in production — it's a security risk; use it only temporarily to confirm SELinux is indeed the cause.

---

### 26. An NFS mount is hanging/not accessible. How do you troubleshoot?
**Answer:**
1. Check network connectivity to NFS server: `ping`, `showmount -e <server>` (lists exported shares).
2. Check NFS service on server: `systemctl status nfs-server`, and firewall allows NFS ports (2049 + rpcbind).
3. On client, check mount status: `mount | grep nfs`, `df -h`.
4. If hanging, check for **stale file handle** errors — may need `umount -f` (force) or `umount -l` (lazy unmount) then remount.
5. Check `/etc/fstab` entry and mount options (e.g., `soft` vs `hard` mount — `hard` retries indefinitely and can hang a process if server is down; `soft` times out, but risks data corruption on write).

---

### 27. DNS resolution is failing on a server. How do you debug it?
**Answer:**
1. Check `/etc/resolv.conf` — correct nameservers configured?
2. Test resolution directly: `dig google.com` or `nslookup google.com` — bypasses local cache/hosts file issues.
3. Check `/etc/hosts` — a wrong static entry can override DNS for a specific hostname.
4. Check `/etc/nsswitch.conf` — confirms lookup order (`hosts: files dns`).
5. Check if it's a **specific domain** issue (upstream DNS problem) or **all domains** (local resolver/network issue).
6. On cloud (AWS), check VPC DNS settings (`enableDnsSupport`, `enableDnsHostnames`) and Security Group allows outbound port 53.

---

### 28. How do you give a user limited sudo access (only specific commands, not full root)?
**Answer:**
- Edit via `visudo` (safely edits `/etc/sudoers`, checks syntax before saving).
- Example: `username ALL=(ALL) NOPASSWD: /usr/bin/systemctl restart myapp, /usr/bin/tail /var/log/myapp.log` — grants only these two commands without password prompt, nothing else.
- Better practice for teams: create a file in `/etc/sudoers.d/` per user/group rather than editing the main sudoers file directly — easier to manage and version control.

---

### 29. Disk I/O seems to be the bottleneck on a server. How do you confirm and analyze it?
**Answer:**
- `iostat -x 1` — check `%util` (close to 100% = disk saturated), `await` (average wait time), `r/s`/`w/s` (read/write ops per sec).
- `iotop` — see which process is generating the most I/O in real time.
- `vmstat 1` — check `wa` column (% CPU time waiting on I/O).
- Fix options: move to faster storage (SSD/gp3 with higher IOPS), spread I/O across multiple disks (RAID/LVM striping), add caching layer, or optimize the application (e.g., reduce unnecessary disk writes, batch writes).

---

### 30. How do you check and change the maximum number of open files/processes a user or process can have?
**Answer:**
- Check current limits: `ulimit -n` (open files), `ulimit -u` (processes) for current shell.
- Check a running process's actual limits: `cat /proc/<pid>/limits`.
- Permanent change: edit `/etc/security/limits.conf` (e.g., `appuser soft nofile 65536` / `appuser hard nofile 65536`), or for systemd services, set `LimitNOFILE=65536` in the unit file (limits.conf doesn't apply to systemd-managed services by default).
- Common scenario: "Too many open files" error in app logs → this is almost always a `ulimit -n` issue.

---

### 31. You're installing a package and get a dependency conflict error. How do you resolve it?
**Answer:**
- Check exact error — usually names the conflicting package/version.
- `yum`/`dnf`: try `yum deplist <package>` to see dependencies, or `dnf install --allowerasing` (careful, can remove conflicting packages).
- `apt`: `apt-get install -f` (fix broken dependencies), check `apt list --installed | grep <package>` for version conflicts.
- If a specific version is pinned/held (`apt-mark hold` / `yum versionlock`), check and release the hold if it's blocking a needed update.
- As last resort, use a **container** or a separate environment (e.g., a different package repo/version) to isolate the conflicting requirement instead of forcing a system-wide change.

---

### 32. What's the difference between `/etc/fstab` entries and manual `mount` command? Why would a manually mounted disk disappear after reboot?
**Answer:** `mount` command mounts a filesystem **temporarily** for the current session only — it does NOT persist across reboot. `/etc/fstab` defines mounts that are **automatically mounted at boot**. If a disk "disappears" after reboot, it's because it was only manually mounted and never added to `/etc/fstab`. Always add an entry (with correct UUID via `blkid`) to `/etc/fstab` for anything that needs to persist, and test with `mount -a` before rebooting to catch fstab syntax errors (a bad fstab entry can even prevent the server from booting).

---

### 33. How do you find which package a specific file belongs to, or what files a package installed?
**Answer:**
- RHEL/CentOS: `rpm -qf /path/to/file` (which package owns this file), `rpm -ql packagename` (list files in a package).
- Debian/Ubuntu: `dpkg -S /path/to/file`, `dpkg -L packagename`.
- Useful for troubleshooting "where did this config file come from" or "what got installed with this package" scenarios.

---

### 34. Explain the Linux file system hierarchy — what goes in `/etc`, `/var`, `/opt`, `/usr`, `/tmp`?
**Answer:**
- `/etc` – system-wide configuration files.
- `/var` – variable data: logs (`/var/log`), spool, cache — data that changes frequently.
- `/opt` – optional/third-party software packages (self-contained apps).
- `/usr` – user programs/binaries, libraries (mostly read-only, shared system resources).
- `/tmp` – temporary files, often cleared on reboot — never store important data here.
- `/home` – user home directories.

---

### 35. A server suddenly can't resolve hostnames but IP-based connections work fine. What could be wrong, and how do you confirm it's DNS?
**Answer:**
- Confirm it's DNS (not general network) by testing: `ping 8.8.8.8` (works, IP-based) vs `ping google.com` (fails, hostname-based) → confirms DNS issue specifically.
- Check `/etc/resolv.conf` for correct/reachable nameservers.
- Check if `systemd-resolved` is running/misconfigured (common on Ubuntu): `systemctl status systemd-resolved`, `resolvectl status`.
- Check firewall/Security Group isn't blocking outbound UDP/TCP port 53.
- Test with a different DNS server directly: `dig @8.8.8.8 google.com` to isolate whether it's your configured resolver or a broader network issue.

---

### 36. How do you securely copy files between two Linux servers, and how do you resume a large interrupted transfer?
**Answer:**
- `scp` for simple secure copy: `scp file.tar user@remotehost:/path/`.
- `rsync` is preferred for large files/directories — it's more efficient (only transfers differences) and **resumable**: `rsync -avz --partial --progress file.tar user@remotehost:/path/` — `--partial` keeps partially transferred data so a re-run continues instead of restarting from zero.

---

### 37. What's the difference between a soft reboot and a hard reboot? When would a server need a hard reboot / power cycle?
**Answer:**
- **Soft reboot** (`reboot`/`shutdown -r now`) – OS gracefully stops services, unmounts filesystems, then restarts. Safe, preferred method.
- **Hard reboot/power cycle** – abrupt power off/on, no graceful shutdown — risk of filesystem corruption/data loss (especially on non-journaled filesystems, though modern ext4/xfs are journaled and mostly recover).
- Needed when the system is completely unresponsive (kernel panic, hung processes not responding to any command, SSH/console both dead) and a normal reboot command can't even be issued.

---

### 38. You need to find all files modified in the last 24 hours in `/var/log`. How?
**Answer:** `find /var/log -type f -mtime -1` — lists files modified within the last 1 day. Useful in incident investigation ("what changed right before the outage"). Variations: `-mmin -60` for last 60 minutes, `-newer /path/reference_file` to find files newer than a specific reference file/timestamp.

---

### 39. How do you check total and per-process memory usage properly, and why does `free -h` sometimes look confusing (low "free" memory even when system is fine)?
**Answer:** Linux uses **free RAM for disk cache/buffers** to speed up I/O — this shows as "used" in older `free` output but is actually reclaimable if an app needs it. Focus on the **"available"** column in `free -h` (modern versions) — that's the real usable memory estimate, not the raw "free" column. `cat /proc/meminfo` gives more granular breakdown if needed.

---

### 40. How do you set up log monitoring to alert when a specific error pattern appears in a log file, without a full ELK stack?
**Answer:**
- Lightweight option: a cron-based script using `grep`/`awk` on new log lines, sending alerts via `mail`/Slack webhook/SNS if pattern matches.
- Better: use **CloudWatch Logs agent** (on AWS) with a **metric filter** on the pattern, tied to a **CloudWatch Alarm** — no need for full ELK for simple pattern alerting.
- For more advanced needs (correlation, dashboards), then consider ELK/EFK or Grafana Loki.

---

*(Questions 41–50 below — quick-fire scenario round)*

### 41. What does `nice` and `renice` do, and when would you use them?
**Answer:** They control **CPU scheduling priority** of a process. `nice -n 10 command` starts a process with lower priority (higher nice value = lower priority, range -20 to 19). `renice -n 5 -p <pid>` changes priority of an already-running process. Use when a background/batch job (e.g., backup script) shouldn't compete with production application processes for CPU.

---

### 42. How do you check which kernel version is running, and how do you safely update it?
**Answer:** `uname -r` shows current kernel. Update via package manager (`yum update kernel` / `apt upgrade`) which installs the new kernel **alongside** the old one (doesn't remove it immediately) — GRUB lets you boot into the old kernel if the new one has issues. Always test in staging before production, and keep at least one older kernel available as fallback.

---

### 43. A file was accidentally deleted on a Linux server (not backed up). Is there any way to recover it?
**Answer:** If the file's inode/data blocks haven't been overwritten yet, tools like `extundelete` (ext3/4) or `testdisk`/`photorec` might recover it — but chances drop fast the longer the disk is used after deletion. Immediately **stop writing to that disk/filesystem** to maximize recovery chances. This is also the strongest argument for proper backup strategy (snapshots/EBS backups) rather than relying on recovery tools.

---

### 44. What is the difference between `su` and `su -` (or `sudo -i`)?
**Answer:** `su username` switches user but keeps the **current shell's environment** (PATH, working directory) — can cause subtle issues (wrong PATH for the new user). `su - username` (or `sudo -i`) starts a **full login shell** for that user — proper environment, correct PATH, home directory — the recommended way to switch users fully.

---

### 45. How would you check if a specific port is being blocked by network/firewall vs the app just not listening?
**Answer:**
1. First check locally on server: `ss -tulnp | grep PORT` — if nothing listening, it's an app issue, not network.
2. If listening locally but unreachable externally: test from another host with `telnet <ip> <port>` or `nc -zv <ip> <port>`.
3. Check Security Group/NACL (cloud) or `iptables`/`firewalld` rules on the server itself.
4. Use `tcpdump -i eth0 port <port>` on the server to see if packets are even arriving — if no packets arrive, it's blocked upstream (SG/NACL/network); if packets arrive but no response, it's likely a local firewall or app-level rejection.

---

### 46. What's the difference between a process and a thread in Linux, from an OS perspective?
**Answer:** A **process** has its own independent memory space (isolated). A **thread** is a unit of execution *within* a process, sharing the same memory space as other threads in that process — cheaper to create/switch between than processes, but a crash/memory corruption in one thread can affect the whole process. `ps -eLf` shows threads per process; `top -H` shows thread-level CPU usage.

---

### 47. Server time is out of sync causing authentication/cert failures. How do you fix and prevent this?
**Answer:** Check current time/sync status: `timedatectl status` or `chronyc tracking`. Fix by ensuring **NTP/chrony** service is running and syncing against a reliable time source (`chronyc sources`). On cloud (AWS), use the Amazon Time Sync Service (`169.254.169.123`). Prevent by ensuring `chronyd`/`ntpd` is enabled at boot on all servers — time drift is a very common, often-overlooked cause of Kerberos/SSL/JWT/cluster failures.

---

### 48. How do you check historical login attempts (successful and failed) on a server for security audit?
**Answer:**
- `last` – successful logins history.
- `lastb` – failed login attempts (needs root, reads `/var/log/btmp`).
- `/var/log/secure` (RHEL) or `/var/log/auth.log` (Debian/Ubuntu) – detailed SSH auth logs including source IP.
- For repeated brute-force attempts, consider **fail2ban** to auto-block IPs after repeated failed attempts.

---

### 49. What is a "orphaned"/dangling symlink, and how do you find all of them on a server?
**Answer:** A symlink pointing to a target that no longer exists (deleted/moved). Find them with: `find / -xtype l 2>/dev/null` — lists all broken symlinks system-wide. Common cause of mysterious "file not found" errors when scripts/configs reference a symlink whose target was moved/deleted during a deployment or cleanup.

---

### 51. How do you check and fix a corrupted filesystem after an unexpected crash/power loss?
**Answer:** On reboot, most systems auto-run `fsck` if the filesystem was marked "dirty" (unclean unmount). Manual check: unmount the filesystem first (`umount /dev/sdX1` — can't fsck a mounted filesystem safely), then run `fsck /dev/sdX1` (or `fsck -y` to auto-fix issues without prompting). For the root filesystem, you may need to boot into **rescue/single-user mode** or run fsck from a live USB/rescue environment since you can't unmount root while it's in use. Always have backups — fsck can sometimes need to delete unrecoverable corrupted data.

---

### 52. What's the difference between `apt-get update` and `apt-get upgrade`?
**Answer:** `apt-get update` – refreshes the **local package index** (metadata about available versions from repositories), doesn't install/upgrade anything itself. `apt-get upgrade` – actually **installs newer versions** of already-installed packages, based on the index. You almost always run `update` before `upgrade` — if you skip `update`, `upgrade` works off stale metadata and might miss available updates.

---

### 53. How do you find and kill all processes matching a name (e.g., all "python" processes)?
**Answer:** `pkill python` (kills by process name match) or `killall python` (similar, slightly different matching behavior). For more control: `ps aux | grep python` to review first, then `kill $(pgrep python)` to kill only the specific PIDs found — safer than `pkill`/`killall` since you can visually verify before killing, avoiding accidentally killing an unrelated process with a similar name (e.g., `grep`'s own process matching itself, or a legitimate different app named similarly).

---

### 54. Your server's `/tmp` directory is filling up disk space with files that should have been cleaned up automatically. Why, and how do you fix it?
**Answer:** Many distros run `systemd-tmpfiles-clean` (via a timer) to auto-clean `/tmp` based on rules in `/etc/tmpfiles.d/` — if this timer/service is disabled or misconfigured, files accumulate. Check: `systemctl status systemd-tmpfiles-clean.timer`. Alternatively, an application might be writing large temp files without cleaning up after itself (a bug) — identify with `du -sh /tmp/* | sort -rh | head`. Fix: enable/fix the tmpfiles cleaner, add a cron job for aggressive cleanup if needed, or fix the misbehaving application.

---

### 55. How do you check which service/process started automatically at boot and identify what's slowing down your server's boot time?
**Answer:** `systemctl list-unit-files --state=enabled` – lists all services enabled to start at boot. `systemd-analyze` – shows total boot time. `systemd-analyze blame` – lists each service by how long it took to start, sorted slowest first — directly identifies the bottleneck. `systemd-analyze critical-chain` – shows the dependency chain that determines the critical path of boot time, useful when the slow part is due to a dependency wait, not just one slow service itself.

---

### 56. What's the difference between `chmod 777` and setting proper least-privilege permissions — why is `777` considered bad practice?
**Answer:** `chmod 777` gives **read, write, and execute to everyone** (owner, group, and all other users) — meaning any user on the system (including a compromised low-privilege account or a malicious process) can modify or execute that file. It's a common "quick fix" for permission errors but bypasses the actual permission model entirely — proper practice is identifying **exactly which user/group** needs access and granting only that (e.g., `chown appuser:appgroup file && chmod 750 file`), keeping the principle of least privilege intact.

---

### 57. How do you check historical resource usage (CPU/memory/disk) on a server for trend analysis, not just real-time?
**Answer:** `sar` (System Activity Reporter, part of `sysstat` package) – collects and stores historical metrics (`sar -u` for CPU history, `sar -r` for memory, `sar -d` for disk) if the `sysstat` collection service is enabled (`systemctl enable --now sysstat`). For richer visual trend analysis, forward metrics to a monitoring stack (**CloudWatch**, **Prometheus + Grafana**, or **Netdata**) rather than relying only on local `sar` logs, which have limited retention by default.

---

### 58. A background process spawned by a script keeps running even after the script/SSH session ends. Why, and how do you prevent it (or intentionally do this)?
**Answer:** Normally, when an SSH session ends, a **SIGHUP** signal is sent to child processes, terminating them — but a process can survive this if started with `nohup command &` (ignores SIGHUP) or `disown` (removes it from the shell's job table so it doesn't receive the signal), or if run inside a **tmux/screen session** that persists independently of the SSH connection. To intentionally run a long task that survives disconnection: `nohup long_task.sh > output.log 2>&1 &`, or better, use `systemd` service/timer for anything meant to run reliably long-term (more robust than nohup for production use).

---

### 59. How do you check if a specific kernel module is loaded, and how do you load/unload one?
**Answer:** `lsmod` – lists all currently loaded kernel modules. `lsmod | grep <name>` to check a specific one. Load a module: `modprobe <module_name>` (handles dependencies automatically) or `insmod <path.ko>` (lower-level, no dependency resolution). Unload: `modprobe -r <module_name>` or `rmmod <module_name>`. To make a module load automatically at boot, add its name to `/etc/modules-load.d/`.

---

### 60. What is the difference between a "hard mount" and a "soft mount" for NFS, and which should you use for a database workload?
**Answer:** **Hard mount** (default) – if the NFS server becomes unreachable, the client process **retries indefinitely**, effectively hanging until the server comes back (or is manually interrupted) — guarantees no silent data loss/corruption. **Soft mount** – gives up after a timeout and returns an I/O error to the application — faster failure but risks data corruption if a write was interrupted mid-operation. For database workloads (or anything requiring data integrity), **always use hard mounts** — a hung process is recoverable, silent data corruption often isn't.

---

### 61. How do you troubleshoot "Too many open files" errors appearing in application logs, beyond just checking `ulimit`?
**Answer:** Beyond checking/raising `ulimit -n`, actually investigate **why** so many files/sockets are open — check if the app is leaking file descriptors (not closing files/sockets properly): `lsof -p <pid> | wc -l` to count open FDs for the process, and `lsof -p <pid>` to see exactly what's open (are there hundreds of duplicate socket connections to the same host, suggesting connections aren't being closed?). Raising the limit is often just a band-aid — the underlying leak should be fixed in the application code if that's the real cause.

---

### 62. How do you securely wipe/delete sensitive data from a disk before decommissioning a server (beyond just `rm`)?
**Answer:** `rm` only removes the file's directory entry — the actual data often remains recoverable on disk until overwritten. For secure deletion: `shred -vfz -n 3 /path/to/file` (overwrites the file multiple times before deletion). For a full disk/volume: `dd if=/dev/zero of=/dev/sdX bs=1M` (overwrites with zeros) or `shred` on the whole device — though on cloud (AWS), simply relying on the fact that EBS volumes are encrypted (if encryption was enabled) and following AWS's data sanitization practices upon volume deletion is generally sufficient and standard practice.

---

### 63. What's the difference between `/proc` and `/sys` filesystems in Linux?
**Answer:** Both are **virtual/pseudo filesystems** (not real disk data, generated by the kernel in real-time). `/proc` – primarily process information (`/proc/<pid>/...`) and some kernel/system info (`/proc/meminfo`, `/proc/cpuinfo`); older, less structured. `/sys` – newer, more structured interface exposing kernel **device/driver** information and tunable parameters (`/sys/class/`, `/sys/block/`), commonly used for hardware-related tuning (e.g., adjusting a disk's I/O scheduler via `/sys/block/sda/queue/scheduler`).

---

### 64. How do you set up log forwarding from a Linux server to a central syslog server?
**Answer:** Configure `rsyslog` (most common modern syslog daemon) in `/etc/rsyslog.conf` or a file in `/etc/rsyslog.d/`, adding a forwarding rule like `*.* @@centralserver:514` (TCP, double `@@`) or `*.* @centralserver:514` (UDP, single `@`) — TCP is more reliable for critical logs since UDP can silently drop packets. Restart rsyslog (`systemctl restart rsyslog`) to apply. On the receiving central server, rsyslog must be configured to **accept** remote logs (`module(load="imtcp")` and `input(type="imtcp" port="514")`).

---

### 65. A server keeps hitting maximum process limit ("fork: retry: Resource temporarily unavailable"). How do you diagnose and fix it?
**Answer:** Check current process count vs limit: `ps aux | wc -l` vs `ulimit -u` (per-user process limit) and `cat /proc/sys/kernel/pid_max` (system-wide max PIDs). Identify what's spawning excessive processes: `ps aux --sort=-pid | head -30` or check for a **fork bomb** (a script/bug rapidly spawning child processes). Fix: kill the runaway process tree, raise `nproc` limit in `/etc/security/limits.conf` if genuinely needed, and check for the root cause (e.g., a cron job spawning duplicate processes because the previous run hadn't finished, indicating a missing lock file check in the script).

---

### 66. How do you check and configure swap space — when is adding swap the right fix vs a band-aid?
**Answer:** Check current swap: `swapon --show` or `free -h`. Add swap file: `fallocate -l 4G /swapfile && chmod 600 /swapfile && mkswap /swapfile && swapon /swapfile`, then add an entry to `/etc/fstab` to persist across reboot. Swap is a reasonable fix for **occasional memory spikes** or as a safety net against OOM kills on memory-constrained systems, but it's a **band-aid, not a fix**, if the application is consistently using more memory than physical RAM available — that indicates you need more RAM, better memory management in the app, or horizontal scaling, since heavy swap usage causes severe performance degradation (disk is much slower than RAM).

---

### 67. What's the difference between `/etc/hosts` and DNS — in what order does Linux check them, and how do you change that order?
**Answer:** `/etc/hosts` is a **local static file** mapping hostnames to IPs, checked before external DNS lookup by default. **DNS** is the network-based dynamic lookup. The order is controlled by `/etc/nsswitch.conf`'s `hosts:` line (e.g., `hosts: files dns` means check `/etc/hosts` first, then DNS) — you can reorder this line to change precedence, e.g., `hosts: dns files` would check DNS first. Useful for local testing (override a hostname to point to a test server via `/etc/hosts` without needing to change actual DNS).

---

### 68. How do you troubleshoot a server that's randomly rebooting with no clear application-level cause?
**Answer:** Check `last reboot` for reboot history/timing patterns. Check `journalctl -b -1` (logs from the *previous* boot, right before the reboot) for the last messages before it went down — look for kernel panics, OOM killer activity, or hardware errors. Check `dmesg` for hardware-related errors (memory, disk). On cloud instances, check the **AWS console's system status checks** for underlying hardware issues, or check if it's an **Auto Scaling Group health check replacement** (not an actual "reboot" but a full instance replacement) misidentified as a reboot. Also check for scheduled reboots (cron, unattended-upgrades auto-reboot config) that might be unintentionally enabled.

---

### 69. What is the difference between `wget` and `curl`? When would you specifically choose one over the other?
**Answer:** Both fetch data over HTTP/HTTPS/FTP, largely overlapping. `wget` – simpler for **downloading files** (recursive downloads, mirroring a website, resuming interrupted downloads by default with `-c`), works well non-interactively for scripted downloads. `curl` – more powerful for **API interactions** (supports more protocols, easier to send custom headers/methods/POST data, better for scripting API calls and inspecting response headers `-I`, `-v`). Rule of thumb: `wget` for downloading files, `curl` for interacting with APIs/web services.

---

### 70. How do you check the actual, real-time bandwidth usage of a specific process or interface on a Linux server?
**Answer:** `iftop` – shows real-time bandwidth usage per connection on an interface. `nethogs` – shows bandwidth usage **per process** (useful to find which specific application/process is consuming network bandwidth). `vnstat` – tracks historical network usage over time (daily/monthly summaries), useful for trend analysis rather than just real-time snapshots.

---

### 71. What's the difference between a "orphan process" and a "zombie process"?
**Answer:** **Orphan process** – its parent process has terminated, but the orphan itself is still running normally; it gets automatically **re-parented to `init`/`systemd` (PID 1)**, which will properly reap it when it eventually exits. Perfectly normal and harmless. **Zombie process** – the process has already **finished execution** but its exit status hasn't been collected by its parent yet, so it lingers in the process table as `<defunct>` — as covered earlier, indicates a parent process bug if there are many of them.

---

### 72. How do you configure a server to automatically restart a critical service if it crashes, without relying on cron-based health checks?
**Answer:** Use **systemd's built-in restart policy** in the service unit file: `Restart=on-failure` (or `always`) combined with `RestartSec=5` (wait 5 seconds before restarting) — systemd handles this natively and immediately upon detecting the process exit, far more responsive than a cron job polling every minute. Add `StartLimitBurst`/`StartLimitIntervalSec` to prevent infinite rapid restart loops if the service is crash-looping due to a persistent bug (systemd will stop trying after N failures within a time window, alerting you instead of endlessly restarting a broken service).

---

### 73. What is "inode exhaustion," and how is it different from disk space exhaustion? How do you diagnose it?
**Answer:** Every file/directory consumes one **inode** (metadata structure), and a filesystem has a **fixed number of inodes** set at creation time — you can run out of inodes (causing "No space left on device" errors) even if there's plenty of free disk **space**, if there are millions of tiny files consuming all available inodes. Diagnose: `df -i` (shows inode usage vs `df -h` which shows byte/space usage) — if `df -i` shows 100% but `df -h` shows plenty of free space, it's an inode exhaustion issue, usually caused by an application creating excessive small files (e.g., session files, cache files) without cleanup.

---

### 74. How do you find out exactly when a server was last rebooted and its total current uptime?
**Answer:** `uptime` – shows current uptime and load average. `who -b` or `last reboot` – shows the exact boot timestamp. `uptime -s` – shows the system start time in a clean, parseable format. Useful for correlating "was this issue related to a recent reboot/patch" during troubleshooting.

---

### 75. What's the difference between "hard limit" and "soft limit" in `ulimit`, and can a regular user increase their own hard limit?
**Answer:** **Soft limit** – the currently enforced limit, which a user **can raise** up to the hard limit value without root privileges. **Hard limit** – the absolute ceiling; a regular (non-root) user **cannot exceed or raise it** themselves — only root can raise a hard limit (typically configured system-wide via `/etc/security/limits.conf`). This two-tier system lets users flexibly adjust within an admin-defined safe ceiling without needing root access for every adjustment.

---

### 76. How do you audit and remove unused/orphaned packages that are no longer needed as dependencies (bloat cleanup)?
**Answer:** Debian/Ubuntu: `apt autoremove` – removes packages that were installed as dependencies but are no longer required by any currently installed package. RHEL/CentOS: `dnf autoremove` (or `package-cleanup --leaves` with yum-utils to identify leaf packages not required by anything else). Always review the list before confirming removal — occasionally a package flagged as "unused" is actually still needed indirectly (rare, but worth a quick sanity check for critical production servers).

---

### 77. What is the purpose of the `/etc/sysctl.conf` file, and give an example of a common production tuning parameter set there?
**Answer:** `/etc/sysctl.conf` (and files in `/etc/sysctl.d/`) configure **kernel runtime parameters** without needing a reboot (`sysctl -p` applies changes). Common production tuning examples: `net.core.somaxconn` (increase max queued connections for high-traffic servers), `vm.swappiness` (control how aggressively the kernel swaps — often lowered on database servers to prefer keeping data in RAM), `fs.file-max` (system-wide max open file limit, relevant when raising per-process `ulimit` isn't enough because the system-wide cap is also too low).

---

### 78. How do you check if a specific TCP connection is in a problematic state (e.g., stuck in TIME_WAIT or CLOSE_WAIT), and what does each state mean?
**Answer:** `ss -tan` or `netstat -tan` shows connection states. **TIME_WAIT** – normal, brief state after a connection closes (waiting to ensure delayed packets don't interfere with a new connection on the same port) — usually harmless unless there are **excessive** numbers of them exhausting available ports (common on servers doing many short-lived outbound connections; tunable via `net.ipv4.tcp_tw_reuse`). **CLOSE_WAIT** – means the **remote side closed the connection but your application hasn't closed its end yet** — a large, growing number of these usually indicates an **application bug** (not properly closing sockets after use), a genuine resource leak worth investigating in the app code.

---

### 79. How do you set up a Linux server to automatically apply security patches without manual intervention, and what's the risk of doing so blindly?
**Answer:** Debian/Ubuntu: `unattended-upgrades` package, configured in `/etc/apt/apt.conf.d/50unattended-upgrades` (can be scoped to security updates only). RHEL/CentOS: `dnf-automatic` with `apply_updates = yes`. **Risk:** blind auto-patching can occasionally break something (a patch changes behavior your app depends on) without anyone reviewing it first — mitigate by scoping auto-updates to **security patches only** (not full version upgrades), testing in a staging environment first if possible, and having monitoring/alerting in place to quickly catch any post-patch issues rather than discovering them much later.

---

### 80. What's the difference between "vertical" log rotation size-based triggers and time-based triggers in `logrotate` — which should you use for a high-traffic web server?
**Answer:** **Time-based** (`daily`/`weekly`) rotates on a schedule regardless of file size — predictable rotation timing, but a very high-traffic day could produce a huge log file before the next scheduled rotation. **Size-based** (`size 100M`) rotates whenever the log hits a size threshold, regardless of time — prevents any single log file from growing unbounded, better protection against sudden traffic spikes filling disk. For a high-traffic web server, often best to combine both (`daily` **and** `size 100M`, whichever triggers first) for both predictable scheduling and a safety net against unexpected volume spikes.

---

*(Questions 81–100 below — advanced/quick-fire scenario round)*

### 81. How do you check which user/process is responsible for high disk write activity slowing down the whole server?
**Answer:** `iotop -o` (shows only processes actively doing I/O, with per-process read/write throughput) is the fastest way to identify the exact culprit process in real time, more directly useful than `iostat` (which shows per-device, not per-process, stats).

---

### 82. What's the difference between a "block device" and a "character device" in Linux?
**Answer:** **Block device** – data accessed in fixed-size chunks/blocks, supports random access (e.g., hard disks, `/dev/sda`) — suitable for filesystems. **Character device** – data accessed as a continuous stream, byte by byte, sequential access only (e.g., `/dev/tty`, `/dev/null`, keyboards/serial ports) — no buffering/random seeking involved.

---

### 83. How do you troubleshoot a situation where `df -h` and `du -sh /` report very different total disk usage numbers?
**Answer:** Common cause: a large file has been **deleted but is still held open by a running process** — `du` walks the actual directory tree (sees only currently-existing file entries), but `df` reports actual disk block usage (which still counts the deleted-but-open file's blocks until the process closes it). Find it: `lsof +L1` or `lsof | grep deleted` — identify the process holding the deleted file open, and either restart that process (releases the space) or, if it's a log file actively being written, consider using `> /path/to/openfile` (truncate without needing to restart) if appropriate.

---

### 84. What is the difference between `apt install package=version` and just `apt install package` — how do you pin a specific version to prevent auto-upgrade?
**Answer:** `apt install package=1.2.3` installs that exact version if available in the repo. To **prevent future automatic upgrades** of that package (hold it at the current version even during `apt upgrade`): `apt-mark hold package_name`. Useful when a specific application requires an exact dependency version and a newer version would break compatibility — release the hold later with `apt-mark unhold` once ready to upgrade deliberately.

---

### 85. How do you check if your server's clock/NTP synchronization is working correctly and how far off it currently is?
**Answer:** `chronyc tracking` (for chrony) shows current offset from reference time, sync status, and stratum level. `timedatectl status` gives a simpler overview (shows if NTP sync is active). If significantly out of sync (`chronyc tracking` shows a large "System time" offset), check `chronyc sources` to verify it's actually reaching valid time servers, and check firewall isn't blocking NTP (UDP port 123).

---

### 86. What's the difference between `systemctl restart` and `systemctl reload` for a service — when would using the wrong one cause a problem?
**Answer:** `restart` – **fully stops and starts** the service (brief downtime, all connections dropped). `reload` – tells the running service to **re-read its configuration without stopping** (e.g., nginx `reload` gracefully applies new config with zero dropped connections) — only works if the service explicitly supports it (defined via `ExecReload=` in its systemd unit). Using `restart` when `reload` would suffice causes **unnecessary downtime/dropped connections** for a simple config change — always check if reload is supported before defaulting to restart on a production service.

---

### 87. How do you check exactly which command-line arguments and environment variables a currently running process was started with?
**Answer:** `cat /proc/<pid>/cmdline` (shows exact command-line arguments, null-separated — use `tr '\0' ' '` to make it readable) and `cat /proc/<pid>/environ` (shows environment variables the process was started with, also null-separated) — extremely useful for debugging "why is this process behaving differently than expected" without needing to have logged the startup command elsewhere.

---

### 88. What's the difference between a symbolic link to a directory vs a bind mount — when would you use bind mount instead of a symlink?
**Answer:** A **symlink** to a directory is just a pointer — some applications/tools that resolve paths strictly (or check for symlinks explicitly for security reasons, like some chroot/container setups) may not follow it correctly. A **bind mount** (`mount --bind /source/dir /target/dir`) makes the target directory literally **be** the same underlying directory as the source (not a pointer, an actual mount) — transparent to all applications with no special symlink-following behavior needed, often used in Docker/container contexts or when a strict tool doesn't handle symlinks the way you need.

---

### 89. How do you diagnose slow SSH login times (takes 10+ seconds to get a password/key prompt)?
**Answer:** Common cause: **reverse DNS lookup delay** — sshd tries to resolve the connecting client's IP back to a hostname for logging, and if DNS is slow/unreachable, it waits before proceeding. Fix: set `UseDNS no` in `/etc/ssh/sshd_config` and restart sshd. Another cause: **GSSAPI authentication** attempts timing out if enabled but not properly configured — check/disable `GSSAPIAuthentication` in sshd_config if not actually using Kerberos. Test directly with `ssh -v` (verbose mode) to see exactly where the delay occurs in the connection handshake.

---

### 90. What's the difference between `find -exec` and piping `find` output to `xargs` — why might one be safer/more efficient than the other?
**Answer:** `find /path -name "*.log" -exec rm {} \;` – runs the command **once per file found** (can be slower for large numbers of files due to repeated process spawning) — using `-exec ... +` instead of `\;` batches multiple files into fewer command invocations, similar to xargs. `find /path -name "*.log" | xargs rm` – pipes filenames to `xargs`, which batches them efficiently, but is **unsafe with filenames containing spaces/special characters** unless you use `find -print0 | xargs -0` (null-delimited) to handle them correctly — always prefer `-print0`/`xargs -0` (or `-exec +`) for safety and efficiency together.

---

### 91. How do you check what a specific systemd service actually does at startup without reading through complex application documentation?
**Answer:** `systemctl cat servicename` – prints the **actual unit file content** directly (ExecStart command, environment, dependencies) — often faster and more accurate than searching documentation, since it shows exactly what command/arguments systemd will run, what user it runs as, and what it depends on (`After=`, `Requires=`).

---

### 92. What is "swappiness," and why might you set it to a low value (like 10) on a database server but leave it default (60) on a general-purpose server?
**Answer:** `vm.swappiness` (0-100) controls how aggressively the kernel swaps memory pages to disk even when RAM isn't fully exhausted — higher value = kernel swaps more readily/eagerly. On a **database server**, you generally want data to stay in RAM for performance (swapping DB working set to disk causes severe latency spikes), so lowering swappiness (e.g., to 1-10) tells the kernel to avoid swapping unless truly necessary. General-purpose servers with more varied/bursty memory usage patterns are usually fine with the default, letting the kernel manage memory more liberally.

---

### 93. How do you verify that a file wasn't corrupted or tampered with during transfer between two servers?
**Answer:** Generate a checksum before transfer: `sha256sum file.tar > file.sha256`. After transfer, verify: `sha256sum -c file.sha256` on the destination — confirms byte-for-byte integrity. `md5sum` is faster but less collision-resistant (fine for accidental-corruption checks, not for security-sensitive integrity verification where SHA-256 is preferred). `rsync` also has built-in checksum verification (`--checksum` flag) if used for the transfer itself.

---

### 94. What's the difference between "signal 15 (SIGTERM)" not working to stop a process, and needing "signal 9 (SIGKILL)" — why would SIGTERM fail?
**Answer:** SIGTERM can be **caught, handled, or ignored** by the process (it's a "polite request" the app can choose to act on, e.g., to clean up gracefully first) — if the process is genuinely hung (stuck in an uninterruptible kernel wait state, `D` state in `ps`, often waiting on unresponsive I/O like a hung NFS mount) or has buggy signal handling that never actually exits, SIGTERM has no effect. **SIGKILL cannot be caught or ignored** — the kernel forcibly terminates the process immediately, which is why it's the fallback when SIGTERM doesn't work, though it means no cleanup happens.

---

### 95. How do you check for and troubleshoot IP address conflicts on a network (two devices claiming the same IP)?
**Answer:** Symptoms: intermittent connectivity, ARP table inconsistencies. Check: `arping -D -I eth0 <ip>` (sends ARP probes, detects if another device responds claiming that IP — a form of duplicate address detection). Check system logs for kernel messages like "duplicate address detected" (`dmesg | grep -i duplicate`). Fix: identify the conflicting device (often a static IP misconfiguration or a DHCP scope overlap) and correct the IP assignment — on cloud instances (AWS), this is rare since IPs are managed by the VPC, but relevant for on-prem/hybrid environments with manually assigned static IPs.

---

### 96. What's the difference between "hard-coding" DNS servers in `/etc/resolv.conf` directly vs using a network manager (NetworkManager/systemd-resolved) to manage it — why does manually editing `/etc/resolv.conf` sometimes get overwritten unexpectedly?
**Answer:** On systems using `NetworkManager` or `systemd-resolved`, `/etc/resolv.conf` is often **auto-generated/managed** by these services (sometimes a symlink to a dynamically generated file) — manual edits get overwritten whenever the network connection is renewed/re-established (e.g., DHCP lease renewal). To make DNS settings persist, configure them through the proper tool (`nmcli` for NetworkManager, or edit the relevant `.network`/DHCP config files) instead of directly editing `/etc/resolv.conf`, or explicitly mark it as immutable (`chattr +i /etc/resolv.conf`) if a static, unmanaged config is genuinely required (use cautiously, as it can cause confusion for other admins later).

---

### 97. How do you determine if a performance issue is caused by CPU, memory, disk, or network — walk through your general diagnostic approach.
**Answer:** Start broad, narrow down: (1) `top`/`htop` – check overall CPU/memory usage and load average first; (2) if load average is high but CPU usage (`%us`) is low, check `%wa` (I/O wait) — points to disk bottleneck, confirm with `iostat -x 1`; (3) if memory usage is near 100% and swap is being used heavily, that's a memory bottleneck — confirm with `free -h` and `vmstat 1` (watch the `si`/`so` swap columns); (4) if none of the above show saturation but the app is still slow, check network with `iftop`/`ss -s` for connection issues or high latency to a dependency. This systematic elimination approach (rather than guessing) is exactly what interviewers want to see in a troubleshooting answer.

---

### 98. What's the difference between "at" and "cron" for scheduling tasks — when would you use `at` instead of cron?
**Answer:** `cron` – schedules **recurring** tasks (daily, weekly, etc.). `at` – schedules a task to run **once**, at a specific future time (e.g., `echo "backup.sh" | at 2:00am tomorrow`) — useful for one-off, non-repeating scheduled tasks (like "restart this service once during tonight's approved maintenance window") without needing to create and then remember to remove a cron entry afterward.

---

### 99. How do you check if a Linux server has been compromised — what are your first diagnostic steps in a suspected security incident?
**Answer:** (1) Check for unexpected processes: `ps aux`, compare against known-good baseline if available; (2) check for unfamiliar user accounts/SSH keys: `cat /etc/passwd`, check `~/.ssh/authorized_keys` for all users for unrecognized entries; (3) check `last`/`lastb` for unusual login patterns/source IPs; (4) check for unexpected listening ports/network connections: `ss -tulnp`, look for unfamiliar processes with outbound connections; (5) check for modified system binaries (compare checksums against known-good versions if available, or use tools like `rkhunter`/`chkrootkit`); (6) check cron jobs for unauthorized entries (`crontab -l` for all users, plus `/etc/cron.d/`). Critically: **isolate the system from the network first** (if safe to do so) to prevent further damage while investigating, and preserve logs/evidence before making changes, rather than immediately "cleaning up" (which destroys forensic evidence needed to understand the full scope of compromise).

---

### 100. Final scenario: Production server is completely unresponsive over SSH, and the application is down. Walk through your full troubleshooting approach from start to finish.
**Answer:**
1. **Check basic connectivity first:** `ping <server-ip>` — is it a network issue or is the server itself down/hung?
2. **Check cloud console (if AWS)** – instance status checks, CPU/network graphs in CloudWatch — often reveals the issue immediately (e.g., 100% CPU, OOM, or a failed status check indicating a hardware/hypervisor issue).
3. **Try alternate access** – AWS Systems Manager Session Manager (works even if SSH/network is misconfigured, since it doesn't rely on port 22), or EC2 Serial Console for very low-level access if the instance is truly hung.
4. **If truly unresponsive:** check if it's worth a **reboot** (via console, not SSH) to restore service quickly, understanding you may lose the chance to diagnose the exact live cause — for critical production, restoring service often takes priority over root-causing immediately, but capture what you can first (logs, metrics screenshots) before rebooting.
5. **After recovery:** review `journalctl -b -1` (previous boot logs) and CloudWatch metrics/logs from the incident window to find the root cause post-recovery.
6. **Communicate status** to stakeholders throughout, not just at the end.
7. **Post-incident:** write a blameless postmortem, add monitoring/alerting for the specific failure mode if it wasn't caught proactively, and implement Auto Scaling/health-check-based auto-recovery if this is a recurring risk, so future incidents self-heal without needing manual SSH troubleshooting at all.
