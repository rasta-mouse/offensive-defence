---
title: "Cobalt Strike Process Inject Kit"
date: 2023-08-09T09:21:07+01:00
draft: false
authors:
    - RastaMouse
---

## Introduction

I was scrolling through one of the social media dumpster fires the other day and whizzed past a post that caught my attention.  The post itself was not paricularly novel - it was a process injection technique implemented as a Cobalt Strike Beacon Object File (BOF).  I had a look at the included [Aggressor](https://hstechdocs.helpsystems.com/manuals/cobaltstrike/current/userguide/content/topics/agressor_script.htm) script and noticed that it was registering a new command with `beacon_command_register`, reading a shellcode (`.bin`) file from disk and executing the BOF via the [beacon_inline_execute](https://hstechdocs.helpsystems.com/manuals/cobaltstrike/current/userguide/content/topics_aggressor-scripts/as-resources_functions.htm?Highlight=inline_execute#beacon_inline_execute) function.  Curiosity piqued, I went in search of some other process injection BOFs across GitHub and the Cobalt Strike [Community Kit](https://cobalt-strike.github.io/community_kit/).  I only looked at a handful of them but found that they all worked in the same way.

I was left wondering why none of them were using the [PROCESS_INJECT_SPAWN](https://hstechdocs.helpsystems.com/manuals/cobaltstrike/current/userguide/content/topics_aggressor-scripts/as-resources_hooks.htm?Highlight=PROCESS_INJECT_SPAWN#PROCESS_INJECT_SPAWN) and [PROCESS_INJECT_EXPLICIT](https://hstechdocs.helpsystems.com/manuals/cobaltstrike/current/userguide/content/topics_aggressor-scripts/as-resources_hooks.htm?Highlight=PROCESS_INJECT_SPAWN#PROCESS_INJECT_EXPLICIT) hooks that have been around since CS 4.5 (circa 2021).  Admittedly, some of the BOFs did pre-date this release but most of them didn't.  My only assumption is that perhaps these hooks are not that well known, so the aim of this post is to provide an overview and why you may want to implement your injection BOFs in this way.

## Spawn vs Explicit

Fundamentally, these hooks allow you to define how both the fork & run and explicit injection techniques are handled when executing post-exploitation commands that perform injection.  Where the BOFs mentioned above only expose shinject-style functionality, these hooks extend the customisation to commands such as `inject`, `dllinject`, `psinject`, `shinject`, `mimikatz`, `powerpick`, and more.  The main notable exception is `execute-assembly`.

The `spawn` (fork & run) variant creates a temporary sacrificial process and injects the post-ex capability into it.  The `explicit` variant injects the post-ex capability into a process that already exists.  Some commands are spawn only (e.g. `powerpick`); some are explicit only (e.g. `inject`); and some can be both (e.g. `mimikatz`).  You can tell which variant a command will use by its arguments - a command that takes a `pid` & `arch` will use the explicit variant, and commands that do not will use spawn.

```text
mimikatz [pid] [arch] [module::command] <args>      <-- EXPLICIT
mimikatz [module::command] <args>                   <-- SPAWN
```

## Inject Kit

The Cobalt Strike [Arsenal](https://download.cobaltstrike.com/scripts) contains a process inject kit to assist with the development of these BOFs.  The default code calls internal Beacon APIs such as `BeaconSpawnTemporaryProcess`, `BeaconInjectTemporaryProcess`, and `BeaconInjectProcess` - the behavours for which are controlled with [Malleable C2](https://hstechdocs.helpsystems.com/manuals/cobaltstrike/current/userguide/content/topics/malleable-c2-extend_process-injection.htm) and other options from the Beacon session.

It's worth noting that you're not limited to using these APIs, but there are some gotchas if you don't use them at all.  For instance, `BeaconSpawnTemporaryProcess` will account for the PPID, spawnto, and blockdll options of the Beacon session.  There is a `BeaconGetSpawnTo` API that will return the current spawnto, but there is no equivilent to fetch the PPID and blockdll settings.  There is also no Aggressor function that can fetch these ([bppid](https://hstechdocs.helpsystems.com/manuals/cobaltstrike/current/userguide/content/topics_aggressor-scripts/as-resources_functions.htm?Highlight=barch#bppid) and [bblockdlls](https://hstechdocs.helpsystems.com/manuals/cobaltstrike/current/userguide/content/topics_aggressor-scripts/as-resources_functions.htm?Highlight=barch#bblockdlls) can only set new values, not return the current values), so they cannot be packed into the BOF arguments either.

It therefore does not seem possible to produce a 1:1 functional relica of the internal Beacon functionality.

Furthermore, you cannot pass any additional [process creation flags](https://learn.microsoft.com/en-us/windows/win32/procthread/process-creation-flags) to `BeaconSpawnTemporaryProcess`, so processes cannot be spawned in a suspended state.  You can call a WinAPI directly, such as [CreateProcessA](https://learn.microsoft.com/en-us/windows/win32/api/processthreadsapi/nf-processthreadsapi-createprocessa) if your injection techniques relies on that, but you will lose the PPID and blockdll integration (you can still hardcode them of course).

In most cases, you will probably want to replace `BeaconInjectTemporaryProcess` and `BeaconInjectProcess` with your own injection logic.  For example:

![](/images/inject-kit/custom-injection.png "Custom Injection Logic")

The actual injection used here is not important - it's just to show that I'm not using the default implementation.  One thing to note is that you should not wait on handles, such as a thread handle.

Once you have custom injection in `process_inject_explicit.c` and `process_inject_spawn.c`, build the kit.

```text
$ ./build.sh /mnt/c/Tools/cobaltstrike/inject
[Process Inject kit] [+] You have a x86_64 mingw--I will recompile the process inject beacon object files
[Process Inject kit] [*] Compile process_inject_spawn.x64.o
[Process Inject kit] [*] Compile process_inject_spawn.x86.o
[Process Inject kit] [*] Compile process_inject_explicit.x64.o
[Process Inject kit] [*] Compile process_inject_explicit.x86.o
[Process Inject kit] [+] The Process inject object files are saved in '/mnt/c/Tools/cobaltstrike/inject'
```

You then load `processinject.cna` via the Script Manager and hey-presto.  Each variant will now use your custom injection.

![](/images/inject-kit/coffee.png "Mimikatz spawn & explict")

## Conclusion

Even though there are some limitations, the process inject kit allows you to apply custom injection techniques to multiple Beacon commands, which is much more versatile than only exposing shinject functionality.  If you're an author of shellcode injection BOFs, I highly recommend building them as part of this kit, rather than standalone.
