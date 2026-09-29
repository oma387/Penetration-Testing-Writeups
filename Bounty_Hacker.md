# TryHackMe Write-Up: Bounty Hacker[cite: 6]

**Scope:** Controlled TryHackMe lab environment[cite: 6]  
**Assessment outcome:** Full compromise to root[cite: 6]

## 1. Executive Summary
During this assessment, a full compromise of the target Linux server was achieved.[cite: 6] The initial breach resulted from an anonymous FTP misconfiguration that exposed sensitive internal documents, including a valid username and a custom password dictionary.[cite: 6] This exposure enabled a targeted SSH brute-force attack.[cite: 6] Privilege escalation to root was subsequently achieved by exploiting a sudo misconfiguration that allowed the compromised user to execute the `tar` archiving utility with administrative privileges.[cite: 6]

## 2. Reconnaissance
The assessment began with an Nmap scan to enumerate open ports and identify running services on the target machine.[cite: 6]

~~~bash
nmap -sC -sV <TARGET_IP>
~~~

Results:[cite: 6]
* **Port 21 (FTP):** vsftpd 3.0.3 (Anonymous login enabled)[cite: 6]
* **Port 22 (SSH):** OpenSSH 7.2p2[cite: 6]
* **Port 80 (HTTP):** Apache httpd 2.4.18[cite: 6]

While the HTTP service hosted a static webpage with no immediate vulnerabilities, the FTP service allowed anonymous access, making it the primary target for enumeration.[cite: 6]

## 3. The Vulnerability & Foothold
I connected to the FTP server using the anonymous account and listed the directory contents.[cite: 6]

~~~bash
ftp <TARGET_IP>
~~~

Inside the server, two critical text files were discovered and downloaded:[cite: 6]
1. `task.txt`: A note outlining system tasks, signed by the user `lin`, confirming a valid system username.[cite: 6]
2. `locks.txt`: A custom list of potential passwords.[cite: 6]

Instead of utilizing a massive, generic dictionary file, I leveraged the custom `locks.txt` wordlist to launch a highly targeted brute-force attack against the SSH service for the user `lin`.[cite: 6]

~~~bash
hydra -l lin -P locks.txt ssh://<TARGET_IP>
~~~

Because the wordlist was tailored to the target environment, Hydra successfully cracked the password in seconds.[cite: 6] I logged into the machine via SSH and retrieved the user flag.[cite: 6]

## 4. Privilege Escalation
Upon gaining low-privileged access, the user's administrative rights were enumerated using `sudo -l`.[cite: 6]

**Results:** User `lin` may run the following commands on bountyhacker: `(root) NOPASSWD: /bin/tar`[cite: 6]

This misconfiguration allowed the user `lin` to run the `tar` backup utility as root without requiring a password.[cite: 6] According to GTFOBins, `tar` contains a checkpoint execution feature that can be abused to execute arbitrary system commands.[cite: 6]

I executed the `tar` utility with flags instructing it to spawn a shell at the first archive checkpoint.[cite: 6] Because the command was run with `sudo`, the resulting shell inherited full root privileges.[cite: 6]

~~~bash
sudo tar -cf /dev/null /dev/null --checkpoint=1 --checkpoint-action=exec=/bin/sh
~~~

This successfully dropped the session into a high-integrity root shell, allowing the retrieval of the final `root.txt` flag and completing the compromise.[cite: 6]

## 5. Remediation
To secure this infrastructure from similar attacks, the administration team should implement the following remediations:[cite: 6]
* **Disable Anonymous FTP:** The FTP server was leaking highly sensitive credentials.[cite: 6] Anonymous logins must be disabled in the FTP configuration to ensure only authenticated employees can access internal file shares.[cite: 6]
* **Eliminate Cleartext Credential Storage:** System passwords should never be stored in cleartext files (like `locks.txt`) on internal servers.[cite: 6] Passwords must be securely stored in an encrypted enterprise password manager.[cite: 6]
* **Restrict Sudo Execution:** The user `lin` was granted broad sudo access to `tar`, which features inherent command execution capabilities.[cite: 6] If the user requires backup capabilities, the sudoers file must be strictly configured to only allow specific, safe `tar` commands (e.g., locking down the exact directory and flags allowed) rather than granting blanket execution rights to the binary.[cite: 6]
