---
layout: post
title: Fun with .NET remoting - Building a RealProxy implementation
date: '2013-12-30 21:19:00'
tags:
- clr
- remoting
---

<p>[Steve’s note: I wrote this post a year or so ago and never had posted it.&nbsp; It’s still relevant today, but I don’t recommend using shady things like this in production code without understanding the implications.]</p>

 <p>A side project I worked on recently involved building an abstraction layer around DB access, specifically calling stored procedures.&nbsp; My goal was to reduce code redundancy and add type-safety to the parameters of stored proc calls.&nbsp; The road I went down involved creating an interface for a group of stored procs, with the interface methods mapping to the stored procs.&nbsp; I might resurrect this as an example and post the code on github, because it actually worked out really well.</p>

 <h2>What's a RealProxy?</h2> <p>The .NET remoting infrastructure is built around 3 key classes:</p>

 <ol> <li>MarshalByRefObject  <ul> <li>This is the "opt-in" marker for by-ref object marshaling. </li></ul> </li><li>RealProxy  <ul> <li>This is the base type for method/constructor call dispatch over remoting.&nbsp; The CLR has special knowledge of this class, and this is the managed entry/exit point for calls dispatched through a TransparentProxy.&nbsp; Calls enter and leave managed code via RealProxy.PrivateInvoke. </li></ul> </li><li>__TransparentProxy  <ul> <li>The other piece of the proxy pair.&nbsp; The CLR has special knowledge of this class in terms of type casting and method invocation. </li></ul></li></ol> <p>Implementing RealProxy only requires you to implement Invoke(IMessage msg) and do whatever you want inside it.&nbsp; In my above example, I took the method name and parameters and handed them off to a SqlCommand and ran ExecuteReader, ExecuteNonQuery, etc.&nbsp; The implementation details for the synchronous case aren't that exciting.</p>

 <h2>Supporting BeginInvoke/EndInvoke</h2> <p>One big feature I wanted to implement was asynchronously executing SQL via BeginInvoke/EndInvoke on the proxy methods.&nbsp; Unfortunately, Microsoft made this impossible to do without resorting to some sketchy reflection.&nbsp; Inside RealProxy.PrivateInvoke, there's a block of code that looks like this:</p>

<pre style="font-family: ; background: white"><font face="Consolas"><span><font color="#0000ff"><font style="font-size: 9.8pt">if</font></font></span><font style="font-size: 9.8pt"> (!<font style="background-color: #ffff00"><span><font color="#0000ff">this</font></span>.IsRemotingProxy()</font> &amp;&amp; ((<font style="background-color: #ffff00">msgFlags &amp; 1</font>) == 1))<br>{<br>&nbsp;&nbsp;&nbsp; Message m = reqMsg <span><font color="#0000ff">as</font></span> Message;<br>&nbsp;&nbsp;&nbsp; AsyncResult ret = <span><font color="#0000ff">new</font></span> AsyncResult(m);<br>&nbsp;&nbsp;&nbsp; ret.SyncProcessMessage(retMsg);<br>&nbsp;&nbsp;&nbsp; retMsg = <span><font color="#0000ff">new</font></span> ReturnMessage(ret, <span><font color="#0000ff">null</font></span>, 0, <span><font color="#0000ff">null</font></span>, m);<br>}</font></font></pre>
<p>There are a few things going on here</p>


<ul>
<li>The constants for msgFlags are defined in System.Runtime.Remoting.Messaging.Message. We'll need this value later on.&nbsp; The interesting values are: 
<ul>
<li><font face="Courier New">BeginAsync</font> (eg a call to BeginInvoke) = 1 
</li><li><font face="Courier New">EndAsync</font> (EndInvoke) = 2 
</li><li><font face="Courier New">OneWay</font> (eg a method with a [OneWay] attribute on it) = 8 </li></ul>
</li><li>The consequence of the block of code above is that the remoting infrastructure will run BeginInvoke synchronously for any RealProxy that isn't a RemotingProxy.&nbsp; I'll get into details on this in a bit. 
</li><li>IsRemotingProxy checks the _flags field on RealProxy.&nbsp; This is set in the RealProxy constructor if the type is RemotingProxy.&nbsp; We can't inherit from RemotingProxy (it's internal), but we can use reflection in our own constructor to set this to RealProxyFlags.RemotingProxy (1).&nbsp; Once we do this, a "normal" Invoke implementation won't work correctly for async invocations. </li></ul>
<h2></h2>
<h2></h2>
<h2></h2>
<h2>How proxy invocation works</h2>
<p>So back to msgFlags and how asynchronous method invocation works with the remoting infrastructure.&nbsp; As I mentioned above, there's two possible code paths for an async invoke on a RealProxy, the first is the "normal" code path that you get when you invoke a delegate bound to a non-RemotingProxy instance.&nbsp; In this case, the code path looks like this:</p>


<ol>
<li>someDelegate.BeginInvoke(...) 
</li><li>Native CLR 
</li><li>RealProxy.PrivateInvoke, msgFlags = 1 (BeginAsync) 
</li><li>PrivateInvoke calls our Invoke(...) 
<ul>
<li>Since our Invoke isn't aware of anything async (in fact, with the public API there's no way to know), it runs synchronously. </li></ul>
</li><li>PrivateInvoke sets up an AsyncResult instance with the result from Invoke(...) and returns this.&nbsp; Note, at this point, everything has run synchronously, our BeginInvoke was exactly the same as an Invoke, except the last step of returning an IAsyncResult.&nbsp; At this point, all our work is done. 
</li><li>We call someDelegate.EndInvoke(iar) 
</li><li>RealProxy.PrivateInvoke, msgFlags = 2 (EndAsync) 
</li><li>PrivateInvoke calls EndInvokeHelper, which takes care of returning the return value or throwing an exception if one occurred. </li></ol>
<p>The pro of this code path is that you (the RealProxy implementer) don't need to worry about the async code path, in fact, you can't.&nbsp; The con is that there's no way to get a truly async invocation with this.</p>


<p>Now, the alternate code path is "enabled" by telling the infrastructure that we're a RemotingProxy.&nbsp; This code path looks roughly like this</p>


<ol>
<li>someDelegate.BeginInvoke(...) 
</li><li>Native CLR 
</li><li>RealProxy.PrivateInvoke, msgFlags = 1 
</li><li>PrivateInvoke calls our Invoke, this time however, it's expecting an IAsyncResult in return. 
<ul>
<li>We set up whatever async stuff we want to do here and start it. 
</li><li><font style="background-color: #ffff00">FUN FACT:</font> You can actually return whatever you want here!&nbsp; The CLR is missing the type check to make sure your ReturnMessage actually returns an IAsyncResult.&nbsp; The resulting chaos that ensues from not returning the correct type is entertaining.&nbsp; I made a post about it on StackOverflow awhile ago <a href="http://stackoverflow.com/questions/194484/whats-the-strangest-corner-case-youve-seen-in-c-or-net/5113304#5113304">here</a>. </li></ul>
</li><li>PrivateInvoke returns, we have an IAsyncResult 
</li><li>We call someDelegate.EndInvoke(iar) 
</li><li>PrivateInvoke calls Invoke, msgFlags = 2 
</li><li>Invoke handles getting the correct return value or throwing an exception if one occurred.&nbsp; Possibly through RealProxy.EndInvokeHelper as above. </li></ol>
<p>As you can see here, there's a lot more burden on us to "get things right." with the implementation.&nbsp; We now need to handle synthesizing an IAsyncResult, invoking a callback (if passed in), raising an exception in the EndInvoke (if thrown), and getting the value back.&nbsp; Also, since none of this is exposed publically via the API, it makes things even more difficult because we now need to rely on reflection.&nbsp; I'm going to drill into step 4 of the second code path and explain how it works.</p>


<h2></h2>
<h2>Implementing an async-aware Invoke</h2>
<p>The first problem of implementing an async-aware Invoke is getting the msgFlags.&nbsp; This is the core of Invoke, and we'll be using it to branch to synchronous/BeginInvoke/EndInvoke/OneWay code paths.&nbsp; Getting the value is pretty easy, looking at System.Runtime.Remoting.Messaging.Message, you can see there's a method called GetCallType.&nbsp; Unfortunately this isn't exposed in any of the public interfaces Message implements, and Message itself is internal, so we need to use reflection to call it. <font face="Consolas"><font style="font-size: 9.8pt">(<span><font color="#0000ff">int</font></span>)msg.GetType().GetMethod(<span><font color="#a31515">"GetCallType"</font></span>).Invoke(msg, <span><font color="#0000ff">null</font></span>) </font></font>will do the trick.</p>


<p>At this point, we need to branch for a few conditions.&nbsp; The 4 main cases are: Begin, End, OneWay, Synchronous.&nbsp; The simplest way is to create a method for each one and call them in Invoke depending on call type.&nbsp; Begin is the most interesting case because it requires the most work, so I'm going to focus on that.</p>


<p>BeginInvoke is responsible for 3 main tasks: creating the IAsyncResult to return, starting the async work, and handling the task completion.&nbsp; It turns out the framework exposes a class System.Runtime.Remoting.Messaging.AsyncResult that wraps a lot of the more complicated async completion logic.&nbsp; For example, the message we get in Invoke doesn't contain the last two "extra" parameters to BeginInvoke, the AsyncCallback delegate and state (it accomplishes this through Message.GetAsyncBeginInfo.)&nbsp; AsyncResult will get these for us, and once we give the AsyncResult the IMessage result, it'll handle invoking the callback if needed.</p>


<p>I’ll write more about EndInvoke at a later point, but it’s a fairly simple implementation.</p>

