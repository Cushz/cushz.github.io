---
layout: post
title:  "Doom-u mikrodalğalı sobada işlətmək əvəzinə debug etmək"
date:   2025-05-10 15:44:35 +0400
categories: reverse-engineering
tags: [doom, reverse-engineering, game-hacking, cheat-engine, x32dbg]
lang: az
ref: doom
permalink: /reverse-engineering/2025/05/10/debugging-doom-instead-of-running-it-on-the-microwave.html
description: "Son vaxtlar hamı Doom-u qeyri-adi cihazlarda — mikrodalğalı sobada, Arduino-da, əmioğlunun köhnə jailbreak edilmiş Nintendo-sunda — işlətməyə çalışırdı…"
canonical_url: "https://medium.com/@Cushz/debugging-doom-instead-of-running-it-on-the-microwave-fd3b6190cecd"
---

<figure>
  <img src="/assets/img/doom/0-6qPIlZRIK78c9wcY.png" alt="screenshot" loading="lazy">
</figure>

Son vaxtlar hamının Doom-u qeyri-adi cihazlarda — mikrodalğalı sobada, Arduino-da, əmioğlunun köhnə jailbreak edilmiş Nintendo-sunda və s. — işlətməyə çalışdığını görürdüm. Bu, artıq nerd-lər arasında bir ənənəyə çevrilib. Mən də düşündüm ki, bəlkə Doom-u işlətmək əvəzinə onu hack etməyə çalışım.

Əvvəlcə oyunun özünü tapmalıyam. Vanilla Doom-u yükləmək üçün çoxlu variant var, amma onu emulator olmadan işlətmək üçün chocolate doom adlı bir şey mövcuddur. Onu hazırlayanlar deyirlər ki, bu, Doom-un demək olar ki, eyni versiyasıdır, yeganə fərq odur ki, Windows 11 kimi ən son əməliyyat sistemlərində işləyə bilir.

Amma nə etdiyimizi daha yaxşı başa düşmək üçün hansısa plan qurmalıyıq. Ən sadə şey odur ki, atəş açanda ammo azalmaq əvəzinə artsın. Bu blog üçün bu kifayət edər. Gəlin başlayaq:

### **Ammo haradadır**

Bir dəqiqə düşünək: proqramın daxili işləmə prinsipi haqqında heç nə bilmiriksə, nəyi axtarmalıyıq? Oyun haqqında məlumat saxlayan çoxlu memory sahəsi var — health, ammo, istifadəçi adı və s. Əslində etməli olduğumuz şey ammo-nun harada saxlandığını və hər dəfə harada dəyişdirildiyini tapmaqdır. Atəş açıb debugger-in köməyi ilə ammo rəqəmini izləyə bilərik, amma bu, olduqca əsəb pozan prosedurdur.  
Onu əl ilə taparaq eqomu doydurmaq əvəzinə, bu memory address-ləri tapmaq üçün sadəcə Cheat Engine-dən istifadə edəcəyəm.

Cheat Engine həm də debugger və memory scanner-dir, process memory-də konkret dəyəri tapmağa kömək edir. Gəlin oyunu açaq, onu debugger-ə attach edək, silahlarla atəş açaq və ammo sayını izləyək.

<figure>
  <img src="/assets/img/doom/1-z8vNkVyRUk3lqB8FO3i9jQ.png" alt="screenshot" loading="lazy">
</figure>

46 axtardığım zaman görürəm ki, dəyər olaraq 46 saxlayan çoxlu memory address var. Cheat Engine-in bir xüsusiyyəti var ki, dəyəri əvvəlki dəyərlə birlikdə yenidən axtarmağa imkan verir. Yəni yenidən atəş açıb hansı address-in eyni anda dəyişdiyini tapa bilərəm.

<figure>
  <img src="/assets/img/doom/1-iiWwSuTNEhuMlEDusXZAtQ.png" alt="screenshot" loading="lazy">
</figure>

Burada biz sadəcə Cheat Engine-ə dedik ki, növbəti scan-da dəyəri 5 vahid azalmış olanı tapsın. Nəticələri daraltmaq həmişə daha yaxşıdır.  
Bəs bunlardan hansı həqiqətən ammo üçün istifadə olunur?

Bu address-lərin hər birinin üzərinə sağ klik etsək, həmin dəyərləri dəyişdirən konkret instruction-u görəcəyik. Amma onlardan ilk ikisinin sayı biz heç nəyə atəş açmasaq belə artacaq. Üçüncüsündə isə say yalnız atəş açdıqda artır. Bizim hədəfimiz budur.

<figure>
  <img src="/assets/img/doom/1-l7c2wkWm9qjldk1OLCxXug.png" alt="screenshot" loading="lazy">
</figure>

<figure>
  <img src="/assets/img/doom/1-aaShV-vOSoIkiI8QbCXOPg.png" alt="screenshot" loading="lazy">
</figure>

Gördüyünüz kimi, onun etdiyi şey **_ebx+edx\*4+000000A4_** ünvanında saxlanılan dəyərdən bir vahid çıxmaqdır.

> RE məsləhəti: ebx-in \[\] içərisində olması o deməkdir ki, ebx hansısa address saxlayır və bu address-də nə saxlandığını yoxlayır. Əgər \[\] olmasaydı, birbaşa address-in özündən çıxacaqdı.

Beləliklə, dəyişikliyə cavabdeh olan address-i tapdıq. İndi nə edək?

### Patching

Bilirik ki, PE32 (eləcə də PE64) executable-larda əsas kod çox vaxt .text section-da saxlanılır. **x32dbg**-nin memory map funksiyasından istifadə edərək doom.exe-nin section-larını görüb analiz edə bilərik.

<figure>
  <img src="/assets/img/doom/1-APbYhjd4-ZFkEnMeCniwZw.png" alt="screenshot" loading="lazy">
</figure>

Əvvəlcədən öyrənmişdik ki, address **_0043057C_** idi.

<figure>
  <img src="/assets/img/doom/1-jolYPOt_A6gjBYEF7LFyJA.png" alt="screenshot" loading="lazy">
</figure>

Bu mərhələdə istədiyimiz kimi işləməsi üçün **_sub_** əmrini **_add_** ilə əvəz edəcəyik. Ya executable-ı birbaşa x32dbg ilə patch edə bilərik, ya da executable hər dəfə işə düşdükdə exploit-imizi inject etmək üçün frida-dan istifadə edə bilərik. Sadəlik naminə birinci variantı seçəcəyik.

Bunu etmək üçün instruction-un üzərinə sağ klik edib **_assemble_** seçimini seçirik və dəyişirik.

<figure>
  <img src="/assets/img/doom/1-fbsqTEvSmeJoK0Ss7lGNnQ.png" alt="screenshot" loading="lazy">
</figure>

Bundan sonra **_ctrl+p_** basırıq və o, kopyasını yaradaraq executable-ımızı patch edəcək.

### Kiçik demo

<figure>
  <img src="/assets/img/doom/1-F-dqI1KmMBi7pV1Jz_G-Yw.gif" alt="screenshot" loading="lazy">
</figure>
