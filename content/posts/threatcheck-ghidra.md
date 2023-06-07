---
title: "Bypassing Defender with ThreatCheck & Ghidra"
date: 2023-06-07T10:16:02+01:00
draft: false
authors:
    - RastaMouse
---

## Intro

It should come as no surprise when payloads generated in their default state get swallowed up by Defender, as Microsoft have both the means and motivation to proactively produce signatures for open and closed source/commericial tooling.  One tactic to get around these is to generate heavily obfuscated, compressed, or encrypted payloads which are unpacked at runtime.  However, [highly entropic](https://practicalsecurityanalytics.com/file-entropy/) payloads can be just as problematc.  Daniel Bohannon and Lee Holmes also wrote a paper called [Revoke-Obfuscation: PowerShell Obfuscation Detection Using Science](https://www.blackhat.com/docs/us-17/thursday/us-17-Bohannon-Revoke-Obfuscation-PowerShell-Obfuscation-Detection-And%20Evasion-Using-Science-wp.pdf) which shows several methods for detecting obfuscated PowerShell scripts.

A philosophy that has stuck with me is to only make the minimum changes necessary to circumvent a particular static signature - i.e. solve the problem with a scalpel rather than a sledgehammer.  This post will present a methodology for identifying "bad bytes" in a payload and finding their location within the compiled binary.  To demonstrate, I will use [ThreatCheck](https://github.com/rasta-mouse/ThreatCheck) and [Ghidra](https://github.com/NationalSecurityAgency/ghidra) to analyze and modify a Beacon payload generated from Cobalt Strike.

## Bad Bytes

This is the easy part.  ThreatCheck works by splitting the binary up into little chunks and scanning each one with Defender.  It will attempt to find the smallest possible chunk that triggers a positive result and prints an array of bytes to the console.

![](/images/threatcheck-ghidra/threatcheck.png "ThreatCheck bad bytes")

ThreatCheck was able to identify a block of code that Defender detects as malicious, but there is no context about which part of the payload it is.  This is where Ghidra comes into play.

## Decompiling

Load the payload into a new project and have Ghidra run through its automated analysis. Then use the **Search Memory** function to find a sequence of bytes output from ThreatCheck.

![](/images/threatcheck-ghidra/search-memory.png "Ghidra Search Memory")

ThreatCheck always attempts to print 1024 bytes to the console by working its way backwards from the end of the "bad byte" range.  So even though 1024 bytes are displayed, it's not an indication that the entirety was malicious.  Since the bad bytes are always at the end of those displayed, use a hex sequence from the bottom rather than the top.  In this example, I'm searching for `8A 54 15 00 32 14 07 88 14 03 48 FF C0 EB E7 48`.

![](/images/threatcheck-ghidra/search-memory-result.png "Ghidra Search Memory result")

I have a single result at `004015d1` and clicking on the row will display the code main Ghidra window. The "Listing" view shows the CPU instructions in a simple list and the "Decompile" view attempts to reverse the code of the function back to its original source.  Now we know that the offending block of code is likely this `for` loop inside `FUN_00401595`.

![](/images/threatcheck-ghidra/decompiled.png "Ghidra decompiled")

The loop comes after a call to `VirtualAlloc` but before a call to `VirtualProtect` and `CreateThread`.  After searching through Cobalt Strike's Artifact Kit source code, we come across the `spawn` function located in `patch.c`.  The loop is responsible for decoding the Beacon shellcode prior to execution and writing it into the allocated memory region.

![](/images/threatcheck-ghidra/decode-payload-routine.png "Artifact Kit")

All we need to do is modify the loop so that it compiles to a different byte sequence (how you do that is up to you).  Then, once the payload has been regenerated, Defender will no longer match on that static signature.

![](/images/threatcheck-ghidra/no-threat.png "No Threat")

## Conclusion

This was a singular example of how to analyse and modify a Beacon payload from Cobalt Strike, but the same methodology can be applied to any payload generating tool for which you can access the source code.  This post demonstrates that complex manipulations are not required to bypass static signatures and why defenders should not soley rely them to detect "well known" tooling.
