---
layout: post
title: Building a mixed-mode stack walker - Part 2
date: '2011-04-28 20:40:00'
tags:
- clr
- dac
- debugging
- diagnostics
- idebugclient
- ixclrdataprocess
- stack-walk
- x64
---

(Part 1 is <a href="{% post_url 2011-04-28-building-a-mixed-mode-stack-walker-part-1 %}">here</a>) 


When I left off in Part 1, I had a stack-walker based on IDebug* that could successfully unwind a mixed-mode stack
  and resolve the native frames to symbols, but the managed sections of the stack were still unresolved. In this post
  I'll talk about how to resolve those managed frames to managed MethodDescs and turn those into names, all without
  using ICorDebug or the CLR profiling APIs. Also, this method will work on both live and dump targets. Just a friendly
  warning here though: none of the proceeding is documented or supported by MS and is subject to change at any time.
  Also, by using headers/idl from the SSCLI, you may be taking a dependency on that licensing (but I’m not a lawyer so
  don’t take my word on that.)

# Starting out – Reversing SOS

When I started the project, I had known a little about how SOS works internally, but nothing substantial. My first
  point was to learn as much about how it worked as possible, then use that to write my own implementation. 

I started by looking at ``mscordacwks.dll`` and ``sos.dll``. I knew the purpose of ``mscordacwks.dll`` was to abstract away the CLR data structures to external tools, so I figured that was the best start. Using dumpbin (part of the Windows
  Platform SDK) I looked at the export table of ``mscordacwks``. Only a few functions are exported (I talked about
  ``OutOfProcFunctionTableCallback`` last post), the most interesting for this project is ``CLRDataCreateInstance``.  


Googling around for that function turned up two interesting hits. The first was a link to <a
    href="http://msdn.microsoft.com/en-us/library/ms230814(v=VS.100).aspx">MSDN</a> which was fairly useless (why is it
  even documented?), and the second was a link to <a
    href="http://www.koders.com/noncode/fid8C64338002FBA00CA9B7FE32D9FE67F6B43DC74B.aspx">clrdata.idl</a> on koderz (a
  great site) from the <a
    href="http://www.microsoft.com/downloads/details.aspx?FamilyId=8C09FD61-3F26-4555-AE17-3121B4F51D4D">SSCLI</a>. For
  those unfamiliar with the SSCLI, it’s basically a dumbed-down version of the .NET 2.0 source MS released under a
  shared-source license. I actually took this opportunity to download the SSCLI, which turned out to be worth it as I
  referred back to the source many times during this project. 

The signature for ``CLRDataCreateInstance`` looks like this: 

```cpp
HRESULT CLRDataCreateInstance (
    [in]  REFIID           iid, 
    [in]  ICLRDataTarget  *target, 
    [out] void           **iface
);
```

So we need to figure out 1) the IID to create, and 2) what a ``ICLRDataTarget`` is.

# Figuring out the IID we want

The implementation of ``CLRDataCreateInstance`` is in ``clr\src\debug\daccess\daccess.cpp`` at the bottom of the file. The
  function creates a ``ClrDataAccess`` object, then QIs it for the IID we passed to ``CLRDataCreateInstance``. The
  implementation of ``ClrDataAccess`` is also in daccess.cpp, and looking at it's implementation of QueryInterface (and
  class declaration), we can see that the only useful interface (to us) it supports is ``IXCLRDataProcess``.
  ``IXCLRDataProcess`` is defined in ``clr\src\inc\xclrdata.idl``. You can use midl to generate a .h file from this .idl file,
  or just use the one included in the SSCLI. This will get us the IID of ``IXCLRDataProcess``
  (``5c552ab6-fc09-4cb3-8e36-22fa03c798b7``). 

# Implementing ICLRDataTarget

``ICLRDataTarget`` is defined in clrdata.idl (and clrdata.h in the platform SDK). The interfaces defines a lot of
  methods, but actually very few of them seem to be used by the IXCLRData* implementations. All we need is: 

* ``GetMachineType``
  * I hard coded mine to return ``IMAGE_FILE_MACHINE_AMD64``, in a cross-platform solution you'd want to return
        ``IMAGE_FILE_MACHINE_I386`` on 32 bit systems as well.
* ``GetPointerSize``
  * This is the easiest one, return ``sizeof(void*)``.
* ``GetImageBase``
  * I chose to take a dependency on IDebug* for this, so I used <a
          href="http://msdn.microsoft.com/en-us/library/ff547126(v=VS.85).aspx">IDebugSymbols3::GetModuleByModuleNameWide</a>.
        Fun fact though: this is only ever called for clr.dll. 
* ``ReadVirtual``
  * You can use <a
          href="http://msdn.microsoft.com/en-us/library/ff554359(v=VS.85).aspx">IDebugDataSpaces::ReadVirtual</a> here.

That's basically all you need to have a working ``ICLRDataTarget`` implementation.


Later on I found out that you can ask WinDBG for an ``IXCLRDataProcess`` through ``Ioctrl`` and ``IG_GET_CLR_DATA_INTERFACE``.
 This has the advantage that WinDBG will try to load the &quot;right&quot; version of mscordacwks for you. However, it 
 doesn't work if you're not running inside WinDBG. Conveniently though, the only case I can think of that you wouldn't be
  running inside windbg would be doing something to a live-process, in which case it's fine to just load mscordacwks from
   the framework directory.
{: .note}

# Putting it together – Resolving a managed IP to a MethodDesc

So at this point we have a working ``ICLRDataTarget`` implementation, we have an IID, and we have a way to create that
  IID. Using ``CLRDataCreateInstance(__uuidof(IXCLRDataProcess), myDataTarget, (PVOID*)&pDac)``, we get an instance of ``IXCLRDataProcess`` bound to our ``IDebugClient`` through our
  ``ICLRDataTarget`` implementation. 

There's a few ways to resolve an IP to a method name now that we have an ``IXCLRDataProcess``, I'll go over two of them.
  The first is to use ``IXCLRDataProcess::GetRuntimeNameByAddress`` and pass an IP. This is probably the simplest method,
  but doesn't get you as much information. In our case however, all I wanted was the name, so this was enough. 

# IXCLRDataProcess::Request

The second brings us to what, in my opinion, is the most powerful feature of ``IXCLRDataProcess``, the ``Request(…)`` method.
  This is basically the ``IOCtrl`` of ``IXCLRDataProcess``; it takes a request code, and and input + output buffer. All the
  valid requests as of .NET 2.0 are defined in ``src\inc\dacprivate.h``, and there's a lot of them. All of the output
  structs contain a Request method which set up the input/output buffers correctly based on the request. 

Through experimentation, I've found a lot of these structs have changed definitions between the Rotor snapshot and
  .NET 4.0. Request returns E_INVALIDARG if the input or output buffers are mis-sized (but not only in that case.)
  There's two ways to figure out the correct buffer sizes: 

* Disassemble the corresponding Request method in mscordacwks and look at what it's expecting for a buffer size.
* Set a breakpoint on ClrDataAccess::Request() and debug windbg+sos calling the method you want. 

I usually went with #1 because it's a little faster. However, you need to be creative figuring out how the structure
  changed, and then adjust the struct in dacprivate.h accordingly. 


Back to resolving our managed IPs. One DAC Request is ``DacpMethodDescData``. This request is an example of one that
  changed between Rotor and .NET 4, the output buffer changed by 8 bytes (a x64 pointer). I removed the
  ``managedDynamicMethodObject`` field from my definition to get it to work. This request struct contains a couple helper
  methods, one being ``RequestFromIP``. Giving this a managed IP will resolve it to a MethodDesc. We can then take the
  ``MethodDescPtr`` from the result and pass it to ``GetMethodName``, also on the ``DacpMethodDescData`` request. 


# Conclusion

We've gone through a lot of work here, but at this point we can resolve any managed IP to a method name. The workflow
  looks like this: 

* Using the ``IDebugClient`` from part 1, create our ``ICLRDataTarget`` implementation. 
* Pass said ``ICLRDataTarget`` to ``CLRDataCreateInstance`` with ``IID = __uuidof(IXCLRDataProcess)``.
* For each frame, call ``GetRuntimeNameByAddress`` with the frame's IP, anything that succeeds is a managed method.
    Also, there may be cases where you'll have both a symbol name and a name from this call, ``GetRuntimeNameByAddress``
    should override the symbol name. 

There's definitely some room for improvement here. One of the biggest downsides is that there's no logic currently to
  find the &quot;right&quot; version of mscordacwks. SOS for example, will try to search around to find the best match,
  I currently just load it from Framework64\v4…\mscordacwks.dll. 

Next up: more advanced CLR inspection with IXCLRDataProcess. 
