# TryHackMe — Publisher | Writeup

**Difficulty:** Easy  
**Target:** Publisher  
**Platform:** TryHackMe  
**Focus:** Web exploitation, SSH, Linux privilege escalation, SUID, and AppArmor

## 1. Reconnaissance

I started by scanning the target to identify open ports and running services.

The scan revealed two open ports:

| Port | Service | Description |
|---|---|---|
| 22 | SSH | OpenSSH |
| 80 | HTTP | Apache web server |

Since HTTP was available, I started investigating the web application.

## 2. Web Enumeration

I opened the website and noticed that it was running a web application.

To discover hidden directories, I used directory enumeration and found an interesting path:

```text
/spip/
```

After exploring the application and inspecting its source code and HTTP response headers, I identified its version:

```text
SPIP 4.2.0
```

This was important because the identified version was associated with a known vulnerability.

## 3. Initial Access — SPIP Remote Code Execution

After researching SPIP 4.2.0, I found **CVE-2023-27372**, an unauthenticated Remote Code Execution vulnerability.

I also found the corresponding Exploit-DB entry:

- Exploit-DB: 51536
- Metasploit module: `exploit/multi/http/spip_rce_form`

I used the Metasploit module to exploit the vulnerable SPIP installation.

One important detail was configuring the correct application path. The application was installed under `/spip/`, so the module's `TARGETURI` needed to point to that directory rather than the website root.

After successfully exploiting the vulnerability, I received a Meterpreter session.

I checked the current user:

```bash
getuid
```

The session was running as:

```text
www-data
```

I had successfully gained initial access to the target.

## 4. Moving from www-data to think

After obtaining the Meterpreter session, I started exploring the filesystem and checking the accessible directories.

While investigating the home directory of the user `think`, I discovered an SSH private key:

```text
/home/think/.ssh/id_rsa
```

I copied the private key to my AttackBox so I could use it for SSH authentication.

However, SSH initially refused to use the key because its permissions were too open.

I fixed that with:

```bash
chmod 600 id_rsa
```

This restricts access to the file owner, which is the permission level OpenSSH expects for a private key.

I then connected to the target:

```bash
ssh -i id_rsa think@<TARGET_IP>
```

After successfully authenticating, I obtained a shell as:

```text
think
```

I checked my identity:

```bash
id
```

The output confirmed that I was logged in as the user `think`, with UID 1000.

## 5. Privilege Escalation Enumeration — Searching for SUID and SGID Binaries

After gaining access as `think`, I started looking for possible Linux privilege-escalation opportunities.

One common enumeration technique is searching for executables with special permissions, particularly **SUID** and **SGID**.

### 5.1 Understanding SUID and SGID

Linux has two important special permission bits:

- **SUID (Set User ID):** When an executable is run, it uses the effective user ID of the file's owner.
- **SGID (Set Group ID):** When an executable is run, it uses the effective group ID of the file's group owner.

For example, if an executable is owned by root and has SUID enabled, it can run with root privileges even when launched by a normal user.

This is why unusual SUID executables are worth investigating.

### 5.2 Searching for SUID executables

I used `find` to search the filesystem for files with the SUID permission bit:

```bash
find / -type f -perm -4000 -ls 2>/dev/null
```

Breaking down the command:

| Argument | Meaning |
|---|---|
| `find /` | Search starting from the filesystem root |
| `-type f` | Search for regular files |
| `-perm -4000` | Find files with the SUID bit enabled |
| `-ls` | Display detailed information |
| `2>/dev/null` | Hide permission-denied errors |

### 5.3 Searching for SGID executables

I also searched for SGID-enabled files:

```bash
find / -type f -perm -2000 -ls 2>/dev/null
```

The permission bits are:

| Permission | Numeric value |
|---|---|
| SUID | `4000` |
| SGID | `2000` |

### 5.4 Investigating interesting results

Finding a SUID binary does not automatically mean that it is vulnerable.

I needed to investigate its ownership, permissions, behavior, and any files or scripts it interacts with.

During this enumeration, I discovered an interesting executable:

```text
/usr/sbin/run_container
```

Its permissions were:

```text
-rwsr-sr-x 1 root root 16760 Nov 14 2023 /usr/sbin/run_container
```

The `s` characters indicated that both SUID and SGID were enabled.

The executable was owned by root, making it particularly interesting.

This became the starting point for my next investigation.

## 6. Investigating run_container

I examined the executable and discovered that it interacted with another file:

```text
/opt/run_container.sh
```

I checked the script's permissions:

```text
-rwxrwxrwx 1 root root 1715 Jan 10 2024 /opt/run_container.sh
```

The important detail was that the script was writable by everyone.

That meant the `think` user could modify a script owned by root.

I also discovered that the `run_container` program executed the script using:

```bash
bash -p
```

The `-p` option enables Bash's privileged mode.

In this context, it prevents Bash from automatically discarding elevated effective privileges under the relevant conditions.

It does not grant root privileges by itself. The important part was that the executable was already running with elevated privileges.

This combination of a root-owned SUID executable and a world-writable script was a serious security weakness.

## 7. Understanding the AppArmor Restriction

Although I had found an interesting privilege-escalation path, the system had another security mechanism enabled: **AppArmor**.

AppArmor is a Linux security module that restricts what applications and processes can access.

It works through security profiles that define which files, directories, and operations a process is allowed to use.

I investigated the active restrictions and found an AppArmor profile associated with:

```text
/etc/apparmor.d/usr.sbin.ash
```

The profile restricted the `ash` shell.

Some directories, including `/opt`, were inaccessible under those restrictions.

This explained why simply trying to access the script through my normal shell was not enough.

I needed to understand how the restrictions were applied and which paths were still accessible.

## 8. Finding the AppArmor Restriction Weakness

While investigating the AppArmor profile, I noticed that its restrictions were not applied consistently to every temporary directory.

In particular, the rules affecting `/var/tmp` were incomplete compared with those affecting `/tmp`.

This was important because `/var/tmp` was still accessible.

The walkthrough by 0xb0b demonstrated that Bash could be copied to `/var/tmp` and executed from there, providing a way to work around the particular shell restriction in this lab.

The key lesson was that security profiles must be examined carefully: a restriction on one directory does not necessarily mean every similar directory is restricted in the same way.

## 9. Exploiting the Writable Script

At this point, I had identified the important pieces:

- A root-owned SUID executable: `/usr/sbin/run_container`
- A world-writable script: `/opt/run_container.sh`
- The executable ran the script using `bash -p`.
- AppArmor restricted the normal shell, but the `/var/tmp` route was available.

The privilege-escalation path relied on modifying the script that the privileged executable would run.

Because the script was executed in the elevated context, the changes it made could create a root-owned file with SUID enabled.

This was the key weakness that allowed the privilege escalation to work.

## 10. Root Access

After using the `/var/tmp` Bash route and the writable script, I was able to obtain a privileged Bash shell.

I checked my identity:

```bash
whoami
```

The result confirmed:

```text
root
```

I had successfully escalated from `think` to root.

## 11. Retrieving the Flags

With root access, I retrieved the root flag from:

```text
/root/root.txt
```

I had already retrieved the user flag during the machine's exploitation process.

Both flags were successfully obtained, completing the room.

---

## Attack Path Summary

```text
Reconnaissance
     |
     v
Ports 22 (SSH) and 80 (HTTP)
     |
     v
Discover /spip/
     |
     v
Identify SPIP 4.2.0
     |
     v
CVE-2023-27372
     |
     v
Metasploit RCE
     |
     v
www-data
     |
     v
Discover think's SSH private key
     |
     v
SSH as think
     |
     v
SUID / SGID enumeration
     |
     v
Discover root-owned run_container
     |
     v
Find world-writable run_container.sh
     |
     v
Investigate AppArmor restrictions
     |
     v
Use the /var/tmp Bash route
     |
     v
Exploit the privileged script execution
     |
     v
ROOT
```

## Key Takeaways

This room taught me several important Linux and web-security concepts:

1. **Web enumeration:** Identifying hidden directories and software versions can reveal vulnerable applications.
2. **CVE research:** Knowing how to connect a software version to a known vulnerability is an important part of penetration testing.
3. **Remote Code Execution:** A vulnerable web application can provide an initial foothold on a target.
4. **SSH authentication:** Private keys can provide access to other system accounts when they are accessible and accepted by the SSH service.
5. **SUID and SGID:** Special permission bits can create privilege-escalation opportunities when executables are improperly configured.
6. **Linux file permissions:** A privileged program interacting with a world-writable script can introduce a serious security weakness.
7. **AppArmor:** Mandatory access-control profiles can restrict processes even when a user has a valid shell.
8. **Bash privileged mode:** The `-p` option affects how Bash handles elevated effective privileges; it does not independently grant root access.

The most important lesson was that Linux privilege escalation is not always about finding a vulnerable binary. Sometimes the weakness comes from the relationship between a privileged executable, the scripts it runs, file permissions, and security policies.
