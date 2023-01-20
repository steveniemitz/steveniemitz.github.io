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

<p>(Part 1 is <a href="/building-a-mixed-mode-stack-walker-part-1">here</a>)</p>

  <p>When I left off in Part 1, I had a stack-walker based on IDebug* that could successfully unwind a mixed-mode stack and resolve the native frames to symbols, but the managed sections of the stack were still unresolved.&#160; In this post I'll talk about how to resolve those managed frames to managed MethodDescs and turn those into names, all without using ICorDebug or the CLR profiling APIs.&#160; Also, this method will work on both live and dump targets.&#160; Just a friendly warning here though: none of the proceeding is documented or supported by MS and is subject to change at any time.&#160; Also, by using headers/idl from the SSCLI, you may be taking a dependency on that licensing (but I’m not a lawyer so don’t take my word on that.)</p>

  <h1>Starting out – Reversing SOS</h1>  <p>When I started the project, I had known a little about how SOS works internally, but nothing substantial.&#160; My first point was to learn as much about how it worked as possible, then use that to write my own implementation.</p>

  <p>I started by looking at mscordacwks.dll and sos.dll.&#160; I knew the purpose of mscordacwks.dll was to abstract away the CLR data structures to external tools, so I figured that was the best start.&#160; Using dumpbin (part of the Windows Platform SDK) I looked at the export table of mscordacwks.&#160; Only a few functions are exported (I talked about OutOfProcFunctionTableCallback last post), the most interesting for this project is CLRDataCreateInstance. </p>

  <p>Googling around for that function turned up two interesting hits.&#160; The first was a link to <a href="http://msdn.microsoft.com/en-us/library/ms230814(v=VS.100).aspx">MSDN</a> which was fairly useless (why is it even documented?), and the second was a link to <a href="http://www.koders.com/noncode/fid8C64338002FBA00CA9B7FE32D9FE67F6B43DC74B.aspx">clrdata.idl</a> on koderz (a great site) from the <a href="http://www.microsoft.com/downloads/details.aspx?FamilyId=8C09FD61-3F26-4555-AE17-3121B4F51D4D">SSCLI</a>.&#160; For those unfamiliar with the SSCLI, it’s basically a dumbed-down version of the .NET 2.0 source MS released under a shared-source license.&#160; I actually took this opportunity to download the SSCLI, which turned out to be worth it as I referred back to the source many times during this project.</p>

  <p>The signature for CLRDataCreateInstance looks like this:</p>

  <pre>HRESULT CLRDataCreateInstance (
    [in]  REFIID           iid, 
    [in]  ICLRDataTarget  *target, 
    [out] void           **iface
);</pre>

<pre><span style="font-family: verdana">So we need to figure out 1) the IID to create, and 2) what a ICLRDataTarget is.</span></pre>

<h1>Figuring out the IID we want</h1>

<p>The implementation of CLRDataCreateInstance is in clr\src\debug\daccess\daccess.cpp at the bottom of the file.&#160; The function creates a ClrDataAccess object, then QIs it for the IID we passed to CLRDataCreateInstance.&#160; The implementation of ClrDataAccess is also in daccess.cpp, and looking at it's implementation of QueryInterface (and class declaration), we can see that the only useful interface (to us) it supports is IXCLRDataProcess.&#160; IXCLRDataProcess is defined in clr\src\inc\xclrdata.idl.&#160; You can use midl to generate a .h file from this .idl file, or just use the one included in the SSCLI.&#160; This will get us the IID of IXCLRDataProcess (5c552ab6-fc09-4cb3-8e36-22fa03c798b7).</p>



<h1>Implementing ICLRDataTarget</h1>

<p>ICLRDataTarget is defined in clrdata.idl (and clrdata.h in the platform SDK).&#160; The interfaces defines a lot of methods, but actually very few of them seem to be used by the IXCLRData* implementations.&#160; All we need is:</p>



<ul>
  <li>GetMachineType 
    <ul>
      <li>I hard coded mine to return IMAGE_FILE_MACHINE_AMD64, in a cross-platform solution you'd want to return IMAGE_FILE_MACHINE_I386 on 32 bit systems as well. </li>
    </ul>
  </li>

  <li>GetPointerSize 
    <ul>
      <li>This is the easiest one, return sizeof(void*). </li>
    </ul>
  </li>

  <li>GetImageBase 
    <ul>
      <li>I chose to take a dependency on IDebug* for this, so I used <a href="http://msdn.microsoft.com/en-us/library/ff547126(v=VS.85).aspx">IDebugSymbols3::GetModuleByModuleNameWide</a>.&#160; Fun fact though: this is only ever called for clr.dll. </li>
    </ul>
  </li>

  <li>ReadVirtual 
    <ul>
      <li>You can use <a href="http://msdn.microsoft.com/en-us/library/ff554359(v=VS.85).aspx">IDebugDataSpaces::ReadVirtual</a> here. </li>
    </ul>
  </li>
</ul>

<p>That's basically all you need to have a working ICLRDataTarget implementation.&#160; (Side note: later on I found out that you can ask WinDBG for an IXCLRDataProcess&#160; through Ioctrl and IG_GET_CLR_DATA_INTERFACE.&#160; This has the advantage that WinDBG will try to load the &quot;right&quot; version of mscordacwks for you.&#160; However, it doesn't work if you're not running inside WinDBG.&#160; Conveniently though, the only case I can think of that you wouldn't be running inside windbg would be doing something to a live-process, in which case it's fine to just load mscordacwks from the framework directory.)</p>



<h1>Putting it together – Resolving a managed IP to a MethodDesc</h1>

<p>So at this point we have a working ICLRDataTarget implementation, we have an IID, and we have a way to create that IID.&#160; <span style="font-family: &#39;Courier New&#39;"><span style="font-family: verdana">Using</span> CLRDataCreateInstance(<span style="color: #0000ff">__uuidof</span>(IXCLRDataProcess), myDataTarget, (PVOID*)&amp;pDac)</span>, we get an instance of IXCLRDataProcess bound to our IDebugClient through our ICLRDataTarget implementation.</p>



<p>There's a few ways to resolve an IP to a method name now that we have an IXCLRDataProcess, I'll go over two of them.&#160; The first is to use IXCLRDataProcess::GetRuntimeNameByAddress and pass an IP.&#160; This is probably the simplest method, but doesn't get you as much information.&#160; In our case however, all I wanted was the name, so this was enough.</p>



<h1>IXCLRDataProcess::Request</h1>

<p>The second brings us to what, in my opinion, is the most powerful feature of IXCLRDataProcess, the Request(…) method.&#160; This is basically the IOCtrl of IXCLRDataProcess; it takes a request code, and and input + output buffer.&#160; All the valid requests as of .NET 2.0 are defined in src\inc\dacprivate.h, and there's a lot of them.&#160; All of the output structs contain a Request method which set up the input/output buffers correctly based on the request.</p>



<p>Through experimentation, I've found a lot of these structs have changed definitions between the Rotor snapshot and .NET 4.0.&#160; Request returns E_INVALIDARG if the input or output buffers are mis-sized (but not only in that case.)&#160; There's two ways to figure out the correct buffer sizes:</p>



<ol>
  <li>Disassemble the corresponding Request method in mscordacwks and look at what it's expecting for a buffer size. </li>

  <li>Set a breakpoint on ClrDataAccess::Request() and debug windbg+sos calling the method you want. </li>
</ol>

<p>I usually went with #1 because it's a little faster.&#160; However, you need to be creative figuring out how the structure changed, and then adjust the struct in dacprivate.h accordingly.</p>



<p>Back to resolving our managed IPs.&#160; One DAC Request is DacpMethodDescData.&#160; This request is an example of one that changed between Rotor and .NET 4, the output buffer changed by 8 bytes (a x64 pointer).&#160; I removed the managedDynamicMethodObject field from my definition to get it to work.&#160; This request struct contains a couple helper methods, one being RequestFromIP.&#160; Giving this a managed IP will resolve it to a MethodDesc.&#160; We can then take the MethodDescPtr from the result and pass it to GetMethodName, also on the DacpMethodDescData request.</p>



<h1>Conclusion</h1>

<p>We've gone through a lot of work here, but at this point we can resolve any managed IP to a method name.&#160; The workflow looks like this:</p>



<ul>
  <li>Using the IDebugClient from part 1, create our ICLRDataTarget implementation. </li>

  <li>Pass said ICLRDataTarget to CLRDataCreateInstance with IID = __uuidof(IXCLRDataProcess). </li>

  <li>For each frame, call GetRuntimeNameByAddress with the frame's IP, anything that succeeds is a managed method.&#160; Also, there may be cases where you'll have both a symbol name and a name from this call, GetRuntimeNameByAddress should override the symbol name. </li>
</ul>

<p>There's definitely some room for improvement here.&#160; One of the biggest downsides is that there's no logic currently to find the &quot;right&quot; version of mscordacwks.&#160; SOS for example, will try to search around to find the best match, I currently just load it from Framework64\v4…\mscordacwks.dll.</p>



<p>Next up: more advanced CLR inspection with IXCLRDataProcess.</p>

