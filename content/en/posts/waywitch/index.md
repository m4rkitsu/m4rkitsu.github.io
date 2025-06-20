
---
title: "WayWitch"
summary: "Very Easy Web Challenge. Based on abusing some bad practices when using JSON Web Tokens."
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

Here we go again with another White-Box Challenge where we will mess around with **JSON Web Tokens**.

## What is a JSON Web Token?

**JSON Web Tokens (JWT)** are a way to securely transmit information between two parties, typically used in authentication systems. When a user logs in (or even visit a website without authentication, as we will see), the server generates a token that contains user-related data (like an ID or role), signs it with a secret or private key, and sends it to the client. The client then includes this token in subsequent requests to prove their identity.


A JWT is composed of three parts: the **header**, the **payload**, and the **signature**, separated by dots. The header specifies the algorithm used to sign the token, the payload carries the actual data (claims), and the signature ensures the token hasn’t been modified.


JWTs are compact, URL-safe, and stateless, meaning the server doesn’t need to store them—everything needed is inside the token. However, this also makes them sensitive to certain attacks if not properly secured.

## First Vulnerability: Hardcoded Secrets (JWT Secret)

As we take a look at the code, we can see that there is a hardcoded **secret**, a really bad practice, as . Secrets, passwords and sentive information should be should be stored in **environment variables**.

![waywitch1 | 500](img/waywitch/waywitch1.png)

Knowing the secret of the JWT allows an attacker to forge a token with another user in the **username** parameter.

When opening the website, we see that a JWT is created with our information.

![waywitch2](img/waywitch/waywitch2.png)

To disect a JWT we can use a website like [jwt.io](https://jwt.io/).

![waywitch3](img/waywitch/waywitch3.png)

As we can see, the body of the JWT is formed by:

- **username**: in this case, guest_6624
- **iat**: which stands for **Issued At**. There is no **exp** (Expiration) parameter, which means that the tokens are valid forever, another vulnerability.

Taking a look at the code in `index.js`, we can see that there is a route to **/tickets**, which "ensures" that the user **admin** user, validating this with the JWT.

![waywitch4 | 600](img/waywitch/waywitch4.png)

If we visit the page with our current user, we will get this message:

![waywitch5 | 600](img/waywitch/waywitch5.png)


We can abuse this by creating a token with the **username** set to admin. To forge our JWT we can use [jwtbuilder](http://jwtbuilder.jamiekurtz.com/). We could have also used [jwt.io](https://jwt.io/) but it won't allow us to create a token with the secret `halloween-secret`, as it is flagged as insecure (shorter that 256 bits).

## Forging Malicious JSON Web Token
This is how I forged the new token.

![waywitch6 | 600](img/waywitch/waywitch6.png)

Let's replace the token with the **DevTools** in the **Storage** section and request to `/tickets` again.

![waywitch7 | 600](img/waywitch/waywitch7.png)
And there we go! We managed to see all tickets and therefore, the flag. 