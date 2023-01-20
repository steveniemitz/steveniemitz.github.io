---
layout: post
title: SPT - A WinDBG extension for debugging .NET applications
date: '2011-06-07 00:36:00'
tags:
- clr
- debugging
- windbg
- x64
- x86
---

*Update: I've written a newer/better version of this for .NET 4.5 and open sourced it, see [later posts](http://blog.steveniemitz.com/introducing-spt-for-net-45/) on my blog for more details.  This version is legaacy and no longer maintained.*

<p>In my spare time I've been working on a WinDBG extension containing some useful methods that automate pretty common .NET debugging tasks for me, and hopefully other people as well.&#160; The extension right now targets .NET 4<strike>/x64 only, as that's my primary development platform.&#160; I'll probably make a x86 build soon (especially if people request it), it shouldn't be very difficult to make it work on 32 bit platforms.</strike>&#160; Currently a few commands support DML (!DumpSqlConnectionPools is the most noticeable), I'll be adding support for DML to other commands soon.</p>
  <p>A few of the commands are targeted at ASP.NET/server apps (!DumpHttpContext, !DumpASPNetRequests, !DumpSqlConnectionPools, !DumpThreadPool for example), but most are fairly general purpose.&#160; In the next few blog posts I'll go into details about the implementation of each of the commands, but for now here's a list of them with quick summaries and examples.&#160; Feel free to comment/email me with any comments/suggestions.</p>
  <p><a href="http://www.steveniemitz.com/Upload/SPT_NET4_x64.zip">Download the extension here (x64)</a></p>
  <p><a href="http://www.steveniemitz.com/Upload/SPT_11_NET4_x86.zip">And the x86 version here</a></p>
  <p>&#160;</p>
  <h1>ASP.NET</h1>  <h2><span style="font-family: &#39;Courier New&#39;"><span style="font-weight: bold">!DumpASPNetRequests</span></span></h2>  <p>Prints out all threads with a HTTP context.    <br />Example output:</p>
  <pre>ThreadID&#160; HttpContext&#160;&#160;&#160;&#160;&#160;&#160; StartTime&#160;&#160;&#160;&#160;&#160;&#160;&#160;&#160;&#160;&#160;&#160;&#160; URL + QS <br />0123&#160;&#160;&#160;&#160;&#160; 0000000002f62b38&#160; 05/29/2011 18:23:02&#160;&#160; /hello/world.aspx&#160;&#160; <br />0124&#160;&#160;&#160;&#160;&#160; 0000000002f62c98&#160; 05/29/2011 18:23:05&#160;&#160; /test.aspx?w00t=1
<br />...</pre>

<pre>&#160;</pre>

<h2><span style="font-family: &#39;Courier New&#39;"><span style="font-weight: bold">!FindHttpContext</span></span></h2>

<p>Prints the context information for the current thread, including the managed thread object address (System.Threading.Thread), ExecutionContext, and Logical/IllogicalCallContext. 
  <br />Example output:</p>


<pre>OS ThreadId: e64, N: 0
Managed Thread @ 0000000002f58580
<br />Thread ExecutionContext @ 0000000002f64af8
<br />IllogicalCallContext @ 0000000002f64c40
<br />HostContext (HttpContext) @ 0000000002f64d80</pre>

<p>Note: of all the commands here, this is the most likely to break from service releases.&#160; It relies on reversed offsets to find the managed thread object from the unmanaged thread object.</p>


<h1>Hashtables/Dictionaries/Etc</h1>

<h2><span style="font-family: &#39;Courier New&#39;"><strong>!DumpDictionary (!DumpHashtable, !ddct, !dht)</strong></span></h2>

<p>Note: I used hashtable/dictionary pretty interchangeably for the rest of this post, anything that supports Hashtables also supports Dictionaries.</p>


<p>Iterates over all the key/value pairs in a dictionary, printing them out, and translating keys/values to strings if possible. 
  <br />Example:</p>


<pre>0:000&gt; !ddct 0000000002f621c0  <br />entry at: 0000000002f622e8 <br />key:      0000000000000001 [(null), 1] <br />value:    0000000002f62128 [System.String, &quot;test&quot;] <br />hashCode: 0x00000001
<p>&#160;</p>
<p>entry at: 0000000002f62300  <br />key:      0000000000000002 [(null), 2] <br />value:    0000000002f62150 [System.String, &quot;two&quot;]  <br />hashCode: 0x00000002</p>
</pre>

<p>These methods have a few switches you can use. </p>


<ul>
  <li>The first is <strong>-short</strong>, when used prints only the key/value/hash tripplet for each entry, useful for feeding into a script. </li>

  <li>The second is <strong>-kdo</strong> and -<strong>vdo</strong>.&#160; These switches allow you to pass a tokenized command in for the key/value in each entry.&#160; For instance, say you want to print out all the values in the dictionary, you could run 

    <p><span style="font-family: &#39;Courier New&#39;">!ddct -vdo &quot;!do %p&quot; 0000000002f621c0</span></p>

  </li>
</ul>

<p>These switches are one of my favorite features and have saved me a ton of time already.</p>


<p>Note: DumpDictionary and DumpHashtable are both actually the same function, the export table links them together.</p>


<p>Note 2: This also supports the HybridDictionary class, just use any of the above functions on it.</p>


<p>&#160;</p>


<h2><span style="font-family: &#39;Courier New&#39;"><strong>!DumpMemoryCache (!dmc)</strong></span></h2>

<p>Basically the same as !DumpDictionary, but goes through all the sub-hashtables inside the memory cache object.&#160; Doesn't support the same switches as !DumpDictionary yet though.</p>


<p>&#160;</p>


<h1>Other misc stuff</h1>

<h2><span style="font-family: &#39;Courier New&#39;"><span style="font-weight: bold">!FullStack (!fs)</span></span></h2>

<p>Prints out the mixed-mode callstack of the current thread. 
  <br />Example:&#160; (parameters snipped by me so it doesn't wrap on my blog)</p>


<pre><span style="font-family: &#39;Courier New&#39;">&#160;&#160;&#160;&#160;&#160;&#160; IP&#160;&#160;&#160;&#160;&#160;&#160;&#160; M&#160; Name 
<br />000000007745153a 0 ntdll!ZwRequestWaitReplyPort+a     <br />0000000076d42f58 0 KERNEL32!ConsoleClientCallServer+54     <br />0000000076d75321 0 KERNEL32!ReadConsoleInternal+1f1     <br />0000000076d8a762 0 KERNEL32!ReadConsoleA+b2     <br />0000000076d58e04 0 KERNEL32!TlsGetValue+899e    <br />000007feef4f17c7 0 clr!DoNDirectCall__PatchGetThreadCall+7b     <br />000007feec6834a1 1 DomainNeutralILStubClass.IL_STUB_PInvoke(...)+101     <br />000007feece2f58a 1 System.IO.__ConsoleStream.ReadFileNative(...)+ba     <br />000007feece2f3f2 1 System.IO.__ConsoleStream.Read(...)+62 
</span></pre>

<p>(Note: I've seen some version of dbghelp have problems resolving native methods in CLR.dll to symbols.&#160; The latest version seems to work fine though.)</p>


<p>&#160;</p>


<h2><span style="font-family: &#39;Courier New&#39;; font-weight: bold">!FormatDT</span></h2>

<p>Given a System.DateTime or TimeSpan, formats it as a string MM/dd/yyyy hh:mi:ss.&#160; I think psscor2 had something similar back in the day.</p>


<p>&#160;</p>


<h2><span style="font-family: &#39;Courier New&#39;"><span style="font-weight: bold">!EvalExpression (!evalexpr)</span></span></h2>

<p>Given an expression consisting of field access or hashtable lookups, evaluates the expression starting at a root object.</p>


<p>Fields are separated by periods just like C++/C#.&#160; Hashtable lookups are specified by the [ ] operator (like in C#) and are done in two ways. </p>


<ul>
  <li>The first is if the hashtable/dictionary key is a string, you can use a quoted string inside the [ ] operator to tell the evaluator you want to match a string.&#160; For example <span style="font-family: &#39;Courier New&#39;">_myHashtable[&quot;steve&quot;].lastName</span> will lookup the value in _myHashtable with the key of &quot;steve&quot;, then evaluate the lastName field.&#160; </li>

  <li>The second is by hashcode.&#160; If the expression inside the [ ] operator is an unquoted number, it's used to do a lookup by hashcode. </li>
</ul>

<p>Also, if the root object itself is a hashtable, it's legal for the expression to begin with simply a [ ] operator.&#160; For example, <span style="font-family: &#39;Courier New&#39;">[&quot;steve&quot;].lastName</span>.</p>


<p>Expressions ending with a &quot;$&quot; sign are evaluated to a string and the result string printed.&#160; If the final result isn't a string object, stuff will probably break.</p>


<p>Note: Hashtable lookups are actually O(n) (you'd need a full-out IL interpreter like in VS to do it in O(1)), so they might get slow on really big dictionaries.</p>


<p>&#160;</p>


<h2><span style="font-family: &#39;Courier New&#39;"><span style="font-weight: bold">!PrintDelegateMethod (!pdm)</span></span></h2>

<p>Attempts to resolve a delegate to a method name.&#160; This should work at least for non-multicast delegates (eg delegates without an invocation list) to static or instance methods.&#160; For instance method calls, the value in _methodPtr is simply a pointer to the managed method and resolved in a way similar to ip2md.&#160; For static methods, the value in _methodPtrAux is an entry into the JIT stub table, and used to get a MethodDesc handle to the method, which is then resolved to a name.</p>


<p>&#160;</p>


<h2><span style="font-family: &#39;Courier New&#39;"><span style="font-weight: bold">!DumpThreadPool</span></span></h2>

<p>Dumps the items currently in the .NET 4.0 ThreadPool queues, resolving work items to method names.&#160; This currently supports work items enqueued via ThreadPool.QueueUserWorkItem and Task[&lt;T&gt;].&#160; These are the only two classes in .NET 4 that currently implement IThreadPoolWorkItem.&#160; This method only works if ThreadPoolGlobals.useNewWorkerQueue = true.</p>


<p>There's a also special case: if a work item was created via BeginInvoke, DumpThreadPool will attempt to resolve it to the real method being called by looking at the MethodCall object's MethodInfo field.&#160; For this to work, the method being called must be System.Runtime.Remoting.Proxies.AgileAsyncWorkerItem.ThreadPoolCallBack, and the callback argument must be a System.Runtime.Remoting.Proxies.AgileAsyncWorkerItem.</p>


<p>&#160;</p>


<h2><span style="font-family: &#39;Courier New&#39;"><span style="font-weight: bold">!DumpSqlConnectionPools (!dsql)</span></span></h2>

<p>Dumps out all SQL connection pools in all app domains of the process.</p>


<p>Example output:</p>


<p><span style="font-family: &#39;Courier New&#39;">0:046&gt;&#160; !dumpsqlconnectionpools; 
    <br />[0] Factory @ 00000001df0614c8 

    <br />Connection String: &lt;snip&gt; 

    <br />PoolGroup:&#160;&#160;&#160; 000000011f0877a8 

    <br />&#160;&#160;&#160; SID:&#160; S-1-5-21-3430578067-4192788304-1690859819-9294 

    <br />&#160;&#160;&#160; Pool: 00000001bf077d08, State: 1, Waiters: 0, Size: 12 

    <br />&#160;&#160;&#160; Connections: 

    <br />&#160;&#160;&#160;&#160;&#160;&#160;&#160;&#160;&#160;&#160;&#160;&#160;&#160; ConnPtr&#160;&#160;&#160;&#160;&#160;&#160;&#160;&#160;&#160;&#160; State&#160; #Async&#160; Created&#160;&#160;&#160;&#160;&#160;&#160;&#160;&#160;&#160;&#160;&#160; Command/Reader&#160;&#160;&#160; Timeout&#160; Text 

    <br />&#160;&#160;&#160;&#160;&#160;&#160;&#160; [000] 00000001bf0788e8&#160;&#160;&#160;&#160;&#160; 1&#160;&#160;&#160;&#160;&#160;&#160; 0&#160; 6/10/2011 17:30:32 

    <br />&#160;&#160;&#160;&#160;&#160;&#160;&#160; [001] 000000015f1ebf80&#160;&#160;&#160;&#160;&#160; 1&#160;&#160;&#160;&#160;&#160;&#160; 0&#160; 6/10/2011 17:30:33 

    <br />&#160;&#160;&#160;&#160;&#160;&#160;&#160; [002] 000000013f30da08&#160;&#160;&#160;&#160;&#160; 1&#160;&#160;&#160;&#160;&#160;&#160; 0&#160; 6/10/2011 17:30:39 

    <br />&#160;&#160;&#160;&#160;&#160;&#160;&#160; [003] 000000013f3315e0&#160;&#160;&#160;&#160;&#160; 1&#160;&#160;&#160;&#160;&#160;&#160; 0&#160; 6/10/2011 17:30:54 

    <br />&#160;&#160;&#160;&#160;&#160;&#160;&#160; [004] 000000011f4852f8&#160;&#160;&#160;&#160;&#160; 1&#160;&#160;&#160;&#160;&#160;&#160; 0&#160; 6/10/2011 17:30:54 

    <br />&#160;&#160;&#160;&#160;&#160;&#160;&#160; [005] 00000001df1b5998&#160;&#160;&#160;&#160;&#160; 1&#160;&#160;&#160;&#160;&#160;&#160; 0&#160; 6/10/2011 17:30:54 

    <br />&#160;&#160;&#160;&#160;&#160;&#160;&#160; [006] 00000000ff41da90&#160;&#160;&#160;&#160;&#160; 1&#160;&#160;&#160;&#160;&#160;&#160; 0&#160; 6/10/2011 17:30:54 

    <br />&#160;&#160;&#160;&#160;&#160;&#160;&#160; [007] 00000001bf1c08f8&#160;&#160;&#160;&#160;&#160; 1&#160;&#160;&#160;&#160;&#160;&#160; 0&#160; 6/10/2011 17:30:54 

    <br />&#160;&#160;&#160;&#160;&#160;&#160;&#160; [008] 000000019f5f8080&#160;&#160;&#160;&#160;&#160; 1&#160;&#160;&#160;&#160;&#160;&#160; 0&#160; 6/10/2011 17:30:54 

    <br />&#160;&#160;&#160;&#160;&#160;&#160;&#160; [009] 000000015f2b4f90&#160;&#160;&#160;&#160;&#160; 1&#160;&#160;&#160;&#160;&#160;&#160; 0&#160; 6/10/2011 17:30:54 

    <br />&#160;&#160;&#160;&#160;&#160;&#160;&#160; [00a] 000000013f344120&#160;&#160;&#160;&#160;&#160; 1&#160;&#160;&#160;&#160;&#160;&#160; 0&#160; 6/10/2011 17:30:54 

    <br />&#160;&#160;&#160;&#160;&#160;&#160;&#160; [00b] 000000019f60ab18&#160;&#160;&#160;&#160;&#160; 1&#160;&#160;&#160;&#160;&#160;&#160; 0&#160; 6/10/2011 17:30:54</span></p>


<p><span style="font-family: &#39;Courier New&#39;">Connection String: &lt;snip&gt;; 
    <br />PoolGroup:&#160;&#160;&#160; 000000017f125730 

    <br />&#160;&#160;&#160; SID:&#160; S-1-5-21-3430578067-4192788304-1690859819-9294 

    <br />&#160;&#160;&#160; Pool: 000000017f4495b8, State: 1, Waiters: 0, Size: 4 

    <br />&#160;&#160;&#160; Connections: 

    <br />&#160;&#160;&#160;&#160;&#160;&#160;&#160;&#160;&#160;&#160;&#160;&#160;&#160; ConnPtr&#160;&#160;&#160;&#160;&#160;&#160;&#160;&#160;&#160;&#160; State&#160; #Async&#160; Created&#160;&#160;&#160;&#160;&#160;&#160;&#160;&#160;&#160;&#160;&#160; Command/Reader&#160;&#160;&#160; Timeout&#160; Text 

    <br />&#160;&#160;&#160;&#160;&#160;&#160;&#160; [000] 000000017f449f50&#160;&#160;&#160;&#160;&#160; 1&#160;&#160;&#160;&#160;&#160;&#160; 0&#160; 6/10/2011 17:30:54 

    <br />&#160;&#160;&#160;&#160;&#160;&#160;&#160; [001] 000000017f471280&#160;&#160;&#160;&#160;&#160; 1&#160;&#160;&#160;&#160;&#160;&#160; 0&#160; 6/10/2011 17:30:54 

    <br />&#160;&#160;&#160;&#160;&#160;&#160;&#160; [002] 00000000ff4305b8&#160;&#160;&#160;&#160;&#160; 1&#160;&#160;&#160;&#160;&#160;&#160; 0&#160; 6/10/2011 17:30:54 

    <br />&#160;&#160;&#160;&#160;&#160;&#160;&#160; [003] 00000000ff442f50&#160;&#160;&#160;&#160;&#160; 1&#160;&#160;&#160;&#160;&#160;&#160; 1&#160; 6/10/2011 17:30:54&#160; 000000017f4372a0&#160;&#160;&#160;&#160;&#160;&#160; 50&#160; do_something_prc</span></p>


<p>What are we seeing here?&#160; Connection pooling is basically a 5 level hierarchy:</p>


<ul>
  <li>Connection String - top level, defines the server and connection options. 
    <ul>
      <li>Pool group - Second level, varies based on the identity used to connect to the DB if using SSPI 
        <ul>
          <li>Pool - Actually contains the SqlConnection instances. 
            <ul>
              <li>SqlConnection - The DB connection instance 
                <ul>
                  <li>Active readers - SqlDataReaders that are actively doing something </li>
                </ul>
              </li>
            </ul>
          </li>
        </ul>
      </li>
    </ul>
  </li>
</ul>

<p>Some fields that probably need some explaining:</p>


<ul>
  <li>Pool -&gt; State 
    <ul>
      <li>Corresponds to the System.Data.ProviderBase.DbConnectionPool+State enum, Initializing, Running, Shutting Down </li>
    </ul>
  </li>

  <li>Connection -&gt; State 
    <ul>
      <li>Corresponds to the System.Data.ConnectionState enum. </li>
    </ul>
  </li>

  <li>Created is the time the connection was first opened in UTC. </li>

  <li>Command/Reader 
    <ul>
      <li>If Async = 1, the pointer is to a command, otherwise it's to a SqlDataReader.&#160; With a SqlCommand, you can get to the reader via the _cachedAsyncState field, with a SqlDataReader, you can get to the command via the _command field. </li>
    </ul>
  </li>
</ul>

<p>Note: This doesn't currently dump connections in the transacted connection pools.</p>


<p>&#160;</p>


<h2><font style="font-weight: bold">!FindTimers</font></h2>

<p>(x86 only for now)</p>


<p>Dumps out all timers registered in the process. </p>


<p>Example:</p>


<p><font face="Courier New">0:006&gt; !findtimers
    <br />Handle&#160;&#160;&#160;&#160;&#160;&#160;&#160;&#160;&#160;&#160;&#160; TimerObj&#160;&#160;&#160;&#160;&#160;&#160;&#160;&#160;&#160; ADID&#160; Period[ms]&#160; State&#160;&#160;&#160;&#160;&#160;&#160;&#160;&#160;&#160;&#160;&#160;&#160; Context&#160;&#160;&#160;&#160;&#160;&#160;&#160;&#160;&#160;&#160; Delegate&#160;&#160;&#160;&#160;&#160;&#160;&#160;&#160;&#160; Method

    <br />&#160;&#160;&#160;&#160;&#160;&#160;&#160;&#160;&#160; 293e48&#160; 00000000022cbbb4&#160;&#160;&#160;&#160; 1&#160;&#160;&#160;&#160;&#160;&#160;&#160; 1000&#160; 00000000022cbb48&#160; 00000000022cbbc8&#160; 00000000022cbb6c&#160; SomeCallback(System.Object)</font></p>


<p>Columns are:</p>


<ul>
  <li>Handle</li>

  <ul>
    <li>The timer handle, can be used to cross reference with an existing TimerBase object</li>
  </ul>

  <li>TimerObj</li>

  <ul>
    <li>A reference to the managed _TimerCallback object</li>
  </ul>

  <li>ADID</li>

  <ul>
    <li>The appdomain ID the timer is associated with</li>
  </ul>

  <li>State</li>

  <ul>
    <li>The state object associated with the constructor.</li>
  </ul>

  <li>Context</li>

  <ul>
    <li>The ExecutionContext captured by the timer.</li>
  </ul>

  <li>Delegate</li>

  <ul>
    <li>The delegate that will be called when the timer fires.
      <br /></li>
  </ul>
</ul>

<h2><span style="font-weight: bold">!SPTHelp</span></h2>

<p>Prints out the built-in help.</p>
