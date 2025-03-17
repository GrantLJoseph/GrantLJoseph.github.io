---
layout: post
title:  "The Worst Segfault* I've Ever Debugged"
date:   2025-03-17 04:25:00 -0400
category: Low-Level Programming
---
A while back, I encountered an error I had never heard of before: a bus error. It reported similarity to a segfault, broke my tooling, and took most of a day to fix, a process that taught me a lot. This is how debugging it went, as best I can recall from memory.

# Let's Look at a Segfault
Below is example C++ code that causes a segfault. I start with a pointer to a valid int before setting a high bit to move it and adding 5 to break 4-byte the alignment.

```cpp
#include <iostream>

int main()
{
    int x{};
    int* ptr{&x};

    ptr = reinterpret_cast<int*>(
        (reinterpret_cast<unsigned long long int>(ptr) | 1ull << 60) + 5
    );

    std::cout << *ptr << '\n';

    return 0;
}
```

When run with GCC's address, leak, and runtime sanitizers, as I was at the time, the following error messages are displayed.
```
/home/grant/src/segfault/src/main.cpp:12:26: runtime error: load of misaligned address 0x100072c663c09025 for type 'int', which requires 4 byte alignment
0x100072c663c09025: note: pointer points here
<memory cannot be printed>
AddressSanitizer:DEADLYSIGNAL
=================================================================
==55662==ERROR: AddressSanitizer: SEGV on unknown address (pc 0x64c13d542563 bp 0x7ffc1b6be3f0 sp 0x7ffc1b6be360 T0)
==55662==The signal is caused by a READ memory access.
==55662==Hint: this fault was caused by a dereference of a high value address (see register values below).  Disassemble the provided pc to learn which register was used.
    #0 0x64c13d542563 in main /home/grant/src/segfault/src/main.cpp:12
    #1 0x72c665e2a1c9 in __libc_start_call_main ../sysdeps/nptl/libc_start_call_main.h:58
    #2 0x72c665e2a28a in __libc_start_main_impl ../csu/libc-start.c:360
    #3 0x64c13d542244 in _start (/home/grant/src/segfault/build/segfault+0x2244) (BuildId: 35c87306194d33738f402782ec488ab90a04a764)

AddressSanitizer can not provide additional info.
SUMMARY: AddressSanitizer: SEGV /home/grant/src/segfault/src/main.cpp:12 in main
==55662==ABORTING
```

At the top, the runtime sanitizer catches the alignment issue, after which the address sanitizer helpfully yells at me about the source of the invalid memory access. As a refresher, a segmentation fault is generally the result of a program accessing memory at an address it is not allowed to access.

# Enter the Bus Error
If only my sanitizers were this helpful back when the bus error struck. Instead of `SEGV`, I was told the error was a bus error. The message was otherwise almost identical. The invalid memory address accessed was shown as 0x0. A null pointer dereference! A classic. The error type was odd, but who cares? Should be an easy fix... right?

Of course not, or this post wouldn't exist. After a while of being unable to figure out how the pointer could possibly be null, I started printing the pointer, only to find it was most definitely not null! The address sanitizer was wrong. I used GDB to confirm the pointer's definitely-not-null property. 

# Address Space
I eventually managed to establish a working theory of what was happening. On a 64-bit system, general-purpose CPU registers are 64 bits wide. Because 2^64 is way more memory addresses than we could ever hope to use, most 64-bit systems only use the lower 48 bits of a pointer to store the actual memory address, resulting in a 48-bit address space. The remaining 16 bits can either be ignored completely or [used to store metadata for things like hardware memory tagging](https://googleprojectzero.blogspot.com/2023/08/summary-mte-as-implemented.html).

While I do not recall any details of the hardware I was using at the time, the conclusion I reached was that the bus error was occurring because the memory address was too high, with all of its 1 bits occurring in the portion of the pointer not actually used to store the address. From the perspective of printing and GDB, the pointer was absolutely non-null. The address sanitizer was probably looking at just the portion actually used by the hardware, which was all 0s.

Once I could confirm the pointer actually was invalid, I found a pointer arithmetic logic error that caused a previously valid pointer to blast off way out of bounds. An unremarkable end to a memorable debugging quest.

# The Missing Piece
The segfault demo above was my best attempt to recreate a bus error on my modern AMD system. On my system today, I only received a segfault, wrapping this in a bit of mystery and preventing me from confirming my old theory. Nevertheless, I hope this post serves as an informative look into a lesser-known error type. I will be eager to investigate in detail any bus errors I encounter in the future.