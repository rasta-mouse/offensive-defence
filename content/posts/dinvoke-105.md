---
title: "D/Invoke v1.0.5"
date: 2023-04-15T21:10:42+01:00
draft: false
authors:
    - RastaMouse
---

This post summerises the recent changes to my [D/Invoke](https://github.com/rasta-mouse/DInvoke/) fork.

## API Hashing

API hashing is already available on methods such as `GetLibraryAddress` and `GetExportAddress`.

```c#
var hAddress = Generic.GetLibraryAddress(
    "kernel32.dll", 
    "2B70CDC3FF17AB3948E01E1A318B8964",
    0xdeadbeef);
```

or

```c#
var hModule = Generic.GetLoadedModuleAddress("kernel32.dll");
var hAddress = Generic.GetExportAddress(
    hModule,
    "2B70CDC3FF17AB3948E01E1A318B8964",
    0xdeadbeef);
```

However, we have lacked a function that has allowed us to get a module handle by hash value.  `GetLoadedModuleAddress` now has an additional overload to support this.

```c#
var hModule = Generic.GetLoadedModuleAddress(
    "56CC05AB6D069D47DF2539FE99937D46",
    0xdeadbeef);
            
var hAddress = Generic.GetExportAddress(
    hModule,
    "2B70CDC3FF17AB3948E01E1A318B8964",
    0xdeadbeef);
```

## DynamicAsmInvoke

`DynamicAskInvoke` is a new method for executing arbitrary assembly stubs.  It requires a `byte[]` stub, function delegate, and parameters.

```c#
[UnmanagedFunctionPointer(CallingConvention.Cdecl)]
private delegate void DummyDelegate();
    
public static void Main()
{
    var stub = new byte[]
    {
        0x90,   // nop
        0xC3    // ret
    };

    var parameters = Array.Empty<object>();
    Generic.DynamicAsmInvoke(
        stub,
        typeof(DummyDelegate),
        ref parameters);
}
```

## GetPebAddress

`GetPebAddress` provides a new method for getting the base address of the PEB (rather than calling the `NtQueryInformationProcessBasicInformation` API).  It uses `DynamicAsmInvoke` to call a `__readgsqword(0x60)` stub.

```c#
var pPeb = Generic.GetPebAddress();
```

## GetSyscallStub

`GetSyscallStub` in the `DynamicInvoke` namespace provides a method, other than manual mapping, for obtaining a (direct) syscall stub for an Nt* API.  This is achieved by walking the PEB until it finds the base address of `ntdll.dll`.  From there it walks the exported functions until the desired API is found.  It reads the corresponding SSN and returns a complete syscall stub that can be passed to `DynamicAsmInvoke`.

```c#
var pPeb = Generic.GetPebAddress();
var stub = Generic.GetSyscallStub(pPeb, "NtAllocateVirtualMemory");
```

It also has an overload to support API hashing.

```c#
var pPeb = Generic.GetPebAddress();
var stub = Generic.GetSyscallStub(
    pPeb,
    "B718585223A73CB5E68746025E55EE46",
    0xdeadbeef);
```

## Closing

Hopefully these additions will prove useful.  The source code can be found on [GitHub](https://github.com/rasta-mouse/DInvoke) and nuget packages on my private [NuGet server](https://nuget.code-offensive.net/?q=DInvoke).  Please feel free to reach out with any comments or feedback.