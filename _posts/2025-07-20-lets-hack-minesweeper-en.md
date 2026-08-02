---
layout: post
title:  "Let’s hack minesweeper"
date:   2025-07-20 18:24:50 +0400
categories: reverse-engineering
tags: [minesweeper, reverse-engineering, ghidra, x32dbg, game-hacking, windows]
lang: en
ref: minesweeper
permalink: /en/reverse-engineering/2025/07/20/lets-hack-minesweeper.html
description: "Remember the old days when you didn’t have any internet but still wanted to play something on the computer? Such as chess, cake cooking and so on. And…"
canonical_url: "https://medium.com/@Cushz/lets-hack-minesweeper-7ec334ebf439"
---

<figure>
  <img src="/assets/img/minesweeper/0-n84wuzZ-MYN7w5xF.gif" alt="screenshot" loading="lazy">
</figure>

Remember the old days when you didn’t have any internet but still wanted to play something on the computer? Such as chess, cake cooking and so on.   
And then, there was a shitty nerve cracking game called minesweeper. You were supposed to blindly choose a cell and hope that it won’t explode. And of course, there was this “smile emoji” mf who did absolutely no help to win the game.  
I was not able to win that game, even now I just can’t. Then I realized, why I don’t apply RE skills in here and decided to write this absolutely useless blogpost.

PS: There are lots of minesweeper reversing blogposts as well, therefore this post will only show my experience and difficulties that I have faced, don’t consider this as a study material.

Hope you will enjoy.

I am going to use Ghidra in order to disassemble / decompile the source code and x32dbg for debugging purposes. In here, function names will be random as my PDB is causing problems (google PDB).

We know 2 things. This game is 2D board, we should expect something like this. And, this game is randomized, meaning that every time there the mines and the flags will be in different places. At this point, there should be random function. Let’s go for it.

Search for rand keyword in full codebase, and you will see 2 random functions, **_rand_** and **_srand_**. first, we will first start with rand. **_something\_random_** is function name that is renamed by me. As it is shown in here, this function is taking parameter and divides the random value with that parameter.

<figure>
  <img src="/assets/img/minesweeper/1-dZTyrgK4yKTWKLBk0HTmEg.png" alt="screenshot" loading="lazy">
</figure>

<figure>
  <img src="/assets/img/minesweeper/1-68kl74E8WHC9xqaay1CMEg.png" alt="something_random is renamed by me" loading="lazy">
  <figcaption>something_random is renamed by me</figcaption>
</figure>

Check the function in which the **_something\_random_** is referenced.

Referenced function is like this:

<figure>
  <img src="/assets/img/minesweeper/1-SQpKKa2k82oYvBJOmrD8ig.png" alt="screenshot" loading="lazy">
</figure>

<figure>
  <img src="/assets/img/minesweeper/1--vZc7Ksmwf95kEtw0qRfKw.png" alt="possible_mine_set is renamed by me as well" loading="lazy">
  <figcaption>possible_mine_set is renamed by me as well</figcaption>
</figure>

```text
iVar1 --> random_value / DAT_..5334
iVar2 --> random_value / DAT_..5338

DAT_01005340 is the pointer to the start of the game board array.
[iVar1 + 1 + (iVar2 + 1) * 32] --> coordinate of one tile
```

You may ask why it is being multiplied with 32, well the game is 32x32 pixels, thats why.

while loop checks if the tile is being set as a mined or not. If mined,  
it will keep looping, if not, it will make it mined by setting LSB. And this  
is done by making OR operation with 0x80.

To understand this better, lets add breakpoints to each of the **_something\_random_** call and see the values.

<figure>
  <img src="/assets/img/minesweeper/1-6aBAA_fjdCIgTXf76qQNDw.png" alt="screenshot" loading="lazy">
</figure>

We see the value that is being pushed to the stack and used as a parameter on the **_something\_random_** function. in both cases, we will get something like this:

```text
ivar1 = random_value / 9
ivar2 = random_value / 9
```

I am guessing that 9 is because of our game scale. It is 9x9 if you remember.

After that, we will see that it goes through the loop 10 times and possibly adds the mines. These mines are being added starting from this address: **_DAT\_01005340._**

**At this point, the logic that I have explained before, starts to execute. if the value is 0x0f, it will convert it to 0x8f my doing OR operation with 0x80.**

Meaning that there is high possibility that if we look for **_DAT\_01005340_**, we will the hex presentation of the game board.

<figure>
  <img src="/assets/img/minesweeper/1-0aoejCJ44zAPY5D80QL7CQ.png" alt="screenshot" loading="lazy">
</figure>

I continued to run the code, and it started to loop ten times between the first two breakpoints and then goes to the last breakpoint. This means, our mines have been set.

0x0F and 0x10 — will be side borders and empty parts (delimiters “\t”)  
 0x0E — **_if we flag_**  
 0x0D — **_if we question_**

0x8F — will be mine locations   
 0x8E — **_if we flag_**  
 0x8D — **_if we question_**

We can easily check it.

<figure>
  <img src="/assets/img/minesweeper/1-fKL5wov0KthsBYv9gu_MAg.png" alt="screenshot" loading="lazy">
</figure>

At this point I can win the game easily, but still, I am lazy to look for the memory dump each time. Let’ s patch this game so that it will be easier for me to just win. For patching idea, there was one researcher who decided to just convert **_0x8f’s_** to the **_0x8e_**. This will mean that when we open the game, we will see the mines flagged already. Let’s do this.

We know that OR operation converts **_0x0f_** to **_0x8f_** right?   
After struggling and using chatgpt, i found that if we XOR **_0x0f_** with **_0x81_**, we will get **_0x8e_**. To do this just make this change as a patch in **_x32dbg_**:

```text
010036FA | or byte ptr ds:[eax],80   --> xor byte ptr ds:[eax],81
```

This is the result:

<figure>
  <img src="/assets/img/minesweeper/1-RwwczteT_z1cN8IzCRcx-g.gif" alt="screenshot" loading="lazy">
</figure>
