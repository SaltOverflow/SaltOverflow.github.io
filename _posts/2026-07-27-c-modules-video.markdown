---
layout: post
title:  "C Modules: Video Overview"
date:   2026-07-27 09:00:00 -500
categories: jekyll update
---
C has a long history, spanning and being influenced by multiple computer architectures. For example, did you know that char isn't necessarily 8 bit? \[[App.1](#appendix-1)\]

Similarly, there was a time when computers couldn't fit code into main memory, so forward declarations were used to handle symbol references.

Many of these limitations are no longer the case, but the language still carries these design decisions. What makes C successful is that it has a huge amount of code and operating systems written in it; this in turn makes it difficult to change or deprecate features. 

With that said, how do we build a module system for C that maintains the ethos and design patterns that C programmers depend on? Specifically, I'm interested in a module system that you can use as a drop-in replacement for header files and forward declarations. This is the problem I've been tackling for the last year, the results of which I presented at ONPLS 2026.

Check out the re-recording of that talk in the link below. The video also contains a link to an implementation if you are interested in trying it out.

https://youtu.be/p8NpyBIRbEQ

The video is only 11 mins long, so I'm only able to go into a high-level overview of the system. The next article will go into the considerations behind making a module system for C, hopefully shedding some light on why certain decisions were taken.

## Appendix 1
The C standard requires the number of bits in a byte (char is a byte) is at least 8. Some historical systems used more, like the Cray-1, used 64 bits. There are even some that used less, like the ICL 1900, which used 6 bits! The latter had a specialized, proprietary C compiler written for it, though this was back in the 1980s. [1] Nowadays, you can pretty much rely on a char being 8 bit, as these exotic systems are obsolete (the first Raspberry Pi was benchmarked as being 4.5 times faster than the Cray-1 [2]). POSIX mandates that a char is 8 bits. [3]

[1] https://kids.kiddle.co/International_Computers_Limited

[2] http://www.roylongbottom.org.uk/Cray%201%20Supercomputer%20Performance%20Comparisons%20With%20Home%20Computers%20Phones%20and%20Tablets.htm#anchor22

[3] "The values for the limits {CHAR_BIT}, {SCHAR_MAX}, and {UCHAR_MAX} are now required to be 8, +127, and 255, respectively." https://pubs.opengroup.org/onlinepubs/9699919799/basedefs/limits.h.html#tag_13_23_10
