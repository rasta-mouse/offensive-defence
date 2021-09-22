---
title: "C# Process Class Primer"
date: 2021-09-22T10:11:26+01:00
draft: false
authors:
    - RastaMouse
---

This post was requested from a [Patron](https://www.patreon.com/_RastaMouse) and will serve as a short primer for using the [Process](https://docs.microsoft.com/en-us/dotnet/api/system.diagnostics.process) class in C#.
This class is part of the [System.Diagnostics](https://docs.microsoft.com/en-us/dotnet/api/system.diagnostics) namespace and is useful for enumerating, starting, and killing processes on a system.

### Enumerating Processes

The `Process` class has 4 methods for enumerating processes on a system (which are quite self-explanatory):

- GetCurrentProcess
- GetProcessesByName
- GetProcessById
- GetProcesses

`Process.GetProcesses()` will return a `Process[]` of every process running on the local system.

```c#
using System.Diagnostics;

namespace ConsoleApp1
{
    internal static class Program
    {
        private static void Main(string[] args)
        {
            var processes = Process.GetProcesses();
        }
    }
}
```

Each instance of a `Process` has properties and methods that can be accessed, such as `ProcessName`, `Id` and `Kill()`.  Your ability to read/call these depends on your privilege and the privilege of the process in question (e.g. you can't just go around killing processes you don't own).

Since this is just an array, iterate over each one and print some information.

```c#
var processes = Process.GetProcesses();

foreach (var process in processes)
{
    Console.WriteLine("{0} : {1}", process.ProcessName, process.Id);
}
```

This will produce very simple output:

```text
Idle : 0
System : 4
Secure System : 104
Registry : 180
```

You can also enumerate the DLLs loaded into a process.

```c#
var process = Process.GetCurrentProcess();

foreach (ProcessModule module in process.Modules)
{
    Console.WriteLine("{0} : 0x{1:X}", module.ModuleName, module.BaseAddress.ToInt64());
}
```

```text
ConsoleApp1.exe : 0x7FF7BB6E0000
ntdll.dll : 0x7FFECDD30000
KERNEL32.DLL : 0x7FFECC780000
KERNELBASE.dll : 0x7FFECB710000
USER32.dll : 0x7FFECBF00000
```

### Starting Processes

The absolute easiest way to start a process is by using `Process.Start()` and passing in the `fileName`.

```c#
Process.Start("notepad");
```

You may also pass in arguments as a `string` or an `IEnumerable<string>`. Some examples:

```c#
Process.Start("notepad", @"C:\Temp\test.txt");
```

```c#
Process.Start("cmd", @"/c calc");
```

```c#
Process.Start(@"C:\Tools\Rubeus\Rubeus\bin\Debug\Rubeus.exe", new[] { "klist" });
```

This method actually returns a new instance of a `Process`, which can be used to access properties/methods for that process instance (as shown above).

```c#
var process = Process.Start("notepad");
Console.WriteLine("Started {0} ({1})", process.ProcessName, process.Id);
```
```text
Started notepad (3640)
```

More granular control can be gained by using the `ProcessStartInfo` class.  This allows you to start processes with hidden windows, redirect std in/out/err, and with alternate credentials.

```c#
var info = new ProcessStartInfo
{
    FileName = "cmd",
    CreateNoWindow = true,
};

var process = Process.Start(info);
Console.WriteLine("Started {0} ({1})", process.ProcessName, process.Id);
process.Kill();
```

```c#
var info = new ProcessStartInfo
{
    FileName = "cmd",
    UserName = "AltRasta",
    PasswordInClearText = "Passw0rd!",
    WorkingDirectory = @"C:\Windows\System32"
};

var process = Process.Start(info);
Console.WriteLine("Started {0} ({1})", process.ProcessName, process.Id);
process.Kill();
```

#### Disposing

An instance of a `Process` has many resources associated with it, including a native handle.  This can be used as part of a process injection workflow, since it's required to allocate and write memory etc.

```c#
var info = new ProcessStartInfo
{
    FileName = "cmd",
    CreateNoWindow = true,
};

var process = Process.Start(info);
Console.WriteLine("Handle: 0x{0:X}", process.Handle.ToInt64());
process.Kill();
```
```text
Handle: 0x298
```

For that reason, `Process` is disposable to allow these associated resources to be freed.  So we should be doing something like:

```c#
using var process = Process.Start(info);
process.Kill();
```
or (depending on your style)
```c#
var process = Process.Start(info);
process.Kill();
process.Dispose();
```

It's worth noting that disposing of a `Process` instance does not kill the process - it simply releases the resources associated with us holding information about it.

#### Capturing Output

Reading from standard out and error are staples of how many "shell" and "run" capabilities work in offensive .NET tooling.  It allows you to spawn hidden processes and capture the output without it ever been seen by a user.

As the current user:

```c#
var info = new ProcessStartInfo
{
    FileName = "cmd",
    Arguments = "/c whoami",
    CreateNoWindow = true,
    RedirectStandardOutput = true,
    UseShellExecute = false
};

using var process = Process.Start(info);
using var sr = process.StandardOutput;
            
var output = sr.ReadToEnd();
Console.WriteLine(output);
```

```text
ghost-canyon\daniel
```

With alternate creds:

```c#
var info = new ProcessStartInfo
{
    FileName = "cmd",
    Arguments = "/c whoami",
    CreateNoWindow = true,
    RedirectStandardOutput = true,
    UseShellExecute = false,
    UserName = "AltRasta",
    PasswordInClearText = "Passw0rd!",
    WorkingDirectory = @"C:\Windows\System32"
};

using var process = Process.Start(info);
using var sr = process.StandardOutput;
            
var output = sr.ReadToEnd();
Console.WriteLine(output);
```

```text
ghost-canyon\altrasta
```

Capturing standard out and error:

```c#
var info = new ProcessStartInfo
{
    FileName = "cmd",
    Arguments = "/c not_a_valid_command",
    CreateNoWindow = true,
    RedirectStandardOutput = true,
    RedirectStandardError = true,
    UseShellExecute = false
};

using var process = Process.Start(info);
using var osr = process.StandardOutput;
using var esr = process.StandardError;
            
var output = osr.ReadToEnd();
var error = esr.ReadToEnd();

var sb = new StringBuilder();
sb.Append(output);
sb.Append(error);

Console.WriteLine(sb.ToString());
```

```text
'not_a_valid_command' is not recognized as an internal or external command, operable program or batch file.
```

Calling an application without cmd:

```c#
var info = new ProcessStartInfo
{
    FileName = "PING",
    Arguments = "127.0.0.1 -n 3",
    CreateNoWindow = true,
    RedirectStandardOutput = true,
    UseShellExecute = false
};

using var process = Process.Start(info);
using var sr = process.StandardOutput;
            
var output = sr.ReadToEnd();
Console.WriteLine(output);
```

```text
Pinging 127.0.0.1 with 32 bytes of data:
Reply from 127.0.0.1: bytes=32 time<1ms TTL=128
Reply from 127.0.0.1: bytes=32 time<1ms TTL=128
Reply from 127.0.0.1: bytes=32 time<1ms TTL=128

Ping statistics for 127.0.0.1:
    Packets: Sent = 3, Received = 3, Lost = 0 (0% loss),
Approximate round trip times in milli-seconds:
    Minimum = 0ms, Maximum = 0ms, Average = 0ms
```

Hopefully this post was useful for those looking to work with the C# Process class.  Become a [Patron](https://www.patreon.com/_RastaMouse) and request a topic of your own.