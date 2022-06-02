---
title: "SharpHound: Technical Details"
date: 2017-10-23T12:00:00-05:00
draft: false
tags: ["bloodhound", "sharphound"]
---

In the [previous](https://blog.cptjesus.com/posts/newbloodhoundingestor) blog post, we focused on SharpHound from an operational perspective, discussing some of the new features, as well as improved features from the original ingestor. In this post, we'll talk more about the technical and underlying changes made to the ingestor that optimize the way data is collected.

## Pure LDAP
In the previous versions of the BloodHound ingestor, and the majority of the tools released, communication with Active Directory is done using the **DirectorySearcher** class in the **System.ActiveDirectory** namespace. In SharpHound, we've transitioned to a lower level API, the **System.ActiveDirectory.Protocols** namespace. DirectorySearcher provides convenience and abstraction, removing the need to handle things like paging, but at the cost of adding ADSI and COM overhead to processing and networking. This results in more traffic, as well as lower performance overall. Using the new namespace, we instead use **LdapConnection** in conjunction with **SearchRequest** and **SearchResponse**.

```csharp
public SearchRequest GetSearchRequest(string filter, SearchScope scope, string[] attribs, string domainName = null, string adsPath = null)
{
    Domain targetDomain;
    try
    {
        targetDomain = GetDomain(domainName);
    }
    catch
    {
        return null;
    }

    domainName = targetDomain.Name;
    adsPath = adsPath?.Replace("LDAP://", "") ?? $"DC={domainName.Replace(".", ",DC=")}";

    var request = new SearchRequest(adsPath, filter, scope, attribs);
    var soc = new SearchOptionsControl(SearchOption.DomainScope);
    
    request.Controls.Add(soc);
    return request;
}

public LdapConnection GetLdapConnection(string domainName = null)
{
    Domain targetDomain;
    try
    {
        targetDomain = GetDomain(domainName);
    }
    catch
    {
        return null;
    }

    var domainController = _options.DomainController ?? targetDomain.Name;

    var identifier = _options.SecureLdap
        ? new LdapDirectoryIdentifier(domainController, 636, false, false)
        : new LdapDirectoryIdentifier(domainController, false, false);

    var connection = new LdapConnection(identifier);
    
    //Add LdapSessionOptions
    var lso = connection.SessionOptions;
    if (!_options.DisableKerbSigning)
    {
        lso.Signing = true;
        lso.Sealing = true;
    }
    
    if (_options.SecureLdap)
    {
        lso.ProtocolVersion = 3;
        lso.SecureSocketLayer = true;
        if (_options.IgnoreLdapCert)
            connection.SessionOptions.VerifyServerCertificate = (con, cer) => true;
    }

    lso.ReferralChasing = ReferralChasingOptions.None;
    return connection;
}

public IEnumerable<SearchResultEntry> DoSearch(string filter, SearchScope scope, string[] props,
            string domainName = null, string adsPath = null, bool useGc = false)
{
    //Get an LDAP Connection
    using (var conn = useGc ? GetGcConnection(domainName) : GetLdapConnection(domainName))
    {
        if (conn == null)
        {
            yield break;
        }

        //Get a SearchRequest with our options
        var request = GetSearchRequest(filter, scope, props, domainName, adsPath);

        if (request == null)
        {
            yield break;
        }

        //Add a page control to get 500 entries at a time
        var prc = new PageResultRequestControl(500);
        request.Controls.Add(prc);

        if (_options.CurrentCollectionMethod.Equals(CollectionMethod.ACL))
        {
            var sdfc =
                new SecurityDescriptorFlagControl { SecurityMasks = SecurityMasks.Dacl | SecurityMasks.Owner };
            request.Controls.Add(sdfc);
        }

        PageResultResponseControl pageResponse = null;
        //Loop to keep getting results
        while (true)
        {
            SearchResponse response;
            try
            {
                response = (SearchResponse)conn.SendRequest(request);
                if (response != null)
                {
                    pageResponse = (PageResultResponseControl)response.Controls[0];
                }
            }
            catch
            {
                yield break;
            }
            if (response == null || pageResponse == null) continue;
            foreach (SearchResultEntry entry in response.Entries)
            {
                yield return entry;
            }

            //Exit the loop when our page response indicates we're out of data
            if (pageResponse.Cookie.Length == 0 || response.Entries.Count == 0)
            {
                yield break;
            }

            prc.Cookie = pageResponse.Cookie;
        }
    }
}
```

In a test environment with about 300,000 users, the pure LDAP method of enumeration took approximately one minute less to retrieve all the data. While this isn't a huge number, those calls add up over time. In this test, the data was not manipulated in any way either, so the actual performance increase is likely slightly higher.

## LDAP Encryption and Site Search
Previously, our LDAP connections were in cleartext, as is the standard for LDAP. The DirectoryServices.Protocols namespace exposes the ability to add Kerberos signing to all our LDAP requests. In SharpHound, even when using port 389, LDAP requests to the Domain Controller are now encrypted by default (special thanks to [Mark Gamache](https://twitter.com/markgamacheNerd) for showing us this trick with the library). This should help avoid simple detection from directory enumeration. Another interesting feature is the addition of site searching for Domain Controllers. Previously, we would use the primary domain controller (PDC) for the domain you were enumerating to grab data. With the new changes, SharpHound will query for your nearest Domain Controller. This should help massively in huge domains that span across the world, as you're now more likely to get a geographically close Domain Controller to pull data from. Additionally, this DC will likely be under less load than the PDC of a domain.

## Caching
One of the most important changes made in SharpHound was the addition of caching. In the PowerShell ingestor, there was a small amount of in-memory caching in different parts of the ingestion process, mainly in the group membership enumeration. Distinguished names were mapped to "display names" and those mappings were saved in a dictionary. In SharpHound, we still maintain that mapping, but we also maintain a mapping of SIDs to "display names". A large portion of the resolution done during enumeration is resolving SIDs, so caching this greatly reduces the number of network requests made. Data is stored in ConcurrentDictionaries, allowing thread-safe access to the data. The cache file is generated using the wonderful protobuf project from Google, implemented in the [protobuf-net](https://www.nuget.org/packages/protobuf-net/) package on nuget. This gives us very fast loading and writing of the cache file, which is essential to keep startup time minimal.

Caching has also been added on a per-run basis to several other parts of enumeration. SharpHound will internally maintain a cache of the result of pings, so systems aren't checked multiple times. DNS resolution is also cached locally.

## New Local Admin Enumeration
This is a feature that will be particularly useful for users of BloodHound outside the US on localized domains. In previous versions of BloodHound, as well as in tools such as PowerView, we use the Windows32 API call, [NetLocalGroupGetMembers](https://msdn.microsoft.com/en-us/library/windows/desktop/aa370601(v=vs.85).aspx). This API call allows you to provide a name for a group you wish to query on a remote host. In any English domain where the group isn't renamed, providing the name **Administrators** will work. Obviously this breaks on localized domains. After examining the results of NetLocalGroupGetMembers in Wireshark, we noticed that the API call actually consists of several calls to the SAMRPC library, a low level library for accessing the remote SAM. You can find the (paltry) documentation for this library [here](https://msdn.microsoft.com/en-us/library/cc245476.aspx). One of the interesting things we noticed was that the **NetLocalGroupGetMembers** call actually resolves the group name you request to a RID and then passes that to a further SAMRPC call.

### The Old Method
Here's the breakdown of the calls used by NetLocalGroupGetMembers when trying to lookup the **Administrators** group. In this example, the system is part of the testlab.local domain, which has a netbios name of TESTLAB. In the interest of brevity, the calls to close handles have been stripped out.

* **SamConnect** - Opens a connection to the remote host
* **SamEnumDomains** - Enumerates the "Domains" available on the system. This returns TESTLAB and built-in on our test system.
* **SamLookupDomain** - Enumerates the TESTLAB domain, as it is the first in the list.
* **SamOpenDomain** - Opens a handle to the TESTLAB domain
* **SamLookupNames** - Looks up the RID for the name **Administrators** which we passed to the function. This returns nothing as **Administrators** is part of the built-in computer domain. (BUILTIN)

At this point we haven't found the group we want, so the process continues

* **SamConnect** - Opens a connection to the remote host.
* **SamOpenDomain** - Opens a handle to the builtin domain (S-1-5-32).
* **SamLookupNames** - Looks up the RID for the name **Administrators**. This function returns 544, because the **Administrators** group is part of this domain.
* **SamOpenAlias** - Opens the alias for Administrators using the RID 544, acquired in the last step.
* **SamQueryAliasInfo** - Queries alias info to verify data (I assume).

At this point, the function has found the right RID. So it continues:

* **SamConnect** - Opens a connection to the remote host.
* **SamOpenDomain** - Opens the handle to the builtin domain (S-1-5-32).
* **SamLookupNames** - Looks up the RID for the name **Administrators**. This function returns 544, because the **Administrators** group is part of this domain.
* **SamOpenAlias** - Opens the RID 544 alias.
* **SamGetMembersInAlias** - Returns the list of SIDs for members in the 544 RID alias.
* **LsaOpenPolicy** - Opens the LSA policy of the remote host.
* **LsaLookupSids** - Converts the list of SIDS from **SamGetMembersInAlias** to names and object types.

Believe it or not, the function actually does more here, even though we actually have all the info we need at this point. Essentially, the function calls a large number of completely unnecessary network calls for our purposes. If you're paying attention, you'll notice a shortcut we can take here. The RID for the local administrators group is always 544.

### The New Method
* **SamConnect** - Opens a connection to the remote host.
* **SamLookupDomainInSamServer** - We pass the machine's Netbios name to this function in order to retrieve the machine SID to filter out local accounts.
* **SamOpenDomain** - Opens the handle to the builtin domain (S-1-5-32)
* **SamOpenAlias** - Opens the RID 544 alias (which we know is the group we want)
* **SamGetMembersInAlias** - Returns the list of SIDS for the members of the alias
* **LsaOpenPolicy** - Opens the LSA policy of the remote host
* **LsaLookupSids** - Converts the list of SIDS from **SamGetMembersInAlias** to names and object types.

This is a significantly reduced number of queries being performed to accomplish the same task. The additional benefit is that since we're no longer querying by name directly, localization of the **Administrators** group is completely irrelevant. Even if the **Administrators** group is renamed to, lets say, **Super Ultra Mega Awesome Users**, SharpHound will have no trouble querying this group directly and getting the necessary data back. The reduction in network overhead is another awesome side effect, as any reduction in the network hit from admin enumeration will eventually result in significant decrease overall, due to the repeated nature of the queries. The full code of the replacement local administrator function can be found in the SharpHound repository [here](https://github.com/BloodHoundAD/SharpHound/blob/1b689d7354407105a7e96aed8b551d6886879731/Sharphound2/Enumeration/LocalAdminHelpers.cs#L50)

## Concurrent Enumeration
In the old ingestor, running “default” collection would run through a set of steps to do collection. Each step would wait until completion to move onto the next step. Group enumeration would start and finish first, followed by the next steps in the process. However, all of these steps rely on the same thing, an LDAP AD entry. By combining the list of properties necessary for each step, a "runner" was designed that accepts an AD entry, and runs it through the appropriate processing steps based on the type of object returned. By only requesting the list of AD object and properties once, we remove a large portion of LDAP overhead incurred by requesting the same objects and properties multiple times. Any reduction in network overhead in an already network heavy tool is a welcome improvement. This has the additional benefit of allowing us to display "progress" and inform the user how many items have been processed, as we now have a discrete representation of a completed object. A completed object has gone through all enumeration steps necessary.

## Threading and Memory Usage
With SharpHound, we targeted .NET 3.5 as a baseline because it should be reasonably available on most systems attackers would want to target. A great resource we use can be found in the MSDN Blog by Aaron Stebner, [Mailbag: What version of the .NET Framework is included in what version of the OS?](https://blogs.msdn.microsoft.com/astebner/2007/03/14/mailbag-what-version-of-the-net-framework-is-included-in-what-version-of-the-os/). The link indicates that Windows 7 onwards will have .NET 3.5.1 or higher by default. Unfortunately for us, most of the new .NET threading features were implemented in .NET 4.0. Thankfully, Microsoft was kind enough to backport several of the key threading improvements in the form of the [TaskParallelLibrary (TPL) Backport for .NET 3.5](https://www.nuget.org/packages/TaskParallelLibrary/). There's a few caveats with using this library, in particular the fact that the behavior of the backported classes is not always identical to the original versions.

The threading capabilities provided by TPL allow us to spin up Tasks instead of traditional threads and allow the scheduler to handle the actual threading on the backend. Tasks are significantly faster and cheaper to start up compared to threads. However, one of the biggest issues we ran into with new the new library is something I'm choosing to call a memory leak, whereas it's considered a design choice by the developers. One of the new collections provided is the BlockingCollection, a collection perfect for implementing the Producer-Consumer model. Using it is as simple as using a regular list, but consumers grab from the **Consuming Enumerable** of the collection.

```csharp
BlockingCollection<string> collection = new BlockingCollection<string>();

foreach (var string in collection.GetConsumingEnumerable()){
   DoSomething(string);
}
```

When the producer signals that the queue has completed input, the enumerable correspondingly notifies consumers that there are no more items coming, which signals the consumers to finish enumeration. Unfortunately, the BlockingCollection in the backported libraries has the downside of holding references to every object that's added to the collection, even ones that have already been removed from the queue. Those familiar with the way garbage collection works in .NET know that an object with a held reference can never be garbage collected, and therefore will continue to consume memory.

As a workaround to this problem, SharpHound uses a Wrapper class to encapsulate items going into any blocking collections.

```csharp
namespace Sharphound2
{
   //This class exists because of a memory leak in BlockingCollection. By setting the reference to Item to null after enumerating it,
   //we can force garbage collection of the internal item, while the Wrapper is held by the collection.
   //This is highly preferable because the internal item consumes a lot of memory while the wrapper barely uses any
   class Wrapper<T>
   {
       public T Item { get; set; }
   }
}
```

The Wrapper represents a very tiny chunk of used memory (bytes) compared to, as an example, a full **SearchResultEntry** returned from LDAP, which has all the properties necessary to carry out enumeration. After enumerating an object, the **Item** in the wrapper is set to **null**, allowing the garbage collector to remove the object and free up memory on its next pass. The BlockingCollection continues to hold a reference to the Wrapper, but at a significantly reduced memory cost. You can see an example of this in the SharpHound code [here](https://github.com/BloodHoundAD/SharpHound/blob/f9574480ac26831f4b03112e1f0c0fe234f2eade/Sharphound2/Enumeration/EnumerationRunner.cs#L828). Huge thanks to the StackOverflow user, usr, on [this post](https://stackoverflow.com/questions/12824519/the-net-concurrent-blockingcollection-has-a-memory-leak).

One of the biggest sources of the memory usage in the original PowerShell ingestor was the need to load the full list of objects to enumerate before starting the actual enumeration steps. The list of all AD objects that match the selected filter for each enumeration step had to be completely streamed through LDAP, and then held in memory while enumeration was performed. Naturally, as the size of the domain increased the number of objects increased as well. In SharpHound, the maximum size of the BlockingCollection used to collect data from LDAP is set to 1000 items. Thanks to the way we stream data from LDAP, the producer will work cooperatively with consumers to keep the input queue full, while only holding 1000 objects at a time. This significantly decreased the memory usage of the application overall. Additionally, many functions were evaluated to ensure that objects in memory would be appropriately disposed of, and small tweaks made in several places.

## Replaced Ping
After running BloodHound on many different networks, we started to notice that hosts sometimes did not respond to ping, but were properly accessible using PowerView commands. After narrowing down culprits, we found that the issue was as simple as our ping check performed before trying to connect to hosts for local admin or session enumeration. In SharpHound, we replaced the ICMP ping method used in the original ingestor with a check if port 445 is open on the target host, something which is far more indicative of if our requests will succeed.

```csharp
internal bool DoPing(string hostname)
{
   try
   {
       using (var client = new TcpClient())
       {
           var result = client.BeginConnect(hostname, 445, null, null);
           var success = result.AsyncWaitHandle.WaitOne(_pingTimeout);
           if (!success)
           {
               return false;
           }

           client.EndConnect(result);
       }
   }
   catch
   {
       return false;
   }
   return true;
}
```

A host that responds to ping but has port 445 closed will still not return data during enumeration. Conversely, a host that has 445 open but does not respond to ping will likely yield valuable data.

## REST API Updates
The original code to do REST API ingestion in the PowerShell ingestor used a naive way of adding objects to the database. The REST API for Neo4j supports loading multiple statements at the same time in the JSON body. Previously, this is how we used this functionality.

```json
{
   "statements" : [
       {
           "statement" : "MERGE (n:User {name:'DOMAIN ADMINS@TESTLAB.LOCAL'})-[r:AdminTo]-(m:Computer {name:'PRIMARY.TESTLAB.LOCAL'})"
       }, 
       {
           "statement" : "MERGE (n:User {name:'DOMAIN ADMINS@TESTLAB.LOCAL'})-[r:AdminTo]-(m:Computer {name:'SECONDARY.TESTLAB.LOCAL'})"
       }
   ]
}
```


While functional and simple, this is a very inefficient method of loading data. Neo4j allows the use of parameters to a query, including a list of parameters. Providing a list of parameters allows Neo4j to perform request caching on the backend, significantly increasing the speed of ingestion. Additionally, treating the properties as parameters ensures that weird characters should get escaped properly, similar to parameterized queries in other query languages.

```json
{
   "statements" : [
       {
           "statement" : "MERGE (n:Group {name:{props.name}})",
           "parameters" : {
               "props" : {
                   "name" : "DOMAIN ADMINS@TESTLAB.LOCAL"
               }
           }
       }
   ]
}
```

## Wrap Up

Building on and learning from the original PowerShell ingestor, we've created SharpHound to resolve the most common issues with the BloodHound project, namely: speed, accuracy, and stability of data collection. We and several folks in the BloodHound community have seen data collection times reduce by orders of magnitude, sometimes completing ingestion in as little as 10 minutes where it previously took hours, or never completed. In this blog post, we outlined several of the key developments in SharpHound that led to this success, and we will continue to improve SharpHound to make it even more reliable and even more user friendly in the future. If you have any suggestions or requests, please feel free to hit up [@CptJesus](https://twitter.com/cptjesus), [@wald0](https://twitter.com/_wald0), or [@harmj0y](https://twitter.com/harmj0y) in the [BloodHound Slack](https://bloodhoundgang.herokuapp.com), or open an issue on the [BloodHound Github Repository](http://github.com/BloodHoundAD/BloodHound).
