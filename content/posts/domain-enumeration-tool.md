---
title: "Domain Enumeration Tool"
date: 2021-04-28T16:32:31+01:00
draft: false
tags:
  - c#
  - .net
  - ldap
authors:
  - rastamouse
---



## Introduction

A couple of days ago (obviously at the time of writing this post), [FortyNorth Security](https://fortynorthsecurity.com/) [tweeted](https://twitter.com/FortyNorthSec/status/1386684411987599361) about a new tool they were releasing called [EDD (Enumerate Domain Data)](https://github.com/FortyNorthSecurity/EDD).  This is a .NET Console Application that aims to provide a .NET alternative to the kind of domain enumeration that [PowerView](https://github.com/PowerShellMafia/PowerSploit/tree/master/Recon) is well known for (a bit like [SharpView](https://github.com/tevora-threat/SharpView)).

I spent some time looking at the EDD codebase and ended up submitting a fairly substantial [PR](https://github.com/FortyNorthSecurity/EDD/pull/1).  Refactoring EDD sparked a fresh interest in Windows domain enumeration via LDAP, so I decided to write a similar tool.  Not because I thought I could do better than EDD, but because it provided an opportunity to do some things that I hadn't before.

1. Learn about LDAP queries

2. Create a NuGet package



The result is the Domain-Enumeration-Tool (we are an imaginative bunch when it comes to naming things).  DET can be found on both [GitHub](https://github.com/ZeroPointSecurity/Domain-Enumeration-Tool) and the [NuGet Gallery](https://www.nuget.org/packages/Domain-Enumeration-Tool/).



## Usage



DET is a .NET Standard library, which can be used with any flavour of .NET project.  Just create your project (e.g. a console app) and install the NuGet package.



### Domain Searcher

The first step is to construct the `DomainSearcher`.

```c#
var searcher = new DomainSearcher();
```



In the background, this creates a new instance of the `System.DirectoryServices.DirectoryEntry` class.  Using the default overload (with no parameters) allows this to be instantiated using the domain and security context of the executing principal.

For instance, if you were executing your console app via a foothold running as a Domain User, the subsequent LDAP queries would be executed within the context of that user.

`DomainSearcher` has additional overloads that allows you to provide a specific LDAP path, as well as a username and password to authenticate with.  These are useful for running enumeration from a machine that is not domain joined.

This would look something like:

```c#
var searcher = new DomainSearcher(
	"LDAP://192.168.1.1,DC=testlab,DC=local",
	"LAB\\user",
	"Passw0rd!");
```



### Users, Computers, Groups, etc

With the `DomainSearcher`, it's time to create the class(es) that correspond to the data type you want to collect.  DET (currently) has classes for `Domain`, `Users`, `Groups`, `Computers`, `GPOs` and `OUs`.  Each of these take the `DomainSearcher` on their constructors.

To search for users, create a `Users` object:

```c#
var users = new Users(searcher);
```

`Users` has a `GetUsers(string[] userNames = null, string[] properties = null)` method.  Both the `userNames` and `properties` parameters are optional.



```c#
var userAccounts = users.GetUsers();
```

will return every property for every user in the domain.



Maybe you do want all the users, but only the `samAccountName` property?  No problem.

```c#
var userAccounts = users.GetUsers(properties: new string[] { "samAccountName" });
```



The `userNames` and `properties` parameters can be used in combination to return the volume of data desired.



### Dictionary<string, Dictionary<string, object[]>>

The format of the data returned from these methods are kind of funky - mostly a result of how the `System.DirectoryServices.DirectorySearcher` works.

The above query creates a data structure that looks like this (assuming only 1 user is present):

```tex
Dictionary<string, Dictionary<string, object[]>>
{
    {
        "LDAP://CN=Administrator,CN=Users,DC=testlab,DC=local",
        Dictionary<string, object[]>
        {
            { "adspath", object[] { "LDAP://CN=Administrator,CN=Users,DC=testlab,DC=local" }},
            { "samaccountname", object[] { "Administrator" }}
        }
    }
}
```



The `ADSPath` is always returned as the dictionary key and is always present in the dictionary of properties.  The data returned for each property is an `object[]` for two reasons:

1. The datatype can be anything, including a `string`, `byte[]`, `DateTime`, etc.
2. Some properties can have more than one value, such as `servicePrincipalName`.



Naturally we can iterate over and/or access any property by its name.

```c#
Console.WriteLine("SamAccountNames:");

foreach (var userAccount in userAccounts.Values)
{
    var sam = userAccount["samaccountname"][0];
    Console.WriteLine($"- {sam}");
}
```



```text
SamAccountNames:
- Administrator
- Guest
- DefaultAccount
- krbtgt
- user1
```



The other classes work in exactly the same way.



### Raw LDAP

DET also has an `LDAP` class with an `ExecuteQuery(string filter, string[] properties = null)` method.  This allows you to execute any LDAP query that you want.

For example - find users that have Kerberos pre-authentication disabled.

```c#
var ldap = new LDAP(searcher);
var filter = "(&(sAMAccountType=805306368)(userAccountControl:1.2.840.113556.1.4.803:=4194304))";

//don't return any properties
var properties = new string[] { "" };
var users = ldap.ExecuteQuery(filter, properties);

foreach (var user in users)
{
    Console.WriteLine(user.Key);
}
```

```text
LDAP://CN=User One,CN=Users,DC=testlab,DC=local
```



## Future Work

Admittedly, there isn't a tremendous amount of functionality here yet.  Over time I hope to add more queries into the various classes for returning useful information.  Specifically on my radar are:

- Kerberos "things" (roastables, delegation etc)
- Domain Trusts
- GPO Links
- DACLs
- LAPS