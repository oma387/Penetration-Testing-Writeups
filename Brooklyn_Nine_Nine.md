# TryHackMe Write-Up: Brooklyn Nine-Nine[cite: 7]

**Scope:** Controlled TryHackMe lab environment[cite: 7]  
**Assessment outcome:** Full compromise to root[cite: 7]

## 1. Executive Summary
During this assessment, a full compromise of the target Linux server was achieved.[cite: 7] The initial foothold was gained due to an anonymous FTP misconfiguration that leaked an internal username and hinted at a weak password policy.[cite: 7] This allowed for a successful SSH brute-force attack.[cite: 7] Privilege escalation to root was achieved by exploiting a sudo misconfiguration that allowed the user to execute the `less` pager utility with administrative privileges.[cite: 7]

## 2. Reconnaissance
The assessment began with an aggressive Nmap scan to identify open ports and running services.[cite: 7]

~~~bash
nmap -Pn -sC -sV <TARGET_IP>
~~~

Results:[cite: 7]
* **Port 21 (FTP):** vsftpd 3.0.3 (Anonymous login enabled)[cite: 7]
* **Port 22 (SSH):** OpenSSH 7.6p1[cite: 7]
* **Port 80 (HTTP):** Apache httpd 2.4.29[cite: 7]

Because Port 21 had Anonymous login enabled, I started enumeration there.[cite: 7]

## 3. The Vulnerability & Foothold
I connected to the FTP server using the anonymous account (leaving the password blank) and listed the hidden files using `ls -la`.[cite: 7]

~~~bash
ftp <TARGET_IP>
~~~

Inside the directory, I discovered a text file named `note_to_jake.txt`.[cite: 7] After downloading and reading the file, I found a message from a manager explicitly stating that Jake's password was too weak.[cite: 7] This gave me two critical pieces of information: a valid SSH username (`jake`) and confirmation that his password was likely in a common dictionary.[cite: 7]

I used Hydra and the `rockyou.txt` wordlist to brute-force the SSH service for the user `jake`.[cite: 7]

~~~bash
hydra -l jake -P /usr/share/wordlists/rockyou.txt ssh://<TARGET_IP>
~~~

Hydra successfully cracked the password in seconds.[cite: 7] I logged into the machine via SSH as the user `jake` and retrieved the user flag.[cite: 7]

## 4. Privilege Escalation
Once I had a low-privileged shell, I checked the current user's administrative permissions using `sudo -l`.[cite: 7]

**Results:** User `jake` may run the following commands on brooklyn_nine_nine: `(ALL) NOPASSWD: /usr/bin/less`[cite: 7]

This misconfiguration allowed the user `jake` to run the `less` command (a file pager) as root without requiring a password.[cite: 7] According to GTFOBins, the `less` utility has an interactive shell escape feature.[cite: 7] If run with sudo, the spawned shell inherits root privileges.[cite: 7]

I executed `less` on a system file and typed `!/bin/sh` to break out of the pager.[cite: 7]

~~~bash
sudo less /etc/hosts
!/bin/sh
~~~

This immediately dropped me into a high-integrity root shell.[cite: 7] I navigated to the `/root` directory and captured `flag.txt`, completing the compromise.[cite: 7]

## 5. Remediation
To secure this server from similar attacks, the system administrator should implement the following changes:[cite: 7]
* **Disable Anonymous FTP:** If public file sharing is not strictly required, anonymous logins should be disabled in the `vsftpd.conf` file to prevent sensitive internal communications from leaking.[cite: 7]
* **Enforce Password Complexity:** The user `jake` was using an easily guessable password.[cite: 7] The organization should enforce strict password complexity requirements and consider disabling password-based SSH authentication entirely in favor of SSH key pairs.[cite: 7]
* **Review Sudo Permissions:** The user `jake` should not be allowed to run binaries with shell-escape features (like `less`, `vim`, or `man`) as root.[cite: 7] The sudoers file should be heavily restricted to only allow the specific commands necessary for the user's job role.[cite: 7]
