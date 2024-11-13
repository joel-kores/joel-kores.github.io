---
title: Paper walkthrough - HTB
author: Kores
date: 2022-05-25 21:18:14 +0300
categories: [Cybersec, HTB]
tags: [Boot2Root]
render_with_liquid: false
---

# Paper Walkthrough - HTB

![](https://i.imgur.com/YBZnYk4.png)

In this machine on port 80 it's first leak the new ```vhost``` called ```office.paper!```
 on responce header ```X-Backend-Server``` after that wordpress version is vernable through Unauthenticated View Private/Draft Posts and we got the hint already with nick comment using the vernability we check the draft message that leak to another vhost and register ourself to that and get the ```directory Path Traversal``` and get the ```.env``` secret and login through ssh and for Privilege escalation we run ```linpeas``` that lead us to ```CVE-2021-3560.``` 
## Emumeration
#### nmap
```bash
┌─[r00t@parrot]─[~/ctf/paper]
└──╼ $sudo nmap -sV -sC -sT -A 10.10.11.143
[sudo] password for r00t: 
Sorry, try again.
[sudo] password for r00t: 
Starting Nmap 7.92 ( https://nmap.org ) at 2022-05-25 21:16 EAT
Nmap scan report for chat.office.paper (10.10.11.143)
Host is up (0.35s latency).
Not shown: 983 closed tcp ports (conn-refused)
PORT      STATE    SERVICE         VERSION
22/tcp    open     ssh             OpenSSH 8.0 (protocol 2.0)
| ssh-hostkey: 
|   2048 10:05:ea:50:56:a6:00:cb:1c:9c:93:df:5f:83:e0:64 (RSA)
|   256 58:8c:82:1c:c6:63:2a:83:87:5c:2f:2b:4f:4d:c3:79 (ECDSA)
|_  256 31:78:af:d1:3b:c4:2e:9d:60:4e:eb:5d:03:ec:a0:22 (ED25519)
80/tcp    open     http            Apache httpd 2.4.37 ((centos) OpenSSL/1.1.1k mod_fcgid/2.3.9)
|_http-server-header: Apache/2.4.37 (centos) OpenSSL/1.1.1k mod_fcgid/2.3.9
|_http-trane-info: Problem with XML parsing of /evox/about
|_http-title: chat.paper.htb
| http-robots.txt: 1 disallowed entry 
|_/
443/tcp   open     ssl/http        Apache httpd 2.4.37 ((centos) OpenSSL/1.1.1k mod_fcgid/2.3.9)
| ssl-cert: Subject: commonName=localhost.localdomain/organizationName=Unspecified/countryName=US
| Subject Alternative Name: DNS:localhost.localdomain
| Not valid before: 2021-07-03T08:52:34
|_Not valid after:  2022-07-08T10:32:34
|_ssl-date: TLS randomness does not represent time
|_http-server-header: Apache/2.4.37 (centos) OpenSSL/1.1.1k mod_fcgid/2.3.9
| tls-alpn: 
|_  http/1.1
1030/tcp  filtered iad1
1123/tcp  filtered murray
1145/tcp  filtered x9-icue
1163/tcp  filtered sddp
2200/tcp  filtered ici
3737/tcp  filtered xpanel
7004/tcp  filtered afs3-kaserver
15660/tcp filtered bex-xr
32769/tcp filtered filenet-rpc
32779/tcp filtered sometimes-rpc21
49154/tcp filtered unknown
50800/tcp filtered unknown
51493/tcp filtered unknown
57797/tcp filtered unknown
No exact OS matches for host (If you know what OS is running on it, see https://nmap.org/submit/ ).
TCP/IP fingerprint:
OS:SCAN(V=7.92%E=4%D=5/25%OT=22%CT=1%CU=42246%PV=Y%DS=2%DC=T%G=Y%TM=628E733
OS:3%P=x86_64-pc-linux-gnu)SEQ(SP=106%GCD=1%ISR=10B%TI=Z%CI=Z%II=I%TS=A)SEQ
OS:(SP=106%GCD=1%ISR=10B%TI=Z%CI=Z%TS=A)OPS(O1=M505ST11NW7%O2=M505ST11NW7%O
OS:3=M505NNT11NW7%O4=M505ST11NW7%O5=M505ST11NW7%O6=M505ST11)WIN(W1=7120%W2=
OS:7120%W3=7120%W4=7120%W5=7120%W6=7120)ECN(R=Y%DF=Y%T=40%W=7210%O=M505NNSN
OS:W7%CC=Y%Q=)T1(R=Y%DF=Y%T=40%S=O%A=S+%F=AS%RD=0%Q=)T2(R=N)T3(R=N)T4(R=Y%D
OS:F=Y%T=40%W=0%S=A%A=Z%F=R%O=%RD=0%Q=)T5(R=Y%DF=Y%T=40%W=0%S=Z%A=S+%F=AR%O
OS:=%RD=0%Q=)T6(R=Y%DF=Y%T=40%W=0%S=A%A=Z%F=R%O=%RD=0%Q=)T7(R=Y%DF=Y%T=40%W
OS:=0%S=Z%A=S+%F=AR%O=%RD=0%Q=)U1(R=Y%DF=N%T=40%IPL=164%UN=0%RIPL=G%RID=G%R
OS:IPCK=G%RUCK=G%RUD=G)IE(R=Y%DFI=N%T=40%CD=S)

Network Distance: 2 hops

TRACEROUTE (using proto 1/icmp)
HOP RTT       ADDRESS
1   374.60 ms 10.10.14.1
2   365.75 ms chat.office.paper (10.10.11.143)

OS and Service detection performed. Please report any incorrect results at https://nmap.org/submit/ .
Nmap done: 1 IP address (1 host up) scanned in 188.39 seconds
┌─[r00t@parrot]─[~/ctf/paper]
└──╼ $
```

We see port 80 and 22 are open
Lets visit port 80 and view the site provided.

![](https://i.imgur.com/plt6gee.png)

We find that it is a simple static website and has nothing interesting.Though port 443 was open, there was nothing usefull, just some certificates.
Running gobuster, there are no path/directories found.
After a little bit of digging using curl to see the response headers, we notice another vhost being exposed. ```curl -I 10.10.11.143```

![](https://i.imgur.com/57vfQiV.png)

Now lets add it to our ```/etc/hosts``` file.

![](https://i.imgur.com/IvVWM8W.png)

Visiting the new domain, we find a different page with a different theme.

![](https://i.imgur.com/RhD1P7j.png)

Looking around the site, we found some interesting comment telling Michael to remove secret from draft.

![](https://i.imgur.com/rH9wml8.png)

This made me want so see what secret was on the draft.Unfortunately, you have to be admin, to view drafts. Looking around at the footer, we see that the site was designed with wordpress. Looking with wappalyzer, we see it is wordpress 5.2.3.

## Exploiting wordpress: CVE-2019-17671
Now lets look for ```wordpress 5.2.3``` vulnerabilities.
I found [this](https://wpscan.com/vulnerability/9909) vulnerability that could allow an unauthenticated user to view private or draft posts due to an issue within WP_Query. So we basically add ```?static=1``` to the end of the url.```http://office.paper/?static=1```
Yeet, we found the secret they were talking about. It is a url with a new ```vhost.``` 

![](https://i.imgur.com/YSlL1Zu.png)

I will add it to my ```/etc/hosts``` file.

![](https://i.imgur.com/h1Ivavl.png)

Now i can visit the new links ```http://chat.office.paper/register/8qozr226AhkCHZdyY``` to see what is there. I find its a registration page where i can create an account.

![](https://i.imgur.com/y6gzGpK.png)

So I will create an account so that I can login to the chat system as an employee.

After loging in, we receive a notification at the ```#general ```Channel whereby we can access the previous chats.

![](https://i.imgur.com/KLCV3M6.png)

Going though the previous chats, I found some bot ```recyclops``` that could allow some LFI. Fortunately, it allow us to send direct messages by clicking on its profile then the message icon. Some if the commands that it allows are like ```list``` and ```file```

![](https://i.imgur.com/wrAvYFC.png)

I decided to see what this commands output.

![](https://i.imgur.com/IUI5fE3.png)

We can see the ```list``` command, lists the file in the current directory and this is interesting.

![](https://i.imgur.com/VVE0vG4.png)

The ```file``` command, prints out the contents of a file so we can read as we can see above.

Now this gives me an idea to try directory ```file transvasal``` using ```../```

![](https://i.imgur.com/nF8M61D.png)

The output, had interesting outputs like the directory ```hubot``` and even ```user.txt``` that returned ```access denied``` when I tried to read it using the ```file``` command.

I gave a look at the ```hubot``` directory where also the results were interesting. There was a ```.env``` file and as ussual, ```.env``` files, are used to store secrets.

![](https://i.imgur.com/9Wq3dn3.png)

Reading the ```.env``` file, I found a ```username``` and ```password.``` I decided to use the credidentials I found to try loging in ```rocket.chat.```

![](https://i.imgur.com/woN5EZQ.png)

Oops! it failed.

I decide to look for a user in the machine using ```file ../../../etc/passwd```

![](https://i.imgur.com/Ze7NUgk.png)

### Getting User

I found a user dwight so I can now ssh into the machine using this use and the password I got previously. ```ssh dwight@10.10.11.143```

![](https://i.imgur.com/rsEyP91.png)

And there was the ```user.txt```

## Privilege Escalation

### Recon with linpeas
I will host a python server on my attack machine ```python3 -m http.server 80``` so that I can use ```wget``` to download linpeas into the victim machine.

![](https://i.imgur.com/w4g5s6s.png)

Now I can successfully run linpease in the victim machine so find any vulnerabilities that I can use.

I found that it was vilnerable to ```CVE-2021-3560``` that is ```Polkit or Pwnkit``` which allows ```unprivileged``` user to call privileged methods using ```DBus.```

![](https://i.imgur.com/ICyzYUG.png)

[Polkit-exploit - CVE-2021-3560](https://github.com/Almorabea/Polkit-exploit)

Now i'll get the python script in the machine and run it.

![](https://i.imgur.com/zvr5ssw.png)

### Root

Finally we get root. Now lets cat the hash.

![](https://i.imgur.com/25Bb5ci.png)

Finally we solved the lab and thank you so much for your time, if you liked this writeup and you feel it’s helpful then please share it with your friends. 
Happy hacking!