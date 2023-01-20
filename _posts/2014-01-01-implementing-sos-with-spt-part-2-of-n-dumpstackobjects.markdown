---
layout: post
title: Implementing SOS with SPT - Part 2 of N - DumpStackObjects
date: '2014-01-01 20:15:00'
tags:
- clr
- debugging
- ixclrdataprocess
- sos
- windbg
---

This is part 2 of N of my SOS series.

- [Part1]({% post_url 2013-12-30-implementing-sos-with-spt-part-1-of-n-dumpobj %})
- [Part2]({% post_url 2014-01-01-implementing-sos-with-spt-part-2-of-n-dumpstackobjects %})
- [Part3]({% post_url 2014-01-04-implementing-sos-with-spt-part-3-of-n-dumpmd-ip2md %})

For part 2, I’m going to talk about ``DumpStackObjects``.  This one is fairly straight forward, and we already have most of the functionality required built in SPT, so all that is left is to hook it up.

The general algorithm for scraping the stack is this:

- Find the stack limits (stackBase, stackLimit)
    - stackBase can be pulled from the current thread’s TEB (thread environment block)  
    - stackLimit is the value of [e/r]sp for that thread.
- Get the bounds of the GC heap segments.  This algorithm can be found in <a href="https://github.com/steveniemitz/SDbgExt2/blob/master/SDbgExt/SDbgExt_EnumHeapObjects.cpp">CSDbgExt::EnumHeapSegments</a>.  
- Start at stackLimit, moving 1 pointer at a time, until we hit stackBase  
    - Read a pointer at our current stack location 
    - Use ``IClrProcess::IsValidObject`` to check if the address is valid or not 
    - If valid, also make sure it’s within the GC heap segments found in (2) 
    - We can find the MT using ``IXCLRDataProcess3::GetObjectData``.  For strings, we can read their value easily using ``IXCLRDataProcess3::GetObjectStringData``.
    
Once we’ve done this, there’s one more thing to consider: some registers may also contain CLR objects.  We can evaluate all the registers in a similar way to #3 above.

You can find the source for this on <a href="https://github.com/steveniemitz/SDbgExt2/blob/master/SDbgExt/DumpStackObjects.cpp">github here</a>.

