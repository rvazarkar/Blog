---
title: "SharpHound: Evolution of the BloodHound Ingestor"
date: 2017-09-19T12:00:00-05:00
tags: ["bloodhound", "sharphound"]
---

Over the past few months, the BloodHound team has been working on a complete rewrite of the C# ingestor. One of the biggest problems end users encountered was with the [current](https://github.com/BloodHoundAD/BloodHound/blob/master/PowerShell/BloodHound.ps1) (soon to be replaced) PowerShell ingestor, particularly in speed of enumeration as well as crippling memory usage. In moderately sized environments, the ingestor would happily eat up gigabytes of memory. There's lots of reasons for this, almost all to do with the limitations of using PowerShell V2 as the base language.

## Limitations of the Current Ingestor
A huge problem with the current ingestor is that PowerShell threading is at best, a hack. Will Schroeder ([@harmj0y](https://twitter.com/harmj0y)) did an amazing job with the initial PowerShell ingestor, and his implementation uses a very hacky workaround using runspaces to create a form of threading, which yielded a pretty substantial speed boost for enumeration when it was first implemented.

However, maintaining the code was a complete nightmare since nothing could be broken off into separate modules, just by the nature of the way we use PowerShell on offensive engagements. Additionally, the runspaces tend to leak unmanaged memory, as well as consume memory themselves, which contributed to the extreme memory usage problem. In light of these problems, the decision was eventually made to stop trying to force the ingestor to behave and just rewrite it from scratch in C#. This gave us the freedom to structure everything the way we wanted to, and to write in the features that we wanted, but couldn't properly implement.

## Something Old
When you first get to use the new ingestor, you'll find that most of your experience will be the same as before. The flags for the rewrite were intentionally kept very similar to the older version. However, there are some new flags and new options that are important to note that fulfill feature requests we've gotten repeatedly from different people. In a future post, we'll look at some of the technical differences between the old and new ingestor, but this post will focus more on operational knowledge. First, we'll talk about old features that have been improved.

### Expanded Threading
Previously, the computer enumeration methods were threaded using runspaces. In the new ingestor, we've implemented threading into ACL and group membership enumeration as well. Domain trust enumeration is still not threaded, but the process has been changed and generally takes a very short amount of time. Threading was not implemented for trust enumeration due to technical limitations as well as there being little to no performance increase displayed. One of the biggest bottlenecks on large domains was the time it took for Group Membership data to be queried from Active Directory. With the threading changes in SharpHound, the process is now significantly faster.

### Concurrent Enumeration
We also decided we didn't particularly like the way you had to wait for one enumeration method to finish to get to another one. The end result of changes made is that all the numeration run concurrently. In the current ingestor, running the default collection would perform trust enumeration, followed by group enumeration and then the computer enumeration methods. Depending on which collection method you use, the C# ingestor will run each object queried from Active Directory through the proper functions all at once.

### Modular Stealth Enumeration
Stealth enumeration has changed as well. We've rolled the changes for parallel enumeration into stealth options, and stealth is no longer a collection method. Instead it's a flag. For example, you could do something like this:

```terminfo
Invoke-BloodHound -CollectionMethod Session -Stealth
```

or if you prefer the executable variant:

```terminfo
.\SharpHound.exe --CollectionMethod Session --Stealth
```

Because the new ingestor is written in C#, there is both a PowerShell script as well as a regular .NET executable available.

### Accurate Local Admin Enumeration
Another issue several users have run into, particularly in areas outside the US, is localization issues with the Local Administrators group on systems. In several languages, the Administrators group was translated to another name. In addition, some organizations rename their Administrators group to other names to obscure the group. A new method of enumeration has been created in SharpHound which completely bypasses either of these changes, which should lead to much more accurate data.

## Something New
Obviously a brand new rewrite couldn't just have upgraded old features, we need some cool new things as well!

### Session Loop
One of the big highlights is a new collection method, something people have asked for before and something that people have used hacky solutions for. Invoking BloodHound with the **SessionLoop** collection method will now put the ingestor in an infinite loop collecting sessions until you tell it to stop. The behavior of this can be configured with a few parameters:

```terminfo
Invoke-BloodHound -CollectionMethod SessionLoop -LoopTime 2 -MaxLoopTime 1d1h10m30s
```

The **LoopTime** parameter controls time in between loops in minutes and the **MaxLoopTime** parameter controls the end time of the looping. The command shown will instruct the ingestor to loop session collection, with a delay of 2 minutes in between each loop, and to continue looping until 1 day, 1 hour, 10 minutes, and 30 seconds have passed, at which point the collection will stop. Here's what it looks like in action:

![SessionLoop]({{ site.url }}/assets/sharphoundintro/sessionloop.png)

### Caching
Another new feature we've added is a caching mechanism. When running the ingestor, a large part of the overhead involved is in doing LDAP requests to resolve SIDs or DistinguishedNames to names we can use in BloodHound. The C# ingestor by default will store mappings for these values in a .bin file. When a first run of BloodHound is finished, the ingestor will write out the file in your current directory. In future runs of BloodHound, the cache file will be loaded and the data will be used whenever possible. This results in a fairly significant speed improvement, as well as a significant reduction in network requests whenever you run BloodHound again with the cache file loaded. You can modify the behavior of the cache with some new flags. For example:

```terminfo
Invoke-BloodHound -CacheFile MyCache.bin -Invalidate
```

The **CacheFile** parameter allows you to change the name of the file for both loading and saving. The **Invalidate** parameter allows you to force the ingestor to ignore any cached data and build a new one. If you don't want to write the cache file out to disk for whatever reason, you can use the **NoSaveCache** parameter:

```terminfo
Invoke-BloodHound -NoSaveCache
```

If you can save the cache file, it's highly recommended you do so if you plan to ever run BloodHound more than once.

### CSV Compression
Yet another new feature deals with getting your CSV data out over your C2 agent. It can be a real pain to download your data, especially if you're using a C2 agent. A new flag is introduced to help deal with this:

```terminfo
Invoke-BloodHound -CompressData
```

When enumeration is finished, you'll be presented with a new file.

![CompressData]({{ site.url }}/assets/sharphoundintro/CompressData.png)

The compressed zip file contains all the CSV files output by your last run, conveniently timestamped to allow you to keep track of what you get from different runs. The compression is smart, and will only zip up files that were actually changed in your last collection. If you look closely at the screenshot you'll see another cool feature we've added, real progress output!

### Progress Output
Another issue many people had with the current ingestor is the lack of progress output. When running the ingestor, it was often difficult to figure out how far along the progress was, or of it was even doing anything at all. With the current ingestor, progress was available with the **Verbose** flag, but would also create a significant amount of other output. The new ingestor has been modified to keep track of progress. Because all the enumeration is collapsed into one, the number of objects shown in the progress represents objects that are fully enumerated through all collection steps. While we still don't have a way to count the total number of objects without a significant amount of overhead, SharpHound will give you output every 30 seconds by default. This can be configured with yet another flag:

```terminfo
Invoke-BloodHound -StatusInterval 15000
```

The **StatusInterval** flag specifies the number of milliseconds in between each progress message. The example above will change the interval to 15 seconds.

### Secure LDAP
While this isn't a feature that was specifically requested much, since we were in the process of rewriting the entire thing anyways, it felt like something to add. Currently, BloodHound (and indeed pretty much every AD reconnaissance tool used currently) uses regular old LDAP on port 389. LDAP is very fast, but it's also completely in plaintext. It's about as secure as FTP. For those more security-minded, the option to enable LDAPs (LDAP Secure) is now available with the **SecureLDAP** flag.

```terminfo
Invoke-BloodHound -SecureLdap -IgnoreLdapCert
```

The **IgnoreLdapCert** option can be used if your organization uses a self-signed certificate for SSL. When the SecureLDAP flag is used, SharpHound will communicate with Domain Controllers over port 636, the default for Secure LDAP.

## Memory and Speed
While it is very difficult to get accurate comparisons between the PowerShell and C# versions of the ingestor due to many factors such as network latency, request caching, and just general variance of systems in Active Directory, the new version is significantly faster than the old version, particularly in the group membership enumeration phase. Some rough numbers from some recent testing are below. Note that an object can be a user, group, or computer, and these stats are based on doing Group Collection, Local Admin Collection, and Session Collection.

| Objects in Domain | Time Taken | Cache Built | Other Comments |
| :---| :---: | :---: |:---|
| ~35000 | 25 minutes 34 seconds | No |  |
| ~3000 | 30 seconds | No | Enumeration over VPN |
| ~370000 (not a typo) | 9 hours 42 minutes | No | Enumeration over VPN. Powershell ingestor never finished after 3 days |
| ~130000 | 40 minutes | No |  |
| ~40500 | 10 minutes | No | Old ingestor took 4 hours 30 minutes. Enumeration over VPN |

On a domain, we also did runs of individual collection methods to determine the difference between the two ingestors.

| Collection Method | Old Ingestor | New Ingestor
| :--- | :---: | :---: |
| Group | 1 minute 10 seconds | 19 seconds |
| LocalGroup | 29 minutes 57 seconds | 6 minutes 21 seconds |
| Session | 29 minutes 1 second | 5 minutes 36 seconds |
| ACL | 10 minutes 20 seconds | 37 seconds |

Additionally, the memory usage issue has been largely solved. In a very large run of SharpHound, the memory usage hovered around 200mb of data used. Several underlying changes were made to the structure of the code to ensure that memory usage would stay much lower, and a few memory leaks were identified and patched. Thanks to these changes, SharpHound should remain stable in the most crazy of environments, and should be runnable without needing a small server farm worth of memory.

## Wrap Up
To wrap up this post, here's a quick rundown of all the flags in SharpHound and what they do.

### Enumeration Options
* **CollectionMethod** - The collection method to use. This parameter will accept a comma seperated list of values. Has the following potential values (Default: Default):
   * **Default** - Performs group membership collection, domain trust collection, local admin collection, and session collection
   * **Group** - Performs group membership collection only
   * **LocalGroup** - Performs local admin collection only
   * **GPOLocalGroup** - Performs local admin collection using Group Policy Objects
   * **Session** - Performs session collection only
   * **ComputerOnly** - Performs local admin collection and session collection
   * **LoggedOn** - Performs privileged session collection (requires admin rights on target systems)
   * **Trusts** - Performs domain trust enumeration for the specified domain
   * **ACL** - Performs collection of ACLs
   * **Container** - Performs collection of Containers
   * **All** - Performs all Collection Methods

* **SearchForest** - Search all the domains in the forest instead of just your current one
* **Domain** - Search a particular domain. Uses your current domain if null (Default: null)
* **Stealth** - Performs stealth collection methods. All stealth options are single threaded.
* **SkipGCDeconfliction** - Skip Global Catalog deconfliction during session enumeration. This can speed up enumeration, but will result in possible inaccuracies in data.
* **ExcludeDc** - Excludes domain controllers from session enumeration (avoids Microsoft ATA flags :) )
* **ComputerFile** - Specify a file to load computer names from

### Connection Options
* **SecureLdap** - Connect to AD using Secure LDAP instead of plaintext LDAP.
* **IgnoreLdapCert** - Ignores LDAP SSL certificate. Use if there's a self-signed certificate for example

### Performance Options
* **Threads** - Specify the number of threads to use (Default: 10)
* **PingTimeout** - Specifies the timeout for ping requests in milliseconds (Default: 250)
* **SkipPing** - Instructs Sharphound to skip ping requests to see if systems are up
* **LoopTime** - The number of minutes in between session loops (Default: 5)
* **MaxLoopTime** - The amount of time to continue session looping. Format is 0d0h0m0s. Null will result in infinite looping (Default: null)
* **Throttle** - Adds a delay after each request to a computer. Value is in milliseconds (Default: 0)
* **Jitter** - Adds a percentage jitter to throttle. (Default: 0)

### Output Options
* **CSVFolder** - Folder in which to store CSV files (Default: .)
* **CSVPrefix** - Prefix to add to your CSV files (Default: "")
* **Uri** - Url for the Neo4j REST API. Format is http(s)://SERVER:PORT (Default: null)
* **UserPass** - Username and password for the Neo4j REST API. Format is username:password (Default: null)
* **CompressData** - Compresses CSV files to a single zip file after completion of enumeration
* **RemoveCSV** - Deletes CSV files from disk after run. Only usable with the **CompressData** flag

### Cache Options
* **CacheFile** - Filename for the Sharphound cache. (Default: BloodHound.bin)
* **NoSaveCache** - Don't save the cache file to disk. Without this flag, BloodHound.bin will be dropped to disk
* **Invalidate** - Invalidate the cache file and build a new cache

### Misc Options
* **StatusInterval** - Interval to display progress during enumeration in milliseconds (Default: 30000)
* **Verbose** - Enables verbose output


We're hoping the new ingestor solves a lot of the problems people are currently having. The newest release of SharpHound can be found in the BloodHound repository under the Ingestors folder, [here](https://github.com/BloodHoundAD/BloodHound/tree/master/Ingestors). The source code can be found [here](https://github.com/BloodHoundAD/Sharphound). As always, you can get pre-compiled releases of the BloodHound user interface for most platforms on the repository at [https://github.com/BloodHoundAD/BloodHound/releases](https://github.com/BloodHoundAD/BloodHound/releases).  We look forward to hearing any feedback anyone has for SharpHound. You can always join us in the BloodHound slack channel. To get yourself an invite, you can go to [https://bloodhoundgang.herokuapp.com](https://bloodhoundgang.herokuapp.com).

#### Updates
Added the DisableKerbSigning option.

Added a link to get SharpHound

Added the CompressData and RemoveCSV Flags. Added the ability to specify multiple CollectionMethods

Added Throttle, Jitter, and multiple CollectionMethods

Change the URI parameter to accept http/https