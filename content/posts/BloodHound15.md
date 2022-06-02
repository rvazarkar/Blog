---
title: "BloodHound 1.5: The Container Update"
date: 2018-03-28T12:00:00-05:00
draft: false
tags: ["bloodhound", "sharphound"]
---

When BloodHound 1.4 came out in October of 2017, the object properties added represented the first major change in the BloodHound database schema since the original creation of the project. Today, we're proud to present BloodHound 1.5, which represents a much larger change in both the database schema, as well as many long standing features of the BloodHound user interface.

# Containers - GPOs and OUs
One of the things the BloodHound team has been talking about for quite a while now is adding GPO and OU objects to the BloodHound schema. With the 1.5 update, this is finally a reality. [BloodHound 1.5](https://github.com/BloodHoundAD/BloodHound/releases/tag/1.5) introduces the ability to collect the structure of domains, including GPOs, what OUs those GPOs apply to, and what objects are contained by the OUs. ACL collection has been modified to collect controllers for GPO objects. This will allow you to explore new attack paths, including modification of GPOs to affect objects further down the tree.

# SharpHound Changes

## New Collection Methods
The 1.5 Release of SharpHound brings two new collection methods. The first, and biggest new addition to the project, is the **Container** collection method. The second is the **All** collection method.

### Container Collection Method
The new **Container** collection method will collect GPO and OU objects in the domain and output two new files: container_structures.csv and container_gplinks.csv. These files contain all the information on how the domain's containers are structured, allowing you to build a graph to examine these new relationships.

![Container Structure](/bloodhound15/container_structure.png)

![Example Graphs](/bloodhound15/outree.png)

Collecting container structure and gplinks is a relatively quick process and only uses LDAP requests to collect all the required data.

### All Collection Method
By popular demand, we've finally introduced the **All** collection method, which as the name implies, will do all forms of enumeration. Enumeration is done in the following order:

* Default Enumeration
* ACL Enumeration
* ObjectProps Enumeration
* Container Enumeration

A great way of doing all the collection and getting the data with minimal leftover artifacts is using the following command:

```posh
Invoke-BloodHound -CollectionMethod All -CompressData -RemoveCSV
```

This will perform all collection methods, compress all the CSVs into a timestamped zip file, and remove all the CSVs after it's completed thanks to the new **RemoveCSV** flag. If you want, you could even add the **NoSaveCache** flag, to remove the cache file from disk as well.

### Specifying Multiple Collection Methods
If you don't want to use the **All** collection method, you can still individually specify multiple collection methods using a comma seperated list

```posh
Invoke-BloodHound -CollectionMethod ACL,ObjectProps
```

The above command will run ACL collection followed by Object Property collection.

## Throttle and Jitter
SharpHound now has throttle and jitter options to slow down and vary the timing of requests made to computers. If you're worried about network load or detection based on quick hits to multiple computers, these options might be useful for you.

```posh
Invoke-BloodHound -Throttle 1500 -Jitter 10
```

The above command will introduce a 1.5 second delay after each computer request, with a 10% variance in the delay.

## Deprecation of REST API ingestion
Unfortunately, we've made the decision to deprecate data collection using the Neo4j REST API. The amount of work required to keep the feature properly supported was deemed not worth it compared to the number of people actually using the feature. Additionally, CSV ingestion is faster, and easier to manage. Historically, we have always recommended people use CSV ingestion anyways to make sure that a copy of the data is retained in case there's a bug in the user interface. As of the 1.5 release, we are no longer supporting the REST API method of ingestion. The entire ingestion method will be removed in a future release.

If you choose to use the REST API in its current form, keep in mind there might be bugs, and **Container** collection is not supported or implemented.

## Bug Fixes
### Group Membership and Local Admin Fixes
One of the coolest features in BloodHound is the ability to visualize cross-domain data. Unfortunately, due to a misunderstanding of the way that Active Directory stores references to group membership, we were improperly collecting this data. The new release of SharpHound has been rewritten to properly collect all group membership data, including Universal and Domain Local groups, which were not being properly ingested. Additionally, we identified an issue where the LDAP query for groups was timing out in the middle of collection. We've upped the default LDAP timeout on our connections significantly, so this should no longer be a problem. For more information, you can see [this GitHub issue](https://github.com/BloodHoundAD/SharpHound/issues/20).

In addition to this group membership fix, we found out that *Domain Local* and *Universal* groups in local admin collection were also not being properly collected. We've also corrected this issue. The end result of these fixes is more accurate data. If you want more information on these groups, you can check out @harmj0y's [blog post](https://www.harmj0y.net/blog/activedirectory/a-pentesters-guide-to-group-scoping/) on the subject.

### Fix for ComputerFile Collection
A reported issue was that the ComputerFile collection method [didn't work](https://github.com/BloodHoundAD/SharpHound/issues/15). The solution to this problem ended up being a weird one, where ping checks took longer when loading from text files as opposed to LDAP. A fix is implemented that will up the default PingTimeout in SharpHound when the ComputerFile option is selected.

### Fix for Session Loop Collection
At some point, the SessionLoop collection method was broken. Specifically, the part that calculated when the session loop would end was completely non-functional. That bug has been fixed, so it should work again properly.

### Additional ACE filtering
We identified several scenarios that resulted in improper ACE collection and ingestion in the interface. A big part of this fix was properly ensuring that the inherited object type of an ACE matches up with the object type actually being enumerated. Previously, this was resulting in false positives when an ACE was set to, for example, "Descendant Group Objects" on a container that contained more than groups. We've taken steps to fix these issues, which should improve the accuracy of data collected and pathfinding.

# BloodHound User Interface Updates
The vast majority of the changes made in 1.5 are in the user interface for BloodHound, and we have several new features to look at.

## GPO and OU Nodes + Owners
To support the addition of containers, we’ve added GPO and OU nodes to the interface. Both types of nodes have their own data displays, similar to existing nodes.

### GPO Nodes and the GpLink Relationship
GPO Nodes are linked to OU nodes using the GpLink relationship, which indicates the GPO is applied to the OU. GPOs can be **Enforced**, which prevents any lower precedence GPO from overriding the settings set. An **Enforced** GPO will also ignore the **blocks inheritance** flag on OU objects it is applied to. Enforced GpLink relationships are represented by a solid line similar to what is used on other relationships. However, GpLink relationships which are *not* set to **Enforced** will be represented with a dashed line to make it clear on the UI.

![GPO Example](/bloodhound15/gpoexample.png)

In the above screenshot, the GpLink relationship between the `DEFAULT DOMAIN POLICY@TESTLAB.LOCAL` node and the `TESTLAB.LOCAL` node is unenforced. The GpLink relationship between `LOCAL ADMINS@TESTLAB.LOCAL` and `TESTOU@TESTLAB.LOCAL` are enforced.

Unlike most nodes in the BloodHound schema, GPOs are indexed using their GUID, instead of their name, as GPO display names are not guaranteed to be unique. Clicking on a GPO node will give you the following information.

![GPO Node Example](/bloodhound15/gponodedata.png)

The query allows you to visualize what objects are actually affected by the GPO, which will allow you to determine what users and/or computers will be affected by modifying the GPO.

### OU Nodes and the Contains Relationship
OU nodes are container objects that allow for management of different units in Active Directory. OU nodes are generally used in conjunction with GPOs to allow delegation of privilege or organization-specific changes to groups of users or computers. OU nodes can **block inheritance**, which prevents GPO settings from being inherited from parent OUs. Similar to GpLink relationships, the Contains relationship is represented as a dotted line if the OU **blocks inheritance**. Domains are also container objects -- similar to OUs -- and have the Contains relationship. They serve as the top level container.

![OU Example](/bloodhound15/ouexample.png)

In the above screenshot, `TESTOU@TESTLAB.LOCAL` blocks inheritance, so the Contains relationship for its children displays this as a dotted-line edge. `TESTLAB.LOCAL` does not block inheritance, so the relationship is displayed normally. Clicking on an OU node will give you the following information. The queries will allow you to find descendant objects for the OU.

![OU Node Example](/bloodhound15/ounodedata.png)

## The Owner Edge
We've also added the **Owns** relationship to the user interface, which indicates a principal that is the owner of another principal.

![Owner Edge](/bloodhound15/owner.png)

The owner of a node by default can modify any property on an object, even if not explicitly granted full control. The **Owns** edge is essentially the equivalent of **GenericAll**.

## Upgraded Search Bar
![New Search Bar](/bloodhound15/15searchbar.gif)

We've upgraded the stock search and pathfinding bars in BloodHound, allowing you to specify the type of node you're searching for in the search bar. By prefixing your search term with a node label, similar to the way you would in Cypher, you can filter your search further. Additionally, we've added the ability to search for OUs and GPOs by GUIDs to allow you to find the right node.

![GUID Search](/bloodhound15/guidsearch.png)

## Upgraded Pre-Built Queries (And a couple new ones)
![New Prebuilt Queries](/bloodhound15/newprebuiltqueries.gif)

We've upgraded the pre-built queries as well to make them more robust and flexible. Pre-built queries can now have any number of filtering steps as needed. Additionally, as shown in the gif above, you can display extra information on the nodes displayed, such as **PwdLastSet**, which is an important factor when deciding which user to target for kerberoasting. Unfortunately, the format of pre-built queries has changed, which means if you've made your own custom queries, you'll have to re-write them to the new format, which should be easier to use.

```json
{
  "name": "Shortest Path from SPN User",
  "queryList":[
      {
          "final": false,
          "title":"Select a domain...",
          "query":"MATCH (n:Domain) RETURN n.name ORDER BY n.name DESC"
      },
      {
          "final": false,
          "title": "Select a user",
          "query": "MATCH (n:User) WHERE n.domain={result} AND n.HasSPN=true RETURN n.name, n.PwdLastSet ORDER BY n.PwdLastSet ASC"
      },
      {
          "final": true,
          "query": "MATCH n=shortestPath((a:User {name:{result}})-[r*1..]->(b:Computer)) RETURN n",
          "startNode":"{}",
          "allowCollapse": true
      }
  ]
}
```

The new format takes a list of queries, with the last query marked as final to tell the UI to graph the query instead of displaying a selection box. Adding properties is as simple as returning the properties in the query, the user interface will do all the work of mapping the properties to the display. The previous values such as allowCollapse and startNode/endNode are still usable.

## New Options Menu
![New Options](/bloodhound15/newoptions.png)

We've added more options to the options menu to help you control the UI's behavior. In particular, we've added options to control how the node and edge labels are displayed. The *ctrl* shortcut is still available for node labels. These options are also persistent, and will save, so you won’t have to keep setting these unless you change them.

## Improved Domain Nodes
![New Domain Node](/bloodhound15/newdomainnode.png)

We've added more data to the Domain nodes, and fixed up the queries on these displays as well. With these additions, it will be easier to locate foreign members of domains that you're interested in. We've also added ACL relationships to the domain nodes, which will allow you to quickly view any controllers for the domain object.

Additionally, we've introduced a new data point on Domain nodes which will calculate all the principals that can DCSync. This will take into account principals grabbing the **DS-Replication-Get-Changes** and **DS-Replication-Get-Changes-All** rights from different places. A prime example of this is the **Domain Controllers** and **Enterprise Domain Controllers** groups, which together provide the rights needed for Domain Controllers to perform replication.

## Domain Property for Nodes
We've modified the ingestion of CSV files to add a new property to nodes, which keeps track of what domain a node belongs to. This allows you to more easily write queries for cross-domain nodes by leveraging this new property. We use this in some of our pre-built queries.

```json
{
  "name": "Users with Foreign Domain Group Membership",
  "queryList": [
      {
          "final": false,
          "title": "Select source domain...",
          "query": "MATCH (n:Domain) RETURN n.name ORDER BY n.name DESC"
      },
      {
          "final": true,
          "query": "MATCH (n:User) WITH n MATCH (m:Group) WITH n,m MATCH p=(n)-[r:MemberOf]->(m) WHERE n.domain={result} AND NOT m.domain=n.domain RETURN p",
          "startNode": "{}",
          "allowCollapse": false
      }
  ]
}
```

To get this property in existing data sets, a re-import of the CSV files will be necessary.

# Wrap-Up
BloodHound 1.5 represents a large milestone in the BloodHound project, and we're hoping the new release will open up new avenues of attack analysis and interesting new paths for compromise, as well as help give the best possible recommendations for both offensive and defensive teams. We also hope that this new release will solve any issues that people might have had in the past.

You can grab compiled versions of the project here or compile it yourself from the BloodHound repository here.

As usual, feel free to join us any time in the [BloodHound Slack Channel](https://bloodhoundgang.herokuapp.com/). The link will allow you to get an invite, and join our active community with over 1200(!) users. It's also the most direct way to get in touch with us for BloodHound issues. We look forward to hearing from any new users and hopefully making BloodHound better for everyone!

# Special Thanks
Thanks to all our beta testers who helped ensure that 1.5 was (relatively) bug free. Special thanks to [@Crypt0-m3lon](https://twitter.com/crypt0_m3lon) for a pull request to fix several issues with the beta.
