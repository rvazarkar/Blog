---
title: "BloodHound 2.0"
date: 2018-08-07T12:00:00-05:00
draft: false
tags: ["bloodhound", "sharphound"]
---

The BloodHound team has been relatively quiet for a while now. It's been 5 months since the release of the Containers update, and outside of some bugfixes, nothing much has changed.

All that is about to change.

We're proud to announce the release of BloodHound 2.0, representing the second major release of the project with tons of new features, bugfixes, and new abuse primitives.

# Major New Features

## Four New Edges
BloodHound 2.0 is adding four new attack primitives of varying complexity and interest. The new attack primitives should help find new paths when executing engagements.

### CanRDP - Remote Desktop Privileges
The CanRDP edge runs from a user or a group to a computer and indicates that the principals are part of the **Remote Desktop Users** local group on the target system. While this does not immediately guarantee privileged access, if you can find a privilege escalation on the target, this results in another attack vector that isn't often looked at, and difficult to enumerate on a large scale. Collection of this edge belongs to the new RDP collection method.

### ExecuteDCOM - Distributed COM
The ExecuteDCOM edge runs from a user or a group to a computer and indicates that the principals are part of the **Distributed COM Users** local group on the target system. Depending on the security descriptor on the target system, users in this group sometimes have remote code execution privileges without corresponding administrative permissions on the target system. This provides another lateral movement method that does not require administrative access. Collection of this edge belongs to the new DCOM collection method.

### ReadLAPSPassword - LAPS Password Readers
The ReadLAPSPassword edge runs from a user or a group to a computer and indicates that LAPS is likely installed on the target computer and the principal can read the password for the local administrator account from LDAP. In addition to this edge, we've enabled other ACL edges on computer objects that allow the ability to gain access to the LAPS password. Reading the password from Active Directory is effectively the same as having admin to a system, and enables collection and evaluation of more attack paths. Collection of this edge and the new ACL edges falls under the existing ACL collection method.

### AllowedToDelegate - Constrained Delegation
A very complicated attack that has several caveats, constrained delegation presents a complex method of compromising a computer, and possibly more. Will Schroeder [(harmj0y)](https://twitter.com/harmj0y) wrote about this permission a while ago and called it ["The most dangerous user right you (probably) have never heard of."](http://www.harmj0y.net/blog/activedirectory/s4u2pwnage/). At its most dangerous, this edge can enable a principal to DCSync without any of the corresponding rights you would expect to see in Active Directory. Collection of this edge takes place during the ObjectProps collection method.

## File Output and Ingestion Changes - JSON and Zip
One of the biggest changes (and the one that required a version bump) is the complete change of the output format for SharpHound. As of 2.0, output from the collector is JSON, and defaults to compressing to a Zip file and removing the JSON files from disk. Since we're using JSON files, appending to files is no longer possible. Instead, each run provides a timestamped Zip file that represents the collection run you performed. Switching to JSON provides far more control over the output and should hopefully fix problems with CSV files and column mismatches, which many users have reported.

While we added the feature a while ago, apparently it slipped under the radar. The user interface has supported Zip ingestion since the 1.4 release in October of 2017, but not many people were aware of it. With the output format switching to Zip files, we're emphasizing this feature and its ease of use benefit. Additionally, we've added drag and drop support to the user interface. You can just drop your file on top of the UI to ingest it instead of using the upload button.

![Drag N' Drop](/bloodhound20/dragndrop.gif)

Ease of use is always something we consider when implementing new features, and we think this streamlines the workflow of getting data into the user interface and the database.

## Edge Filtering
Sometimes, you want to execute an attack path that doesn't use certain attack types. Maybe you want to just execute a path that uses one of the new edges that may or may not result in admin rights. Before 2.0, this required writing a custom cypher query to filter out edges you weren't interested in. BloodHound 2.0 has a built in way to filter edges from ShortestPath queries that applies to everything in the UI, including pre-built queries.

![Edge Filtering UI](/bloodhound20/edgefilter.png)

Changes you make in this UI element will persist across uses of BloodHound and are saved similar to other settings.

![RDP Rights](/bloodhound20/dfmcanrdp.png)

In the above screenshot, the user dfm has RDP privileges to WINDOWS1. However, if we don't want to use RDP, we can filter out the edge type and see if dfm still has a path:

![Filter RDP](/bloodhound20/edgefilter.gif)

After stripping the RDP edge, we find that dfm still has a path to the computer!

## Edge Addition/Deletion
Sometimes you find a weird path that doesn't really fit in the context of the BloodHound graph. Maybe you know a user has admin on a system using some kind of SQL abuse or a third-party application. In BloodHound 2.0, you can easily add this to the graph using the built-in workflow to add an edge. To access this UI, right click anywhere on the graph where nothing else exists (the stage menu) to open the new context menu.

![Stage Menu](/bloodhound20/stagemenu.png)

Clicking on **Add Edge** will open a new window that allows you to add edges between two nodes of your choice.

![Add Edge](/bloodhound20/addedgemenu.png)

This new feature allows you to easily model attack paths that don't follow traditional paths. Along with this feature, we also added the ability to delete edges straight from the user interface. Let’s say you find a user that is a local administrator to a system, but is disabled and interfering with pathfinding. Now you can easily right click on the edge to delete it.

![Edge Menu](/bloodhound20/deleteedgemenu.png)

If you add or delete an edge, you can click the **Reload Query** button in the stage menu to re-run the current query that's present in the UI.

![Edge Menu](/bloodhound20/reloadquery.gif)

## Node Addition/Deletion/Modification
While this feature has less applications in our opinion, there are certainly times when modifying nodes is useful. The stage menu also allows you to add a node, which opens the **Add Node** UI.

![Add Node](/bloodhound20/addnode.png)

Deleting a node can be done by right clicking on any node and using the context menu.

![Node Menu](/bloodhound20/nodemenu.png)

You can also modify the properties on nodes, which will be saved in the database, from the same menu. This is helpful if you perform a targeted kerberoast attack for example, where you can set the HasSPN property of the user to allow pathfinding from kerberoastable users.

![Edit Node](/bloodhound20/editnode.png)

## Owned/High Value Properties
Tom Porter [(porterhau5)](https://twitter.com/porterhau5) did some fantastic work on forking BloodHound and adding the ability to mark users/computers that have been "owned". We decided that this feature is awesome and integrated it into the main fork with some modifications. User and computer nodes now have the owned property, and are marked on the user interface with a small skull. We've also added the ability to mark nodes as "High Value", which is represented with a diamond on the node.

![Owned/High Value Nodes](/bloodhound20/ownedhighval.png)

When ingestion is done, all user/computer nodes are initially marked as not owned. Certain groups are automatically marked as High Value, including Domain Admins, Enterprise Admins, and other groups that lead to high privileges. Along with these new properties, we've introduced new pre-built queries that work with these properties.

![New Pre-Built Queries](/bloodhound20/newprebuilt.png)

You can easily set these properties on nodes from the right click context menu as well, which will save into the database for future use.

![Node Menu](/bloodhound20/nodemenu.png)

## Edge Help
One of our favorite new features is the built in edge abuse help now available in the BloodHound user interface. Right clicking on any edge will present you with the option to click on **Help**.

![Edge Help](/bloodhound20/edgehelp.png)

Each help menu presents you with general information about the edge, as well as detailed information on how to abuse the privileges. Along with these comes OPSEC considerations you should be aware of, as well as references to other resources. Abuse info is tied to the type of the target node, so you'll always have relevant information for the attack you're trying to execute.

![Abuse Info](/bloodhound20/edgeabuseex.png)

## Merged LDAP Queries
In the initial release of SharpHound, one of the features that saved network overhead as well as time was unified LDAP queries. When running Default collection, only one LDAP query was performed instead of four. Eventually, the ability to specify multiple collection methods using a comma separated list was added in BloodHound 1.5 and each collection method ran as a separate query. With 2.0, SharpHound now resolves all selected collection methods and dynamically builds a LDAP filter that encompasses the data and properties from all of them.

![Resolved Collection Methods](/bloodhound20/resolvedcollection.png)

This once again ensures that all collection is performed with a single query, optimizing network usage.

## Persistent LDAP Connection
Ben Campbell [@Meatballs__](https://twitter.com/Meatballs__) pointed out that SharpHound spins up a large number of LDAP connections when doing enumeration, which causes a significant amount of overhead on building Kerberos handshakes. He provided an [alternative solution](https://github.com/BloodHoundAD/SharpHound/pull/30) which maintains a persistent LDAP connection. 

As of 2.0, SharpHound will use a cached LDAP connection to request resources which should speed up enumeration time in group enumeration at the very least, and possibly in more areas.

## Notes and Pictures
Sometimes, you want to keep notes for your operation. Maybe you want to note how a user was owned or where you got a credential from. Maybe a file in a share that was particularly useful. With the 2.0 release, we've added a Notes field to every node that persists in the database.

![Notes](/bloodhound20/notes.png)

We've also added the ability to attach pictures to nodes. You can drag and drop an image file into the node information window to attach it. This feature is all managed locally, so the image configuration is on a per-user basis and will not sync between users.

![Attach Pictures](/bloodhound20/attachpic.gif)

The images will have a copy button and a delete button. The copy button will copy the full path to the file on disk to your clipboard.

## Language Independent Pre-Built Queries + Data
When switching to the JSON format, one of the top priorities was ensuring that the objectsid parameter is stored for every node. This allows queries to use the objectsid instead of the name of the object. Domain Admins is now referenced by its objectsid that ends in -512, meaning it will work regardless of the actual name of the object in the database. Additionally, we've switched the encoding of files output by the ingestor to UTF8 in order to properly support foreign language domains.

# Minor Updates
These updates, while not as significant, are still cool additions to the project which some will find useful.

## Dark Mode (Beta)
We've added a dark mode feature to the user interface which should make it significantly easier on the eyes in dark environments. We don't anticipate many people using this because hackers definitely don't sit in dark rooms. This feature likely has some bugs associated with it, so if you find some, let us know!

![Dark Mode](/bloodhound20/darkmode.png)

## New Properties and Lowercase Property Names
New properties have been added to every single node type.

* Users
   * Description
   * AdminCount
   * UserPassword (This property can sometimes have the real password for the user and be in plaintext, readable by any user!)
   * Title
   * HomeDirectory
* Groups
   * Description
   * AdminCount
* Computers
   * ServicePrincipalNames
* GPO
   * GPCFileSysPath
   * Description
* OU
   * Description
* Domains
   * Description

All properties are now stored in all lower case in the database for consistency.

## Random File Names, Encrypted Zip, ZipFileName
A feature request we've had several times is the ability to randomize file names to prevent EDR solutions from signaturing the filenames. We've introduced the **RandomFileNames** parameter with 2.0 which will generate meaningless filenames to throw off signatures.

![Random Filenames](/bloodhound20/random.png)

Additionally, the **EncryptZip** parameter was added which generates a random 10 character password for the Zip file. This Zip file can *not* be uploaded to the interface directly and must be unzipped manually.

![Zip Password](/bloodhound20/zippassword.png)

The **ZipFileName** parameter allows you to manually specify the name of the Zip file that’s created if you want to change the default.

## LDAPUser + LDAPPass Parameters
Another feature request we've received is to specify a credential for querying LDAP, similar to the Credential parameter in PowerView. We've added this in the form of the **LdapUser** and **LdapPass** parameters, which will create a NetworkCredential for use in LDAP queries.

Note: NetworkCredentials do NOT apply to API calls to remote systems such as NetSessionEnum or queries for Local Groups. These will likely fail if you use this method instead of the more recommended runas /netonly method.

## Hierarchical Default Layout
The default layout was the Force Directed layout for quite a while. We decided that the hierarchical view offers more in most situations, and that many users weren't even aware of the ability to change layouts. We've made hierarchical view the default going forward.

## Removed Node Sibling Collapse
Node Sibling Collapse was one of the features that was added in order to reduce the load required for the user interface to draw graphs. In practice, this logic was almost never applied or useful, and resulted in a huge computation load when processing data. We've completely removed the feature which should speed up pre-display data processing.

## Removed DCSync Edge
We've removed the DCSync edge in favor of the *Find Principals with DCSync Rights* query. In practice, the DCSync edge was inaccurate and only covered a small number of cases. Many of the principals that have DCSync privileges by default are now marked as high value, and other principals can be manually marked as such.

## Added GPOs Affecting this OU
This new query allows you to see the effective GPOs applied to an OU. This can help you isolate what GPO is making a particular change, or figure out what GPOs you can modify for a specific attack vector.

## Renamed LoopTime to LoopDelay
LoopTime was a terrible parameter name. LoopDelay makes far more sense.

## Brought some sanity to the rescaler
![Zip Password](/bloodhound20/camera.gif)

Anyone who has tried to make cool pictures in the UI knows how annoying the camera can be. We've finally brought some sanity to the rescale middleware so you should have a bit more freedom in moving stuff without your mouse pointer rocketing off the screen.

# Bugfixes
* All ACL edges have been examined to ensure that privileges were properly calculated for abusable principals
* Fixed Global Catalog searching to properly use the domain root
* Filtered out ACE's without the AccessAllowed flag
* Filtered out ACE's with InheritOnly set
* Escaped search terms so they don't interfere with the search regex
* Fixed stealth container enumeration so it doesn't break
* Fixed an issue causing duplicated Owns edges
* Fixed a bug causing duplicate GPO/OU Nodes
* Fixed an issue with the DCSync query
* Fixed Well-Known principals in local groups not having a domain
* Pruned some pre-built queries and fixed a couple others
* General code cleanup and updated libraries
* Fixed collection of Enterprise Domain Controllers (it kind of works)
* Updated the Foreign Admins query on domain nodes so it doesn't take ages

# Wrap Up
## SharpHound Flags
Since we've made tons of changes, here's the updated list of flags for SharpHound.

### Enumeration Options
* **CollectionMethod** - The collection method to use. This parameter accepts a comma separated list of values. Has the following potential values (Default: Default):
  * **Default** - Performs group membership collection, domain trust collection, local admin collection, and session collection
  * **Group** - Performs group membership collection
  * **LocalAdmin** - Performs local admin collection
  * **RDP** - Performs Remote Desktop Users collection
  * **DCOM** - Performs Distributed COM Users collection
  * **GPOLocalGroup** - Performs local admin collection using Group Policy Objects
  * **Session** - Performs session collection
  * **ComputerOnly** - Performs local admin, RDP, DCOM and session collection
  * **LoggedOn** - Performs privileged session collection (requires admin rights on target systems)
  * **Trusts** - Performs domain trust enumeration
  * **ACL** - Performs collection of ACLs
  * **Container** - Performs collection of Containers
  * **ObjectProps** - Collects object properties such as LastLogon and DisplayName
  * **DcOnly** - Performs collection using LDAP only. Includes Group, Trusts, ACL, ObjectProps, Container, and GPOLocalGroup.
  * **All** - Performs all Collection Methods except GPOLocalGroup
* **SearchForest** - Search all the domains in the forest instead of just your current one
* **Domain** - Search a particular domain. Uses your current domain if null (Default: null)
* **Stealth** - Performs stealth collection methods. All stealth options are single threaded.
* **SkipGCDeconfliction** - Skip Global Catalog deconfliction during session enumeration. This can speed up enumeration, but will result in possible inaccuracies in data.
* **ExcludeDc** - Excludes domain controllers from enumeration (avoids Microsoft ATA flags :) )
* **ComputerFile** - Specify a file to load computer names/IPs from
* **OU** - Specify which OU to enumerate

### Connection Options
* **DomainController** - Specify which Domain Controller to connect to (Default: null)
* **LdapPort** - Specify what port LDAP lives on (Default: 0)
* **SecureLdap** - Connect to AD using Secure LDAP instead of regular LDAP. Will connect to port 636 by default.
* **IgnoreLdapCert** - Ignores LDAP SSL certificate. Use if there's a self-signed certificate for example
* **LDAPUser** - Username to connect to LDAP with. Requires the LDAPPass parameter as well (Default: null)
* **LDAPPass** - Password for the user to connect to LDAP with. Requires the LDAPUser parameter as well (Default: null)
* **DisableKerbSigning** - Disables LDAP encryption. Not recommended.

### Performance Options
* **Threads** - Specify the number of threads to use (Default: 10)
* **PingTimeout** - Specifies the timeout for ping requests in milliseconds (Default: 250)
* **SkipPing** - Instructs Sharphound to skip ping requests to see if systems are up
* **LoopDelay** - The number of seconds in between session loops (Default: 300)
* **MaxLoopTime** - The amount of time to continue session looping. Format is 0d0h0m0s. Null will loop for two hours. (Default: 2h)
* **Throttle** - Adds a delay after each request to a computer. Value is in milliseconds (Default: 0)
* **Jitter** - Adds a percentage jitter to throttle. (Default: 0)

### Output Options
* **JSONFolder** - Folder in which to store JSON files (Default: .)
* **JSONPrefix** - Prefix to add to your JSON files (Default: "")
* **NoZip** - Don't compress JSON files to the zip file. Leaves JSON files on disk. (Default: false)
* **EncryptZip** - Add a randomly generated password to the zip file.
* **ZipFileName** - Specify the name of the zip file
* **RandomFilenames** - Randomize output file names
* **PrettyJson** - Outputs JSON with indentation on multiple lines to improve readability. Tradeoff is increased file size.

### Cache Options
* **CacheFile** - Filename for the Sharphound cache. (Default: BloodHound.bin)
* **NoSaveCache** - Don't save the cache file to disk. Without this flag, BloodHound.bin will be dropped to disk
* **Invalidate** - Invalidate the cache file and build a new cache

### Misc Options
* **StatusInterval** - Interval to display progress during enumeration in milliseconds (Default: 30000)
* **Verbose** - Enables verbose output

## Final Thoughts
BloodHound 2.0 represents one of the biggest releases to date with many new features and a completely new output format. It should also enable significant analysis opportunities in the UI while lowering the barrier to entry for database modifications. As usual, you can grab compiled versions of the UI and the collector, or self-compile, from our GitHub repository [here](https://github.com/BloodHoundAD/BloodHound/releases).

You can always find us in the [BloodHound Slack Channel](https://bloodhoundgang.herokuapp.com/). The link will allow you to get an invite and join our active community with over 1700 users. We look forward to hearing from new and old users alike on their thoughts on 2.0.

