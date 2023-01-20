---
layout: post
title: Implementing SOS with SPT - Part 1 of N - DumpObj
date: '2013-12-30 20:59:00'
tags:
- clr
- debugging
- ixclrdataprocess
- sos
- windbg
---

I decided to have some fun re-implementing SOS methods using SPT.  I chose DumpObj to start with because it’s one of the more complicated methods in SOS.  Hopefully this will be part 1 of a multipart series.

Let’s start by looking at some sample output from !do on a hashtable.

```
0:036> !do 11faa984 
Name:        System.Collections.Hashtable
MethodTable: 7370d288
EEClass:     7337c4a0
Size:        52(0x34) bytes
File:        C:\Windows\Microsoft.Net\assembly\GAC_32\mscorlib\v4.0_4.0.0.0__b77a5c561934e089\mscorlib.dll
Fields:
      MT    Field   Offset                 Type VT     Attr    Value Name
7370ae80  4000c01        4 ...ashtable+bucket[]  0 instance 11faa9b8 buckets
7370c770  4000c02       18         System.Int32  1 instance        2 count
7370c770  4000c03       1c         System.Int32  1 instance        1 occupancy
7370c770  4000c04       20         System.Int32  1 instance        2 loadsize
737070b4  4000c05       24        System.Single  1 instance 0.720000 loadFactor
7370c770  4000c06       28         System.Int32  1 instance        2 version
73707208  4000c07       2c       System.Boolean  1 instance        0 isWriterInProgress
733223ec  4000c08        8 ...tions.ICollection  0 instance 00000000 keys
733223ec  4000c09        c ...tions.ICollection  0 instance 00000000 values
7370222c  4000c0a       10 ...IEqualityComparer  0 instance 00000000 _keycomparer
7370b350  4000c0b       14        System.Object  0 instance 00000000 _syncRoot
```

We can go through this line by line and figure out what’s going on.


1. **MethodTable**: This can be found on an object via ``IXCLRDataProcess3::GetObjectData``.
1. **Name**: This is the type name, it can be obtained via ``IXCLRDataProcess3::GetMethodTableName ``with the method table from above.
1. **EEClass**: ``IXCLRDataProcess3::GetMethodTableData ``will give you the EEClass, as well as the Module (which we need in a minute).
1. **Size**: This is on the ClrObjectData we got in #1.
1. **File**: Given the Module (from #3), we can get the assembly that contains it via ``IXCLRDataProcess3::GetModuleData``. Once we have the assembly, ``IXCLRDataProcess3::GetAssemblyName`` will give you the full path to an assembly.
1. **Field Name**: This one is a little interesting.  As far as I can tell, IXCLRDataProcess doesn’t directly expose a way to get a field name, however, we can use IMetaDataImport to get the values.  The method of getting an instance of IMetaDataImport is fairly simple, once you know it’s implemented by the module itself.  We can use ``IXCLRDataProcess3::GetModule ``(with the module address from #3) to get an IUnknown.  That can be QI’d for IID_IMetaDataImport.  Once we have that pointer, we can GetMemberProps to get the name.

## Fields

<p>Fields are the fun part.  The whole algorithm for iterating over a classes fields is in <a href="https://github.com/steveniemitz/SDbgExt2/blob/master/SDbgCore/src/ClrProcess_Reflection.cpp#L39" target="_blank">ClrProcess::FindFieldByNameExImpl</a>.  he basic idea is:</p>


1. Get to the root type in the inheritance hierarchy
1. For each type, traverse the linked list of FieldDescs (obtained via GetMethodTableFieldData.FirstField, stop traversing when you’ve seen all instance fields and static files on that type. 
1. Once you’ve seen them all, go to the next type in the hierarchy and repeat #2.  

*Note, NumInstanceFields includes the fields inherited from parent classes as well.*  This is why we’re starting at the root (System.Object) and moving down, otherwise we wouldn’t know when to stop traversing each list of fields.

<p>Once we have all the fields, we can start drawing each line of output.  Most are fairly trivial to get.</p>


- **MethodTable**: Using the ClrFieldDescData (from ``IXCLRDataProcess3::GetFieldDescData) ``saved during iterating over the fields, we can get the FieldMethodTable.
- **Field**: This is the token for the field.
- **Offset**: This is simply the Offset from the field data.  However, instance fields need to be offset by sizeof(void*) to account for the type handle (MethodTable *) at the start of every object.  Static fields do not need to be offset since they aren’t actually stored on the object instance.
- **Type**: This is the same as the typename from above, using the FieldMethodTable.
- **VT**: As far as I can tell, there’s no simple “IsValueType” flag anywhere.  However, you can use the FieldType to figure it out.  It corresponds to the CorElementType enum in corhdr.h.  Anything before and including ELEMENT_TYPE_R8 is a value type, as well as ELEMENT_TYPE_VALUETYPE.  Everything else can be counted as class.
- **Attr**: I’ve only used two values here, “shared” (for IsStatic, Is[Context,Thread]Local), or “instance” for anything else.
- **Value**: The algorithm to read the values falls into two categories, which further break down into more subcategories.  The top level categories are instance vs static.  We’re not even going to try to get values for context local and thread local fields.

## Instance Fields

<p>Instance fields break down into 3 more subcategories, class, primitive, and value type.</p>

- Class is the simplest, just read one pointer at ``obj+offset+sizeof(void*)``.  Display that value
- ValueType is similar, but one less level of indirection.  ValueTypes are stored in-type, not as a pointer to another object on the heap.  The address shown under value is simply ``obj+offset+sizeof(void*)``.  This address could then be passed to !dumpvc to get the value.
- Primitive.  For these, we need to compute the size of the field via ``ClrProcess::GetSizeForType.``  This method is simply a switch statement with the built-in primitives and their respective sizes.  Once we have a size, we can read S bytes at the correct offset (as above) into the object.


## Static Fields

Static fields break down similar, but there is already logic in ``ClrProcess::GetStaticFieldValue`` to handle it.  The complication of static fields is we need to consider all app domains.  ``ClrProcess::GetStaticFieldValues`` handles all of this for us.  We can simply call it and display the results.  One thing I’ve noticed is that SOS displays “NotInit” on some fields, I’m not sure if it’s just doing a null check here, or there’s more logic to figure out if a domain has somehow initialized a static field yet.


<p>Putting that all together with some pretty formatting, we’ve got the same information as SOS’s !dumpobj.</p>


<p>I’ve pushed this to my <a href="https://github.com/steveniemitz/SDbgExt2" target="_blank">github repo</a> in case anyone wants to check it out.</p>
