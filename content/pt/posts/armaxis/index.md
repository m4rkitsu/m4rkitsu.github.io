
---
title: "Armaxis"
summary: "Un desafío web simple en el que abusamos de un IDOR para acceder a una cuenta privilegiada, para aprovechar un LFI mediante inyección de Mardown"
categories: ["Post","Blog",]
tags: ["hackthebox challenge"]
#externalUrl: ""
#showSummary: true
draft: false
date: 2025-06-07

---

| Machine Info   |            |
| -------------- | ---------- |
| **Platform**   | HackTheBox |
| **Category**   | Web        |
| **Difficulty** | Very Easy  |

- - -

En este desafío, tenemos dos puertos disponibles:

- El primero es la propia web.
- El segundo es una bandeja de entrada de correo, utilizada para recibir tokens de cambio de contraseña.

Nuestro correo es `test@email.htb`.

## Primera vulnerabilidad: Insecure Direct Object Reference

Podemos cambiar la contraseña de la cuenta de administrador. Esto se logra solicitando un token de cambio de contraseña y usándolo en la cuenta de administrador. La cuenta de administrador está codificada en el código proporcionado, como podemos ver:

![armaxis1](img/armaxis/armaxis1.png)

Registremos la cuenta con `test@email.htb` y solicitemos un token para cambiar la contraseña.

![armaxis2](img/armaxis/armaxis2.png)

Capturemos la solicitud de restablecimiento de contraseña con Burp.

![armaxis3](img/armaxis/armaxis3.png)

No hay validación en el campo de correo electrónico en relación con el token solicitado, por lo que podemos cambiar la contraseña de una cuenta con privilegios.

![armaxis4](img/armaxis/armaxis4.png)

Iniciar sesión como usuario con privilegios nos permite usar una nueva función llamada "Enviar arma". Veamos cómo podemos aprovecharla.

Este formulario está diseñado para agregar armas a una lista.

![armaxis8](img/armaxis/armaxis8.png)

## Segunda vulnerabilidad:  Local File Inclusion via Markdown / HTML Injection

La vulnerabilidad reside en esta línea del archivo `markdown.js`, que ejecuta un comando con la variable `url` sin ninguna corrección:

![armaxis9](img/armaxis/armaxis9.png)

Al abusar de Markdown, podemos usar curl dentro del servidor para obtener archivos como `flag.txt`. También podríamos aprovechar esto para la inyección HTML, ya que se interpretan las etiquetas `<>`.

![armaxis5](img/armaxis/armaxis5.png)

La bandera se incrustará en el HTML y no se mostrará en el contenido del gráfico. Solo tenemos que hacer clic en el enlace para abrir la bandera en texto plano.

![armaxis6](img/armaxis/armaxis6.png)

¡Listo!

![armaxis7](img/armaxis/armaxis7.png)