## Target: 10.81.130.75
### Recon
- ``rustscan -a 10.81.130.75 -- -A``
### Enumeration
PORT   STATE SERVICE REASON  VERSION
22/tcp open  ssh     syn-ack OpenSSH 8.2p1 Ubuntu 4ubuntu0.13 (Ubuntu Linux; protocol 2.0)
80/tcp open  http    syn-ack Golang net/http server (Go-IPFS json-rpc or InfluxDB API)
Web Findings:
- Port 80 is open and leads us to a website for "Overpass" _(Overpass allows you to securely store different passwords for every service, protected using military grade cryptography to keep you safe.)_
- `dirsearch` shows us an /admin page with a logon form.
	- Brute-forcing did not work.
### Vulnerabilities
 Looking at a guide, I learned that there is a login.js file that uses a vulnerable mechanism to validate credentials. If the response doesn't contain the text "Incorrect Credentials", then the user is assigned a SessionToken. We can inspect the /admin page and insert an arbitrary session token, then refresh the page to get access.
[_"It would first check if the response contained the string \u201cIncorrect Credentials\u201d when the page was loaded, and if not, it would set the cookie value of SessionToken with the response. This allows an attacker to create a cookie called SessionToken with any random value other than \u201cIncorrect Credentials\u201d to bypass the authentication mechanism."_](https://medium.com/@kamalabdulkadir/overpass-tryhackme-pentest-report-f8e900abdf68) 
### Exploitation
Once we get past the login form, we are greeted with a message to a user James containing an encrypted private key. We can use John the Ripper to crack the pass to the key.
``python3 /opt/john/ssh2john.py id_rsa > ssh.hash``
``john --wordlist=/usr/share/wordlists/rockyou.txt ssh.hash``
We crack the pass which is _james13_, and login.
``ssh -i id_rsa james@10.81.130.75``
**Note: ssh usernames are case sensitive**
### Post Exploitation
`user.txt` is in James' home directory.
### Privilege Escalation
By looking in the `todo.txt` file in the home directory we see a message mentioning something about a build script. They mention it is automated so it's probably a cron job.
Yep, in `/etc/crontab` we see that there is a `buildscript.sh` file that runs periodically.
Also, `/etc/hosts` is writeable at our current privilege level.
What we can do is create the file path on our local machine and host a reverse shell called `buldscript.sh` that will connect back to a waiting listener.

Boom, root.
