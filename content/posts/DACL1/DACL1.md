---
title: "DACL 1 Attacks and sacrificial process"
date: 2025-11-12T21:09:18+03:00
draft: false
tags: ["AD","ACL","TTP","Lateral Movement"]
---

## INTRO

This is one of the machine that test my skills in DACL and lateral movement,
however, I will write my experiences and my challenges that I faced, let's say it is like an engagement for pen-tester for AD networks, let's me show you the AD network that we have:
```
domain: inlanefreight.local
DC: dc01.inlanefreight.local
computers: WS01, REMOTE_SVC
```

## Enum

I have got a credential to start the machine that is **carlos:Pentesting01**
using this account let's stat an initial enumerations

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

Adding an SPN tricks Active Directory into believing that Mathew's account is a service account. This allows us, as an attacker, to request a service ticket (TGS) for Mathew's "service". The domain controller (KDC) then creates this TGS and, most importantly, encrypts it using a hash derived from the service account's password—in this case, **Mathew's own account password**.

Use hashcat to crack it "the hash is TGS which is "
`hashcat -m 13100 --force hashcatme.txt /usr/share/wordlists/rockyou.txt --force`

![](/posts/DACL1/5.png)

 We have mathew's password, in the attack path from bloodhound shown that any one in Network Admins can read the LAPS, this is  security feature that automatically manages and randomizes the password of the local administrator account on devices
Now mathew have `WriteOwner` over the Network Admins object, we cannot directly go and add our self in the group, WriteOwner is mean we can change the ownership of the Network Admins object. After this we can get most of the  right if we own the object

![](/posts/DACL1/8.png)

So we put our self as owner of the object, now we need select which right that we need, in `dacledit.py` i write `-action write` which by default will chose full control right

![](/posts/DACL1/10.png)

We can remotely read the LAPS for the WS01 because the password is stored in the computer attribute which we can read it from LDAP protocol in DC

![](/posts/DACL1/12.png)

I forget to tell that we have only one public IP to attack which it DC01 10.129.205.122
in the enterprise there is a pivot port in this IP that is 13889 if we connect to this port it will direct us to the WS01 machine in 3389 port which is for the WS01 RDP.
We used `Administrator:9P-82cdtoI2.Iq` we catch this from LDAP abuse

![](/posts/DACL1/14.png)

So we get in the machine with local admin, we can start dumping the credentials with `mimikatz`, and I found a cached domain user with is `jose` and he's NT HASH

![](/posts/DACL1/15.png)

I want to explain two terms here:
* PtH (Pass the Hash): We found jose NT hash we can pass it to get session under he's name and he's privileges
* OPtH (Over-pass the hash): is using jose NT hash to request from the DC to get the TGT that is for jose, then we can use the TGT to request TGS, we can use it remotely also from kali

## Second Attack path

getting back to the bloodhound we see a new attack path.

![](/posts/DACL1/16.png)

Now we have WriteDACL, this right is direct we can edit the security descriptor to abuse it. but in the past we was facing a user and we used SPN abuse, now what?
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
We started a jose session using he's NT hash, and we can now start a new session under `remote_svc$`. new windows but now with computer
### Second Sacrificial process
```
.\mimikatz.exe privilege::debug "sekurlsa::pth /user:remote_svc /domain:inlanefreight.local /ntlm:0077EF1652B20A7845C5B6AFB3C1718D /run:powershell.exe" exit
```

![](/posts/DACL1/21.png)

Even if you see administrator account in the new PtH window, it is still the **network token and all auth over the network is from REMOTE_SVC$**. - Perfectly normal for PtH. you can verify using `net user /dom` it will goes with the `remote_svc` not the local user

![](/posts/DACL1/22.png)

this computer, can control MicrosoftSync group, that can do DCsync attack which can dump any user NT hash in domain joined network 


in the new windows for remote_svc I have grant my self fullcontrol and add my self in the group that for dcsync

![](/posts/DACL1/23.png)

## DCsync

let's try close the windows and run it again and test dcsync using mimikatz
`mimikatz # lsadump::dcsync /domain:inlanefreight.local /user:krbtgt`

![](/posts/DACL1/24.png)

I got `krbtgt`!

Thank you for reading, if there is any misinformation contact with my in LinkedIn.

