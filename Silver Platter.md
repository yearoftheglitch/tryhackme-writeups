## Target: 10.66.179.5
### Recon
- Nmap: nmap -A 10.66.179.5 -p- -oA nmap
- Banners:
- Web Enumeration:
### Enumeration
- Open Ports:
	- **22**/tcp, **ssh**
	- **80**/tcp, **http**, nginx/1.18.0 (Ubuntu)
	- **8080**/tcp, **http-proxy**
- Web Findings:
	- On the contact page of the port 80 site, there are a couple of clues.
		- Potential username _scr1ptkittie_
		- Potential platform/tool to exploit _Silverpeas_
### Vulnerabilities
- Silverpeas has a vulnerability that allows login bypass. We can execute this by intercepting a login request with Burp's proxy and removing the password field in the request.
- Silverpeas also has a vulnerability that allows us to read all messages by visiting a certain URL: http://[Target_IP]:8080/silverpeas/RSILVERMAIL/jsp/ReadMessage.jsp?ID=[messageID]
### Exploitation
- To log in as admin, we exploited the authentication bypass vulnerability and logged in as SilverAdmin.
- We then visited the vulnerable message URL and incremented the message number until we discover an SSH username and password.
- Sure enough, we can log in with credentials _tim_:_THM{c4ca4238a0b923820dcc509a6f75849b}_
### Post Exploitation
- Was not able to run ``sudo -l`` as we do not know user tim's password. 
- ``cat /etc/crontab`` did not produce fruit.
- Nothing stuck out from ``find / -perm -u=s -type f 2>/dev/null``
- A look at ``/etc/passwd`` shows a user named tyler with some sort of root privs.
- What did prove interesting was a simple ``id`` command, which produced ``uid=1001(tim) gid=1001(tim) groups=1001(tim),4(adm)``. the _adm_ apparently indicates that tim has read-access to the log files on the system.
	- ``cat /var/log/auth* | grep -i pass`` - This shows us anything in the logs that includes the term _pass_.
	- Sure enough, we find that _"_Zd_zx7N823/"_ is tyler's password.
### Privilege Escalation
- We are now able to login to tyler's account.
- A ``sudo -l`` reveals that tyler has unrestricted sudo privileges.
- A quick look at GTFOBins shows us that we can run ``sudo sudo /bin/sh`` to get root.
- We do so, and it works.
![[Screenshot 2026-01-03 at 6.07.37 PM.png]]

### Flags
- User: _THM{c4ca4238a0b923820dcc509a6f75849b}_
- Root: _THM{098f6bcd4621d373cade4e832627b4f6}_
### Notes
- Key takeaways:
	- Don't skim. Being careless will lead to missing clues. 
	- ``/etc/passwd`` & ``id`` need to be religiously checked, make it a priority to fully memorize the components of the passwd file. 
