---
title: "SharpSploit v1.6 Updates"
date: 2020-06-09T13:12:56+01:00
authors:
    - rastamouse
tags:
    - "c#"
    - ".net"
    - "sharpsploit"
draft: false
---

[Ryan Cobb](https://twitter.com/cobbr_io) has just tagged the release of SharpSploit [v1.6](https://github.com/cobbr/SharpSploit/releases/tag/v1.6), which comes with a number of cool changes.  The most significant of which includes some very clever Dynamic Invocation functionality that [TheWover](https://twitter.com/TheRealWover) has [blogged about](https://thewover.github.io/Dynamic-Invoke/).  My contributions were relatively minor and will be the subject of this post.

## Enhanced WMI Output

SharpSploit has had a `WMIExecute` method in the `SharpSploit.LateralMovement` namespace for as long as I can remember.  It took in a target `ComputerName`, a `Command` to execute and optional plaintext creds, but it returned a `bool`.

```c#
public static bool WMIExecute(string ComputerName, string Command, string Username = "", string Password = "")
```

The remaining method looked a bit like this:

```c#
try
{
    // do the WMI magic

    Console.WriteLine("Win32_Process Create returned: " + outParams["returnValue"].ToString());
    Console.WriteLine("ProcessID: " + outParams["processId"].ToString());
    return true;
}
catch
{
    Console.Error.WriteLine("WMI Exception:" + e.Message);
    return false;
}
```

There were a few issues with this, at least from my perspective.

1. It's writing to the application's console.

    We may not have visibility of the console if SharpSploit is being executed in a RAT such as Convenant's Grunt and if we've injected the RAT into a process that doesn't even have a console, this may crash the process.

2. There is no validation on the `returnValue` before returning `true`.

    Veterans of WMI will know that there are a few [return codes](https://docs.microsoft.com/en-us/windows/win32/wmisdk/wmi-return-codes) that are possible and not all of them indicate successful execution.  This is particularly troublesome when built into places like Covenant's [WMI lateral movement task](https://github.com/cobbr/Covenant/blob/master/Covenant/Data/Tasks/SharpSploit.LateralMovement.yaml#L24-L31), as the task can report execution was successful when it wasn't.

To try and improve this, the method was reworked to return a `SharpSploitResultList` of type `WmiResult`.  `WmiResult` is a simple class that contains two properties for the actual `ReturnValue` and `ProcessID`.

```c#
SharpSploitResultList<WmiResult> wmiResult = new SharpSploitResultList<WmiResult>();

try
{
    // do the WMI magic

    wmiResult.Add(new WmiResult
    {
        ReturnValue = outParams["returnValue"].ToString(),
        ProcessID = outParams["processId"].ToString()
    });
}
catch { }

return wmiResult;
```

And this is how it looks:

```text
C:\>WmiDemo.exe WIN-CJ8120QPH84 C:\Windows\System32\win32calc.exe

ReturnValue  ProcessID
-----------  ---------
0            1196
```

```text
PS C:\Users\Administrator> hostname
WIN-CJ8120QPH84

PS C:\Users\Administrator> Get-Process -Id 1196

Handles  NPM(K)    PM(K)      WS(K)     CPU(s)     Id  SI ProcessName
-------  ------    -----      -----     ------     --  -- -----------
    125      13     4792      11804       0.03   1196   0 win32calc
```

## Reverse Port Forwarding

The larger change on my part came with the addition of the `SharpSploit.Pivoting` namespace and new methods for creating, listing and stopping reverse port forwards.  I won't cover the code in intricate detail - just some usage examples.

### Starting

Creating a new reverse port forward is as easy as:

```c#
ReversePortForwarding.CreateReversePortForward(8080, "httpbin.org", 80);
```

Where `8080` is the bind port, `httpbin.org` is the forward host (IP addresses and domain names are supported) and `80` is the forward port. This method returns a `bool`.

If successful, you will see port `8080` bound on the host.

```text
C:\>netstat -anp tcp | findstr 8080
  TCP    0.0.0.0:8080           0.0.0.0:0              LISTENING
```

### Listing

You can list current reverse port forwards with:

```c#
var list = ReversePortForwarding.GetReversePortForwards();
Console.WriteLine(list);
```

This returns a `SharpSploitResultList` of type `ReversePortFwdResult`. This class contains the `BindAddress`, `BindPort`, `ForwardAddress` and `ForwardPort`.

```text
BindAddresses  BindPort  ForwardAddress  ForwardPort
-------------  --------  --------------  -----------
0.0.0.0        8080      52.6.108.56     80
```

This test machine cannot talk to the Internet directly.

```text
PS C:\Users\Administrator> curl http://httpbin.org/base64/SFRUUEJJTiBpcyBhd2Vzb21l
curl : The remote name could not be resolved: 'httpbin.org'
```

But it can via the reverse port forward.

```text
PS C:\Users\Administrator> $data = curl http://DESKTOP-U3N86EQ:8080/base64/SFRUUEJJTiBpcyBhd2Vzb21l
PS C:\Users\Administrator> $data.RawContent
HTTP/1.1 200 OK
Connection: keep-alive
Access-Control-Allow-Origin: *
Access-Control-Allow-Credentials: true
Content-Length: 18
Content-Type: text/html; charset=utf-8
Date: Tue, 09 Jun 2020 13:53:48 GMT
Server: gunicorn/19.9.0

HTTPBIN is awesome
```

### Deleting

There are two methods for removing reverse port forwards.  `FlushReversePortFowards` (returns `void`) will indiscriminately remove all of them and `DeleteReversePortForward` (returns `bool`) will remove a single reverse port forward by its bind port.

```c#
ReversePortForwarding.DeleteReversePortForward(8080);
```

Obviously when the reverse port forward is stopped, the port is released.

## Conclusion

The updates to WMI are more quality of life changes and should be reflected in the relevant Covenant Tasks soon.  I hope that the new reverse port forwarding can help bring some additional capabilities to similar tools that leverage SharpSploit.
