<h1 align="center">🐧 Linux System Administration & Web Server Security Lab</h1>

<p align="center">
  Hands-on lab covering SSH hardening, privilege management, LAMP/WordPress deployment, and blue-team detection of privilege escalation.
</p>

<p align="center">
  <img alt="Linux" src="https://img.shields.io/badge/Linux-Debian%2FUbuntu-FCC624?logo=linux&logoColor=black">
  <img alt="Apache" src="https://img.shields.io/badge/Apache2-D22128?logo=apache&logoColor=white">
  <img alt="MariaDB" src="https://img.shields.io/badge/MariaDB-003545?logo=mariadb&logoColor=white">
  <img alt="WordPress" src="https://img.shields.io/badge/WordPress-21759B?logo=wordpress&logoColor=white">
  <img alt="Focus" src="https://img.shields.io/badge/Focus-Security-red">
</p>

---

## 📖 About

This repository documents a hands-on Linux administration lab done with a **security-first mindset**. The goal was not just to get things working, but to understand **why** each command and configuration choice matters.

The lab follows the full lifecycle of a server: preparing it for remote access, managing users and privileges safely, deploying a web stack, hardening it, and finally looking at what an attacker's privilege-escalation and persistence activity looks like from a defender's point of view.

**Author:** Hazem Akkouh
**Environment:** Debian/Ubuntu VM accessed over SSH

---

## 🎯 What's Covered

| # | Topic | Key Skills |
|---|---|---|
| 1 | **SSH setup & remote access** | `openssh-server`, `ufw`, `ss`, service management |
| 2-3 | **User & sudo management, safe renaming** | `usermod`, `groupmod`, avoiding lockouts |
| 4 | **Vim recovery tricks** | Editing root-owned files with `:w !sudo tee %` |
| 5 | **Auditing users & sudo access** | `/etc/passwd`, `getent`, `last`, `lastb`, `auth.log` |
| 6 | **Removing a logged-in user safely** | `deluser --remove-home`, orphan cleanup |
| 7-8 | **Shell productivity & filesystem hierarchy** | `bash-completion`, `fzf`, `du`, `ls -l` |
| 9 | **SSH key-based authentication** | `ssh-keygen`, `ssh-copy-id` |
| 10 | **Locking down root** | `passwd -l`, `nologin`, `PermitRootLogin no` |
| 11 | **Apache2 + WordPress + MariaDB deployment** | Virtual hosts, `.htaccess`, Basic Auth, DB hardening |
| 12 | **Configuration files reference** | What each config controls and its security impact |
| 13 | **Blue-team notes** | Privilege escalation & persistence indicators, `auditd` events |

---

## 📄 Full Report

The complete write-up, with every command and explanation, is here:

👉 **[Read the full lab report](./LAB_REPORT.md)**

---

## 🔐 Key Security Takeaways

- **Always use `visudo`** to edit `/etc/sudoers`. A syntax error can lock out every admin.
- **Create a fallback sudo user** before renaming or deleting the account you're using.
- **Disable root SSH login** (`PermitRootLogin no`) and lock the root password.
- **Use key-based SSH authentication** instead of passwords where possible.
- **Apache security only works if every layer lines up:** `DocumentRoot`, global `<Directory>` rules, and `.htaccess`.
- **Delete test files** like `info.php` and `testdb.php` right after use. They leak configuration details.
- **Detect by event type, not command name.** Monitoring `execve`, `chmod`, `chown`, `setuid`, and file writes is harder to evade than matching a list of "bad commands."

---

## 🧰 Tools & Technologies

- **OS:** Debian / Ubuntu
- **Remote access:** OpenSSH
- **Firewall:** UFW
- **Web server:** Apache2
- **Database:** MariaDB
- **CMS:** WordPress
- **Auth:** SSH keys, Apache Basic Auth (`htpasswd`)
- **Monitoring concepts:** `auditd`, `auth.log`, `last` / `lastb`

---

## 🗂️ Repository Structure

```
.
├── README.md         # You are here
└── LAB_REPORT.md     # Full step-by-step lab documentation
```

---

## ⚠️ Disclaimer

This lab was performed in an **isolated virtual machine** for educational purposes as part of cybersecurity coursework. The credentials and IP addresses shown are lab values only. Do not use the commands here on systems you do not own or have explicit permission to test.

---

## 📬 Contact

**Hazem Akkouh**
