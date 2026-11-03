# TryHackMe: Library — Beginner-Friendly Write-up

**Platform:** TryHackMe  
**Room:** Library  
**Difficulty:** Easy  
**Target:** Linux  
**Objective:** Retrieve the user and root flags.

## Overview

This room demonstrates a basic penetration-testing workflow: reconnaissance, web enumeration, initial access through SSH, and Linux privilege escalation.

My path through the room was:

1. Scan the target and identify its services.
2. Explore the website and identify a potential username.
3. Use Hydra and the RockYou wordlist to obtain SSH credentials.
4. Log in as `meliodas` and retrieve `user.txt`.
5. Inspect local sudo permissions.
6. Discover that a root-owned Python script can be replaced because it is located in a user-controlled directory.
7. Execute the replacement script through the permitted sudo rule and obtain root access.

## 1. Reconnaissance

I began by scanning the target to identify open ports and the services running on it.

```bash
nmap -sC -sV <TARGET_IP>
```

The scan identified two relevant services:

| Port | Service | Version |
|---|---|---|
| 22 | SSH | OpenSSH 7.2p2 |
| 80 | HTTP | Apache 2.4.18 |

**What the options mean:**

- `-sC` runs Nmap's default scripts.
- `-sV` attempts to identify service versions.
- `<TARGET_IP>` is the IP address assigned to the target machine.

Port 80 provided a website to investigate, while port 22 indicated that SSH could be a potential route to initial access.

## 2. Web Enumeration

I opened the target's website in a browser and explored its blog content.

While browsing, I identified several possible usernames, including `meliodas`. Usernames found on a website can sometimes be useful during subsequent authentication testing.

I also checked the site's `robots.txt` file:

```text
http://<TARGET_IP>/robots.txt
```

The file contained a reference to `rockyou`, which suggested a possible connection to the commonly used `rockyou.txt` password wordlist.

This was a useful clue, but it did not prove that any particular account had a weak password. I still needed to test the hypothesis.

## 3. Initial Access Through SSH

Since SSH was open and I had identified a potential username, I tested the account using Hydra and the RockYou wordlist.

```bash
hydra -l meliodas -P /usr/share/wordlists/rockyou.txt ssh://<TARGET_IP>
```

**Command breakdown:**

- `-l meliodas` specifies the username.
- `-P` specifies a file containing candidate passwords.
- `ssh://<TARGET_IP>` identifies the SSH service being tested.

Hydra found valid credentials for the account.

I then connected to the machine:

```bash
ssh meliodas@<TARGET_IP>
```

After authenticating successfully, I obtained a shell as `meliodas`.

## 4. Retrieving the User Flag

After logging in, I checked the current directory and listed its contents:

```bash
pwd
ls -la
```

The home directory contained `user.txt` and `bak.py`.

I retrieved the user flag with:

```bash
cat user.txt
```

At this point, initial access was complete. The next objective was to investigate whether the account could obtain higher privileges.

## 5. Enumerating Sudo Permissions

I ran:

```bash
sudo -l
```

This command lists the commands the current user is permitted to execute through `sudo`.

The relevant output was:

```text
(ALL) NOPASSWD: /usr/bin/python* /home/meliodas/bak.py
```

This permission is worth examining carefully.

- `(ALL)` means the permitted command can be run as any user, including root.
- `NOPASSWD` means the matching command does not require a password.
- `/usr/bin/python*` matches Python executable paths covered by the rule.
- `/home/meliodas/bak.py` specifies the script path.

The rule did not grant unrestricted permission to run every Python script as root. It allowed a matching Python executable to run this particular script with elevated privileges.

## 6. Inspecting bak.py

I examined the script:

```bash
cat /home/meliodas/bak.py
```

The original code was:

```python
#!/usr/bin/env python
import os
import zipfile

def zipdir(path, ziph):
    for root, dirs, files in os.walk(path):
        for file in files:
            ziph.write(os.path.join(root, file))

if __name__ == '__main__':
    zipf = zipfile.ZipFile('/var/backups/website.zip', 'w', zipfile.ZIP_DEFLATED)
    zipdir('/var/www/html', zipf)
    zipf.close()
```

The script creates a ZIP archive of the website directory at `/var/www/html` and writes it to `/var/backups/website.zip`.

On its own, this is a normal backup operation. The important question was whether the script and its location could be manipulated in a way that affected its execution with root privileges.

## 7. Understanding the File Permissions

I checked the script's ownership and permissions:

```bash
ls -l /home/meliodas/bak.py
```

The output showed that the file was owned by root and was not writable by `meliodas`.

I then checked the parent directory:

```bash
ls -ld /home/meliodas
```

The directory was owned by `meliodas`, and its permissions allowed its owner to create and remove directory entries.

This distinction is central to the vulnerability.

**File permissions** govern access to a file's contents. **Directory permissions** govern operations on entries within the directory, such as creating, deleting, and renaming files.

Consequently, a user may be unable to edit a root-owned file directly but still be able to remove or replace it if the parent directory's permissions allow that operation.

The fact that the script was owned by root was therefore not enough to make this setup safe.

## 8. Privilege Escalation: Replacing the Script

I used the directory-permission weakness to replace the original `bak.py` with my own Python script at the same path.

The significant combination was:

- The sudo rule permitted execution of `/home/meliodas/bak.py` as root.
- The home directory was controlled by `meliodas`.
- The script could be replaced even though its original contents were not directly writable.

After replacing the script, I invoked the permitted command using the absolute paths:

```bash
sudo /usr/bin/python3 /home/meliodas/bak.py
```

This executed the replacement script with elevated privileges and gave me a root shell.

### Why the absolute path matters

The sudo rule specified `/home/meliodas/bak.py`, not simply `bak.py`.

Using an absolute path makes it clear which script is being invoked and matches the path in the permission rule. This was also relevant when troubleshooting my earlier attempts, which used a relative path.

### Why this is a security misconfiguration

A privileged script should not be stored in a directory that an unprivileged user can control if that user can replace the script and then execute it with elevated privileges.

The combination of a permissive sudo rule and an unsafe file location allowed a low-privileged account to obtain root access.

## 9. Retrieving the Root Flag

Once I had obtained the root shell, I verified the current user:

```bash
whoami
```

The output confirmed that the shell was running as `root`.

I then retrieved the root flag:

```bash
cat /root/root.txt
```

The room was complete.

## Lessons Learned

### 1. Enumerate before exploiting

The open ports, website content, and `robots.txt` provided clues that helped determine where to investigate next.

### 2. Treat password clues as hypotheses

A username and a wordlist hint can suggest a potential authentication weakness, but the weakness must still be verified.

### 3. Read sudo rules carefully

`sudo -l` can reveal commands that have been granted excessive privileges. The exact executable and argument paths matter.

### 4. Check directory permissions, not just file ownership

A file owned by root may still be replaceable if it is stored in a directory controlled by another user.

### 5. Review the full execution chain

The risk did not come from Python alone. It came from the combination of a privileged execution rule, a replaceable script, and a user-controlled directory.

## Conclusion

