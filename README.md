# 🧀 Cheese CTF — TryHackMe Writeup

A comprehensive walkthrough of the **Cheese CTF** room on TryHackMe.

This machine focuses on a realistic web-to-system attack chain involving:

* 🔎 Network & service enumeration
* 💉 SQL Injection (SQLi)
* 📂 Local File Inclusion (LFI)
* ⛓️ PHP Filter Chain exploitation
* 💻 Remote Code Execution (RCE)
* 🔑 SSH Key Injection
* 🐧 Linux Privilege Escalation
* ⚙️ Systemd Timer misconfiguration

> **Difficulty:** Medium
> **Target:** Linux
> **Primary Skills:** Web Exploitation, PHP, Linux Privilege Escalation

---

## 📌 Attack Chain

The overall exploitation path was:

```text
Nmap Enumeration
      ↓
Web Application Discovery
      ↓
SQL Injection
      ↓
LFI
      ↓
PHP Filter Chain
      ↓
Remote Code Execution
      ↓
SSH Access
      ↓
SSH Key Injection
      ↓
Systemd Timer Enumeration
      ↓
Timer Modification
      ↓
Root Access
```

---

# 1. 🔎 Enumeration

The first step was to identify open ports and running services.

```bash
nmap -sC -sV -p- <TARGET_IP>
```

### What the command does

* `-sC` → Runs Nmap's default NSE scripts.
* `-sV` → Attempts to identify service versions.
* `-p-` → Scans all TCP ports from `1-65535`.

The scan revealed the services exposed by the target. The web service became the primary attack surface and was investigated further.

### Initial Recon Checklist

After discovering the web server, the following areas were checked:

```text
→ Web application
→ Robots.txt
→ Hidden directories
→ Application parameters
→ Login functionality
→ Source code
→ Included files
```

---

# 2. 🌐 Web Application Enumeration

Opening the discovered HTTP service in a browser revealed the target web application.

At this stage, the goal was not immediately to exploit the application, but to understand how it processed user-controlled input.

Particular attention was given to URL parameters and application functionality that appeared to retrieve information from the backend.

---

# 3. 💉 SQL Injection

One of the application parameters was found to be vulnerable to **SQL Injection**.

SQL Injection occurs when an application directly incorporates user-controlled input into a SQL query without properly validating or parameterizing it.

A vulnerable application may effectively construct a query similar to:

```sql
SELECT * FROM users WHERE username = '<USER_INPUT>';
```

If the input is not properly handled, an attacker may alter the intended SQL logic.

### Testing SQLi

A basic SQLi test can be performed by supplying characters that affect SQL syntax and observing whether the application's response changes.

For example:

```text
'
```

If the application behaves differently or produces a database-related error, this can indicate that the parameter may be interacting with a SQL query.

### Enumeration

Once SQL Injection was confirmed, the database structure and available information could be enumerated using appropriate SQLi techniques.

The important finding from this stage was that database-level access could be leveraged to obtain information useful for the next stage of the attack.

---

# 4. 📂 Local File Inclusion (LFI)

Further investigation revealed a **Local File Inclusion** vulnerability.

LFI occurs when a web application allows an attacker to influence which local file is loaded by the server.

A vulnerable implementation may conceptually look like:

```php
include($_GET['page']);
```

If user input is passed directly into the `include()` function, local files may become accessible.

A typical vulnerable request might look like:

```text
/index.php?page=<FILE>
```

The objective at this stage was to determine which local files could be read and whether the LFI could be escalated beyond simple file disclosure.

---

# 5. ⛓️ PHP Filter Chain Exploitation

The LFI vulnerability became significantly more interesting because the application was running PHP.

PHP stream filters can sometimes be abused to transform file contents before PHP processes them.

A commonly used technique involves:

```text
php://filter
```

For example, the general concept is:

```text
php://filter/convert.base64-encode/resource=<FILE>
```

This can allow source-code disclosure because PHP source is encoded before being returned.

### Why This Matters

Source-code disclosure can reveal:

* Database credentials
* Application logic
* File paths
* Authentication mechanisms
* Internal endpoints
* Dangerous PHP functionality

In this challenge, PHP filter-chain techniques were taken further to construct a payload capable of reaching **Remote Code Execution**.

> **Important:** PHP filter-chain exploitation is highly dependent on the PHP environment and available filters. The exact payload should therefore be treated as environment-specific rather than a universal exploit.

---

# 6. 💻 Remote Code Execution

After successfully exploiting the PHP filter chain, the vulnerability chain resulted in **Remote Code Execution** on the target.

The next objective was to convert this limited execution capability into a more stable shell.

A reverse shell generally works by making the compromised server establish an outbound connection to an attacker-controlled listener.

Before executing a reverse-shell payload, the attacker would prepare a listener:

```bash
nc -lvnp <PORT>
```

The resulting connection provided shell access to the target system.

---

# 7. 🔑 SSH Key Injection

With shell access established, the next objective was to obtain a more reliable method of authentication.

SSH uses public/private key cryptography for authentication.

The basic concept is:

```text
Private Key → Attacker
Public Key  → Target
```

The public key can be placed in the target user's:

```text
~/.ssh/authorized_keys
```

If SSH key authentication is enabled and the corresponding private key is available, the user can authenticate without a password.

### Why SSH Access Is Useful

Compared with a temporary reverse shell, SSH provides:

* More stable sessions
* Better terminal interaction
* Easier command execution
* Persistence for the compromised account

After configuring the SSH key appropriately, SSH access to the target user was established.

---

# 8. 🐧 Linux Privilege Escalation

Once authenticated as a low-privileged user, the focus shifted to privilege escalation.

The first step was basic system enumeration.

Useful commands include:

```bash
id
whoami
uname -a
sudo -l
```

Additional enumeration was performed to identify:

```text
→ SUID binaries
→ Writable files
→ Cron jobs
→ Systemd services
→ Systemd timers
→ Interesting capabilities
→ Misconfigured permissions
```

The key finding was a **Systemd Timer** that could be abused due to insecure file permissions.

---

# 9. ⚙️ Systemd Timer Misconfiguration

Systemd timers are commonly used to execute tasks at scheduled intervals.

A timer is typically associated with a service:

```text
timer
  ↓
service
  ↓
command/script
```

The important discovery was that a component executed by the timer could be modified by the compromised user.

This is a classic privilege-escalation mistake:

```text
Root-controlled scheduled task
             ↓
User-writable script/configuration
             ↓
Attacker-controlled command
             ↓
Code execution with root privileges
```

### Enumeration

Systemd timers can be enumerated with:

```bash
systemctl list-timers --all
```

Individual units can then be inspected using:

```bash
systemctl cat <TIMER_NAME>
```

and:

```bash
systemctl cat <SERVICE_NAME>
```

The goal was to identify:

1. Which timer executes.
2. Which service it triggers.
3. Which script/command is executed.
4. Whether that file is writable by the current user.

---

# 10. 👑 Privilege Escalation to Root

After identifying the writable component executed by the privileged Systemd service, it was possible to modify the execution path.

When the timer triggered the service, the modified command executed with elevated privileges.

This resulted in a root shell.

The final privilege level was verified with:

```bash
whoami
```

Expected result:

```text
root
```

---

# 11. 🏁 Flags

After obtaining the required access, the flags could be collected from their respective locations.

```bash
whoami
id
```

Then search the relevant directories/files discovered during enumeration.

> Replace this section with the actual flag values if you want your GitHub repository to contain the complete CTF solution.

---

# 🧠 Key Takeaways

This machine demonstrates how several seemingly separate vulnerabilities can be chained together into a complete compromise.

### 1. SQL Injection

Improperly handled database input can expose application data and potentially reveal information useful for further exploitation.

### 2. LFI

A file inclusion vulnerability can expose sensitive local files and, under certain conditions, become a stepping stone toward code execution.

### 3. PHP Filter Chains

PHP stream filters can be abused in specific environments to manipulate file processing and potentially turn LFI into RCE.

### 4. SSH Key Injection

Adding an attacker's public key to a user's `authorized_keys` can provide reliable SSH authentication when permissions and configuration allow it.

### 5. Systemd Timer Misconfiguration

Scheduled tasks running with elevated privileges become dangerous when the executed scripts or configuration files are writable by unprivileged users.

---

# 🛡️ Defensive Perspective

The same vulnerabilities can be prevented through proper security controls.

### SQL Injection

Use parameterized queries/prepared statements:

```text
User Input
    ↓
Prepared Statement
    ↓
Database
```

Never directly concatenate untrusted input into SQL queries.

### LFI

* Use allowlists for files.
* Avoid passing raw user input to `include()`.
* Restrict accessible paths.
* Disable unnecessary PHP wrappers/streams where appropriate.

### SSH

* Protect `~/.ssh` permissions.
* Restrict write access to `authorized_keys`.
* Monitor unauthorized key changes.
* Use centralized authentication where appropriate.

### Systemd

Scheduled scripts executed as root should be writable only by trusted administrators.

Check permissions regularly:

```bash
ls -la <SCRIPT>
```

A root-executed script should **not** be writable by an unprivileged user.

---

# 📚 Skills Practiced

```text
✔ Nmap Enumeration
✔ Web Application Enumeration
✔ SQL Injection
✔ Local File Inclusion
✔ PHP Stream Filters
✔ PHP Filter Chain Exploitation
✔ Remote Code Execution
✔ Reverse Shells
✔ SSH Authentication
✔ Linux Enumeration
✔ Systemd Enumeration
✔ Privilege Escalation
```

---

## Conclusion

The **Cheese CTF** demonstrates an important penetration-testing concept: a serious compromise does not always depend on a single critical vulnerability.

Instead, attackers can chain multiple weaknesses:

```text
SQLi
 ↓
Information Disclosure
 ↓
LFI
 ↓
PHP Filter Chain
 ↓
RCE
 ↓
SSH Access
 ↓
Systemd Misconfiguration
 ↓
Root
```

The challenge is a good practical exercise for understanding how web vulnerabilities can eventually lead to complete Linux system compromise.

> **Disclaimer:** This write-up is intended for educational purposes and authorized CTF/lab environments such as TryHackMe.
