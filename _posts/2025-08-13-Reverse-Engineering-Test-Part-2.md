---
layout: post
title: Reverse Engineering - Test Class - Part 2
---

# Edits from previous

I only added `__declspec(noinline)` before every function in the header in order to prevent inlining

# Differences

First of all, both decompilers show the decompilation much clearer

![image.png](https://github.com/rtlcopymemory/rtlcopymemory.github.io/blob/master/images/Test2/1.png?raw=true)

![image.png]([image%201.png](https://github.com/rtlcopymemory/rtlcopymemory.github.io/blob/master/images/Test2/2.png?raw=true))

I have also now decompiled it using IDA:

![image.png]([image%202.png](https://github.com/rtlcopymemory/rtlcopymemory.github.io/blob/master/images/Test2/3.png?raw=true))

However now mentions of the `vftable` are almost complete gone

# Dynamic Analysis

x64dbg is smart and already comments on the side of the pure virtual call what it will be, but let’s analyze it further

![image.png]([image%203.png](https://github.com/rtlcopymemory/rtlcopymemory.github.io/blob/master/images/Test2/4.png?raw=true))

We look at RAX (which we know thanks to the x64dbg comments, contains the pointer to the `vftable`) `00007FF62D0133F0` (.rdata)

![image.png](https://github.com/rtlcopymemory/rtlcopymemory.github.io/blob/master/images/Test2/5.png?raw=true)

It contains a pointer (Little Endian): `00007FF62D0115D0` (.text)

![image.png]([image%205.png](https://github.com/rtlcopymemory/rtlcopymemory.github.io/blob/master/images/Test2/6.png?raw=true))

![image.png](https://github.com/rtlcopymemory/rtlcopymemory.github.io/blob/master/images/Test2/7.png?raw=true)

Once there, if we right-click and disassembly:

![image.png](https://github.com/rtlcopymemory/rtlcopymemory.github.io/blob/master/images/Test2/8.png?raw=true)

![image.png](https://github.com/rtlcopymemory/rtlcopymemory.github.io/blob/master/images/Test2/9.png?raw=true)

We can easily spot our marker bytes:

![image.png](https://github.com/rtlcopymemory/rtlcopymemory.github.io/blob/master/images/Test2/10.png?raw=true)

For completeness we take a screenshot of the memory map too:

![image.png](https://github.com/rtlcopymemory/rtlcopymemory.github.io/blob/master/images/Test2/11.png?raw=true)

# Class method calling convention

We can also notice that for every call of a method, the pointer to the class (usually referred to as `this`) is passed in `ECX` , this is also valid for virtual functions.

Ghidra cleverly names the first parameter `this` in the decompilation

![image.png](https://github.com/rtlcopymemory/rtlcopymemory.github.io/blob/master/images/Test2/12.png?raw=true)

While Binanry Ninja leaves it to the analyst

![image.png](https://github.com/rtlcopymemory/rtlcopymemory.github.io/blob/master/images/Test2/13.png?raw=true)

This is all consistent with what declared by the [official Microsoft documentation](https://learn.microsoft.com/en-us/cpp/build/x64-calling-convention?view=msvc-170) where simple the class instance is just passed as first parameter:

![image.png](https://github.com/rtlcopymemory/rtlcopymemory.github.io/blob/master/images/Test2/14.png?raw=true)

# Conclusions
The big takeaways from this are:
- `ECX` on class methods is a pointer to the class base address (`this`)
- `vftable` isn't obvious in most reverse engineering occasions
