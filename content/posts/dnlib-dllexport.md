---
title: "Adding DLL Exports with dnlib"
date: 2020-12-29T10:38:26Z
draft: false
tags:
    - "c#"
    - ".net"
    - "dnlib"
authors:
    - RastaMouse
---

### Intro

[dnlib](https://github.com/0xd4d/dnlib) is an insanely powerful library for reading and writing .NET assemblies.  It can be used to modify existing assemblies or even create new ones from scratch.  It's also used within other .NET modification/disassembly packages such as [ConfuserEx](https://github.com/yck1509/ConfuserEx) and [dnSpy](https://github.com/dnSpy/dnSpy).

I'm in the process of porting the SharpC2 payload generation capabilities from using [roslyn](https://github.com/dotnet/roslyn) to dnlib, and thought I would write about a fun use case.

### Starting from a .NET "template"

Using dnlib to create an entire payload stager from nothing, whilst possible, is just a bit much for any sane individual.  It's far easier to start with a compiled assembly as a template and just make the desired modifications before outputting a new assembly.

For this example, we'll start with the following compiled to an `AnyCPU` .NET Class Library.

```c#
using System.Windows.Forms;

namespace DemoDLL
{
    public class Demo
    {
        public static void Execute()
        {
            MessageBox.Show("Hello from DemoDLL");
        }
    }
}
```

Of course, if we try and use rundll32 to run the `Execute` method, it will fail.

![](/images/dnlib-dllexport/fail.png "Missing entry: Execute")

### dnlib "ConsoleApp"

We can now create a console application that will load and modify this DLL (in the case of SharpC2, the Team Server uses dnlib to modify the stager template on request).  dnlib has a convenient [Nuget package](https://www.nuget.org/packages/dnlib/) that we can install.

First we create a module definition for the assembly by calling `ModuleDefMD.Load()`, which will contain every single little piece of information about the assembly.

```c#
var module = ModuleDefMD.Load(@"C:\Temp\DemoDLL.dll");
```

Then we want to find a reference to the `Execute` method, as this is the method we want to expose via the export.  I typically do this by using Linq to find the type (or class) first, which is just `Demo`, within which we'll find the method.

```c#
var type = module.GetTypes().FirstOrDefault(t => t.Name == "Demo");
var method = type.Methods.FirstOrDefault(m => m.Name == "Execute");
```

Each method has `ExportInfo` (`MethodExportInfo`) and `IsUnmanagedExport` (`bool`) properties.  As expected, `ExportInfo` is currently `null` and `IsUnmanagedExport` is `false`.

```text
method.ExportInfo
null

method.IsUnmanagedExport
false
```

We can change that by simply instantiating a new instance of `MethodExportInfo` and setting `IsUnmanagedExport` to `true`.

```c#
method.ExportInfo = new MethodExportInfo();
method.IsUnmanagedExport = true;
```

Next, we modify the `PEHeadersOptions` and `Cor20HeaderOptions` to:

1. Change the architecture from `AnyCPU` to `x64`.
2. Remove the `ILOnly` flag.

This can be done using `ModuleWriterOptions`.

```c#
var opts = new ModuleWriterOptions(module);
opts.PEHeadersOptions.Machine = dnlib.PE.Machine.AMD64;
opts.Cor20HeaderOptions.Flags = 0;
```

In my case, the only flag that was set in the `Cor20HeaderOptions` was `ILOnly`, so I cleared it by setting the flags to `0`.

Finally, we can write the module to a new DLL file.

```c#
module.Write(@"C:\Temp\DemoDLLExported.dll", opts);
```

Now rundll32 will be able to execute this method.

![](/images/dnlib-dllexport/success.png "MsgBox")

### Export Name

By default, `MethodExportInfo` will use the method name as the export name, but it has a few optional overloads that can change that behaviour.  For instance, it can take a string or ordinal.

```c#
method.ExportInfo = new MethodExportInfo("SomethingRandom");
```

![](/images/dnlib-dllexport/somethingrandom.png "SomethingRandom")

I think this would make for an interesting user-supplied parameter when generating this style of payload.

### Final Code

```c#
using dnlib.DotNet;
using dnlib.DotNet.Writer;

using System.Linq;

namespace ConsoleApp
{
    class Program
    {
        static void Main(string[] args)
        {
            var module = ModuleDefMD.Load(@"C:\Temp\DemoDLL.dll");

            var type = module.GetTypes().FirstOrDefault(t => t.Name == "Demo");
            var method = type.Methods.FirstOrDefault(m => m.Name == "Execute");

            method.ExportInfo = new MethodExportInfo();
            method.IsUnmanagedExport = true;

            var opts = new ModuleWriterOptions(module);
            opts.PEHeadersOptions.Machine = dnlib.PE.Machine.AMD64;
            opts.Cor20HeaderOptions.Flags = 0;

            module.Write(@"C:\Temp\DemoDLLExported.dll", opts);
        }
    }
}
```