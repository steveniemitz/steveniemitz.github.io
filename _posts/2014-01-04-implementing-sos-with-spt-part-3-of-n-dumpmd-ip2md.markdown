---
layout: post
title: Implementing SOS with SPT - Part 3 of N - DumpMD & IP2MD
date: '2014-01-04 15:36:00'
tags:
- clr
- debugging
- ixclrdataprocess
- sos
- windbg
---

This is part 3 of N of my SOS series

- [Part1]({% post_url 2013-12-30-implementing-sos-with-spt-part-1-of-n-dumpobj %})
- [Part2]({% post_url 2014-01-01-implementing-sos-with-spt-part-2-of-n-dumpstackobjects %})
    
It's a Saturday, so I'm going to do a nice easy one today, DumpMD, and IP2MD.  I group these together because ip2md is just a level of indirection from DumpMD.

Let's start with a quick refresher of ``!dumpmd``:

```
0:036> !dumpmd 5b2351ec 
Method Name:  System.Web.HttpContext.Init(System.Web.HttpRequest, System.Web.HttpResponse)
Class:        5b220e1c
MethodTable:  5b458094
mdToken:      06002627
Module:       5b201000
IsJitted:     yes
CodeAddr:     5b3bb0f0
Transparency: Safe critical
```

Most things here are self explanatory. The only thing many people may not be familiar with is the "Transparency" row. Transparency is a core part of the .NET security model, I'm not going to get into it, but you can read about it <a href="http://msdn.microsoft.com/en-us/library/ee191569(v=vs.110).aspx">on MSDN</a>.

DumpMD is very simple because almost all of the displayed fields are obtained via ``IXCLRDataProcess3::GetMethodDescData``, which takes a methodDesc address and returns a ClrMethodDescData structure.  Name we can get via ``IXCLRDataProcess3::GetMethodDescName.``  The EEClass ("Class") can be retrieved via IXCLRDataProcess3::GetMethodTableData with the methodTable address on ClrMethodDescData.  All we're left with is Transparency, which we can get with ``IXCLRDataProcess3::GetMethodDescTransparencyData.``


IP2MD adds one layer on top of DumpMD, it figures out the MD first, then runs the rest.  Given an arbitrary IP, IXCLRDataProcess3 has a convenient method to resolve it to a MethodDesc, unsurprisingly named GetMethodDescPtrFromIP.  Calling this will give us the MethodDesc address for that IP, which we can then pass to DumpMD.


<p>As always, the source for this is <a href="https://github.com/steveniemitz/SDbgExt2/blob/master/SDbgExt/DumpMD.cpp">on my</a> <a href="https://github.com/steveniemitz/SDbgExt2/blob/master/SDbgExt/IP2MD.cpp">github</a>. </p>

