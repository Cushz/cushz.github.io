---
layout: post
title:  "Debugging Doom instead of running it on the microwave"
date:   2025-05-10 15:44:35 +0400
categories: reverse-engineering
tags: [doom, reverse-engineering, game-hacking, cheat-engine, x32dbg]
lang: en
ref: doom
permalink: /en/reverse-engineering/2025/05/10/debugging-doom-instead-of-running-it-on-the-microwave.html
description: "So, recently I was seeing that everyone was trying to run doom in unusual devices such as microwave, Arduino, your cousin’s old jailbroken Nintendo and…"
canonical_url: "https://medium.com/@Cushz/debugging-doom-instead-of-running-it-on-the-microwave-fd3b6190cecd"
---

<figure>
  <img src="/assets/img/doom/0-6qPIlZRIK78c9wcY.png" alt="screenshot" loading="lazy">
</figure>

So, recently I was seeing that everyone was trying to run doom in unusual devices such as microwave, Arduino, your cousin’s old jailbroken Nintendo and such. This actually became a tradition among nerds. And I just thought that maybe I should try hacking doom.

First I have to find the game itself. There are lots of options to download the doom vanilla but in order to run it without emulator there is apparently a thing called chocolate doom. People who made this are saying that it is almost identical version of the doom with one difference with being able to run on latest operating systems such as windows 11 and so.

But we have to create some kind of plan in order to have better idea about what we actually do. The most basic thing is to increase the ammo when I shoot instead of decreasing. This should be enough for the blog. Lets dive into it:

### **Where is the ammo**

Let’s think for a minute what we should look for if we don’t know anything about the inner working of the program. So, there are lots of memory spaces which contain data about the game, including health, ammo, user name and etc. Basically what we should do is to find where our ammo is saved and changed each time. We can shoot and keep track of ammo digit with the help of the debugger but this is actually very annoying procedure.  
Instead of satisfying my ego by manually finding it, I will just use Cheat engine to find these memory addresses.

Cheat engine is also a debugger and memory scanner which helps to find specific value in the process memory. Let’s open the game, attach it to the debugger, shoot guns and keep track of the ammo count.

<figure>
  <img src="/assets/img/doom/1-z8vNkVyRUk3lqB8FO3i9jQ.png" alt="screenshot" loading="lazy">
</figure>

When i search for 46 i see that there are lots of memory addresses that are storing 46 as a value. Cheat engine has a feature which helps you the search the value again but with the previous value as well. This means, I can shoot again and find what address is being simultaneously changed.

<figure>
  <img src="/assets/img/doom/1-iiWwSuTNEhuMlEDusXZAtQ.png" alt="screenshot" loading="lazy">
</figure>

In here, we simply told cheat engine to find next scan where the value is being decreased by 5. It is better to have a narrowed down results.   
But which one is actually used for ammo.

if we right click on each of these addresses we will see the specific instruction that changes these values. But the first two of them will have increased count even though we are not shooting anything. But for the third one, the count is increased only when we shoot. This is our target.

<figure>
  <img src="/assets/img/doom/1-l7c2wkWm9qjldk1OLCxXug.png" alt="screenshot" loading="lazy">
</figure>

<figure>
  <img src="/assets/img/doom/1-aaShV-vOSoIkiI8QbCXOPg.png" alt="screenshot" loading="lazy">
</figure>

As you can see, what it does, is to subtract one from the value that is being stored in **_ebx+edx\*4+000000A4._**

> RE tip: the ebx being inside \[\] means that ebx is storing some address and it checks what is being stored on this address. If there wasn’t any \[\], it will directly substract it from the address itself

Now, we got the address that is responsible for change. Now what?

### Patching

We know that, in PE32 executables, (also in PE64) main code is being stored in .text most of the time. By using memory map functionality of **x32dbg**, we can see the sections of the doom.exe and analyze it.

<figure>
  <img src="/assets/img/doom/1-APbYhjd4-ZFkEnMeCniwZw.png" alt="screenshot" loading="lazy">
</figure>

Previously we have learned that the address was, **_0043057C_**.

<figure>
  <img src="/assets/img/doom/1-jolYPOt_A6gjBYEF7LFyJA.png" alt="screenshot" loading="lazy">
</figure>

At this point we will change **_sub_** command to **_add_** to make it work as we want. We can either patch the executable directly with x32dbg, or we can use frida to inject our exploit each time the executable is being used. For the sake of the simplicity we will go for the first option.

To do this, we will right click on the instruction and choose **_assemble_** option and change it.

<figure>
  <img src="/assets/img/doom/1-fbsqTEvSmeJoK0Ss7lGNnQ.png" alt="screenshot" loading="lazy">
</figure>

After that press **_ctrl+p_** and it will patch our executable by creating a copy.

### Little demo

<figure>
  <img src="/assets/img/doom/1-F-dqI1KmMBi7pV1Jz_G-Yw.gif" alt="screenshot" loading="lazy">
</figure>
