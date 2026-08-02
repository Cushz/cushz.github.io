---
layout: post
title:  "Gəlin minesweeper-i hack edək"
date:   2025-07-20 18:24:50 +0400
categories: reverse-engineering
tags: [minesweeper, reverse-engineering, ghidra, x32dbg, game-hacking, windows]
lang: az
ref: minesweeper
permalink: /reverse-engineering/2025/07/20/lets-hack-minesweeper.html
description: "İnternetin olmadığı, amma yenə də kompüterdə nəsə oynamaq istədiyin köhnə günləri xatırlayırsan? Şahmat, tort bişirmə oyunları və s. Bir də minesweeper…"
canonical_url: "https://medium.com/@Cushz/lets-hack-minesweeper-7ec334ebf439"
---

<figure>
  <img src="/assets/img/minesweeper/0-n84wuzZ-MYN7w5xF.gif" alt="screenshot" loading="lazy">
</figure>

İnternetin olmadığı, amma yenə də kompüterdə nəsə oynamaq istədiyin köhnə günləri xatırlayırsan? Şahmat, tort bişirmə oyunları və s.  
Bir də minesweeper adlı əsəb pozan bir oyun vardı. Sən kor-koranə bir xana seçməli və partlamayacağına ümid etməli idin. Əlbəttə, bir də oyunu udmağa heç bir köməyi olmayan o “gülən emoji” vardı.  
Mən o oyunu uda bilmirdim, indi də bacarmıram. Sonra düşündüm ki, niyə burada RE bacarıqlarımı tətbiq etmirəm və bu tamamilə faydasız blog yazısını yazmaq qərarına gəldim.

Qeyd: minesweeper reversing haqqında çoxlu blog yazısı var, ona görə də bu yazı yalnız mənim təcrübəmi və qarşılaşdığım çətinlikləri göstərəcək — bunu tədris materialı kimi qəbul etməyin.

Ümid edirəm ki, xoşunuza gələcək.

Source code-u disassemble / decompile etmək üçün Ghidra-dan, debugging üçün isə x32dbg-dən istifadə edəcəyəm. Burada function adları təsadüfi olacaq, çünki PDB-m problem yaradır (PDB-ni google edin).

İki şeyi bilirik. Bu oyun 2D board-dur, ona görə də buna uyğun nəsə gözləməliyik. Həmçinin bu oyun randomize olunub, yəni hər dəfə mina və bayraqlar fərqli yerlərdə olacaq. Bu nöqtədə mütləq random function olmalıdır. Gəlin ona baxaq.

Bütün codebase-də rand açar sözünü axtarın və iki random function görəcəksiniz: **_rand_** və **_srand_**. Əvvəlcə rand ilə başlayacağıq. **_something\_random_** mənim tərəfimdən dəyişdirilmiş function adıdır. Burada göründüyü kimi, bu function parametr qəbul edir və random dəyəri həmin parametrə bölür.

<figure>
  <img src="/assets/img/minesweeper/1-dZTyrgK4yKTWKLBk0HTmEg.png" alt="screenshot" loading="lazy">
</figure>

<figure>
  <img src="/assets/img/minesweeper/1-68kl74E8WHC9xqaay1CMEg.png" alt="something_random adını mən dəyişmişəm" loading="lazy">
  <figcaption>something_random adını mən dəyişmişəm</figcaption>
</figure>

**_something\_random_**-un referans edildiyi function-a baxaq.

Referans edilən function belədir:

<figure>
  <img src="/assets/img/minesweeper/1-SQpKKa2k82oYvBJOmrD8ig.png" alt="screenshot" loading="lazy">
</figure>

<figure>
  <img src="/assets/img/minesweeper/1--vZc7Ksmwf95kEtw0qRfKw.png" alt="possible_mine_set adını da mən dəyişmişəm" loading="lazy">
  <figcaption>possible_mine_set adını da mən dəyişmişəm</figcaption>
</figure>

```text
iVar1 --> random_value / DAT_..5334
iVar2 --> random_value / DAT_..5338

DAT_01005340 is the pointer to the start of the game board array.
[iVar1 + 1 + (iVar2 + 1) * 32] --> coordinate of one tile
```

Niyə 32-yə vurulduğunu soruşa bilərsiniz — oyun 32x32 piksel olduğu üçün.

while loop yoxlayır ki, tile mina kimi işarələnib, ya yox. Əgər minalıdırsa,  
loop davam edəcək; deyilsə, LSB-ni set etməklə onu minalı edəcək. Bu isə  
0x80 ilə OR əməliyyatı aparmaqla həyata keçirilir.

Bunu daha yaxşı başa düşmək üçün gəlin hər bir **_something\_random_** çağırışına breakpoint qoyaq və dəyərlərə baxaq.

<figure>
  <img src="/assets/img/minesweeper/1-6aBAA_fjdCIgTXf76qQNDw.png" alt="screenshot" loading="lazy">
</figure>

Stack-ə push edilən və **_something\_random_** function-da parametr kimi istifadə olunan dəyəri görürük. Hər iki halda belə bir nəticə alacağıq:

```text
ivar1 = random_value / 9
ivar2 = random_value / 9
```

Güman edirəm ki, 9 oyunun ölçüsünə görədir. Yadınızdadırsa, o, 9x9-dur.

Bundan sonra görəcəyik ki, loop 10 dəfə işləyir və çox güman ki, minaları əlavə edir. Bu minalar bu address-dən başlayaraq əlavə olunur: **_DAT\_01005340._**

**Bu nöqtədə əvvəl izah etdiyim məntiq işə düşür. Əgər dəyər 0x0f-dirsə, 0x80 ilə OR əməliyyatı aparmaqla onu 0x8f-ə çevirəcək.**

Yəni böyük ehtimalla **_DAT\_01005340_**-a baxsaq, oyun board-unun hex təsvirini görəcəyik.

<figure>
  <img src="/assets/img/minesweeper/1-0aoejCJ44zAPY5D80QL7CQ.png" alt="screenshot" loading="lazy">
</figure>

Kodu işlətməyə davam etdim və o, ilk iki breakpoint arasında on dəfə loop etməyə başladı, sonra isə sonuncu breakpoint-ə keçdi. Bu o deməkdir ki, minalarımız yerləşdirilib.

0x0F və 0x10 — yan sərhədlər və boş hissələr olacaq (delimiter “\t”)  
 0x0E — **_bayraq qoysaq_**  
 0x0D — **_sual işarəsi qoysaq_**

0x8F — mina yerləri olacaq  
 0x8E — **_bayraq qoysaq_**  
 0x8D — **_sual işarəsi qoysaq_**

Bunu asanlıqla yoxlaya bilərik.

<figure>
  <img src="/assets/img/minesweeper/1-fKL5wov0KthsBYv9gu_MAg.png" alt="screenshot" loading="lazy">
</figure>

Bu mərhələdə oyunu asanlıqla uda bilərəm, amma yenə də hər dəfə memory dump-a baxmağa tənbələm. Gəlin bu oyunu patch edək ki, sadəcə udmaq daha asan olsun. Patch ideyası üçün bir tədqiqatçı sadəcə **_0x8f_**-ləri **_0x8e_**-yə çevirmək qərarına gəlmişdi. Bu o deməkdir ki, oyunu açdığımız zaman minaları artıq bayraqlanmış görəcəyik. Gəlin bunu edək.

Bilirik ki, OR əməliyyatı **_0x0f_**-i **_0x8f_**-ə çevirir, elə deyilmi?  
Bir az əlləşdikdən və ChatGPT-dən istifadə etdikdən sonra tapdım ki, **_0x0f_**-i **_0x81_** ilə XOR etsək, **_0x8e_** alacağıq. Bunu etmək üçün **_x32dbg_**-də sadəcə bu dəyişikliyi patch kimi tətbiq edin:

```text
010036FA | or byte ptr ds:[eax],80   --> xor byte ptr ds:[eax],81
```

Nəticə budur:

<figure>
  <img src="/assets/img/minesweeper/1-RwwczteT_z1cN8IzCRcx-g.gif" alt="screenshot" loading="lazy">
</figure>
