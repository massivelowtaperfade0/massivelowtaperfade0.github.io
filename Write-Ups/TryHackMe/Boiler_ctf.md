
---

layout: page
title: "Boiler CTF"
platform: "TryHackme"
difficulty: "Medium"
categories: [tryhackme, ctf]
tags: [enumeration, suid, gtfobins, nmap, gobuster, cyberchef]

---
### About this Challenge
**Boiler CTF** by [MrSeth6769](https://tryhackme.com/p/MrSeth6797)is a medium-difficulty TryHackMe room designed to strengthen web enumeration and Linux pentesting skills.  
The challenge focuses on identifying misconfigurations and abusing poor privilege management within the server’s operating system.

This room reinforces the importance of **thorough enumeration** before exploitation and follows a standard VAPT-style methodology.

VAPT workflow:
	- Reconnaissance
	- Enumeration
	- Exploitation
	- Initial Access
	- Privilege Escalation

> Note: This write-up intentionally includes missteps, dead ends, and corrections to reflect a real VAPT workflow and learning process.

---
### Reconnaissance

We begin with a simple NMAP scan with following flags,
 - -sS - TCP SYN (stealth) scan
 - -sV - Service Version Detection
 - -sC - Default NSE scripts
 - -Pn - Skip host discovery
 - -oN - Save in an output file (*optional*)
```bash
nmap -sS -sV -sC -Pn <target-ip> -oN scan.txt
```

![Nmap Scan](Write-Ups/Assets/images/Boiler_ctf/Nmap_Scan.png)

Taking a look at the scan results we can see that, three ports are open i.e. 21, 80 and 10000, which as per the `-sV` flag are running, one FTP and two HTTP services (one at port 80 and another at higher port 10000)

> **Key finding:**
>Anonymous FTP login is enabled. This allows unauthenticated users to access the FTP service, which may lead to sensitive information disclosure and/or unauthorized file uploads.
>By default, Nmap scans the top 1000 most common ports, which caused us to initially miss the SSH service running on a high port 55007.

---
### Enumeration

#### FTP Enumeration:
Since anonymous login is enables lets start with FTP
- To authenticate, we connect to the FTP service and log in using the username `anonymous`.
- Upon listing the directory contents, a hidden file named `.info.txt` is discovered.
- The file contains text that appears unreadable at first glance but does not resemble a complex hash.
- Based on the character distribution, the text resembles a simple ROT-based substitution cipher.
- Decoding the text using **ROT13** reveals a readable message.
- CyberChef can be used for quick decoding, but the reader is encouraged to attempt this manually.
- I tried to enumerate further, but didn't find anything further of value with FTP

```bash
ftp $IP 
Name: Anonymous
230 Login successful.
ftp> ls -la
drwxr-xr-x    2 ftp      ftp          4096 Aug 22  2019 .
drwxr-xr-x    2 ftp      ftp          4096 Aug 22  2019 ..
-rw-r--r--    1 ftp      ftp            74 Aug 21  2019 .info.txt
226 Directory send OK.
ftp> get .info.txt
ftp> exit

cat .info.txt 
Whfg jnagrq gb frr vs lbh svaq vg. Yby. Erzrzore: Rahzrengvba vf gur xrl!
```

#### Web Enumeration

Since we have two HTTP service running, i started off with the native port 80
 - The native port 80 is hosting Default Apache webpage serving static HTTP content, with nothing much to leverage
 
 ![Default Page](../../Assets/images/Boiler_ctf/Default_Page.png)
 
 - Nmap scan also show presence of `/robots.txt` but it turns out to be a rabbit hole (*Not surprising at all, as the contents of the file speaks for itself, but we still had to verify*)
 
 ![robots.txt](../../Assets/images/Boiler_ctf/robot_txt.png)
 
 - We used `gobuster` for brute-forcing directory discovery for static HTTP webpage on PORT 80
	 
 ![gobuster-v1](../../Assets/images/Boiler_ctf/Gobuster_1.png)
 
 - Results shows, it's serving two web-directories, `/joomla` and `/manual` respectively
 - The `/manual` didn't have much information for initial access, so we move to `/joomla`
 - It does look interesting, considering it's a free, open source CMS used to building and managing websites
 
 ![Joomla](../../Assets/images/Boiler_ctf/joo../..
- I ran an `gobuster` scan on `<target-id>/joomla` specifically to see it's sub-directories

```bash
gobuster dir -w /usr/share/wordlists/dirbuster/directory-list-2.3-medial.txt -u http://<target-ip>/joomla
```


![joomla_subdir_v1](../../Assets/images/Boiler_ctf/joomal_dir1.png)

- Based on the initial results, I visited each sub-directory manually and wasted a ton of my time but found nothing
- I ran another `gobuster` scan but this time with a different directory wordlist and found new directories to take a look at

```bash
gobuster dir -w /usr/share/wordlists/dirb/common.txt -u http://<target-ip>/joomla
```

- This scan resulted in new finding which are as follow:

![joomla_dirs2](../../Assets/images/Boiler_ctf/joomal_dirs2.png)

- Again we went through the sub-directories, manually searching for some new findings, and this time after examining a few sub-directories, we have our first exploitation vector

---
### Exploitation

In our last stage of Web Enumeration we found, our first exploitation vector `/_test`
- The `/_test` sub-directory, is serving `sar2html` utility, which is used to "Collect SAR Data"
- Exploit-DB confirms, this is a vulnerable utility, and attacker can leverage it for RCE (*Remote Code Execution)*

![Exploit-DB](../../Assets/images/Boiler_ctf/exploit-db.png)

- Exploit-DB provide clear and concise, steps to reproduce exploit for this vulnerable service, I encourage you to visit `exploit-db` and refer to this exploit's functionality
- Utilizing this method, we are able to perform directory listing on remote server using following crafted URL

```URL
http://<target-ip>/joomla/_text/index.php?plot;ls
```

- It successfully list contents of the remote server directory, and we see one interesting file named `log.txt`, similarly we can execute other bash command such as `pwd`, `whoami`, `cat`, etc.
- It's always recommended to use URL encoding before pushing any URL payload, again you can use `CyberChef` for URL-Encoding as well

```
http://<target-ip>/joomla/_text/index.php?plot;cat%20log.txt
```

- We examines the contents of the interesting looking file, and it reveals some login credentials for one of the users on the remote server

![log_txt](../../Assets/images/Boiler_ctf/log_txt.png)

---
### Initial Access

- We use this credentials to login via SSH  as a legitimate user. (*Note... SSH is running on port 55007 as per earlier findings*)

```bash
ssh -p 55007 [REDACTED]@<target-ip>
```

- Provide the *password* found in the file `log.txt` for that user, and we have an active SSH connection with the remote server, verify it with `whoami`

![user1](../../Assets/images/Boiler_ctf/user_1.png)

- Listing contents of the default home directory for the logged in user, we see an BASH script

![user2](../../Assets/images/Boiler_ctf/user_2.png)

- The contents of the file reveals yet another users credentials, so now we have credentials for two active users
- Before switching to another user, it's always recommended to check privileges and permission of the current log in system, again in encourage you to perform following commands on the current user and see for yourself
	- `whoami`
	- `id`
	- `sudo -l`
	- `ls -la`
	- `ls /root`
- Now that we have credentials for the other users, we can use `su` command and provide found name and password (without '#'), and switch to that user to examine contents to it's home directory
- Switching to the other users, and navigating to it's home directory, we can see some hidden files/folders
- Reading contents of a hidden file is similar to that of a normal file

```bash
cat .secret
```

![secret](../../Assets/images/Boiler_ctf/secret.png)

- And just like that can have access to our first sensitive information on the target system
- ( *Since i wasn't aware that the `.secret` file was the `user.txt` flag, i ran a command to search for it, only after a solid 10 minutes when i couldn't find it, that I saw that the content of `.secret` matched the placeholder for `uset.txt` on TryHackMe* )

---
### Privilege Escalation

In attempt to examining privilege and permission of newly found user, again we run

```bash
sudo -l
```

And we got trolled again,

Since we didn't have any `sudo` permission, our next resort is to find what could be used with `sudo/root`  privileges, without having to be a `sudoer` to do so.

We look for command with [SUID](https://www.hackingarticles.in/linux-privilege-escalation-using-suid-binaries/) bit set, so we utilize `find` command to search for them

```bash
find / -perm -4000 -type f 2>/dev/null
```

The above command can be broken into:
- `find` in `/` (root) directory, with `-perm`( permission ) set to `-4000` (SUID Bit enabled) and the `type` to be `f`(file) and redirect all the error to `2>/dev/null`(void) directory instead of printing it on console
- Refer to `man find` for more details

![find](../../Assets/images/Boiler_ctf/find.png)

The `find` command is marked with an SUID bit to be executed as root
What This means is that, `find` command can be used with the privileges that of a root user and can be utilized to read and write contents of a file and also to spawn a bash shell with ROOT user privileges
To know more about Privilege Escalation, File Read, File Write, SUDO, Shell, etc. using various commands, refer to [GTFOBins](https://gtfobins.org/) 
We see `find` is listed on [GTFOBins](https://gtfobins.org/), we utilize it to spawn a `bash` shell as ROOT

![gtfobins](../../Assets/images/Boiler_ctf/gtfobins.png)

Upon executing command shown in the above image we get a shell as root

![root_txt](../../Assets/images/Boiler_ctf/root_txt.png)

Since i know, where the `root.txt` was in the `/root` directory, i was able to read it's content without having to look for it's path
But since we have SUID enable `find` command, we are able to list the contents for `/root` directory without any `sudo` privileges, as shown in the above command

---

### Impact

Threat Vector : sar2html
Privilege Vector : find (SUID)
Information Leak : Plain text password in unprotected files

- Attacker can read and write files via anonymous FTP access.

- Attacker can achieve RCE through vulnerable `sar2html` deployment.

- Attacker can escalate privileges via misconfigured SUID `find` binary.

- Sensitive credentials stored in plaintext enable lateral movement.

##### Security Strengths

Webmin panel is safer and follows SSL certificate and encryptions standard, running on HTTPS(secure alternative as to HTTP)
SSH requires password authentication (Prone to Brute-Forcing, but still some authentication is implemented)

---
### Notes & Reflection

- Missed SSH initially due to relying on default Nmap scan range.
- Spent excessive time manually enumerating Joomla directories before changing wordlists.
- Initially failed to recognize `.secret` as the user flag.
- Reinforced the importance of:
  - Full port scans
  - Wordlist selection
  - Checking SUID binaries early

---
##### With this we can conclude our completion of Boiler CTF room on `TryHackMe` 
I also want to point out that, if you were stuck, it's okay, we all learn somewhere, take this write-up as a way of learning something new, the key to enumeration is to look, leverage, and level up