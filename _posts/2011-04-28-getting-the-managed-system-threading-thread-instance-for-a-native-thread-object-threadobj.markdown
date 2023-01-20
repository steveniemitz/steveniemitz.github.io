---
layout: post
title: Getting the managed System.Threading.Thread instance for a native thread object
  (ThreadObj)
date: '2011-04-28 20:41:00'
tags:
- clr
- debugging
- sos
- x64
- x86
---

<p>There are times when I&rsquo;ve needed to find the managed thread object backing the native object output in !threads.&nbsp; The way I used to get it was to dump all thread objects (!dumpheap -mt &lt;thread MT&gt;) and match the managed thread Id field with the output from !threads.&nbsp; However, there&rsquo;s a better way.&nbsp;</p>

<p>There&rsquo;s a method on the native thread object &ldquo;clr!Thread::GetExposedObjectRaw&rdquo;, which is a simple 3 instruction method.&nbsp; Disassembling it we can see that a pointer to the managed thread object is 0x228 bytes into the native thread object.&nbsp; So, simply, the managed thread object is @ poi(poi(ThreadObj+0x228)).</p>

<p>I assume this is also there in the 2.0 CLR but I haven&rsquo;t looked.&nbsp; Also, this offset is for the 64 bit CLR, on a 32 bit system it&rsquo;s at ThreadObj+0x160.&nbsp; Obviously these offsets are subject to change with any CLR patch, so don&rsquo;t depend on them in any production code unless you understand the risks.</p>
