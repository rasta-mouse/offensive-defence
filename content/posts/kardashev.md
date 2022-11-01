---
title: "The Information Security Kardashev Scale"
date: 2022-11-01T10:22:58Z
draft: false
authors:
    - RastaMouse
---

The Kardashev scale is a method of measuring a civilization's level of technological advancement based on the amount of energy it is able to use.  [Kurzgesagt](https://kurzgesagt.org/) have a fun video on the subject if you're not familiar with the premise.

| ![](/images/kardashev/kardashev.png "Kardashev Scale") |
|:--:|
| [What Do Alien Civilizations Look Like?](https://www.youtube.com/watch?v=rhFK5_Nx9xY&) |

I found myself wondering if such a scale could be applied to those of us working in information security.  To that end, I took a broad look at the type of connections I have across Twitter, LinkedIn, Discord, Slack, etc, and attempted to categorise them based on how their contributions have advanced the information security field.

> \> *This is just for fun, please don't lynch me.*

## Type 1

Type 1's are actively trying to break into into the field or are still new to it - perhaps just graduating from full-time education or coming in as a career change.  They learn from existing resources (courses, books, videos, mentors, etc) and can almost never contribute anything new back.

The aim of those in this phase is to obtain a good foundational understanding of their area of study - either to land their first job or to work competently in a junior role that they've already obtained.

Once in a role, those that are exposed to richer and more varied experiences typically grow faster than those that aren't.

## Type 2

Type 2's have solid contextual knowledge of their field which allows them to expand their own thinking.  If a Type 1 is taught that A = B and B = C; a Type 2 has enough experience and awareness to conclude that A = C.  That may not be a great analogy, but if you've ever come up with a new idea and asked yourself why you hadn't thought of it before, then this could be an example of your knowledge growth allowing you to form links that were previously out of reach.

This process allows Type 2's to begin expanding their own knowledge independently - challenging assumptions, coming up with and testing their own ideas, building various scripts or toolsets, and adding their own twists onto existing methodologies.  The output of which may be published and consumed by others, which often represents their first meaningful contributions to the field.

## Type 3

Type 3's are excellent practioners who are extremely adapatable with exceptional problem solving abilities.  They set themselves apart from Type 2's by investing significant time and effort into security research (either independant or building off the research of other Type 3 and 4's).

Two examples of this are the "An ACE Up the Sleeve" and "Certified Pre-Owned" whitepapers from Will Schroeder, Andy Robbins and Lee Christensen.  These are exceptional because they exposed new attack surfaces that nobody was looking at prior to their publications which had a profound impact on how we view, test and audit security controls around those components.

## Type 4

Type 4's are pure security researchers - the James Forshaw's of the world.  They dedicate all of their time to finding new flaws and vulnerabilities, and reporting them to the relevant vendor.  They're not practitioners (in that they don't do penetration testing etc for a living) and therefore only release proof of concept code for the issues that they find.

Nevertheless, they provide the foundations for brand new attack primitives (e.g. [Kerberos Relaying](https://googleprojectzero.blogspot.com/2021/10/using-kerberos-for-authentication-relay.html)), which are often weaponised into fully ledged tools by Type 3's (and sometimes Type 2's).
