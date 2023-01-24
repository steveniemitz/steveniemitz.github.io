---
layout: post
title: Building a mixed-mode stack walker - Part 1
date: '2011-04-28 20:38:00'
tags:
- clr
- debugging
- diagnostics
- idebugclient
- stack-walk
- x64
---

A project I've been working on recently is a tool to capture the stack trace of all running threads in a
process. The tool is used in response to a monitoring event to gather information about the process at the time
of the event firing. Gathering this information needs to be fast (sub-second, preferably < 100ms), so using
CDB, loading SOS (or sosex) and running ``~*e!clrstack`` or ``~*e!mk`` or similar wasn't an option, since it takes far too long. Also, as a secondary goal I wanted to be able to allow this to operate on a dump file as well as a live
process, and also be as non-invasive as possible. That ruled out using the CLR profiling APIs or MDbg (as a side
note, it seems like MDbg tends to randomly kill OS handles in the process it's attached to).
  

# Try #1
My initial attempts were to use <a
    href="http://msdn.microsoft.com/en-us/library/ms680650(v=vs.85).aspx">dbghelp!StackWalk64</a> to get the full
  callstack, however, it had a lot of trouble traversing through managed frames on an x64 process. I'll talk a
  little bit about how x64 stack walking works and what the problems I ran into were.

## An aside on x64 stack unwinding
<p>In the x64 ABI, there's <a href="http://msdn.microsoft.com/en-us/library/ms235286(v=VS.100).aspx">only one calling
    convention</a>, and all code generators must use this convention in order for stack unwinding to work
  reliably. An interesting part of the convention is how unwind data is stored so stack unwinding can happen at
  runtime. x64's calling convention doesn't use a base pointer for each frame (EBP in x86), so there needs to be
  data somewhere about how to find the return address of each frame on the stack. This data is actually baked into
  the PE header of every DLL/EXE. </p>
 
> "But Steve! How do you unwind a managed callstack? There's no PE to embed the unwind data into, since it's JITed at runtime!"

Well, now we're jumping into semi-undocumented-land. A function exists ([RtlInstallFunctionTableCallback](http://msdn.microsoft.com/en-us/library/ms680595(VS.85).aspx)) that allows
  systems doing runtime codegen to handle the function table data themselves. There's actually a great blog post
  that goes into more detail <a
    href="http://blogs.msdn.com/b/ntdebugging/archive/2010/05/12/x64-manual-stack-reconstruction-and-stack-walking.aspx">here</a>
  The CLR uses this to install a callback function to provide function table data when requested.

>  "But Steve! How can you run that code when you're not in the same process!?!" (eg a debugger)

Well thankfully the people at Microsoft thought about that, the last parameter of ``RtlInstallFunctionTableCallback`` is
the name of a DLL that exports a function named ``OutOfProcFunctionTableCallback``. Debuggers/etc can use this
callback function to access the function tables in cases where there isn't a live process or code can't be run in the
process. If you look at the exports (``dumpbin /exports``) on mscordacwks.dll, you'll see it exports
``OutOfProcFunctionTableCallback``. 

For &quot;normal&quot; (native) x64 frames, dbghelp provides <a
    href="http://msdn.microsoft.com/en-us/library/ms681327(v=vs.85).aspx">SymFunctionTableAccess64</a> to resolve an IP
  to a function table entry (StackWalk64 calls this internally, it's usually passed as parameter 7
  ``FunctionTableAccessRoutine``). However, the built in functions seem to break down on a mixed mode
  stack. In my attempts I couldn't get StackWalk64 to get past certain managed frames. I got as far as
  trying to reverse engineer the function table linked list (you can get it with <a
    href="http://msdn.microsoft.com/en-us/library/bb432427(v=vs.85).aspx">RtlGetFunctionTableListHead</a>) and manually
  calling the callback in mscordacwks from my own callback installed with <a
    href="http://msdn.microsoft.com/en-us/library/ms681361(v=VS.85).aspx">SymRegisterFunctionEntryCallback64</a>, but
  was never successful. If anyone knows how to get ``SymFunctionTableAccess64`` to &quot;play nice,&quot; with managed
  code, I'd be interested to hear.
  

# Try #2

As an alternative, I looked into using the <a href="http://msdn.microsoft.com/en-us/library/ff549827(v=vs.85).aspx">IDebugClient</a> APIs exposed in
dbgeng.dll. This DLL is the core of windbg, cdb, etc and actually, the IDebug* APIs are very easy to use.&#160;
The biggest bonus is that any code you write using these APIs instantly supports both live and dump debugging
(assuming you stick to only the APIs, ironically, the steps I describe below only work on live targets, but is fairly
easy to adapt for dump debugging too).

The IDebugClient COM object (and others) are all created through <a
    href="http://msdn.microsoft.com/en-us/library/ff540469(v=VS.85).aspx">DebugCreate</a>, and the workflow here is
  pretty simple.

* ``DebugCreate`` an ``IDebugClient``
* Call ``AttachProcess`` on the client (in my case I used ``DEBUG_ATTACH_NONINVASIVE | DEBUG_ATTACH_NONINVASIVE_NO_SUSPEND``
  which makes sure the debugger doesn't actually do anything to the process).
* QI the ``IDebugClient`` for ``IDebugControl4`` (or DebugCreate it)
* For each thread in the target process:
  * ``OpenThread`` 
  * ``SuspendThread`` 
     * ``GetThreadContext``
     * <a
        href="http://msdn.microsoft.com/en-us/library/ff545748(v=VS.85).aspx">IDebugControl4::GetContextStackTrace</a>

  * ``ResumeThread``
  * ``CloseHandle``

Following this simple(?) 9 step process will get you a nice ``DEBUG_STACK_FRAME[]`` for each thread in the target
  process. In my tests, the whole step 4 (the only invasive part of the process) took basically no measurable
  amount of time. The slow part is symbol resolution (might need to hit a symbol server).

You might be curious why IDebug* is able to walk mixed-mode stacks correctly while StackWalk64 can't. Well, if
you put a breakpoint on ``OutOfProcFunctionTableCallback`` in mscordacwks, you can see that ``IDebugControl`` is passing in a custom function table callback (``dbgeng!SwFunctionTableCallback``) to ``StackWalk64``, and not just using the stock
``SymFunctionTableAccess64`` function in dbghelp. I suspect there's some magic occurring inside internally that gets
everything to work.

# Putting it together: Symbol resolution

The final step of the native stack walk is resolving the IPs for the native frames to symbol names.&#160;
  ``IDebugSymbols`` makes this simple, with <a
    href="http://msdn.microsoft.com/en-us/library/ff547186(v=VS.85).aspx">IDebugSymbols3::GetNameByOffsetWide</a>.&#160;
  This is basically the equivalent to <a
    href="http://msdn.microsoft.com/en-us/library/ms681323(v=VS.85).aspx">SymFromAddr</a> (but supports unicode
  symbols). Again, you can just QI the ``IDebugClient`` instance from step 1 for ``IDebugSymbols3`` then call
  ``GetNameByOffsetWide`` for each frame's IP. It will fail for some of the managed frames (some frames, such as ones
  in ngen'd assemblies might resolve &quot;successfully&quot;) but will hopefully succeed for all the native frames.

<p>Note you probably need to set up the symbol client with <a
    href="http://msdn.microsoft.com/en-us/library/ff556802(v=VS.85).aspx">IDebugSymbols::SetSymbolPath</a>. One
  big &quot;gotcha&quot; with symbol server access is that, if your process is running as a service, the symbol server
  will try to use a proxy server unless explicitly told not to. A full explanation is on <a
    href="http://msdn.microsoft.com/en-us/library/ff539229(VS.85).aspx">MSDN here</a>.</p>
<p>At this point, we have a full stack for every thread, and have resolved all the native frames to symbols.&#160;
  Threads running managed code still have big gaps of unresolved frames. Next up: Resolving the managed frames and
  getting more CLR diagnostic info.</p>


<p>(Part 2 <a href="{% post_url 2011-04-28-building-a-mixed-mode-stack-walker-part-2 %}">here</a>)</p>