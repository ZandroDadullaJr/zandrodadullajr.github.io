---
title: "TryHackMe - Investigating Windows Walkthrough"
date: 2025-09-09 00:00:00 +0800
categories: [Writeups, TryHackMe]
tags: [tryhackme,blueteam,easy,ctf,dfir]
pin: false
image:
  path: /assets/img/thm-investigating-windows/immo-wegmann-OIUpXdhfJ1w-unsplash.jpg
  alt: Photo by <a href="https://unsplash.com/@tinkerman?utm_content=creditCopyText&utm_medium=referral&utm_source=unsplash">Immo Wegmann</a> on <a href="https://unsplash.com/photos/a-person-holding-a-pencil-and-a-broken-laptop-OIUpXdhfJ1w?utm_content=creditCopyText&utm_medium=referral&utm_source=unsplash">Unsplash</a>
      
---
Room: [Investigating Windows](https://tryhackme.com/room/investigatingwindows)<br>
Author: [tryhackme](https://tryhackme.com/p/tryhackme), [umairalizafar](https://tryhackme.com/p/umairalizafar), [Aashir.Masood](https://tryhackme.com/p/Aashir.Masood)<br>
Difficulty: <span style="color: #03b303"> **Easy** </span>

The **Investigating Windows** room on TryHackMe is designed to teach essential blue team skills in analyzing and investigating a Windows system after suspicious activity. As defenders, it’s critical to understand where to look for evidence, how to interpret Windows artifacts, and how to uncover potential persistence mechanisms left behind by an attacker.  

In this walkthrough, we’ll be covering two approaches:  

- **GUI-based investigation** using built-in Windows tools like Event Viewer and Task Scheduler  
- **Command-line investigation** leveraging tools such as PowerShell and CMD to gather insights quickly and effectively  

By the end of this write-up, you’ll gain hands-on experience with:  

- Reviewing and analyzing **Windows Event Logs** for signs of malicious activity  
- Identifying **persistence mechanisms** commonly abused by attackers  
- Using the **command line** to perform forensic investigation and threat hunting  

This room is an excellent opportunity to build foundational **SOC and incident response skills** that directly apply to real-world investigations.  

## Connecting to the Machine
If you're using Linux, you can connect to the machine using `xfreerdp`

```bash
xfreerdp /v:10.201.73.18 /u:Administrator /p:letmein123! /dynamic-resolution +clipboard
```

If you're using Windows, you can connect using Remote Desktop. Open the Windows Run dialog box by pressing `Windows Key + R` and type `mstsc` and hit enter. Enter the machine credentials and connect.

## Question 1: Whats the version and year of the windows machine?

### GUI
Open the Windows Run dialog box by pressing `Windows Key + R` and type `winver`

![image.png](/assets/img/thm-investigating-windows/q1-gui.png)

### Command-line
Use the `Get-ComputerInfo` cmdlet in PowerShell and look for the `OSName` object.

```powershell
Get-ComputerInfo | Select-Object OSName
```
![image.png](/assets/img/thm-investigating-windows/q1-cli.png)

### Flag
<details> 
  <summary><b>Question 1 Flag</b></summary>
   Windows Server 2016 
</details>

## Question 2: Which user logged in last?
### GUI

Open the Windows Run dialog box by pressing `Windows Key + R` and type `eventvwr` or search for `Event Viewer` in the Windows search bar. Once Event Viewer is open, go to `Windows Logs > Security`, click the `Filter Current Log` and enter the event ID `4624`. The event ID 4624 is generated when a logon session is created.

![image.png](/assets/img/thm-investigating-windows/q2-gui-1.png)

Once the logs are filtered to the correct event ID, look for the last user logon that is not `SYSTEM`

![image.png](/assets/img/thm-investigating-windows/q2-gui-2.png)

### Command-line
Using PowerShell, we will be filtering logs using the command below. The `Get-WinEvent` cmdlet is used to pull Windows logs entries and looks for logs under the Security log and filters those logs for event ID 4624. It will then list the last 5 entries and gets the value of its 5th index to show the names of the logon account.

```powershell
Get-WinEvent -FilterHashtable @{LogName='Security'; Id=4624} | Select-Object -First 5 | Select-Object {$_.Properties[5].Value} | Format-List
```
![image.png](/assets/img/thm-investigating-windows/q2-cli.png)

### Flag
<details> 
  <summary><b>Question 2 Flag</b></summary>
   Administrator
</details>

## Question 3: When did John log onto the system last?
### GUI
Still on the Event Viewer and filtered to event ID 4624, click Find on the Actions section and enter "*John*". It will show you the last successful sign-in of the account John.

![image.png](/assets/img/thm-investigating-windows/q3-gui-1.png)

### Command-line
For the command-line method, we will be using 3 methods. The first is by using the Net command which is the quickest way to get the last logon date. The other 2 methods uses the `Get-WinEvent` cmdlet but uses different filtering methods. 

The Filterhashtable is the one we're familiar with and it utilizes a PowerShell hash table, which is a collection of key-value pairs. The XPath Query is a query language for selecting data/elements from an XML document. Filters are applied to the structured XML representation of event data, allowing for more complex and granular selections based on the XML hierarchy and content.

Using Net command
```powershell
net user john
```
![image.png](/assets/img/thm-investigating-windows/q3-cli-1.png)

Using Get-WinEvent FilterHashtable
```powershell
Get-WinEvent -FilterHashtable @{LogName='Security'; ID=4624} | Where-Object { $_.Properties.Value -contains "John" }
```

![image.png](/assets/img/thm-investigating-windows/q3-cli-2.png)

Using Get-WinEvent XPath Query
```powershell
Get-WinEvent -LogName Security -FilterXPath "*[System[(EventID=4624)]] and *[EventData[Data[@Name='TargetUserName']='John']]" | Select-Object TimeCreated, Id, Message
```

![image.png](/assets/img/thm-investigating-windows/q3-cli-3.png)

### Flag
<details> 
  <summary><b>Question 3 Flag</b></summary>
   03/02/2019 5:48:32 PM
</details>

## Question 4: What IP does the system connect to when it first starts?
For this question, it already gives us a clue on where to look. The question suggests that we need to look for startup programs. In Windows, Startup programs are primarily found in the Startup folders (user-specific and system-wide) and the Windows Registry.

### GUI
For the purposes of this write-up, we already looked into the startup folders using `shell:startup` and `shell:common startup` but nothing interesting pops up. We then look into the Registry editor for the startup programs. In the Run dialog, enter `regedit` to open Registry editor. Registry run keys can be found in the following location.

```text
HKEY_LOCAL_MACHINE\Software\Microsoft\Windows\CurrentVersion\Run
HKEY_LOCAL_MACHINE\Software\Microsoft\Windows\CurrentVersion\RunOnce
HKEY_CURRENT_USER\Software\Microsoft\Windows\CurrentVersion\Run
HKEY_CURRENT_USER\Software\Microsoft\Windows\CurrentVersion\RunOnce
```
![image.png](/assets/img/thm-investigating-windows/q4-gui.png)

### Command-line
For the command-line approach, we will be using the `Get-WmiObject` cmdlet.

```powershell
Get-WmiObject Win32_StartupCommand | Select *
Get-WmiObject Win32_StartupCommand | Select Name, Command, Location | Format-List
```
![image.png](/assets/img/thm-investigating-windows/q4-cli.png)

### Flag
<details> 
  <summary><b>Question 4 Flag</b></summary>
   10.34.2.3
</details>

## Question 5: What two accounts had administrative privileges (other than the Administrator user)?
### GUI
For the GUI approach, we will be using the Local Users and Group console. To open the console, enter the keyword `lusrmgr.msc` in the Run dialog. Go to the `Groups` folder and open the `Administrators` group.

![image.png](/assets/img/thm-investigating-windows/q5-gui.png)

### Command-line

```powershell
net localgroup administrators
```
![image.png](/assets/img/thm-investigating-windows/q5-cli.png)

### Flag
<details> 
  <summary><b>Question 5 Flag</b></summary>
   Guest, Jenny
</details>

## Question 6: Whats the name of the scheduled task that is malicous.
We will be looking into the Task Scheduler for persistence mechanisms. 

### GUI
To open Task Scheduler console, open the Run dialog and enter `taskschd.msc`. When we reviewed the scheduled tasks there's actually 2 malicious and 1 suspicious tasks enabled. 

**Malicious**: Clean file system, GameOver<br>
**Suspicious**: falshupdate22

![image.png](/assets/img/thm-investigating-windows/q6-gui-1.png)

The **Clean file system** scheduled task is malicious as it runs a PowerShell version of netcat (`nc.ps1`), listening on port 1348 (`-l 1348`)

![image.png](/assets/img/thm-investigating-windows/q6-gui-2.png)

The **falshupdate22** scheduled task is suspicious to me as it runs a hidden PowerShell. However, the command portion (`-c`) is empty making it benign.

![image.png](/assets/img/thm-investigating-windows/q6-gui-3.png)

The **GameOver** scheduled task is malicious as it runs mimikatz to dump passwords from LSASS. 

![image.png](/assets/img/thm-investigating-windows/q6-gui-4.png)

We can also check the script on the **Clean file system** scheduled task. We can confirm that the script is called **powercat** which is a PowerShell version of Netcat.

![image.png](/assets/img/thm-investigating-windows/q6-gui-5.png)

Since TryHackMe only accepts a task name with 3 words, we will be focusing on the **Clean file system** scheduled task. Also take note when was the tasks created as this will helpful later on.

### Command-line
For the command-line approach, we will be using the `Get-ScheduledTask` PowerShell cmdlet.

```powershell
Get-ScheduledTask -TaskPath "\" | Select-Object Taskname, Description, Author, Date, State
Get-ScheduledTask -TaskName "Clean file system" | Select-Object -ExpandProperty Actions
```
![image.png](/assets/img/thm-investigating-windows/q6-cli.png)

### Flag
<details> 
  <summary><b>Question 6 Flag</b></summary>
   Clean file system
</details>

## Question 7: What file was the task trying to run daily?
As we discovered from the previous question, the file is located in `C:\TMP` directory. See the Flag for the file name.

### Flag
<details> 
  <summary><b>Question 7 Flag</b></summary>
  nc.ps1
</details>

## Question 8: What port did this file listen locally for?
As we discovered from question 6, the scheduled task is running a PowerShell version of netcat listening to a port. The `-l` flag enables listening mode, where it listens/waits for connections on the specified port. See the flag for the port number.

### Flag
<details> 
  <summary><b>Question 8 Flag</b></summary>
  1348
</details>

## Question 9: When did Jenny last logon?
### GUI
This is similar to Question 3. Unfortunately, no logs were found which suggests that the user never logged in or the logs for the user have been removed. We can consult the command-line for more info.

![image.png](/assets/img/thm-investigating-windows/q9-gui.png)

### Command-line
Using the Net command, we can see that Jenny never logged in.

```powershell
net user jenny
```

![image.png](/assets/img/thm-investigating-windows/q9-cli.png)

### Flag
<details> 
  <summary><b>Question 9 Flag</b></summary>
   Never
</details>

## Question 10: At what date did the compromise take place?
### GUI
Looking at the date modified/date created property of the malicious file we found earlier on `C:\TMP`, we can assume that the compromise took place on that date. The date created on the malicious tasks are on the same date as well.

![image.png](/assets/img/thm-investigating-windows/q10-gui-1.png)

### Command-line

```powershell
Get-ChildItem C:\TMP
```

![image.png](/assets/img/thm-investigating-windows/q10-cli.png)

### Flag
<details> 
  <summary><b>Question 10 Flag</b></summary>
   03/02/2019
</details>

## Question 11: During the compromise, at what time did Windows first assign special privileges to a new logon?
### GUI
Some helpful clues here are the timeline of the compromise and the "*assigning special priveleges to a new logon*". It is likely that we will need to review the event logs for this so we need to search what Event ID we will be using. Searching for the clue "*assigning special priveleges to a new logon Event ID*" online will give us the Event ID [4672](https://learn.microsoft.com/en-us/previous-versions/windows/it-pro/windows-10/security/threat-protection/auditing/event-4672).

![image.png](/assets/img/thm-investigating-windows/q11-gui.png)

Next is the timeframe of the compromise. Recalling the previous questions, we have the creation date of the malicious scheduled task (2019-03-02 4:56:16 PM) and the earliest modified date of the files in the TMP directory (2019-03-02 4:37 PM). We now have a timeframe we can use to filter the event logs.

With all these infromation, we can filter for the event ID `4672` from `2019-03-02 4:37 PM` to `2019-03-02 4:56: PM`.

Unfortunately that did not worked as TryHackMe is looking for a different time based on the hint provided (00/00/0000 0:00:49 PM) 😅. I adjusted the timeframe to cover the whole day of the compromise.

![image.png](/assets/img/thm-investigating-windows/q11-gui-2.png)

![image.png](/assets/img/thm-investigating-windows/q11-gui-3.png)

### Command-line
For the command-line, we will be using again the `Get-WinEvent` PowerShell cmdlet with the StartTime and EndTime parameters.

```powershell
Get-WinEvent -FilterHashTable @{LogName='Security'; 'Id'= '4624'; 'StartTime' = Get-Date "2019-03-02 00:00:00"; 'EndTime' = Get-Date "2019-03-02 23:59:59"; }
```
![image.png](/assets/img/thm-investigating-windows/q11-cli.png)

### Flag
<details> 
  <summary><b>Question 11 Flag</b></summary>
   03/02/2019 4:04:49 PM
</details>

## Question 12: What tool was used to get Windows passwords?
### GUI
Recalling when we investigated the Scheduled task on Question 6 and we found another malicious task named `GameOver` where the action properties points to a file called `mim.exe` and the parameters are similar to the parameters used in mimikatz. Mimikatz is a tool used to extract passwords from memory and applications. The action in the scheduled task runs the mimikatz function to dump all credentials stored in the LSASS and saves it in the text file `C:\TMP\o.txt`.

There's no `o.txt` in the `C:\TMP` folder but there's `mim-out.txt`. Checking its content, we confirmed that it's mimikatz.

![image.png](/assets/img/thm-investigating-windows/q12-gui.png)

### Command-line
We can also verify this by getting the file hash and checking it against [Virus Total](https://www.virustotal.com/gui/file/f8f1c210a8c863efc0f6b8ac3553030a14a702ce8cf573cb5e9cd58f70c7c622).

```powershell
get-filehash .\mim.exe | Format-List
```

![image.png](/assets/img/thm-investigating-windows/q12-cli.png)
![image.png](/assets/img/thm-investigating-windows/q12-cli-2.png)

### Flag
<details> 
  <summary><b>Question 12 Flag</b></summary>
   Mimikatz
</details>

## Question 13: What was the attackers external control and command servers IP?
This question took me some time to answer. Tried looking into the netstat for any TCP/IP connections but none were found that matches the octets in the answer textbox. I eventually discovered that we can also look into the host file for any changes or DNS cache poisoning that points to the threat actor's C2 servers.

### GUI
Navigate to `C:\Windows\System32\drivers\etc\hosts`

![image.png](/assets/img/thm-investigating-windows/q13-gui.png)


### Command-line
We can also use the command-line to view the contents of a file.

```powershell
type C:\Windows\System32\drivers\etc\hosts
```
![image.png](/assets/img/thm-investigating-windows/q13-cli.png)


### Flag
<details> 
  <summary><b>Question 13 Flag</b></summary>
   76.32.97.132
</details>

## Question 14: What was the extension name of the shell uploaded via the servers website?
This also took me some time as I did not see the IIS running on the server. For Windows, the primary file directory for the default website in IIS is typically located at `C:\inetpub\wwwroot`. This is the root folder where web content for the default website is stored.

### GUI
Navigate to `C:\inetpub\wwwroot`

![image.png](/assets/img/thm-investigating-windows/q14-gui.png)

### Command-line
Similar to the command-line approach on Question 10, we can use the Get-ChildItem PowerShell cmdlet. 

```powershell
Get-ChildItem C:\inetpub\wwwroot
```

![image.png](/assets/img/thm-investigating-windows/q14-cli.png)

### Flag
<details> 
  <summary><b>Question 14 Flag</b></summary>
   .jsp
</details>

## Question 15: What was the last port the attacker opened?
Reviewing the `netstat` or the `Get-NetTCPConnection` cmdlet did not yield the answer that TryHackMe wanted. Also, Windows does not log *new firewall rule addded* events by default ([Event ID 4946](https://learn.microsoft.com/en-us/previous-versions/windows/it-pro/windows-10/security/threat-protection/auditing/event-4946)). Instead, we went to the Windows Firewall to manually review the rules.

### GUI
To open the Windows Firewall with Advanced Security console, enter `wf.msc` in the Run Dialog. Go to Inbound Rules and double click the top most rule. New rules are typically added to the top of the list. Go to Protocols and ports and check the Local port. Alternatively, you can also see the local port in the Inbound rules table under the Local port column.

![image.png](/assets/img/thm-investigating-windows/q15-gui.png)

### Command-line
With the help of ChatGPT, we were able to create a PowerShell one-liner that lists all inbound firewall rules and their ports. Unfortunately, using this method alone doesn't show the last port that was opened.

```powershell
Get-NetFirewallRule -Direction Inbound | ForEach-Object { [PSCustomObject]@{RuleName=$_.DisplayName; Port=(Get-NetFirewallPortFilter -AssociatedNetFirewallRule $_).LocalPort -join ", " } } | Format-Table -AutoSize
```
![image.png](/assets/img/thm-investigating-windows/q15-cli.png)

### Flag
<details> 
  <summary><b>Question 15 Flag</b></summary>
   1337
</details>

## Question 16: Check for DNS poisoning, what site was targeted?
This question can be answered by recalling the investigation on Question 13. The threat actors were able to perform DNS cache poisoning by modifying the host file.

### Flag
<details> 
  <summary><b>Question 16 Flag</b></summary>
   google.com
</details>
<br>
Thank you for reading my write-up! I hope you learned a thing or two on how to investigate a compromised Windows machine using GUI or command-line approach.

## References
Getting OS Properties: [https://learn.microsoft.com/en-us/powershell/module/microsoft.powershell.management/get-computerinfo?view=powershell-7.5](https://learn.microsoft.com/en-us/powershell/module/microsoft.powershell.management/get-computerinfo?view=powershell-7.5)

Event ID 4624: [https://learn.microsoft.com/en-us/previous-versions/windows/it-pro/windows-10/security/threat-protection/auditing/event-4624](https://learn.microsoft.com/en-us/previous-versions/windows/it-pro/windows-10/security/threat-protection/auditing/event-4624)

Get-WinEvent: [https://learn.microsoft.com/en-us/powershell/module/microsoft.powershell.diagnostics/get-winevent?view=powershell-7.5](https://learn.microsoft.com/en-us/powershell/module/microsoft.powershell.diagnostics/get-winevent?view=powershell-7.5)

FilterHashTable: [https://learn.microsoft.com/en-us/powershell/scripting/samples/creating-get-winevent-queries-with-filterhashtable?view=powershell-7.5](https://learn.microsoft.com/en-us/powershell/scripting/samples/creating-get-winevent-queries-with-filterhashtable?view=powershell-7.5)

XPath Query: [https://devblogs.microsoft.com/scripting/understanding-xml-and-xpath/](https://devblogs.microsoft.com/scripting/understanding-xml-and-xpath/)

Startup Applications: [https://support.microsoft.com/en-us/windows/configure-startup-applications-in-windows-115a420a-0bff-4a6f-90e0-1934c844e473](https://support.microsoft.com/en-us/windows/configure-startup-applications-in-windows-115a420a-0bff-4a6f-90e0-1934c844e473)

Event ID 4672: [https://learn.microsoft.com/en-us/previous-versions/windows/it-pro/windows-10/security/threat-protection/auditing/event-4672](https://learn.microsoft.com/en-us/previous-versions/windows/it-pro/windows-10/security/threat-protection/auditing/event-4672)

Mimikatz: [https://www.varonis.com/blog/what-is-mimikatz](https://www.varonis.com/blog/what-is-mimikatz)

Event ID 4946: [https://learn.microsoft.com/en-us/previous-versions/windows/it-pro/windows-10/security/threat-protection/auditing/event-4946](https://learn.microsoft.com/en-us/previous-versions/windows/it-pro/windows-10/security/threat-protection/auditing/event-4946)

Windows Firewall Rule Added: [https://what2log.com/windows/logs/WFRuleAdd/](https://what2log.com/windows/logs/WFRuleAdd/)