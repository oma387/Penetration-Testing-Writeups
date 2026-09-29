# TryHackMe Write-Up: RootMe[cite: 9]

**Scope:** Controlled TryHackMe lab environment[cite: 9]  
**Assessment outcome:** Full compromise to root[cite: 9]

## 1. Executive Summary
During this assessment, a full compromise of the target Linux web server was achieved.[cite: 9] The initial foothold was established by exploiting an insecure file upload vulnerability on a hidden web panel, utilizing an extension bypass technique to execute a malicious payload.[cite: 9] Privilege escalation to root was subsequently achieved by abusing a system misconfiguration where the Python interpreter was assigned SUID (Set Owner User ID) permissions.[cite: 9]

## 2. Reconnaissance
The assessment began with an Nmap scan to map the external attack surface and identify running services.[cite: 9]

~~~bash
nmap -sC -sV <TARGET_IP>
~~~

Results:[cite: 9]
* **Port 22 (SSH):** OpenSSH 7.6p1[cite: 9]
* **Port 80 (HTTP):** Apache httpd 2.4.29[cite: 9]

With the web server identified, a directory brute-force attack was launched to discover hidden administrative or development paths not linked on the main page.[cite: 9]

~~~bash
gobuster dir -u http://<TARGET_IP> -w /usr/share/wordlists/dirb/common.txt
~~~

Results:[cite: 9]
* `/panel/` (A hidden file upload interface)[cite: 9]
* `/uploads/` (The publicly accessible directory where uploaded files are stored)[cite: 9]

## 3. The Vulnerability & Foothold
Navigating to the `/panel/` directory revealed a web form allowing users to upload files.[cite: 9] To test for Remote Code Execution (RCE), a standard PHP reverse shell (`shell.php`) was uploaded.[cite: 9] The application rejected the file, indicating a basic blacklist filter was in place to block `.php` extensions.[cite: 9]

To bypass this restriction, the file extension was altered to an alternative PHP execution format: `shell.phtml`.[cite: 9] The web application failed to filter this extension and successfully uploaded the payload.[cite: 9]

A Netcat listener was established on the attacking machine to catch the incoming connection:[cite: 9]

~~~bash
nc -lvnp 1234
~~~

The payload was triggered by navigating to `http://<TARGET_IP>/uploads/shell.phtml`.[cite: 9] The connection was successfully caught, providing a reverse shell as the low-privileged `www-data` service account.[cite: 9]

To ensure the shell was fully interactive and stable (allowing for command history and clearing the screen), it was upgraded using Python's pseudo-terminal utility:[cite: 9]

~~~bash
python -c 'import pty; pty.spawn("/bin/bash")'
~~~

## 4. Privilege Escalation
With a stable foothold, the file system was enumerated for privilege escalation vectors.[cite: 9] A search for binaries with the SUID bit set was executed to identify programs that run with the permissions of their owner (which is often root).[cite: 9]

~~~bash
find / -type f -perm -04000 -ls 2>/dev/null
~~~

**Results:** The output revealed that `/usr/bin/python` possessed SUID permissions.[cite: 9] This is a critical security flaw, as Python can execute system commands directly.[cite: 9]

Consulting GTFOBins for the exact syntax, a Python one-liner was executed to spawn a new shell.[cite: 9] Because the Python binary had SUID enabled, this new shell inherited root privileges.[cite: 9]

~~~bash
python -c 'import os; os.execl("/bin/sh", "sh", "-p")'
~~~

This command successfully escalated privileges, dropping into a high-integrity root shell where `/root/root.txt` was captured, completing the compromise.[cite: 9]

## 5. Remediation
To secure this web server, the development and infrastructure teams must address the following vulnerabilities:[cite: 9]
* **Implement Strict File Upload Whitelisting:** The web application currently relies on a flawed blacklist (blocking `.php`) to prevent malicious uploads.[cite: 9] This must be replaced with a strict whitelist that only accepts necessary file types (e.g., jpg, png).[cite: 9] Furthermore, the `/uploads/` directory should be configured so that the web server cannot execute scripts within it.[cite: 9]
* **Remove SUID Permissions from Python:** Granting SUID permissions to interpreters like Python, Bash, or Perl allows any standard user to easily escalate to root.[cite: 9] The SUID bit must be removed immediately using the command `chmod -s /usr/bin/python`.[cite: 9] If specific scripts require root execution, sudo rules should be explicitly defined for those scripts rather than the entire Python binary.[cite: 9]
