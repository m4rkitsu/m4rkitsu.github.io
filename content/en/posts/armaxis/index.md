
---
title: "Armaxis"
summary: "A simple web challenge where we abuse and IDOR to access a privileged account, to take advantage of a LFI via Mardown Injection"
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

In this challenge we have two available ports:

- The first one it's the Web itself
- The second one it's a mail Inbox, used to receive password change tokens

Our mail is `test@email.htb`.

## First Vulnerability - Insecure Direct Object Reference

We are able to change the password of the admin account. This is made by requesting a password change token and using it against the admin account. The admin account is hardcoded in the given code, as we can see:

![[img/armaxis/armaxis1.png]]

Let's register the an account with `test@email.htb` and request a token to change the password.

![[img/armaxis/armaxis2.png]]

Let's catch the `reset-password` request with Burp

![[img/armaxis/armaxis3.png]]

There is no validation in the `email` field in correlation with the requested token, so we can change the password of a privileged account.

![[img/armaxis/armaxis4.png]]

Login as a privileged user allows us to use a new function called `Dispatch Weapon`. Let's see how we can abuse this.

This form is made to add weapons to a list. 

![[img/armaxis/armaxis8.png | 300]]

## Second Vulnerability - Local File Inclusion via Markdown / HTML Injection

The vulnerability resides in this line in the `markdown.js` file, which executes a command with the input of the `url` variable without any sanitization:

![[img/armaxis/armaxis9.png]]

Abusing markdown, we can curl inside the server to fetch files like the `flag.txt`. We could also leverage this to HTML Injection, as `<>` tags are interpreted.

![[img/armaxis/armaxis5.png]]

The flag will be embeded in the HTML and will not actually display in the content in the chart. All we have to do is click the link and open the plain-text flag.

![[img/armaxis/armaxis6.png]]

There we go!

![[img/armaxis/armaxis7.png]]