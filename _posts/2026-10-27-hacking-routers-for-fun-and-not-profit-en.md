---
layout: post
title:  "Hacking routers for fun and not profit"
date:   2026-10-27 12:00:00 +0400
categories: exploit-development
tags: [iot, firmware-analysis, command-injection, mips, ghidra, router]
lang: en
ref: routers
permalink: /en/exploit-development/2026/10/27/hacking-routers-for-fun-and-not-profit.html
description: "Recently i realized that doing tutorials and course materials is not that interesting in terms of binary exploitation, so i wanted to choose real targets — which led me to IoT devices, especially routers…"
canonical_url: "https://medium.com/@Cushz/hacking-routers-for-fun-and-not-profit-f37da822f745"
---

<figure>
  <img src="/assets/img/routers/1-kBvAeebtnXoE_8CmknkDcA.gif" alt="screenshot" loading="lazy">
</figure>

## Introduction

Recently, I was just realizing that doing tutorials and learning from course materials are not interesting and beneficial in terms of binary exploitation, therefore I wanted to choose real targets to see what is happening in the real products and that led me to choose IoT devices, especially routers. If you look at Pwn2Own or other initiatives, they are mostly appreciating router exploits more than any other IoT devices, so why not?

While there are lots of vendors such as Tp-Link, Broadcom and such, I decided to choose Tenda. There were two conditions that I had to set before choosing a vendor.  
1) It should be used in the wild  
2) There should be public firmware

If you have worked with IoT devices, you probably know that they are working with firmware. Some of the vendors are publishing the firmware already, and others are not publishing, in which you have to do some kind of UART/JTAG magic in order to dump it from the hardware, but i don’t have such equipments so I had to find one with public firmware.

For these 2 conditions, Tenda was the perfect match. Also in our country, tenda is popular which made me more interested in this vendor.

Anyways, while I was looking for the Tenda routers and their public firmwares, I realized that, some of the firmwares are SSL encrypted. It is basically like you have all the source code but it is password protected zip file and you can’t decrypt it. Therefore, i had to choose one which is not ssl encrypted. That’s when I came across to this:

<figure>
  <img src="/assets/img/routers/1-3N0xx4Y1HPYfk-m6d6Snxg.jpeg" alt="W18E v2.0 AC1200 Dual Band Gigabit Enterprise Wireless Router" loading="lazy">
  <figcaption>W18E v2.0 AC1200 Dual Band Gigabit Enterprise Wireless Router</figcaption>
</figure>

In firmware research, you usually have 2 steps that you mostly follow.  
1) Extract the firmware  
2) Reverse engineer the firmware  
3) Write the PoC to see if it is actually vulnerable

For step 3, i don’t have the hardware itself, therefore I had to emulate it with the help of the QEMU (thanks to Fabrice Bellard).

you may download the exact firmware from here: [https://www.tendacn.com/material/show/824455349567557?proPath=W18EV2](https://www.tendacn.com/material/show/824455349567557?proPath=W18EV2)

## Extracting the firmware

After downloading the firmware, I have used binwalk in order to extract the firmware.

```
~/Firmware_Hacking ❯ binwalk -e US_W18EV2_kf_V16.01.0.20\(4766\)_HighPower\ \(1\).bin
........
~/Firmware_Hacking ❯ cd _US_W18EV2_kf_V16.01.0.20\(4766\)_HighPower\ \(1\).bin.extracted/
~/Firmware_Hacking/_US_W18EV2_kf_V16.01.0.20(4766)_HighPower (1).bin.extracted ❯ ls
20F29F.squashfs  80  80.7z  squashfs-root  squashfs-root-0
```

extraction will give you some warning but you may ignore them.
There are lots of binary files inside squashfs-root that we may analyze.

```
~/Firmware_Hacking/_US_W18EV2_kf_V16.01.0.20(4766)_HighPower (1).bin.extracted/squashfs-root/bin ❯ ls
3322ip      cameraDiscovery  dhcp6c       dhcps            httpd                 l2tp_dhcpc_wan3  netcheck     pppd6_wan3        pptp_dhcpc_wan4  set_proc.sh   tendaupgrade    wpad
88ip        cfm              dhcp6c_wan1  dhcps-guest      igmpproxy             l2tp_dhcpc_wan4  netctrl      pppd6_wan4        pptp_wan1        setmac        type_identify   wps_msg
acbs        cfmd             dhcp6c_wan2  discovery-dhcp   igmpproxy.sh          lan_dhcps0       network      pppdForPppServer  pptp_wan2        sfwifi_ate    ucloud          wserver
acbs_cli    checklock        dhcp6c_wan3  discovery-pppoe  imsd                  lldpd            ngrokc       pppdForPptp       pptp_wan3        sh            ucloudapp       wserver_notify.sh
appmgr      clear-table      dhcp6c_wan4  dnrd             inadyn                logdebug         ntdd         pppd_wan1         pptp_wan4        show-table    uhttpd          xl2tpcltpppd
arpbrocast  db_dhcpc_wan1    dhcp6s       dnrd-guest       install_kernel_ko.sh  login            onvifproxy   pppd_wan2         pptpd            sntp          unset-value     xl2tpd-client
arpgateway  db_dhcpc_wan2    dhcpcd       dnrd-wisp        ip                    login.sh         phddns       pppd_wan3         pptppppd         sqlite3       vconsole        xl2tpd-server
ash         db_dhcpc_wan3    dhcpcd_ac    flash            ip6tables             logserver        phddns_wan1  pppd_wan4         pptppppd-server  ssdpd         wcli            xl2tpd_wan1
ate         db_dhcpc_wan4    dhcpcd_lan   flash_erase      iptables              miniupnpd        phddns_wan2  pppd_wisp         pservice_tool    steer_cli.sh  wdtctl          xl2tpd_wan2
ate_cmd     db_pppd_wan1     dhcpcd_wan1  flash_unlock     iw                    monitor          phddns_wan3  pppnotify         publish          steerd        wifi            xl2tpd_wan3
ate_init    db_pppd_wan2     dhcpcd_wan2  get-value        jshn.sh               msgagent         phddns_wan4  pptp_callmgr      radvd            subscribe     wifi_init.sh    xl2tpd_wan4
ate_server  db_pppd_wan3     dhcpcd_wan3  gnway            km_cmd                msgd             pppd         pptp_dhcpc_wan1   restart_imsd.sh  swconfig      wifibase        xl2tpdpppd
authmgr     db_pppd_wan4     dhcpcd_wan4  hostapd          l2tp_dhcpc_wan1       multiWAN         pppd6_wan1   pptp_dhcpc_wan2   sdwanctl         tcpdump       wpa_cli
burn_make   debug            dhcpcd_wisp  hostapd_cli      l2tp_dhcpc_wan2       net_config.sh    pppd6_wan2   pptp_dhcpc_wan3   set-value        temp          wpa_supplicant
```

In here, I decided to focus on the httpd binary. Because if you look for the previous researches which have been done on Tenda, they were mostly targeting this binary. It is the core web server used by Tenda, for handling administrative functionalities. It is also famous for backdoors and such.

## Reverse engineering httpd

When i added this file to ghidra, I saw that this is different architecture (MIPS) which haven’t work on it before, but let’s hope it won’t be difficult.

<figure>
  <img src="/assets/img/routers/1-oOCOA7RxHKcHVgHNhBLUnQ.png" alt="Ghidra output" loading="lazy">
  <figcaption>Ghidra output</figcaption>
</figure>

Considering that it has the huge codebase, it would not be wise to just randomly look all of the functions. In binary exploitation, the main point is to find something where user input it being executed without validation. For finding it in languages like C/C++ and such you have lots of options but some of them are:  
1) usage of scanf, snprintf, memcpy and such.  
2) Exported functions with names such as Get…., Set…., Do…. and so on.

Basically, you either find the native vulnerable functions being used or custom functions which could be vulnerable.

After little bit of research, I came across to this. One interesting function was doSystemCmd. The name already tells me what this function could do.
For example:

<figure>
  <img src="/assets/img/routers/1-GaZ9K5LUhLK0dkf6rhXOgQ.png" alt="screenshot" loading="lazy">
</figure>

In this function, doSystemCmd, basically executes system commands. This is crucial and will be needed in the future.

Now, I have to find all the cross references where this function could take user input instead of hardcoded system commands. Luckily, I found the one where the arguments were not hardcoded. It was a function called do_packet_action.

<figure>
  <img src="/assets/img/routers/1-fqNhREOkyXbXpF_sUyVfSw.png" alt="code snippet from do_packet_action()" loading="lazy">
  <figcaption>code snippet from do_packet_action()</figcaption>
</figure>

There is a draft idea that do_packet_action implements the packet capture functionality. Its job is to build and run the tcpdump command. But the problem is it is taking attacker-controlled input to that string when it builds the command string.  
Flow is basically like this:  
1) The code reads values (net.check.protocol, net.check.target.device and etc) via GetValue and places them in the stack  
2) These values are being read and placed to the tcpdump with snprintf. Especially the ones with …. %s host %s …. where the second %s is acStack_254 (net.check.protocol) and third %s is local_314 (net.check.target.device)  
3) builded tcpdump command is being executed with doSystemCmd

In here, the problem is %s fields are not being validated or sanitized. This makes it potentially vulnerable to command injection.

Last thing left is to find how net.check.protocol and such values are being set, and after small reference based search, I have found out that it is being set in a function called formSetNetCheckTools().

<figure>
  <img src="/assets/img/routers/1-tCZQeB4W2UqaLJvInlqPJA.png" alt="Code snippet from formSetNetCheckTools()" loading="lazy">
  <figcaption>Code snippet from formSetNetCheckTools()</figcaption>
</figure>

This function takes parsed JSON request body and set it to the respective values. For example:  
action = net.check.action  
type = net.check.type  
targetDevice = net.check.target.device

But where this JSON request body comes from?

Each function has a slot the program uses to call it. I searched for who uses formSetNetCheckTools’s slot. It’s registered by a line that effectively says: register formSetNetCheckTools in the set table under the name diagnose.

<figure>
  <img src="/assets/img/routers/1-SjDEqhzzv5Eh_BIsiBlcMg.png" alt="Code snippet from System_Manage_Register()" loading="lazy">
  <figcaption>Code snippet from System_Manage_Register()</figcaption>
</figure>

These modules are being set with formSetModules before being registered.

<figure>
  <img src="/assets/img/routers/1--gC8-dDO1UfruPs1Jx-YkQ.png" alt="Code snippet from formSetModules()" loading="lazy">
  <figcaption>Code snippet from formSetModules()</figcaption>
</figure>

<figure>
  <img src="/assets/img/routers/1-yOoE1Kr_XvxoNnQi81oC-Q.png" alt="string search for modules" loading="lazy">
  <figcaption>string search for modules</figcaption>
</figure>

Function’s first argument is being parsed with cJSON_Parse. In another function, formSetModules is being registered as setModules.

So now, we roughly have an idea what is happening out here. Let’s summarize in forward direction:

1. We send a POST request to /goform/setModules
2. body is parsed with cJSON_Parse(formSetmodules)
3. Parsed values are being set to a internal value in formSetNetCheckTools
4. These values are used in the building of tcpdump command
5. Builded command is executed with doSystemCmd

Our potential PoC will look like this:

```
POST /goform/setModules
 {"module":"diagnose","action":"start","interface":"…",
 "protocol":"…","targetDevice":"1.1.1.1;telnetd -l/bin/sh -p9999;"}
```

which should give me shell.

## Testing the PoC

For emulating this router, I will use QEMU and FirmAE. It is basically emulation tool for IoT firmwares.  
Basic build and run script (thanks to claude) is enough for me to emulate the environment.

<figure>
  <img src="/assets/img/routers/1-YGJC1fOQ5x-cmc04IoI1Hg.png" alt="build script" loading="lazy">
  <figcaption>build script</figcaption>
</figure>

<figure>
  <img src="/assets/img/routers/1-liMSINL8CV2zd9W44NbRyw.png" alt="login page for tenda" loading="lazy">
  <figcaption>login page for tenda</figcaption>
</figure>

After building, we should be able to see the login page of the router in port 8080.

Now, let’s run our PoC. But keep in mind that some of the commands such as id or whoami are not existent on the firmware, therefore don’t try to inject those.

<figure>
  <img src="/assets/img/routers/1-NmpU7mpfa8IKJOG2fIvTBA.png" alt="screenshot" loading="lazy">
</figure>

<figure>
  <img src="/assets/img/routers/1-jMtSjAFW67OO7kEmm_90OQ.png" alt="PoC demonstration" loading="lazy">
  <figcaption>PoC demonstration</figcaption>
</figure>

On this device, uid_0 means it is root.

As a conclusion, this is an unauthenticated (if it is in factory state) root command injection. The PoC and harness (if you want to test it without the real hardware) is also published on my github page as well.

[https://github.com/Cushz/CVE-2026-118238](https://github.com/Cushz/CVE-2026-118238)
