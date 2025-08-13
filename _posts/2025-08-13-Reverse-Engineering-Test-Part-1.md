---
layout: post
title: Reverse Engineering - Test Class - Part 1
---

# Premise
I decided to start documenting what I do when I reverse engineer things in my free time in order to start building a portfolio I can show to others.  
My hope with this is to have tangible and proven experience to eventually move into the professional world of Reverse Engineering.

# Source Files
As a starting point, even if this isn't my first time looking at reversing executables, I decided to write some C++ code and then pass it to Ghidra and Binary Ninja so to see first hand how code gets transformed and optimized by compilers.
I used multiple decompilers to use this oppostunity to compare them and see the strength of each one.
The source code can be found [at this GitHub Repo](https://github.com/rtlcopymemory/ClassREStudy/tree/master).

# Precompiled Executable
[https://github.com/rtlcopymemory/ClassREStudy/releases/tag/0.0.1](https://github.com/rtlcopymemory/ClassREStudy/releases/tag/0.0.1)

# Analysis
The above files have been compiled using Visual Studio 2022 in Release mode

![image.png](https://github.com/rtlcopymemory/rtlcopymemory.github.io/blob/master/images/Test1/1.png?raw=true)

- Compiler Output
    
    ```
    1>------ Build started: Project: ClassREStudy, Configuration: Release x64 ------
    1>ClassREStudy.cpp
    1>TestClass.cpp
    1>Generating code
    1>Previous IPDB not found, fall back to full compilation.
    1>All 29 functions were compiled because no usable IPDB/IOBJ from previous compilation was found.
    1>Finished generating code
    1>ClassREStudy.vcxproj -> C:\Users\Sesilaso\source\repos\ClassREStudy\x64\Release\ClassREStudy.exe
    ========== Build: 1 succeeded, 0 failed, 0 up-to-date, 0 skipped ==========
    ```
    

## Decompilation
I’ve used both the Free version of **Binary Ninja** and **Ghidra** to decompile and see the differences between the two decompilers which in this case isn’t much.

The major issue I have spotted once decompiling these is that the **compiler automatically inlined** most if not all of the functions leaving only trace of the virtual functions in the `vftable` structure.

This can be noted thanks to my addition of volatile char arrays inside of each function denoting their beginning as a sort of “signature” for each function (This is just done to facilitate the reverse engineering).

- Binary Ninja
    
    ![image.png](https://github.com/rtlcopymemory/rtlcopymemory.github.io/blob/master/images/Test1/2.png?raw=true)
    
- Ghidra
    
    ![image.png](https://github.com/rtlcopymemory/rtlcopymemory.github.io/blob/master/images/Test1/3.png?raw=true)
    

Between the 2, Ghidra does a way better job ignoring the useless array assignment when decompiling to pseudo C rather than Binary Ninja where it shows all the assignments

![image.png](https://github.com/rtlcopymemory/rtlcopymemory.github.io/blob/master/images/Test1/4.png?raw=true)

Whichever is more useful or correct really depends on the context.

Besides the optimizations, the interesting part is how a class is initialized and allocated in memory

![image.png](https://github.com/rtlcopymemory/rtlcopymemory.github.io/blob/master/images/Test1/5.png?raw=true)

![image.png](https://github.com/rtlcopymemory/rtlcopymemory.github.io/blob/master/images/Test1/6.png?raw=true)

A new call allocates `0x10` (16) bytes of memory, 8 (64 bits) of which get instantly assigned the value of the pointer to the `vftable` (Virtual function Table)

`0x2a` = 42 in decimals

We notice how Binary Ninja puts the 42 assignment in the middle of the array assignment (as that’s where the Assembly also has is). This would have been much clearer in Ghidra or without the noise of the arrays.

This assignment is done by the constructor which also has the byte sequence `0xaa 0xbb 0xcc 0xdd 0xee 0xff` and is also inlined.

The most interesting thing here is the offset from rax which is the base memory address at which the object has been allocated from the `new` instruction. Binary Ninja here is slightly more intuitive as it clearly uses the offset in bytes (`rax + 8`) rather than Ghidra’s cast to a 64bit pointer and subsequent offset of 1 (1 * 64bit = 1 * 8 bytes).

This value is then assigned `123` and read back to be printed.

This reading back operation is quite hard to see in Binanry Ninja’s High Level IL or C Pseudocode as it’s bascially skipped but is still visible in the Disassembly

![image.png](https://github.com/rtlcopymemory/rtlcopymemory.github.io/blob/master/images/Test1/7.png?raw=true)

While it’s easily visible in Ghidra where it changes from the pointer arithmetics notation used above for the assignments to an equivalent array notation.

![image.png](https://github.com/rtlcopymemory/rtlcopymemory.github.io/blob/master/images/Test1/8.png?raw=true)

## Static Functions
As they’re also inlines we only see the assignments

![image.png](https://github.com/rtlcopymemory/rtlcopymemory.github.io/blob/master/images/Test1/9.png?raw=true)

![image.png](https://github.com/rtlcopymemory/rtlcopymemory.github.io/blob/master/images/Test1/10.png?raw=true)

## Virtual Functions
Both decompilers use offsets from `rax` which we have seen after the allocation has been assigned the `vftable`

![image.png](https://github.com/rtlcopymemory/rtlcopymemory.github.io/blob/master/images/Test1/11.png?raw=true)

![image.png](https://github.com/rtlcopymemory/rtlcopymemory.github.io/blob/master/images/Test1/12.png?raw=true)

However Ghidra here helps us a little more in the disassembler where it correctly comments next to the `call` instructions how it’s related to the `vftable`

One of the most interesting facts here to note is that those functions are still present in the general code and thanks to debug information are easily spotted:

![image.png](https://github.com/rtlcopymemory/rtlcopymemory.github.io/blob/master/images/Test1/13.png?raw=true)

Here we understand the importance and usage of the `vftable`.

![image.png](https://github.com/rtlcopymemory/rtlcopymemory.github.io/blob/master/images/Test1/14.png?raw=true)

Being nothing more than just a list of pointers to functions, this is how C++ implements the sharing of functionality given by inheritence. It’s also very important to keep in mind how this was the **first** thing copied into the allocated memory space.

# Conclusions
While this study has been useful, my next step is to try and make the compiler not optimize the code by inlining all the functions and try this again.
