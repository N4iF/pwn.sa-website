---
title: "DACL 1 Attacks and sacrificial process"
date: 2025-11-12T21:09:18+03:00
draft: false
tags: ["AD","ACL","TTP","Lateral Movement"]
---

## INTRO
This is DACL I skill assessment from HackTheBox, one of the machines that tested my skills in **DACL** and lateral movement, Below I will write my experiences and my challenges that I faced, let me show you the AD network that we have:
```
domain: inlanefreight.local
DC: dc01.inlanefreight.local
computers: WS01, REMOTE_SVC
```

## Enum

I have got a credential to start the machine that is **carlos:Pentesting01**
using this account let's start an initial enumeration

![](/posts/DACL1/1.png)
![](/posts/DACL1/2.png)

## First Attack path
We gathered the information using `bloodhound-python` remotely to use it in Bloodhound CE

![](/posts/DACL1/3.png)

Now let's talk about the DACL. As we saw, our user **Carlos** (the Subject) has GenericWrite permission over **Mathew** (the Object). This permission is called an "Access Right".

To imagine how this works, think of Mathew's user account having a security rulebook attached to it—this is its **Security Descriptor**. A key chapter in that rulebook is the **DACL**, which is a list of rules. One of those rules says, "Carlos has GenericWrite permission on me." Every object has a rulebook like this describing who can access it and what they can do.

So, in our enumeration phase, we read the Security Descriptor of our targets to find what rights we have over them.

let's move on to attacking Mathew.

![](/posts/DACL1/4.png)

To understand what's happening here, our goal is to get a service ticket (TGS) that we can crack offline to reveal Mathew's password. Here’s how we did it:

We used our GenericWrite permission to edit one of Mathew's **user attributes**, making him vulnerable to a **Kerberoasting** attack. Specifically, we added a Service Principal Name (SPN) to his servicePrincipalName attribute.

When we add an SPN to a user account, Active Directory starts issuing Kerberos service tickets for that account. This makes the account eligible for Kerberoasting attacks, allowing us to request and crack those tickets offline. It doesn’t turn the user into a “service account,” it just enables ticket issuance that attackers can abuse.

Use hashcat to crack it "the hash is TGS which is "
`hashcat -m 13100 --force hashcatme.txt /usr/share/wordlists/rockyou.txt --force`

![](/posts/DACL1/5.png)

We have mathew's password, in the attack path from bloodhound shown that any one in Network Admins can read the LAPS, this is  security feature that automatically manages and randomizes the password of the local administrator account on devices
The WriteOwner permission allows us to take ownership of an object, but ownership alone doesn’t instantly give full control. Once we become the owner, we can modify the object’s DACL and grant ourselves any rights we need — that’s what truly gives us control.

![](/posts/DACL1/8.png)

So we put our self as owner of the object, now we need select which right that we need, in `dacledit.py` i write `-action write` which by default will chose full control right

![](/posts/DACL1/10.png)

We can remotely read the LAPS for the WS01 because The password is stored by LAPS in the computer object’s ms-Mcs-AdmPwd attribute. It can only be read if your account has the correct LDAP read rights it’s not readable by everyone by default. WE HAVE THIS RIGHT NOW

![](/posts/DACL1/12.png)

I forget to mention that we have only one public IP to attack which it DC01 10.129.205.122
in the enterprise there is a pivot port in this IP that is 13889 if we connect to this port it will direct us to the WS01 machine in 3389 port which is for the WS01 RDP.
We used `Administrator:9P-82cdtoI2.Iq` we catch this from LDAP abuse

![](/posts/DACL1/14.png)

So we get in the machine with local admin, we can start dumping the credentials with `mimikatz`, and I found a cached domain user with is `jose` and his NT HASH

![](/posts/DACL1/15.png)

I want to explain two terms here:
* PtH (Pass the Hash): We found jose NT hash we can pass it to get session under his name and his privileges
* OPtH (Over-pass the hash): is using jose NT hash to request from the DC to get the TGT that is for jose, then we can use the TGT to request TGS, we can use it remotely also from kali

## Second Attack path

getting back to the bloodhound we see a new attack path.

![](/posts/DACL1/16.png)

Now we have WriteDACL, this right is direct we can edit the security descriptor to abuse it. but in the past we were facing a user and we used SPN abuse, now what?
Also we have NT hash for jose, we don't have plaintext password? how can we abuse this relationship?
Local administrator cannot enumerate the AD? 

in this challenge I used **sacrificial process**, create a new process under jose account using only NT hash, open a process like PowerShell or CMD. this will create a token for jose. let's combine sacrificial process + PtH

### First Sacrificial process
in mimikatz you can do Sacrificial process

![](/posts/DACL1/18.png)

We get a new PowerShell windows. however if you run `whoami` you will see administrator. but if you connect to any network resources you will be **impersonating jose**

Let's get back to the attack path. We have WriteDACL permission over the TechSupport group, which in turn can read a GMSA password. If you have WriteDACL, you can grant yourself full control because it lets you edit the security descriptor of the object. We'll use this power to add ourselves to that group.

![](/posts/DACL1/19.png)

Now, I should be able to read the GMSA password, but as you can see in the screenshot, my first attempt fails with "Access is denied". **Why?**

This is a critical concept: when you add yourself to a new group, your current logon session's security token is **not** automatically updated. The PowerShell window I'm using is still running with my old token for `jose`, which doesn't know I'm now a member of `TechSupport`.

To fix this, you have to get a new token. The easiest way is to start a new logon session. After using a tool like `Mimikatz` to create a new process with an updated token for `jose`, we can simply run the GMSA password reader command again, and this time it will succeed.

![](/posts/DACL1/20.png)

I got them, the old and the new NT hash that they used in `remote_svc$` computer.
We started a jose session using his NT hash, and we can now start a new session under `remote_svc$`. new windows but now with computer
### Second Sacrificial process
```
.\mimikatz.exe privilege::debug "sekurlsa::pth /user:remote_svc /domain:inlanefreight.local /ntlm:0077EF1652B20A7845C5B6AFB3C1718D /run:powershell.exe" exit
```

![](/posts/DACL1/21.png)

Even if you see administrator account in the new PtH window, it is still the **network token and all auth over the network is from REMOTE_SVC$**. - Perfectly normal for PtH. you can verify using `net user /dom` it will goes with the `remote_svc` not the local user

![](/posts/DACL1/22.png)

this computer, can control MicrosoftSync group, that can do DCsync attack which can dump any user NT hash in domain joined network 


in the new windows for remote_svc I have grant myself fullcontrol and add my self in the group that for dcsync

![](/posts/DACL1/23.png)

## DCsync

let's try close the windows and run it again and test dcsync using mimikatz
Running DCSync requires replication privileges (like Replicating Directory Changes and Replicating Directory Changes All) or an ACL that grants them. Local admin rights alone aren’t enough. WE GOT THIS RIGHT

`mimikatz # lsadump::dcsync /domain:inlanefreight.local /user:krbtgt`

![](/posts/DACL1/24.png)

I got `krbtgt`!

## Defender side
To defend against these DACL abuses, restrict who has GenericWrite, WriteDACL, or WriteOwner permissions, monitor for unexpected DACL or ownership changes, and alert when new SPNs are created on regular user accounts. Regular permission reviews and logging can prevent these privilege escalation paths.

Thank you for reading, if there is any inaccuracies, feel free to contact me in LinkedIn.
resource: https://academy.hackthebox.com/module/219/section/2329
