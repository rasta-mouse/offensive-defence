---
title: "Offensive Defence"
date: 2020-06-05T10:04:14+01:00
authors:
    - cyberzombie
draft: false
---

So welcome along to our joint blog with [@_RastaMouse](/authors/rastamouse/) [@Rythmstick](/authors/rythmstick/) and [@Furby](/authors/furby/), I'm [@CyberZombi3](/authors/cyberzombie/) we hope to blog about all levels of Red Team and Blue teaming, our combined skillsets, areas of expertise and desire to gain knowledge from one another should provide a little something for everyone so with that said this is my first blog here, which can also can be found at  [cyberzombi3.co.uk](https://cyberzombi3.co.uk/)

I usually blog in the style of a basic guide, for this blog though I wanted to throw out a few suggestions around Offensive Defence (very apt don’t you think) Think how an attacker starts their mission, with some recon right. So wouldn’t it be great to have an early warning system for a potential phishing attack, maybe a password spray against your O356 environment. How do we go about this ?

## Fake Profiles

What we want to do is setup a fake profile and also a blank user account, I don’t believe you can create a Linkedin profile as it would be against their terms and conditions however there’s nothing to say that you can have a fake COO or another high value target listed on your own corporate website. To sit along side this you would also have an AD user for the fake target. It’s worth mentioning that you should maybe configure the account logon hours to zero, give the account access to as little as possible just to be on the safe side.

What you can then do with your logging / monitoring platform of choice is to monitor for emails being sent to that user, also monitor for login attempts via your O365 system and or others.

## High Value AD Account

Similar to the above, create an AD account that looks highly valuable to an attacker, think something along the lines of the name SuperAdmin, again monitor for login attempts of this user. The bonus with this one is that if there were some potential malicious activity you would have to assume that the attacker is already on your systems, be it an external attacker or a malicious insider. You would also want your server admins and potentially other admins to be aware of this account and forbidden from using it. This would help with any false positives.

## Fake News

You could also give yourselves an idea of whether or not your website is being actively scanned around the times or merges / takeovers or some other exciting time for the business.

On your corporate website in the latest new section or deeper within the site you could list some exciting new projects with code names, think Greek gods and names like “The Messiah Project” it peaks an interest. Again via your monitoring platform of choice you would look out for these codenames, if you receive an alert you can be sure that your sure that your website has been scanned and that someone is looking to collect further information on it.

I guess the limits are kind of endless but I just wanted to bring your attention to such methods as I don’t think I have come across to many posts along these lines, be creative. There’s always the good old honeypots too.
I hope this was of some help to someone out there.

Thanks

CyberZombi3
