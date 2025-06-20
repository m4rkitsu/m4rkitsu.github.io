
---
title: "WayWitch"
summary: "Oculto en las sombras, un aquelarre de brujas se comunica mediante símbolos arcanos, cuyos mensajes están envueltos en capas de oscuros encantamientos. Estos símbolos encantados protegen sus crípticas conversaciones, ocultando siniestros planes que amenazan con desenvolverse bajo el velo de la noche. Sin embargo, rumores sugieren que sus hechizos protectores son defectuosos, lo que permite a los forasteros forjar sus propios encantos."
categories: ["Post","Blog",]
tags: ["hackthebox challenge"]
#externalUrl: ""
#showSummary: true
draft: false
date: 2025-06-20

---

| Machine Info   |            |
| -------------- | ---------- |
| **Platform**   | HackTheBox |
| **Category**   | Web        |
| **Difficulty** | Very Easy  |

- - -

Aquí vamos de nuevo con otro Challenge White Box donde experimentaremos con **JSON Web Tokens**.

## ¿Qué es un JSON Web Token?

Los **JSON Web Tokens (JWT)** son una forma de transmitir información de forma segura entre dos partes, y se utilizan habitualmente en sistemas de autenticación. Cuando un usuario inicia sesión (o incluso visita un sitio web sin autenticación, como veremos), el servidor genera un token que contiene datos relacionados con el usuario (como un ID o un rol), lo firma con una clave secreta o privada y lo envía al cliente. El cliente incluye este token en solicitudes posteriores para comprobar su identidad.

Un JWT se compone de tres partes: el **encabezado**, la **carga útil** y la **firma**, separadas por puntos. El encabezado especifica el algoritmo utilizado para firmar el token, la carga útil contiene los datos reales (reclamos) y la firma garantiza que el token no se haya modificado.

Los JWT son compactos, seguros para URL y sin estado, lo que significa que el servidor no necesita almacenarlos; todo lo necesario está dentro del token. Sin embargo, esto también los hace vulnerables a ciertos ataques si no se protegen adecuadamente.

## Primera vulnerabilidad: Hardcoded Secrets (JWT Secret)

Al examinar el código, podemos ver que hay un **secreto** codificado, una práctica muy poco recomendable, ya que los secretos, las contraseñas y la información confidencial deberían almacenarse en **variables de entorno**.

![waywitch1 | 500](img/waywitch/waywitch1.png)

Conocer el secreto del JWT permite a un atacante falsificar un token con otro usuario en el parámetro **nombre de usuario**.

Al abrir el sitio web, vemos que se crea un JWT con nuestra información.

![waywitch2](img/waywitch/waywitch2.png)

Para analizar un JWT, podemos usar un sitio web como [jwt.io](https://jwt.io/).

![waywitch3](img/waywitch/waywitch3.png)

Como podemos ver, el cuerpo del JWT está formado por:

- **nombre de usuario**: en este caso, guest_6624
- **iat**: que significa **Emitido en**. No hay parámetro **exp** (Expiración), lo que significa que los tokens son válidos indefinidamente, otra vulnerabilidad.

Al observar el código en `index.js`, podemos ver que hay una ruta a **/tickets**, que "asegura" que el usuario **admin** sea el usuario, validándolo con el JWT.

![waywitch4 | 600](img/waywitch/waywitch4.png)

Si visitamos la página con nuestro usuario actual, recibiremos este mensaje:

![waywitch5 | 600](img/waywitch/waywitch5.png)

Podemos aprovechar esto creando un token con el **nombre de usuario** establecido como admin. Para falsificar nuestro JWT, podemos usar [jwtbuilder](http://jwtbuilder.jamiekurtz.com/). También podríamos haber usado [jwt.io](https://jwt.io/), pero no nos permitirá crear un token con el secreto `halloween-secret`, ya que está marcado como inseguro (menor de 256 bits).

## Creación de JSON Web Token Malicioso
Así es como falsifiqué el nuevo token.

![waywitch6 | 600](img/waywitch/waywitch6.png)

Reemplacemos el token con **DevTools** en la sección **Storage** y volvamos a solicitar `/tickets`.

![waywitch7 | 600](img/waywitch/waywitch7.png)
¡Listo! Logramos ver todos los tickets y, por lo tanto, la bandera.