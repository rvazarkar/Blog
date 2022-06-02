---
title: "BloodHound 2.1: The Fix Broken Stuff Update"
date: 2019-03-12T12:00:00-05:00
draft: false
tags: ["bloodhound", "sharphound"]
---

Sometimes, you have to step back and look at your code you wrote a while ago. Usually, it's not pretty. Sometimes, it's just flat out wrong. This is one of those times. The 2.1 release of BloodHound has a large focus on bug fixes, and a couple new features including a new attack primitive. This post is going to cover changes we've made since the release of BloodHound 2.0, including some of the incremental changes in between.

# New Attack Primitive - AddAllowedToAct/AllowedToAct
The BloodHound team has been looking for a generic computer ACL attack primitive for quite a while. Thanks to the excellent work of Elad Shamir ([@elad_shamir](https://twitter.com/elad_shamir)), one has finally been found, with additional weaponization and simplification done by Will Schroeder ([@harmj0y](https://twitter.com/harmj0y)). In BloodHound 2.0, we added collection for computers with LAPS, allowing users to determine which principals could read those passwords. The new attack primitive is for **Resource Based Constrained Delegation**, which allows for a generic attack against computer objects provided you can write to the **msds-AllowedToActOnBehalfOfOtherIdentity** property, and you control a user with a ServicePrincipalName set. If you fulfill these conditions, you can gain privileged code execution on the computer itself. BloodHound will also collect users that already have this permission set in the form of the AllowedToAct edge.

![New Edge](/bloodhound21/allowedtoact.png)

For more information on the attack primitive, you should read the incredibly detailed post by Elad Shamir which can be found [here](https://shenaniganslabs.io/2019/01/28/Wagging-the-Dog.html), or the post by Will Schroeder showing a [case study of the attack](https://posts.specterops.io/a-case-study-in-wagging-the-dog-computer-takeover-2bcb7f94c783).

As usual, you can right click on the edge to open the help window and get more information as well.

![Help Text](/bloodhound21/help.png)

# SharpHound Changes
Quite a few new things have been added to SharpHound, either expanding existing functionality, or fixing other stuff.

### Rewrite of ACL Logic
This is probably the biggest fix in 2.1. Windows ACLs are confusing at best, and downright impossible to understand at worst. After a few conversations with [@_dirkjan](https://twitter.com/_dirkjan), who recently wrote the excellent [BloodHound-Python](https://github.com/fox-it/BloodHound.py) project, it was determined that parts of the BloodHound ACL collection logic were either wrong, or missing elements. Unfortunately, the existing logic for collecting ACLs was a mess, and difficult to understand. At the suggestion of Vincent Le Toux ([@mysmartlogon](https://twitter.com/mysmartlogon)), we switched the way we process ACEs to a more modern approach using a different .NET library, which simplified things greatly. As of BloodHound 2.1, all the ACL logic has been rewritten from scratch and covers some edge cases which were missed previously.

### Better Domain Controller Search Code
[@Crypt0-m3lon](https://twitter.com/crypt0_m3lon) came up with the concept for a much more thorough way of searching for usable domain controllers. Previously, SharpHound would just check the primary domain controller and assume it was available. With the new logic, SharpHound will grab a list of domain controllers available for each domain being enumerated, starting with the primary domain controller, and do a quick port check to see if the LDAP service is available. Once the first one is found, it will cache that domain controller for that domain and use that for LDAP queries. If the user supplies a domain controller via the **--DomainController** flag, that will override any other domain controller selection logic.

### Fixed Trust Collection
[@Crypt0-m3lon](https://twitter.com/crypt0_m3lon) also took a look at the logic for collecting domain trusts and realized there were several issues with it. He submitted a very thorough [pull request](https://github.com/BloodHoundAD/SharpHound/pull/61), along with testing to fix the issues with trust collection, which should now be much more accurate.

### GPO Collection for RDP/DCOM Groups
Thanks to the initial work of [@jonas2k](https://github.com/jonas2k), GPO collection was expanded to include the new groups that were added in BloodHound 2.0. Practically, this means that you'll get better data from just GPO collection than you would have previously. It also expanded the logic to include situations where users/groups were added using GPO to the **member** property instead of the **memberof** property. And since more data is generally better, this was a great addition to the collector.

### New Cache File Naming
Mariusz Banach ([@mgeeky](https://github.com/mgeeky))[informed us](https://github.com/BloodHoundAD/SharpHound/issues/48) that some EDR/HIPS solutions were flagging runs of SharpHound based on the name of the cache file that was created (BloodHound.bin). A new method of generating the cache file name which is unique to each system has been implemented. Thanks to [@mgeeky](https://github.com/mgeeky) for the initial issue, as well as for a start to the solution.

![Cache File](/bloodhound21/cachefile.png)

### LdapFilter Parameter
Lee Christensen ([@tifkin_](https://twitter.com/tifkin_)) added the LdapFilter parameter to SharpHound, which allows you to fine tune your collection using the existing LDAP syntax. You can use this to collect single objects, or anything else you could do with LDAP filters. The new filter is appended to the LDAP filter that SharpHound automatically generates for collection.

```posh
SharpHound.exe --LdapFilter "(samaccountname=test2)"
```

### Last Logon Timestamp Fix
A user on the BloodHound slack ([@bluecurby](https://twitter.com/bluecurby)) pointed out that our logic for collecting the Last Logon attribute was collecting the **lastlogon** LDAP attribute. Unsurprisingly, Active Directory stores the actual last logon of the user in the attribute **lastlogontimestamp** and replicates that properly. Thanks to the pull request from bluecurby, the last logon value is now accurate.

### Fixes for ComputerFile Collection
ComputerFile collection is a feature that the BloodHound team very rarely uses, so it doesn't get very much attention. However, Clément Notin ([@cnotin](https://twitter.com/cnotin)), who is a regular contributor to the BloodHound project, went through the ComputerFile enumeration method and fixed several bugs, with the most egregious one being a complete failure in binary logic which led to the ComputerFile method basically ignoring your CollectionMethod flag. He also submitted several fixes to ensure that object names will be properly upper cased, helping to ensure consistency.

### Fixes for LdapUser/LdapPassword on Non-Domain Joined Computers
One of the users of BloodHound, [@webr0ck](https://twitter.com/webr0ck) tracked down an issue with running SharpHound on non-domain joined computers using the LdapUser/LdapPass parameters, and submitted a [pull request](https://github.com/BloodHoundAD/SharpHound/pull/59) to fix the issue. SharpHound should run properly on non-domain joined systems now.

# BloodHound Changes
### Database Warmup Button
Some queries in the BloodHound UI can take quite a bit of time to complete, and we're always looking for ways to optimize the performance of the graph. We've added a button under the database information tab which will allow you to "warm up" the database by loading everything into memory. Obviously on large databases, this will load quite a bit into memory. We tested this one one of the largest databases we've used to date, with upwards of 600,000 nodes and 10 million relationships, and Neo4j grabbed about 6.5 GB of RAM when everything was fully loaded. In return, queries like shortest paths to domain admins were approximately 75% faster. Obviously, results will vary, but the option is there for those who are willing to sacrifice RAM in exchange for performance.

![Warmup DB](/bloodhound21/newdbinfo.png)

### Query Optimizations and New Queries
Several of the prebuilt queries in BloodHound have been reworked or optimized to greatly increase performance. In particular, the DCSync queries have seen a massive performance increase. We've also added some new queries on different tabs, which should give you more information when viewing nodes.

### Better/Fixed Upload Logic
A frequent complaint from users was that uploading of files was very slow. This was a result of a compromise that had to be made after finding a domain where GPO files resulted in over 100000 entries being set in single GPO. After some tweaking, the ingestion logic has been changed so it will be significantly faster, but still parse massive files properly. There were also a few issues with the ingestion logic from 2.0 that were fixed. Another shoutout to [@_dirkjan](https://twitter.com/_dirkjan) for some of those fixes.

### Dont Require Preauth Flag, Sensitive Flag
We've added a few new flags to user objects, particularly the **dontreqpreauth** and **sensitive** properties on objects. The first flag allows you to find users that are ASREP Roastable, which is similar to Kerberoasting. You can find a post about it [here](https://www.harmj0y.net/blog/activedirectory/roasting-as-reps/). The second flag corresponds to the "This account is sensitive and can't be delegated" checkbox in active directory. 

### Updated Help Text
We've updated the help text for several different attack primitives. [@harmj0y](https://twitter.com/harmj0y) went ahead and added more information to the constrained delegation attack that was added in BloodHound 2.0, and should make it easier to exploit. We've tweaked help text a bit here and there for clarity.

### Update Query Debug Mode
The query debug mode has been a valuable tool for people attempting to learn cypher and understand how the interface does queries. Unfortunately, it was never updated properly to deal with the edge filtering logic that was introduced in 2.0. We've corrected that issue in 2.1, and also upgraded the output so it properly formats the parameters into the query. The result is you get the full query instead of one missing edge specifications and parameter values.

As an example, the output of the Shortest Paths to Domain Admins query used to look like this:

```
MATCH (n:Group {name:{name}}),(m),p=shortestPath((m)-[r:{}*1..]->(n)) WHERE NOT m=n RETURN p
```

With the updates, it'll look like this instead:

```
MATCH (n:Group {name:"DOMAIN ADMINS@TESTLAB.LOCAL"}),(m),p=shortestPath((m)-[r:MemberOf|HasSession|AdminTo|AllExtendedRights|AddMember|ForceChangePassword|GenericAll|GenericWrite|Owns|WriteDacl|WriteOwner|CanRDP|ExecuteDCOM|AllowedToDelegate|Contains|GpLink|ReadLAPSPassword|AddAllowedToAct*1..]->(n)) WHERE NOT m=n RETURN p
```

# Final Thoughts
BloodHound 2.1 isn't quite as world changing as the 2.0 release, but the fixes should make analysis more efficient and less error-prone. The accuracy of data should also be better overall. We're interested to know if anyone gets to actually exploit the new constrained delegation attack, as it represents one of the most complex attack primitives in the graph at this point. As usual, you can grab compiled versions of the [user interface](https://github.com/BloodHoundAD/BloodHound/releases) and the [collector](https://github.com/BloodHoundAD/BloodHound/tree/master/Ingestors) from here, or self-compile from our GitHub repository for [BloodHound](https://github.com/BloodHoundAD/BloodHound) and [SharpHound](https://github.com/BloodHoundAD/SharpHound).

We want to particularly thank the community for a lot of suggestions and fixes, which helped simplify the development cycle for the BloodHound team for this release. Many important fixes were either directly contributed by the community or found and reported.

You can always find us in the [BloodHound Slack Channel](https://bloodhoundgang.herokuapp.com/), which just recently passed the 3000 user mark. The link will allow you to get an invite and join our active community. We look forward to hearing from new and old users alike as we work to make our networks a safer place.

