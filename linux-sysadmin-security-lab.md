# Linux System Administration & Web Server Security Lab

**Author:** *Hazem Akkouh*
**Environment:** Debian/Ubuntu VM (target `192.168.159.111`), SSH remote access
**Scope:** SSH hardening, user/privilege management, Apache + WordPress + MariaDB deployment, and blue-team indicators of privilege escalation

---

## Table of Contents

1. [Overview](#overview)
2. [Part 1 :SSH Setup & Remote Access](#part-1--ssh-setup--remote-access)
3. [Part 2 :User & Sudo Privilege Management](#part-2--user--sudo-privilege-management)
4. [Part 3 :Safe User Renaming](#part-3--safe-user-renaming)
5. [Part 4 :Vim Recovery Tricks (Editing Root-Owned Files)](#part-4--vim-recovery-tricks-editing-root-owned-files)
6. [Part 5 :Auditing Users & Sudo Access](#part-5--auditing-users--sudo-access)
7. [Part 6 :Safely Removing a Logged-In User](#part-6--safely-removing-a-logged-in-user)
8. [Part 7 :Shell Quality-of-Life (Autocomplete)](#part-7--shell-quality-of-life-autocomplete)
9. [Part 8 :Filesystem Hierarchy Review](#part-8--filesystem-hierarchy-review)
10. [Part 9 :SSH Key-Based Authentication](#part-9--ssh-key-based-authentication)
11. [Part 10 :Locking Down the Root Account](#part-10--locking-down-the-root-account)
12. [Part 11 :Apache2 + WordPress + MariaDB Deployment](#part-11--apache2--wordpress--mariadb-deployment)
13. [Part 12 :Configuration Files Reference](#part-12--configuration-files-reference)
14. [Part 13 :Blue Team Notes: Privilege Escalation & Persistence Indicators](#part-13--blue-team-notes-privilege-escalation--persistence-indicators)
15. [Lessons Learned](#lessons-learned)

---

## Overview

This lab documents a hands-on exercise in Linux system administration with a security-first mindset. It covers the full lifecycle of preparing a server for remote access, managing users and privileges safely, deploying a LAMP-based web application (Apache + MariaDB + WordPress), hardening the resulting stack, and finally reviewing what an attacker's privilege-escalation and persistence activity looks like from a defender's (SOC/`auditd`) perspective.

The goal was not just to get things "working," but to understand **why** each command and configuration choice matters from a security standpoint.

---

## Part 1 :SSH Setup & Remote Access

Initial reconnaissance and enabling SSH access on the target machine.

```bash
ip a                              # Identify network interfaces / IP address
sudo systemctl status ssh         # Check if SSH service is active
sudo apt update
sudo apt install openssh-server -y
sudo systemctl start ssh
sudo systemctl enable ssh         # Persist across reboots
sudo ss -tulnp | grep 22          # Confirm SSH is listening on port 22
sudo ufw status
sudo ufw allow ssh                # Allow SSH through the firewall
sudo ufw reload
```

Connecting from the attacking/admin machine and basic recon on the target:

```bash
ssh user@192.168.159.130
whoami
cat /etc/passwd    # Enumerate all system accounts
cat /etc/shadow    # Requires root :view password hashes
useradd -m -s /bin/bash dev   # Create a new user 'dev' with a home dir and bash shell
```

**Security takeaway:** exposing SSH means the attack surface grows immediately :firewall rules, key-based auth, and disabling root login (covered later) are essential follow-ups, not optional extras.

---

## Part 2 :User & Sudo Privilege Management

Before renaming or deleting accounts, always create a **temporary sudo-capable user** so you never lock yourself out.

```bash
sudo adduser bob
sudo usermod -aG sudo bob
# Logout and log back in as bob before continuing
```

---

## Part 3 :Safe User Renaming

Renaming a user (`user` → `admin`) without breaking ownership of the home directory or primary group.

```bash
# Step 1: rename the account
sudo usermod -l admin user

# Step 2: rename & move the home directory
sudo usermod -d /home/admin -m admin

# Step 3: rename the matching primary group (if group name == old username)
sudo groupmod -n admin user

# Step 4: verify
id admin
ls /home
```

---

## Part 4 :Vim Recovery Tricks (Editing Root-Owned Files)

A common gotcha: opening a root-owned config file in Vim **without** `sudo`, then being unable to save.

**Solution 1 :reopen with sudo (simplest):**
```
Esc
:q!
sudo vim /etc/ssh/sshd_config
Esc
:wq
```

**Solution 2 :save in place without exiting (advanced):**
```
Esc
:w !sudo tee %
[enter password when prompted]
:q!
```

---

## Part 5 :Auditing Users & Sudo Access

A checklist used to review who has elevated privileges on the box.

```bash
# All system users (including service accounts)
cat /etc/passwd

# Only "real" human users (UID >= 1000)
awk -F: '$3 >= 1000 {print $1}' /etc/passwd

# Group membership for a given user
groups username
groups admin

# Who is in the sudo / admin groups?
getent group sudo
getent group admin

# Review sudoers safely
sudo cat /etc/sudoers      # read-only inspection
sudo visudo                # the ONLY safe way to edit sudoers (syntax-checked)

# Remove a user from a privileged group
sudo deluser userX sudo
sudo deluser userX admin
groups userX               # verify removal

# SSH root-login policy
sudo grep PermitRootLogin /etc/ssh/sshd_config
sudo systemctl restart ssh

# Login activity
last                       # successful logins
sudo lastb                 # failed login attempts
who                        # currently logged-in users
sudo cat /var/log/auth.log | grep sudo   # sudo usage history

ss -tulpn | grep ssh       # confirm SSH service state
```

**Target security posture for this lab:**
- `root` account exists but is not used interactively
- `admin` is the only account in the `sudo` group
- All other accounts are excluded from `sudo`/`admin`
- `PermitRootLogin no` in `sshd_config`
- SSH restarted after every config change
- Login activity actively monitored (`last`, `lastb`, `auth.log`)

---

## Part 6 :Safely Removing a Logged-In User

Deleting an account that may still have active processes/sessions requires care to avoid orphaned processes or file-permission issues.

```bash
# 1. Reboot to guarantee the target user has no active processes
sudo reboot

# 2. Log back in as 'admin' BEFORE the target user ('bob') logs back in

# 3. (Optional) Move bob off his own primary group first
sudo groupadd tempgroup
sudo usermod -g tempgroup bob

# 4. Delete the user and their home directory
sudo deluser --remove-home bob

# 5. Clean up the now-orphaned group
sudo delgroup bob
```

---

## Part 7 :Shell Quality-of-Life (Autocomplete)

Not security-critical, but useful for working efficiently at the CLI.

```bash
sudo apt update
sudo apt install bash-completion -y
echo '[[ $PS1 && -f /usr/share/bash-completion/bash_completion ]] && source /usr/share/bash-completion/bash_completion' >> ~/.bashrc
source ~/.bashrc
sudo apt install fzf -y     # optional: fuzzy history/command search
```

---

## Part 8 :Filesystem Hierarchy Review

Reviewing what lives under the standard Linux directory tree, and checking sizes/permissions as a sanity check.

```bash
ls /bin   /etc   /usr   /var   /home   /root
cd /bin   /etc   /usr   /var   /home
sudo cd /root       # root's home requires elevated privileges

du -sh /bin /etc /usr /var /home /root   # disk usage
ls -l  /bin /etc /usr /var /home /root   # permission review
```

---

## Part 9 :SSH Key-Based Authentication

Moving from password auth to key-based auth for the `dev` account.

```bash
# Check for existing keys
ls ~/.ssh/

# Generate a new RSA keypair
ssh-keygen -t rsa
# -> private key: ~/.ssh/id_rsa
# -> public key:  ~/.ssh/id_rsa.pub

# Push the public key to the remote 'dev' account
ssh-copy-id dev@192.168.159.130

# Passwordless login test
ssh dev@192.168.159.130

# Escalate to root locally (requires root password)
su root

# Edit SSH server config if needed (port, PasswordAuthentication, etc.)
vim /etc/ssh/sshd_config

exit   # leave root
exit   # close SSH session
```

---

## Part 10 :Locking Down the Root Account

Steps taken to prevent direct/interactive root logins while keeping the account itself intact for ownership purposes.

```bash
su root
cd ~
vim /etc/sudoers        # inspection only :NOT the recommended edit method
visudo                  # correct, syntax-safe way to edit sudoers

groups dev
groups root

usermod -aG sudo dev    # grant dev sudo rights
groups dev               # confirm

# Lock the root password (disables password login for root)
sudo passwd -l root

# Change root's shell to a non-login shell (defense in depth)
sudo chsh root
# set to: /usr/sbin/nologin
```

> **Recovery note:** if `/usr/sbin/nologin` locks you out of `su root` entirely and you need it back, edit `/etc/passwd` directly and change root's shell field back to `/bin/bash`:
> ```bash
> sudo vim /etc/passwd
> ```

---

## Part 11 :Apache2 + WordPress + MariaDB Deployment

### 11.1 Connectivity & user context
```bash
ssh root@192.168.159.130   # denied :root SSH disabled, as intended
ssh dev@192.168.159.130    # normal login path

su admin                   # switch to a more privileged account
sudo -i                    # interactive root shell for admin tasks
```

### 11.2 Locating the web root & WordPress files
```bash
sudo ls -al /var/www/
ls -al /usr/share/wordpress
ls -al /var/www/html/
ls -l  /var/www/wordpress
cd /var/www/wordpress/
cd /etc/apache2/
```

### 11.3 Wiring WordPress into the Apache web root
```bash
sudo ln -s /usr/share/wordpress /var/www/html/wordpress
sudo ln -s /usr/share/wordpress /var/www/wordpress
```

### 11.4 Key configuration edits
```bash
sudo nano /etc/wordpress/config-192.168.159.130.php   # DB credentials
sudo vim  /etc/apache2/apache2.conf                    # global security directives
sudo nano /etc/apache2/sites-available/000-default.conf # DocumentRoot / VirtualHost
sudo nano /etc/apache2/conf-available/security.conf     # hide server version info
vim .htaccess                                           # per-directory auth rules
```

### 11.5 File ownership for Apache
```bash
sudo chown -R www-data:www-data /usr/share/wordpress
sudo chown www-data:www-data /etc/wordpress/config-192.168.159.130.php
```

### 11.6 Database service (MariaDB)
```bash
sudo systemctl start mariadb
sudo systemctl enable mariadb
sudo systemctl status mariadb
sudo mysql_secure_installation   # remove anonymous users, secure install
```

### 11.7 Creating the WordPress database
```sql
sudo mysql -u root -p

CREATE DATABASE wordpress DEFAULT CHARACTER SET utf8 COLLATE utf8_unicode_ci;
CREATE USER 'wordpressuser'@'localhost' IDENTIFIED BY 'yourpassword';
GRANT ALL PRIVILEGES ON wordpress.* TO 'wordpressuser'@'localhost';
FLUSH PRIVILEGES;

-- if authentication later fails:
ALTER USER 'wordpressuser'@'localhost' IDENTIFIED BY 'root';

-- verify accounts & auth plugin
SELECT user, host, authentication_string, plugin FROM mysql.user;
```

### 11.8 Applying config & enabling modules
```bash
sudo systemctl restart apache2
sudo systemctl status apache2
sudo a2enmod rewrite          # required for WordPress permalinks
```

### 11.9 Basic Authentication in front of the site
```bash
sudo apt update
sudo apt-get install apache2-utils -y
sudo htpasswd -c /etc/apache2/.htpasswd dev
```

### 11.10 System control
```bash
reboot
systemctl reboot -i           # force reboot, ignoring active-session warnings
```

---

## Part 12 :Configuration Files Reference

| File | Edited With | Purpose | Security Impact |
|---|---|---|---|
| `/etc/apache2/sites-available/000-default.conf` | `nano` | Defines `DocumentRoot` / VirtualHost | Wrong root path = every downstream rule (auth, permissions) silently fails to apply |
| `/etc/apache2/apache2.conf` | `vim` | Global `<Directory>` access rules | `AllowOverride All` enables `.htaccess`; `Require all denied` blocks system dirs; `Options -Indexes` stops directory listing |
| `.htaccess` (in WordPress dir) | `vim` | Per-directory override | Adds `AuthType Basic` / `Require valid-user`; only works if parent config allows overrides |
| `/etc/apache2/.htpasswd` | `htpasswd` | Encrypted credential store | Apache checks this before granting access via Basic Auth |
| `/etc/wordpress/config-<ip>.php` | `nano` | WordPress DB connection | Wrong/weak credentials here = DB compromise risk or "Error establishing a database connection" |
| `/etc/apache2/conf-available/security.conf` | `nano` | Server hardening (`ServerSignature Off`, `ServerTokens Prod`) | Hides Apache/OS version, reducing recon value for attackers |
| `/var/www/html/info.php`, `testdb.php` | `nano` | Temporary PHP/DB test scripts | **Must be deleted after testing** :they leak PHP config and DB details |
| `/etc/sudoers` | `visudo` **only** | Controls who can run privileged commands | Misconfiguration can either lock out admins or grant unintended root access |

**Request flow, end to end:**
1. Apache reads `000-default.conf` → resolves the site's document root
2. Global rules from `apache2.conf` are applied
3. If overrides are allowed, Apache reads the local `.htaccess`
4. `.htaccess` enforces Basic Auth against `.htpasswd`
5. WordPress connects to MariaDB using `config-<ip>.php`
6. `security.conf` suppresses version banners in all responses
7. `sudoers` governs who can manage any of the above

---

## Part 13 :Blue Team Notes: Privilege Escalation & Persistence Indicators

The final part of this lab flips perspective from *builder* to *defender* :recognizing the command patterns and file changes that typically show up during a Linux compromise, so they can be mapped to detection rules (e.g. `auditd`, EDR, SIEM correlation).

### Categories of suspicious activity to monitor

**Unauthorized privilege changes**
- Sudden use of `sudo su`, `sudo -i`, or repeated `sudo -l` checks
- SUID/SGID bit changes (`chmod u+s`, `chmod 4755`) or capability grants (`setcap cap_setuid+ep`)
- Unexpected `usermod -aG sudo`, new `useradd` calls, or direct edits appending entries to `/etc/passwd`

**Execution of unexpected binaries**
- Scripts run from world-writable locations like `/tmp` or `/dev/shm`
- Download-and-execute chains (`wget`/`curl` piped straight into `bash`)
- "Living-off-the-land" interpreter one-liners (`bash -i`, `python -c`, `perl -e`) or reverse-shell primitives (`nc -e`, `socat ... EXEC:`)

**Modification of sensitive system files**
- Edits to `/etc/passwd`, `/etc/shadow`, `/etc/sudoers`, `/etc/group`
- Use of `chattr +i` to make a tampered file immutable and harder to revert

**Persistence mechanisms**
- New or modified cron entries (`crontab -e`, writes to `/var/spool/cron/root`)
- Rogue systemd services created and enabled
- Payloads appended to shell startup files (`~/.bashrc`, `~/.profile`)

**Permission/ownership tampering**
- Overly permissive `chmod 777`, unexpected `chown root`, binaries copied into `/usr/bin` or `/bin`

**Common recon commands preceding an escalation attempt**
```bash
id
whoami
groups
uname -a
ps aux
sudo -l
```

### What actually matters for detection

Rather than memorizing an attacker's exact command list, effective monitoring focuses on the **underlying syscalls/events** that `auditd` (or an EDR) can reliably capture:

| Event type | What it flags |
|---|---|
| `execve` | Any process/command execution |
| `chmod` / `fchmod` | Permission changes |
| `chown` | Ownership changes |
| `open` / `write` | File content modification |
| `setuid` / `setgid` | Privilege escalation attempts |

This event-based view is more resilient than string-matching specific commands, since attackers can trivially rename binaries or use alternate syntax, but the underlying kernel-level actions are much harder to hide.

---

## Lessons Learned

- **Never edit `/etc/sudoers` directly** :always use `visudo` to avoid a syntax error locking every admin out.
- **Always create a fallback sudo user** before renaming or deleting the account you're currently using.
- **Reboot before deleting a logged-in user** to guarantee no orphaned processes are left holding file handles.
- Apache's behavior is only as secure as its **weakest applied config layer** :`DocumentRoot`, global `<Directory>` rules, and `.htaccess` all have to align, or protections silently don't apply.
- **Delete test/debug files** (`info.php`, `testdb.php`) immediately after use :they're a common source of accidental information disclosure.
- Defense is stronger when built around **event types** (`execve`, `setuid`, file writes) rather than a static list of "bad commands," since the latter is trivial to evade.

---

*This report was compiled from personal lab notes for educational purposes as part of ongoing cybersecurity coursework.*
