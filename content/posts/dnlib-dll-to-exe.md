---
title: "DLL's to EXE's with dnlib"
date: 2020-12-30T20:25:03Z
draft: false
authors:
    - RastaMouse
tags:
    - c#
    - .net
    - dnlib
---

### Intro

Continuing with my dnlib kick, this post will demonstrate how to "convert" a .NET DLL to an EXE.  We'll use the same DLL code as in the [previous](../dnlib-dllexport) post.

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

If you've dabbled in C# enough, you'll likely already know that within IDEs (such as Visual Studio), you can easily toggle the output type of a C# app between Console Application, Windows Application and Class Library.  DLLs don't require anything special and the only real "requirement" of a console app is for it to have an Entry Point, e.g.

```c#
static void Main(string[] args)
{
    // do stuff
}
```

To convert a DLL to an EXE, all we really have to do is provide an entry point and change the "module type" of the assembly.  We'll make our entry point run the existing `Execute` method.

### Creating Main

To create the new `Main` method, we use the `MethodDefUser` method in dnlib, which allows us to craft arbitrary methods.  This method takes a name, a signature and a set of optional flags.  The signature defines both the return and input types for the method.  `Main` will return `void` and take in a `string[]` called `args`.

```c#
var main = new MethodDefUser("Main", MethodSig.CreateStatic(module.CorLibTypes.Void, new SZArraySig(module.CorLibTypes.String)))
{
    Attributes = MethodAttributes.Static,
    ImplAttributes = MethodImplAttributes.IL | MethodImplAttributes.Managed
};
```

Add the parameter(s) with `ParamDefUser` which can take a name and position.  We use position `1` because `0` is reserved for the method's return type.

```c#
main.ParamDefs.Add(new ParamDefUser("args", 1));
```

Those steps have only defined the signature - now we need to give it a body (the "stuff" to execute).  Create a new (empty) `CilBody` and assign it to `main`'s `Body`.

```c#
var mainBody = new CilBody();
main.Body = mainBody;
```

Now we can add instructions to the body - since we want to execute an existing method, we need a reference to it.  As stated in the previous post, I just use Linq.

```c#
var exec = type.Methods.FirstOrDefault(m => m.Name == "Execute");
```

Then we can add a `call` `opcode` to the body, passing in the reference to the target method (followed by a `ret`).

```c#
mainBody.Instructions.Add(OpCodes.Call.ToInstruction(exec));
mainBody.Instructions.Add(OpCodes.Ret.ToInstruction());
```

The last two steps are to add this new `Main` method to the `Demo` type (class) and then specify it as the module's `EntryPoint`.

```c#
type.Methods.Add(main);
module.EntryPoint = main;
```

### ModuleKind

Now we're ready to write the assembly to a new file.  First define a new `ModuleWriterOptions` instance.

```c#
var opts = new ModuleWriterOptions(module);
```

`opts.ModuleKind` is currently set to `Dll` and `opts.PEHeadersOptions.Characteristics` has the flags `ExecutableImage | LargeAddressAware | Dll`.

Change `ModuleKind` to `Console` and strip the `Dll` flag from the `PEHeaderOptions`.

```c#
opts.ModuleKind = ModuleKind.Console;
opts.PEHeadersOptions.Characteristics = dnlib.PE.Characteristics.ExecutableImage | dnlib.PE.Characteristics.LargeAddressAware;
```

Finally, write the assembly to a new EXE.

```c#
module.Write(@"C:\Temp\DemoEXE.exe", opts);
```

### dnSpy

This is how the assembly looks in dnSpy.  We didn't mess with changing the namespace or anything, so there are still references to the original `DemoDLL` name.

![](/images/dnlib-dll-to-exe/dnspy.png "dnSpy")

And executing the assembly works as expected.

![](/images/dnlib-dll-to-exe/exec.png "DemoEXE")

### Final Code

```c#
using dnlib.DotNet;
using dnlib.DotNet.Emit;
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

            // Create the Main signature
            var main = new MethodDefUser("Main", MethodSig.CreateStatic(module.CorLibTypes.Void, new SZArraySig(module.CorLibTypes.String)))
            {
                Attributes = MethodAttributes.Static,
                ImplAttributes = MethodImplAttributes.IL | MethodImplAttributes.Managed
            };

            // Add Main param
            main.ParamDefs.Add(new ParamDefUser("args", 1));

            // Create Main body
            var mainBody = new CilBody();
            main.Body = mainBody;

            // Instance of Execute method
            var exec = type.Methods.FirstOrDefault(m => m.Name == "Execute");

            // Add Call and Ret instructions to Main body
            mainBody.Instructions.Add(OpCodes.Call.ToInstruction(exec));
            mainBody.Instructions.Add(OpCodes.Ret.ToInstruction());

            // Add Main method and set EntryPoint
            type.Methods.Add(main);
            module.EntryPoint = main;

            var opts = new ModuleWriterOptions(module);
            opts.ModuleKind = ModuleKind.Console;
            opts.PEHeadersOptions.Characteristics = dnlib.PE.Characteristics.ExecutableImage | dnlib.PE.Characteristics.LargeAddressAware;

            module.Write(@"C:\Temp\DemoEXE.exe", opts);
        }
    }
}
```