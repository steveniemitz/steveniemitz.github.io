---
layout: post
title: SPT Series Part 0 - Supporting methods
date: '2011-06-14 00:37:42'
---

<p>Before I begin talking about how I've implemented some of the commands in SPT, I need to talk about some of the supporting methods that make doing things much easier.&#160; The IXCLRDataProcess/Dacp*Request API is not very user friendly and very verbose, by creating wrappers around them, we can spend more time building extensions and less time worrying about implementation.</p>  <p>The first thing we need to do is get an instance of an IXCLRDataProcess.&#160; This is the core interface we'll be using for everything.&#160; See my previous posts for more information on it.&#160; Conveniently, WinDBG has a semi-undocumented IoCtrl request to give us an instance of this object.&#160; It internally handles creating an ICLRDataTarget implementation, finding and loading the correct mscordacwks dll, and calling CLRDataCreateInstance for you.&#160; Using it is simple, the IoCtrl request code is documented, so we simply need to make a request and call Ioctl. <font size="1">(Note: Ioctl is defined in wdbgexts.h)</font></p>  <pre style="font-family: ; background: white"><font face="Consolas"><font style="font-size: 9.8pt">HRESULT InitIXCLRDataFromWinDBG(IXCLRDataProcess **ppDac)<br />{<br />&#160;&#160;&#160; WDBGEXTS_CLR_DATA_INTERFACE ixDataQuery;<br /> <br />&#160;&#160;&#160; ixDataQuery.Iid = &amp;<span><font color="#0000ff">__uuidof</font></span>(IXCLRDataProcess);<br />&#160;&#160;&#160; <span><font color="#0000ff">if</font></span> (!Ioctl(IG_GET_CLR_DATA_INTERFACE, &amp;ixDataQuery, <span><font color="#0000ff">sizeof</font></span>(ixDataQuery)))<br />&#160;&#160;&#160; {<br />&#160;&#160;&#160;&#160;&#160;&#160;&#160; <span><font color="#0000ff">return</font></span> E_FAIL;<br />&#160;&#160;&#160; }<br /> <br />&#160;&#160;&#160; *ppDac = (IXCLRDataProcess*)ixDataQuery.Iface;<br />&#160;&#160;&#160; <br />&#160;&#160;&#160; <span><font color="#0000ff">return</font></span> S_OK;<br />}</font></font></pre>

<p>Next, we'll need an easy way to read the value of a field from a managed object.&#160; This is a multi-step process:</p>

<ol>
  <li>Make sure the memory address (lets call it pObj) is pointing at a real managed object.</li>

  <li>Get the MethodTable for the object. </li>

  <li>Find the field by name, note: this may require traversing up the type hierarchy looking at parent classes for their fields as well. </li>

  <li>Figure out the field offset from the start of the managed object. </li>

  <li>Read the value from the target process's memory. </li>
</ol>

<h2>1/2 - Validate the address and get the MethodTable</h2>

<p>This is the easiest part.&#160; One of the Dacp requests, (they're all defined in dacpriv.h in the SSCLI), is DacpObjectData, which most closely maps to !DumpObj in SOS.&#160; Requesting this with our pObj will give us a few useful things.&#160; 1) The MethodTable of the object, 2) if the object isn't a managed object, it'll return a failure code.</p>

<h2>3 - Find the field by name</h2>

<p>This is by far the hardest.&#160; It's an 8 step recursive process:</p>

<ol>
  <li>Use a DacpMethodTableData request to get the parent method table of the current MT.</li>

  <ol>
    <li>If the parent MT isn't NULL, call ourselves recursively.&#160; This will allow us to start at System.Object and work up, the reason for doing this will make sense later.</li>
  </ol>

  <li>Get the number of static + instance fields on the current class.&#160; The DacpMethodTableData request has an EEClass in the return information, and we can use DacpEEClassData to get the field information and module information.&#160; Important data here is: FirstField, wNumInstanceFields, wNumStaticFields, Module.</li>

  <li>Get an instance of IMetaDataImport.&#160; We can use DacpModuleData and the Module token from our EEClass data to get an instance of the IXCLRDataModule containing the EEClass.&#160; This can then be QI'd to an IMetaDataImport object.</li>

  <li>Iterate over the fields (they're a linked list, start a FirstField and keep going) until you hit (already seen fields from previous recurison) + wNumInstanceFields + wNumStaticFields.</li>

  <ol>
    <li>The condition needs some explaining.&#160; Imagine a 2 class hierarchy, Foo and Bar, where Bar inherits Foo.&#160; Foo defines 2 fields, a and b, and Bar defines one, c. If we look at the EEClass data on Bar, we'll see wNumInstanceFields = 3, but traversing the field list will only give us 1 field, since the other 2 are on the Foo EEClass.&#160; This is where the recursion comes in, by traversing from the bottom of the type hierarchy up and keeping count of how many fields we've seen, by the time we visit Bar we've already looked at Foo and seen 2 fields, so that means we only need to look at one field on Bar, which is exactly how many fields are valid in the linked list.</li>

    <li>Static fields are simpler because they don't inherit up the type hierarchy.&#160; There's no need to keep track of the number of fields you've seen in previous recursion steps.</li>
  </ol>

  <li>For each field, use DacpFieldDescData to get the FieldDesc for it.&#160; We'll use this to figure out 1) if the field is static/instance, 2) the field token (&quot;mb&quot;), 3) the pointer to the next field.</li>

  <li>Use the IMetaDataImport object we got in step 3 to get the field name for the field token.&#160; I used <a href="http://msdn.microsoft.com/en-us/library/ms230840.aspx">IMetaDataImport::GetMemberProps</a>, GetFieldProps might work too?</li>

  <li>Compare to the input field name</li>

  <li>Continue if not a match.</li>
</ol>

<h2>4/5 - Phew, figure out the field offset and read the value!</h2>

<p>Compared to step 3, this is a walk in the park.&#160; There's only two conditions we need to worry about.&#160; 1) If the type is a value type, dwOffset on the field is relative to the object address. 2) If it's a reference type, dwOffset is relative to the start of the object + sizeof(void*).&#160; This is because reference types have a methodtable pointer at the start of their instance.</p>

<p>To read the value from the target process, we can use <a href="http://msdn.microsoft.com/en-us/library/ff554359(v=vs.85).aspx">IDebugDataSpaces::ReadVirtual</a> and pass it the computed address from above.</p>