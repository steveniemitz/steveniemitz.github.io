---
layout: post
title: Introducing SPT for .NET 4.5
date: '2013-06-30 21:11:34'
---

<p>This has been a post long in the making.&#160; I’d written this in December/January as a complete re-do of SPT for .NET 4.0,&#160; <em>(Side Note: Interestingly enough, in the mean time MS released ClrMD, which is similar to the direction I was going, but still significantly different (I’m focusing more on a lower level interface to be plugged into WinDBG, ClrMD is focused more on a managed vector).) </em>but never released it publically.</p>

  <p>I’ve decided this time around to release the entire library under the GPL.&#160; For anyone who would like to use this in commercial software, contact me and we can probably work something out.&#160; I’ll be pushing the full source tree to my github account shortly.&#160; <a href="https://github.com/steveniemitz/SDbgExt2"><u><font color="#0066cc">https://github.com/steveniemitz/SDbgExt2</font></u></a>.</p>

  <p>The initial release supports most of the original SPT commands:</p>

  <ul>   <li>DumpAspNetRequests</li>    <li>DumpDictionary</li>    <li>DumpSqlConnectionPools (aka !sqlblame)</li>    <li>DumpThreadPoolQueues</li>    <li>FindHttpContext</li>    <li>GetDelegateMethod</li> </ul>  <p>In the next few days, I’ll be posting more about SDbgExt2 (and the WinDBG extension, SPT), it’s inner workings, and how the new IXCLRDataProcess3 interface works.&#160; My main design goals for SDbgExt2 were:</p>

  <ul>   <li>Easily support a managed wrapper interface</li>    <li>Separate the core debugging interface from the “value add” interface.&#160; As you can see, SDbgCore is a fairly small static library, which SPT links against.</li>    <li>Provide a cleaner code base which is easier to maintain and enhance in the future, as well as which abstracts IXCLRDataProcess as much as possible.</li>    <li>Have near-100% code coverage via unit tests from the start.&#160; This is mostly achieved with pre-canned mini-dumps which the unit tests can operate on.&#160; (<em>Side note: for now I’m not releasing the mini-dumps as they contain machine-specific data, I may in the future release them if I can prove they contain no personal data, but for now the unit tests are being released as more of a reference guide.)</em></li> </ul>  <p><strong>I’m also posting pre-compiled, ready to run SPT.dll binaries for use in WinDBG (or elsewhere).</strong></p>

  <p>Download here:&#160; <a href="http://www.steveniemitz.com/Upload/SPT_2_NET45_x86.zip">x86</a> / <a href="http://www.steveniemitz.com/Upload/SPT_2_NET45_x64.zip">x64</a></p>

  <p>As always, feel free to post here or email me if you have any questions / comments.</p>

