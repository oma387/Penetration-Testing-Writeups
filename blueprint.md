# TryHackMe Write-Up: Blueprint[cite: 5]

**Scope:** Controlled TryHackMe lab environment[cite: 5]  
**Assessment Outcome:** Full compromise to `NT AUTHORITY\SYSTEM`[cite: 5]

## 1. Executive Summary
During the assessment of the Blueprint machine, several exposed Windows services were identified.[cite: 5] The primary attack path was the web application running on port 8080, where an osCommerce 2.3.4 installation exposed useful application functionality and a vulnerable file-upload path.[cite: 5] A PHP payload was uploaded using a PHP-compatible extension and executed to obtain a reverse shell.[cite: 5] The resulting shell ran with SYSTEM privileges, allowing credential extraction and recovery of the Lab user password.[cite: 5]

## 2. Reconnaissance
A full TCP scan with default scripts and version detection was used to map the attack surface.[cite: 5]

~~~bash
nmap -sC -sV -p- <TARGET_IP>
~~~

Important results included IIS on port 80, SMB on 139/445, MySQL on 3306, and Apache/PHP on port 8080.[cite: 5] The initial SMB checks did not produce a useful foothold, so attention moved to the reachable web application on 8080.[cite: 5]

## 3. Web Enumeration
Port 80 returned an IIS response without a useful application surface.[cite: 5] Port 8080 exposed the osCommerce application and its documentation, providing the key technology fingerprint:[cite: 5]
* **URL:** `http://<TARGET_IP>:8080`[cite: 5]
* **Application:** `osCommerce 2.3.4`[cite: 5]

Directory enumeration was used to discover these additional application paths and exposed resources:[cite: 5]

~~~bash
gobuster dir -u http://<TARGET_IP>:8080 -w /usr/share/wordlists/dirb/common.txt
~~~

## 4. Foothold
Through the discovered osCommerce application paths, a vulnerable file-upload feature was identified.[cite: 5] The web application accepted PHP-family extensions such as `.php4` and `.php5`.[cite: 5] This was treated as a potential server-side code-execution path.[cite: 5] 

To bypass standard filters, a PHP reverse-shell payload was uploaded using one of these accepted PHP-compatible extensions.[cite: 5] The validation sequence confirmed the uploaded file was stored, reachable, and actually executed by the server.[cite: 5]

Listener setup on the attacker machine:[cite: 5]
~~~bash
nc -lvnp 4444
~~~

When the uploaded payload was triggered, the target connected back to the Netcat listener.[cite: 5] After receiving the reverse shell, the current security context was verified:[cite: 5]
~~~cmd
whoami
~~~
**Output:** `nt authority\system`[cite: 5]

The shell was already running as `NT AUTHORITY\SYSTEM`, so no additional privilege-escalation exploit was required.[cite: 5]

## 5. Post-Exploitation & Password Recovery
With SYSTEM access, local account credential material could be collected.[cite: 5] Using a Windows credential-dumping method such as Meterpreter `hashdump` or SAM/SYSTEM hive extraction revealed the Lab account NTLM hash:[cite: 5]
~~~text
Lab NTLM: 30e87bf999828446a1c1209ddde4c450
~~~
The NTLM hash was cracked offline, recovering the Lab account password: `googleplus`[cite: 5]

## 6. Flags
The Administrator desktop contained the root flag:[cite: 5]
~~~cmd
type C:\Users\Administrator\Desktop\root.txt.txt
~~~
**Output:** `THM{aeale3ce6fe7f89e10cea833ae009bee}`[cite: 5]

## 7. Attack Chain
* Nmap reconnaissance[cite: 5]
* Identify web application on 8080[cite: 5]
* osCommerce 2.3.4 enumeration[cite: 5]
* Validate PHP-compatible file upload[cite: 5]
* Upload PHP reverse-shell payload[cite: 5]
* Reverse shell[cite: 5]
* `NT AUTHORITY\SYSTEM`[cite: 5]
* Extract Lab NTLM hash[cite: 5]
* Crack hash (`googleplus`)[cite: 5]
* Recover root.txt[cite: 5]

## 8. Remediation
* **Patch the Application:** Upgrade or replace vulnerable osCommerce versions and remove obsolete components.[cite: 5]
* **Secure File Uploads:** Allow only required extensions, validate server-side, rename uploads, store them outside executable web directories, and disable script execution in upload locations.[cite: 5]
* **Remove Installation Files:** Delete installer and documentation paths after deployment.[cite: 5]
* **Least Privilege:** Restrict unnecessary SMB/MySQL access and apply least-privilege controls.[cite: 5]
