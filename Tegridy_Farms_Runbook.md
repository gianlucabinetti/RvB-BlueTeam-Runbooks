# RvB Competition Runbook — Team 2: Tegridy Farms

> **Cal Poly Pomona** · SWIFT × ISACA Red vs Blue · February 28, 2026

**My Machine:** Tegridy Farms · `192.168.1.15` · Kubuntu · **Scored service:** DNS (port 53)

> One of two runbooks I authored as **Team Lead** for this competition. Each defended machine in our environment had a dedicated runbook; remaining machines were covered by teammates.

---

## Table of Contents

1. [The Job in One Sentence](#the-job-in-one-sentence)
2. [Schedule](#schedule)
3. [Phase 1 — First 30 Minutes](#phase-1--first-30-minutes)
4. [Phase 2 — Ongoing Monitoring](#phase-2--ongoing-monitoring-every-2030-minutes)
5. [DNS-Specific Troubleshooting](#dns-specific-troubleshooting)
6. [Emergency Procedures](#emergency-procedures)
7. [Quick Command Reference](#quick-command-reference)
8. [Final Notes](#final-notes)

---

## The Job in One Sentence

**Keep DNS running on port 53.** That's it. Everything else is just protecting that one thing.

>  If DNS goes down you lose **5 points every minute**. Keep it up no matter what.

>  After changing any password - submit a **PCR** on the scoreboard or DNS checks will fail.

>  If anything breaks - ping the team chat; we have your back.

---

## Schedule

| Time | Action |
|---|---|
| 1:30 PM | Briefing - get VPN creds, meet team |
| **2:00 PM** | **COMPETITION STARTS - execute steps below immediately** |
| 4:30 PM | Competition ends |

---

## Phase 1 - First 30 Minutes

>  **Do everything in this exact order. Do not skip steps.**

### Step 1 - Connect to Your Machine

VPN on → open terminal → SSH into your machine:

```bash
ssh <username>@192.168.1.15
```

Username and password will be provided in your team channel at competition start.

### Step 2 - Run Inventory (See What's On The Machine)

```bash
bash inventory.sh | tee /tmp/baseline.txt
```

This shows all open ports, running services, and users. Save it as your baseline and read it carefully - anything unexpected is suspicious. You'll compare this baseline against active ports, services, and users later to flag anything that wasn't there before.

If you need to pull the script down first:

```bash
curl https://pastebin.com/raw/<your-paste-id> -o inventory.sh
```

### Step 3 - Run `firstrun.sh` (Installs Tools and Hardens SSH)

```bash
bash firstrun.sh
```

This automatically installs security tools, backs up important files, hardens the SSH config, and downloads monitoring tools. Let it finish completely before moving on.

### Step 4 - Rotate All Passwords

```bash
bash passwd.sh | tee /tmp/passwords.txt
```

Saves new passwords to `/tmp/passwords.txt`. Open that file and save the passwords somewhere safe, like your phone notes.

>  **IMMEDIATELY after** - go to `http://172.16.215.250`, log in, and submit a PCR:

```
root,NewPasswordHere
otheruser,NewPasswordHere
```

One user per line. If you skip this, the scoring engine will fail your DNS checks.

### Step 5 — Apply Firewall

>  **FIRST** — edit `ufw.sh` and make sure **port 53 is allowed**, or you will lose scoring.

Full `ufw.sh` for your machine:

```bash
sudo ufw --force reset
sudo ufw default deny incoming
sudo ufw default allow outgoing
sudo ufw allow 22
sudo ufw allow 53
sudo ufw deny 4444
sudo ufw enable
sudo ufw status verbose
```

Then run it:

```bash
bash ufw.sh
```

>  After enabling the firewall - open a **second terminal** and verify you can still SSH in **before** closing your first session.

### Step 6 - Verify DNS is Running

```bash
systemctl status bind9
```

If `bind9` isn't found, try:

```bash
systemctl status named
```

Should show **active (running)**. If not — restart it:

```bash
systemctl restart bind9
```

Then verify port 53 is listening:

```bash
ss -tunp | grep :53
```

Should show something listening on `0.0.0.0:53` or `*:53`.

### Step 7 — Check for Backdoors

```bash
cat /etc/passwd | grep -E '/bin/.*sh'
```

Look for any users you don't recognize with a login shell. If you see something unexpected, tell the team.

```bash
cat /root/.ssh/authorized_keys 2>/dev/null
```

Should be empty or contain only expected keys. If you see a key you didn't add — delete it.

```bash
crontab -l && cat /etc/crontab
```

Check for any scheduled tasks. Note what's there so you can spot changes later.

---

## Phase 2 — Ongoing Monitoring (Every 20–30 Minutes)

### Check DNS is Still Running

```bash
systemctl status bind9
ss -tunp | grep :53
```

If DNS is down - restart immediately:

```bash
systemctl restart bind9
```

### Check Scoreboard

Go to `http://172.16.215.250` and verify your DNS service checks are passing. If they're failing, check the debug info on the scoreboard.

### Check for New Backdoors

```bash
cat /root/.ssh/authorized_keys
crontab -l
ls /etc/cron.d/
```

Anything new that wasn't there before is suspicious - tell the team.

### Check Active Connections

```bash
ss -tunp
```

Look for any outbound connections to unknown external IPs. If you see `bash` or `python` connecting out - tell the team immediately.

### Check Login Attempts

```bash
bash injects/login.sh
```

Shows how many successful and failed SSH attempts. A high failed count means the red team is brute forcing.

---

## DNS-Specific Troubleshooting

### DNS Won't Start

```bash
journalctl -u bind9 -n 50
```

Shows the last 50 log lines - look for error messages and tell the team what you see.

### DNS Config Check

```bash
named-checkconf
```

Checks the DNS config for errors. If it returns nothing - the config is fine. If it shows errors — tell the team.

### DNS Not Responding on Port 53

```bash
ufw allow 53
systemctl restart bind9
```

Make sure the firewall allows port 53 and restart the service.

---

## Emergency Procedures

### Machine Completely Broken

Request a snapshot revert from **@Green Team** in your team channel. Costs 500 points but better than losing 5 points per minute. After revert - redo **ALL** steps from Phase 1.

### Locked Out of SSH

Use the Proxmox console go to `https://proxmox.sdc.cpp`, find **Tegridy Farms**, and open the console. You can access the machine directly without SSH.

### Red Team Active on Your Machine

Comment out both instances of these lines in `ipban.sh` (so it doesn't reboot mid-incident):

```bash
sleep 2
reboot
```

Get their IP:

```bash
ss -tunp
```

Then ban them:

```bash
bash ipban.sh <their_IP>
```

This blocks them. Then kill the process to make sure any running tasks are cancelled:

```bash
ss -tunp | grep <their_IP>
kill -9 <PID1> <PID2> ...
```

---

## Quick Command Reference

```bash
systemctl status bind9          # check DNS
systemctl restart bind9         # restart DNS
ss -tunp | grep :53             # verify port 53 listening
ufw status verbose              # check firewall
ufw allow 53                    # open port 53 if blocked
systemctl status ssh            # check SSH
tail -f /var/log/auth.log       # watch login attempts live
cat /root/.ssh/authorized_keys  # check for dropped keys
```

---

## Final Notes

The job is to keep DNS up on port 53. The scripts do most of the heavy lifting. When in doubt, ping the team chat and we'll help troubleshoot.

-  Run `inventory.sh` first before touching anything
-  Submit a PCR after `passwd.sh` — the most important step
-  Check `systemctl status bind9` every 20–30 minutes
-  something breaks — ask the team

**Lock it down.**
