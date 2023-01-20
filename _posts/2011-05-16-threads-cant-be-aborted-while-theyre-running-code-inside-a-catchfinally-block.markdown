---
layout: post
title: Threads can't be aborted while they're running code inside a catch/finally
  block
date: '2011-05-16 01:05:00'
tags:
- clr
- threading
- x64
---

<h1>(and more on the exciting world of thread aborts)</h1>
<p>It logically makes sense, if threads could be aborted while they were inside catch or finally blocks, those constructs would be fairly useless.&nbsp; For example:</p>

<pre style="background: white;"><span style="font-family: Consolas;"><span style="font-size: 9.8pt;">&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp; <span><span style="color: #2b91af;">SafeHandle</span></span> sh = <span><span style="color: #0000ff;">null</span></span>;<br />&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp; <span><span style="color: #0000ff;">try</span></span><br />&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp; {<br />&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp; sh = AllocateUnmanagedResource();<br />&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp; }<br />&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp; <span><span style="color: #0000ff;">finally</span></span><br />&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp; {<br />&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp; FreeUnmanagedResource(sh);<br />&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp; }</span></span></pre>
<p>If threads were abortable in finally blocks, there would be no guarantee that the resource allocated in the try block would ever be freed (note, this pattern is basically a using block). This rule doesn't apply to "rude" thread aborts, which are used in some hosting scenarios (SQLCLR for instance) and basically boil down to a call to TerminateThread(), but code enabling rude aborts generally is written to not even allow code that could leak to run.&nbsp; For instance, SQLCLR uses the CLR verifier and HPA to ensure code running in its domains won't leak if threads are terminated.</p>

<p>I came across this trying to investigate why some requests in an ASP.NET app would never time out. The culprit was code someone wrote similar to this:</p>

<pre style="background: white;"><span style="font-family: Consolas;"><span style="font-size: 9.8pt;">&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp; <span><span style="color: #0000ff;">try</span></span><br />&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp; {<br />&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp; RunSlowOperation();<br />&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp; }<br />&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp; <span><span style="color: #0000ff;">catch</span></span> (<span><span style="color: #2b91af;">Exception</span></span>)<br />&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp; {<br />&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp; <span><span style="color: #0000ff;">try</span></span><br />&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp; {<br />&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp; RunOtherSlowOperation();<br />&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp; }<br />&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp; <span><span style="color: #0000ff;">catch</span></span> (<span><span style="color: #2b91af;">Exception</span></span>) <br />&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp; { <br />&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp; <span><span style="color: #008000;">// Give up</span></span><br />&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp; }<br />&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp; }</span></span></pre>
<p>The idea was to try to do a (possibly long running) operation in one way, and if it failed, try it a different way. Besides being a horrible abuse of exception handling, the thread stops being abortable while RunOtherSlowOperation is executing, and thus can't be timed out by ASP.NET.&nbsp; The solution is simple, refactor the code to be something like this:</p>

<pre style="background: white;"><span style="font-family: Consolas;"><span style="font-size: 9.8pt;">&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp; <span><span style="color: #0000ff;">bool</span></span> doOtherStuff = <span><span style="color: #0000ff;">false</span></span>;<br />&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp; <span><span style="color: #0000ff;">try</span></span><br />&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp; {<br />&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp; RunSlowOperation();<br />&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp; }<br />&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp; <span><span style="color: #0000ff;">catch</span></span> (<span><span style="color: #2b91af;">Exception</span></span>)<br />&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp; {<br />&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp; doOtherStuff = <span><span style="color: #0000ff;">true</span></span>;<br />&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp; }<br />&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp; <span><span style="color: #0000ff;">if</span></span> (doOtherStuff)<br />&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp; {<br />&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp; <span><span style="color: #0000ff;">try</span></span><br />&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp; {<br />&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp; RunOtherSlowOperation();<br />&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp; }<br />&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp; <span><span style="color: #0000ff;">catch</span></span> (<span><span style="color: #2b91af;">Exception</span></span>)<br />&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp; {<br />&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp; <span><span style="color: #008000;">// Give up</span></span><br />&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp; }<br />&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp; }</span></span></pre>
<p>This way the time in catch blocks is reduced to a trivial amount of time.</p>

<p>An interesting side note is that I've never been able to find any accurate information online about how thread aborts work and abortable conditions.&nbsp; Most documentation only talks about aborting code inside an alertable wait/sleep/join (most locking primitives enter an alertable state) using APCs and <a href="http://msdn.microsoft.com/en-us/library/ms684954(v=vs.85).aspx">QueueUserAPC</a>.&nbsp; This isn't the whole story however.&nbsp; Consider this trivial case:</p>

<pre style="background: white;"><span style="font-family: Consolas;"><span><span style="color: #0000ff;"><span style="font-size: 9.8pt;">while</span></span></span><span style="font-size: 9.8pt;"> (<span><span style="color: #0000ff;">true</span></span>)<br />{<br />&nbsp;&nbsp;&nbsp; x++;<br />}</span></span></pre>
<p>This thread never enters an alertable wait state where it could process an APC, but it's definitely abortable.&nbsp; As far as I can tell, this feature is NOT implemented in the SSCLI 2.0.&nbsp; The code is in Thread::UserAbort, src\clr\vm\threads.cpp, around line 5048 and doesn't seem to handle this case.</p>

<p>I haven't spent a lot of time looking at the actual implementation but I suspect it works similar to this:</p>

<p>(Updated: 6/14/11, I looked into it and this is basically how the CLR does it.)</p>

<ol>
<li>Ensure the target thread is at a safe point (running managed code) </li>
<li>Suspend the thread </li>
<li>Use <a href="http://msdn.microsoft.com/en-us/library/ms679362(v=vs.85).aspx">GetThreadContext</a> to extract the IP and any registers that will be touched by the next steps from the thread's context. </li>
<li>Generate a small stub to:      <ol>
<li>Set up the stack to correctly unwind back to the IP saved in (3) (as if the stub had been called from the target thread). </li>
<li>Throw a thread abort exception. </li>
</ol> </li>
<li>Use <a href="http://msdn.microsoft.com/en-us/library/ms680632(v=vs.85).aspx">SetThreadContext</a> to change the IP (and maybe some other registers if needed) to point to the stub's entry point. </li>
<li>Resume the thread </li>
</ol>
<p>When the target thread resumes after step 7, it'll begin running inside out stub, which will handle setting up the stack and throwing the ThreadAbortException.</p>

<p>The (possibly) full list of abortable states that will abort (almost) immediately is as follows:</p>

<ol>
<li>Performing an alertable wait/sleep/join [will abort when the APC gets processed, typically immediately] </li>
<li>In managed code at a safe point&nbsp; (not in a catch/finally for example) [will abort immediately] </li>
<li>Suspended for GC [will abort when the GC finishes] </li>
</ol>
<p>Then there's a couple strange cases that don't abort immediately:</p>

<ol>
<li>User suspended (with Thread.Suspend()) threads will be aborted when they are resumed, but the call to Abort() will also throw an exception.</li>
<li>Threads in native code will be aborted when they transition back into the CLR.</li>
</ol>
<p>Hopefully this clears up some confusion.</p>
