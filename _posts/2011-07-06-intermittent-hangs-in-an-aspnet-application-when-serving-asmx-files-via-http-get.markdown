---
layout: post
title: Intermittent hangs in an ASP.NET application when serving asmx files via HTTP
  GET
date: '2011-07-06 23:03:07'
tags:
- asp-net
- debugging
---

<p>I was recently debugging an intermittent, brief hang (5-15 seconds) in an ASP.NET application that would occur once every few days.&#160; The application was fully precompiled (non-updatable) and running .NET 4.0.</p>

  <p>When the hang occurred, most threads would be sitting in a stack similar to this: (method signatures slightly modified for visibility reasons)</p>

  <p><font face="Courier New"><font style="background-color: #ffff00">System.Web.Compilation.BuildManager.GetBuildResultFromCacheInternal        <br /></font>System.Web.Compilation.BuildManager.GetVPathBuildResultFromCacheInternal       <br />System.Web.Compilation.BuildManager.GetVPathBuildResultInternal       <br />System.Web.Compilation.BuildManager.GetVPathBuildResultWithNoAssert       <br />System.Web.Compilation.BuildManager.GetVirtualPathObjectFactory       <br />System.Web.Compilation.BuildManager.CreateInstanceFromVirtualPath       <br />System.Web.UI.PageHandlerFactory.GetHandlerHelper       <br />&lt;snip&gt;</font></p>

  <p>With one thread in a stack like this:</p>

  <p><font face="Courier New">System.Threading.WaitHandle.InternalWaitOne      <br />System.CodeDom.Compiler.Executor.ExecWaitWithCaptureUnimpersonated       <br />System.CodeDom.Compiler.Executor.ExecWaitWithCapture       <br /><font style="background-color: #ffff00">Microsoft.CSharp.CSharpCodeGenerator.Compile        <br />Microsoft.CSharp.CSharpCodeGenerator.FromFileBatch         <br />Microsoft.CSharp.CSharpCodeGenerator.CompileAssemblyFromFileBatch         <br />System.Web.Compilation.AssemblyBuilder.Compile         <br />System.Web.Compilation.BuildProvidersCompiler.PerformBuild         <br />System.Web.Compilation.BuildManager.CompileWebFile         <br />System.Web.Compilation.BuildManager.GetVPathBuildResultInternal         <br /></font>System.Web.Compilation.BuildManager.GetVPathBuildResultWithNoAssert       <br />System.Web.Compilation.BuildManager.GetVPathBuildResult       <br />System.Web.UI.PageParser.GetCompiledPageInstance       <br />System.Web.UI.PageParser.GetCompiledPageInstance       <br />System.Web.Services.Protocols.DocumentationServerProtocol.GetCompiledPageInstance       <br />System.Web.Services.Protocols.DocumentationServerProtocol.Initialize       <br />System.Web.Services.Protocols.ServerProtocolFactory.Create       <br />System.Web.Services.Protocols.WebServiceHandlerFactory.CoreGetHandler       <br />System.Web.Services.Protocols.WebServiceHandlerFactory.GetHandler       <br />System.Web.Script.Services.ScriptHandlerFactory.GetHandler       <br />&lt;snip&gt;</font></p>

  <p>(note, I didn't have parameters to these stack traces, had I, the diagnostic would have been much simpler in hindsight)</p>

  <p>Every time this happened it'd be an ASMX file triggering the compile, never any other type.&#160; I was pretty perplexed by this because:</p>

  <ol>   <li>The application is fully precompiled, so ASP.NET should never attempt to compile anything. </li>    <li>There's not really anything to compile in our ASMX files (they're just pointers to a class in a DLL), so even if ASP.NET did try to compile it, it shouldn't take any time. </li> </ol>  <p>After a few days of continuing to be perplexed, I had a breakthrough.</p>

  <p>In ASP.NET, if you just hit an ASMX file via a browser, you get a pretty web page that lists all the methods on the service and allows you to invoke methods through the browser.&#160; This page is actually an ASPX file (named <em>DefaultWsdlHelpGenerator.aspx</em>) that lives in the Config directory of whatever framework you're running (in my case it was C:\Windows\Microsoft.NET\Framework64\v4.0.30319\Config, ymmv).&#160; Bingo, that's what's getting compiled!</p>

  <p>Once I figured this out, the solution was fairly simple.&#160; ASP.NET allows you to specify the WSDL help generator file via <a href="http://msdn.microsoft.com/en-us/library/ycx1yf7k(v=VS.100).aspx">system.web/webServices/wsdlHelpGenerator</a> in your web.config.&#160; I simply copied DefaultWsdlHelpGenerator.aspx into my application's root directory and added the wsdlHelpGenerator element to my web.config.&#160; Now, when my application needs to generate a WSDL file or page, ASP.NET uses the precompiled help generator instead of the default one and doesn't trigger a compile.&#160; <strong>Problem solved!</strong></p>

  <p>The moral of the story here: if you have a precompiled site with asmx services and care about short hangs, do what I outlined above.&#160; If you don't precompile, this is the least of your worries.</p>

