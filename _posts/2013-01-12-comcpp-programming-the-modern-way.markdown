---
layout: post
title: COM/C++ Programming – the modern way
date: '2013-01-12 14:35:00'
tags:
- atl
- c
- com
---

<p>For a new project I&rsquo;ve been working on I was forced into working with the world of C++ and COM for a sizable portion of it.&nbsp; In my past life I&rsquo;d previously worked on some large-ish projects involving COM (office interop in particular), but never on a fresh codebase from the ground up.</p>

<p>I&rsquo;ve found that life in &ldquo;modern COM&rdquo; is significantly different than the &ldquo;old school&rdquo; COM I had worked with before.&nbsp; A lot of this is probably very obvious to people working with COM/C++ for years, but for someone working in a mostly managed environment without much enterprise C++ experience, a lot of it wasn&rsquo;t obvious.&nbsp; Below are some observations.</p>

<h2>Smart pointers are ridiculously good.</h2>
<p>I know some people disagree with this (like <a href="http://blogs.msdn.com/b/ericlippert/archive/2003/09/16/53016.aspx">Eric Lipert</a>), but I can honestly say that my code using <em>CComPtr&lt;T&gt;</em> and equivalents has at least 95% less bugs than code using AddRef() and Release() everywhere.&nbsp; Further, the code is significantly more readable, and I probably was 50% more productive.&nbsp; Not having to reason about the correctness of a function (are all objects being released?) in almost all cases allows much more time for actually solving problems.</p>

<p>Further, using STL containers for dynamic memory allocation is significantly better than manually allocating storage.&nbsp; Need a dynamic LPWSTR?&nbsp; Use a vector&lt;WCHAR&gt; (although technically you can use a std::wstring and pass its buffer around, its not supported, and functions like length() behave differently than expected.&nbsp; Need a BYTE[]?&nbsp; Use a vector.&nbsp; You get the point.</p>

<h2>Lambda Expressions in C++11 are amazing.</h2>
<p>It&rsquo;s amazing, coming from C#, how much time and code lambda expressions can save you.&nbsp; Lambda expressions get you two major things.</p>

<ol>
<li>Anonymous, inline functions</li>
<li>Closures</li>
</ol>
<p>Manually creating closure classes and functors, or constantly casting PVOID state objects in callbacks wastes time and makes code ugly.&nbsp; Anonymous functions make using algorithms that take predicates or other functions much easier.&nbsp; Converting code from callback + PVOID pairs to use functors and lambdas made my code cleaner and easier to read.&nbsp; Being able to convert almost ANY instance method to a functor (or even normal function) with a couple lines of code is also a huge win.</p>

<h2>The ATL is useful, but horribly documented.</h2>
<p>I started my project not planning on using ATL, but after implementing <em>IUnknown</em> for the 10th time, decided this was a mistake.&nbsp; Unfortunately, I don&rsquo;t think there&rsquo;s a single good guide online for making a simple COM object with ATL.&nbsp; In addition, almost every tutorial I could find went something like &ldquo;open up the &lt;something&gt; wizard and click next.&rdquo;&nbsp; I hate wizards when it comes to coding with a passion so I promptly ignored the tutorials.&nbsp; Here&rsquo;s my 3 step guide:</p>

<ol>
<li>Inherit CComObjectBase and your COM interface (or just IUnknown) </li>
<li>Use <a href="http://msdn.microsoft.com/en-us/library/xfb1zk2x.aspx">BEGIN_COM_MAP, COM_INTERFACE_ENTRY, END_COM_MAP</a> to implement your interface </li>
<li>Create instances of your object with CComObject&lt;YourClass&gt;::CreateInstance. </li>
</ol>
<p>That&rsquo;s it!&nbsp; Why is that not on the front page of the ATL docs on MSDN?&nbsp; Note, you also need to define a CComModule somewhere in your library or you&rsquo;ll get null reference exceptions in AddRef() on create.&nbsp; Also, don&rsquo;t forget ATL starts your refcount at 0, so you should AddRef immediately after CreateInstance.</p>

<h2>IDL makes interface with .NET trivial</h2>
<p>MIDL + TLBImp can round trip COM objects (and data structs) with almost 100% fidelity.&nbsp; There are some gotchas though:</p>

<ul>
<li>It seems like MIDL randomly will capitalize/lowercase members of structs and parameters.&nbsp; I couldn&rsquo;t figure out why, or the pattern, which is annoying. </li>
<li>TLBImp (because of limitations of type libraries) can&rsquo;t handle variable sized OUT arrays, and instead define them as simple out variables in the managed interface.&nbsp; The only way around this is to edit the IL (with ILdasm, ILasm), and add <em>[] marshal([+1]) </em>to the parameter.&nbsp; The +1 tells the marshaller the index of the size parameter, 0 indexed left to right.&nbsp; For example, you might end up with: <em>[out] valuetype int<strong>[] marshal([+1])</strong> pValues</em> </li>
</ul>
<h2>Visual Studio 2012 C++ Unit Testing is really good</h2>
<p>TDD is a great paradigm, but until 2012 there were few (no?) good, integrated solutions to unit test C++ code.&nbsp; Visual Studio 2012 integrates C++ unit tests seamlessly into the IDE, just like managed unit tests.</p>

<h2>Conclusion</h2>
<p>The project that I started using these new technologies in is basically a rewrite of SPT for .NET 4.5 (I&rsquo;ll be blogging about that soon).&nbsp; It&rsquo;s codebase is significantly cleaner and less buggy than the first version of SPT.&nbsp; High code coverage with unit tests and smart pointers handling most COM objects contributed substantially to that.</p>
