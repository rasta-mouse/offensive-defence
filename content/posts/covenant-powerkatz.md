---
title: "Updating Mimikatz in Covenant"
date: 2022-05-13T10:38:32+01:00
draft: false
authors:
    - RastaMouse
---

There's an [issue](https://github.com/cobbr/Covenant/issues/358) on Covenant's GitHub asking how to update the bundled version of Mimikatz.  This post will address that question and also outline how I went about finding the answer.  I cloned the dev branch to look at this, because I really didn't feel like installing .NET Core 3.1 which main is still using.  Running a mimikatz command, I can see the build is from Feb 2021.

![](/images/covenant-powerkatz/katz-2021.png "Mimikatz Feb 2021")

Covenant uses [SharpSploit](https://github.com/cobbr/SharpSploit) as a dependancy, which itself has Mimikatz (or more accurately, Powerkatz) built-in.  You can find the different builds of Powerkatz inside the `ReferenceSourceLibraries\SharpSploit` directory.

```
PS C:\Tools\Covenant\Covenant\Data\ReferenceSourceLibraries\SharpSploit\SharpSploit\Resources> ls

    Directory: C:\Tools\Covenant\Covenant\Data\ReferenceSourceLibraries\SharpSploit\SharpSploit\Resources

Mode                 LastWriteTime         Length Name
----                 -------------         ------ ----
-a---          13/05/2022    10:33        1116672 powerkatz_x64.dll
-a---          13/05/2022    10:33         677330 powerkatz_x64.dll.comp
-a---          13/05/2022    10:33         936960 powerkatz_x86.dll
-a---          13/05/2022    10:33         608852 powerkatz_x86.dll.comp
```

My assumption was just to replace these files with updated builds from the Mimikatz repo, and hey-presto. However, UserXGnu stated that that does not actually work.  I tried this myself, and sure enough, Covenant wasn't using them.

The next place I looked was the task definitions for the various Mimikatz commands, such as LogonPasswords.  These can be found at [https://github.com/cobbr/Covenant/blob/master/Covenant/Data/Tasks/SharpSploit.Credentials.yaml](https://github.com/cobbr/Covenant/blob/master/Covenant/Data/Tasks/SharpSploit.Credentials.yaml).  Task definitions in Covenant are quite large, but in this case, we're looking for references (no pun intended) to where they're pulling external resources.

I found these enteries, which show x86 and x64 builds being pulled from a different location on disk.

```
EmbeddedResources:
  - Name: SharpSploit.Resources.powerkatz_x86.dll
    Location: SharpSploit.Resources.powerkatz_x86.dll
  - Name: SharpSploit.Resources.powerkatz_x64.dll
    Location: SharpSploit.Resources.powerkatz_x64.dll
```

```
PS C:\Tools\Covenant\Covenant\Data\EmbeddedResources> ls

    Directory: C:\Tools\Covenant\Covenant\Data\EmbeddedResources

Mode                 LastWriteTime         Length Name
----                 -------------         ------ ----
-a---          13/05/2022    10:27        1451520 SharpSploit.Resources.powerkatz_x64.dll
-a---          13/05/2022    10:28        1212416 SharpSploit.Resources.powerkatz_x86.dll
```

These are the actual DLLs you need to replace. Interestingly, they are read directly from disk each time, so you can replace them whilst Covenant and Grunts are running in case you need to change them on-the-fly.

![](/images/covenant-powerkatz/katz-2022.png "Mimikatz May 2022")