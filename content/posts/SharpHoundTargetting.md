---
title: "SharpHound: Target Selection and API Usage"
date: 2018-03-05T12:00:00-05:00
draft: false
tags: ["bloodhound", "sharphound"]
---

One of the most common questions we get from BloodHound users is related to how collection is done, as well as what targets are selected for different collection methods. In this post, we're going to detail what each collection method does, particularly which API calls are used for each different step, as well as the detailed target selection logic.

## What does each collection method do?
The SharpHound collector has several discrete steps which run simultaneously to collect different data necessary for the graph. The overall breakdown falls into a few categories: Local Admin Collection, Group Membership Collection, Session Collection, Object Property Collection, ACL Collection, and Trust Collection.

### Local Admin Collection
Local Admin Collection is done using two different methods, depending on if the stealth option is specified. Local admin collection without the stealth option will first query Active Directory for a list of computers. The list of computers is passed to the enumeration runner which reaches out to each computer and does the following actions:

* Performs a TCP connect on port 445 to check if the host is alive
* If the host is alive, performs a modified version of the [NetLocalGroupGetMembers NETAPI32](https://msdn.microsoft.com/en-us/library/windows/desktop/aa370601(v=vs.85).aspx) API call. For more information on the modified API call, see the "New Local Administration" section of [this blog post](https://blog.cptjesus.com/posts/sharphoundtechnical) or the code [here](https://github.com/BloodHoundAD/SharpHound/blob/947b0d42399be23530975f719dce33a9e8b6e66d/Sharphound2/Enumeration/LocalAdminHelpers.cs#L50)
* Takes the data returned from the NetLocalGroupGetMembers call and resolves SIDs to actual users, and filters out local accounts.

If stealth collection is specified, SharpHound will instead query the domain controller for a list of all Group Policy Container objects and their corresponding **gpcfilesyspath** attribute which indicates where the actual group policy file is located in the domain controller $SYSVOL directory. Each group policy file will be enumerated, looking for the pattern *S-1-5-32-544__Members*, which indicates the local Administrators group. The files are processed to determine which computers these GPOs are applied too.

### Session Collection
Session collection is the same regardless of if stealth is specified or not. However, BloodHound exposes two different methods of querying session information for computers. Both methods start by checking for port 445, but then diverge.

#### Default Session Collection
The default method of session collection uses the [NetSessionEnum NETAPI32](https://msdn.microsoft.com/en-us/library/windows/desktop/bb525382(v=vs.85).aspx) call. An important distinction with the NetSessionEnum API call is that it does not allow you to directly query a system to ask who is logged on. Instead, it allows you to query a system for what network sessions are established to that system and from where. Network sessions are created when network resources, such as a file share, are accessed. A good example is mounted home drives that reside on the domain controller.

As an example, the API call can be run like this:

```csharp
NetSessionEnum("primary.testlab.local", null, null, 10, out IntPtr ptrInfo, -1, out int EntriesRead, out int _, ref resumeHandle);
```

The fourth parameter of the API call is the **level** of the API call, with 10 being the only level that gives the data necessary for BloodHound in an unauthenticated manner.

Running this against a domain controller with a mounted shared drive will give you results similar to this:

```
sesi10_cname - 192.168.1.10
sesi10_username - rvazarkar
sesi10_time - 0
sesi10_idle_time - 0
```

The sesi10_cname parameter indicates *where* the session is coming from, so resolving this IP address to a hostname allows us to correlate sessions to remote hosts. This is frequently the reason why you might not see logon sessions that you know exist. If an outward network session doesn't exist, the unauthenticated collection methods of SharpHound have no way of enumerating this data. It is also important to note that the username parameter does NOT have a domain associated with it, which means that session information has an element of guesswork involved. SharpHound uses the global catalog to attempt to deconflict users, and figure out which domain is the proper one, but there's no guarantee that it will pick the correct one.

#### LoggedOn Session Collection
The **LoggedOn** collection method is a much more precise collection method that returns session information by asking computers who is actually logged in to the system. The caveat is that this level of collection **requires administrative** privileges on hosts that you want to collect data from. This session collection is ideal for defenders, or for additional data collection after getting Domain Admin. The **LoggedOn** collection method uses two different methods to collect data. The first method is using the [NetWkstaUserEnum NETAPI32](https://msdn.microsoft.com/en-us/library/windows/desktop/aa370669(v=vs.85).aspx) API call.

An example of this API call is as follows:

```csharp
NetWkstaUserEnum("primary.testlab.local", 1, out IntPtr intPtr, -1, out int entriesRead, out int _, ref resumeHandle);
```

The second parameter of the API call is the **level** of the API call, with 1 returning more data than 0. Running this against a system results in data like this:

```csharp
wkui1_username - rvazarkar
wkui1_logon_domain - TESTLAB
wkui1_oth_domains -
wkui1_logon_server - PRIMARY
```

The secondary enumeration method used by the LoggedOn collection method is using the Remote Registry. SharpHound will attempt to open the Users hive of the Remote Registry if it is enabled and will look for subkeys that match the SID format. These correspond to logged-on users. In rare occasions, this data is actually accessible without Administrative privileges. You can verify if this condition occurs in your environment by running the LoggedOn collection method from an unprivileged user and checking the resulting data for results.

### Group Membership Collection
All domain group membership collection is done through LDAP. SharpHound will ask the domain controller for a list of every group, user, and computer object in the Domain, and use the **MemberOf** property to resolve group membership. Group membership collection does not require touching any system other than the Domain Controller.

### ACL Collection
All AD object ACL collection is done through LDAP. SharpHound will ask the Domain Controller for a list of every user, group, computer, and domain object in the domain, and use the **NTSecurityDescriptor** property to resolve the access control list. ACL collection does not require touching any system other than the Domain Controller.

### Trust Collection
Trust collection is performed using the [DsEnumerateDomainTrusts NETAPI32](https://msdn.microsoft.com/en-us/library/ms675976(v=vs.85).aspx) API call. An example of running this query:

```csharp
DsEnumerateDomainTrusts("testlab.local", 63, out IntPtr ptr, out int domainCount);
```

The second parameter is a set of flags that specify which trust types to return. 63 corresponds to all possible flags:
* DS_DOMAIN_IN_FOREST
* DS_DOMAIN_DIRECT_OUTBOUND
* DS_DOMAIN_TREE_ROOT
* DS_DOMAIN_PRIMARY
* DS_DOMAIN_NATIVE_MODE
* DS_DOMAIN_DIRECT_INBOUND

This will return all possible domain types

### Object Property Collection
All property collection is done through LDAP. SharpHound will ask the Domain Controller for a list of every user and computer object in the domain, and request several different properties for each object.

For user objects, the following properties are used:
* SamAccountName
* DistinguishedName
* SaMAccountType
* PwdLastSet
* LastLogon
* SidHistory
* UserAccountControl
* Mail
* ObjectSid
* ServicePrincipalName
* DisplayName

For computer objects, the following properties are used:
* SaMAccountName
* DistinguishedName
* SaMAccountType
* ObjectSid
* UserAccountControl
* DNSHostName
* OperatingSystemServicePack
* OperatingSystem

## What systems do each collection methods target?
SharpHound changes target selection significantly based on the flags provided. The default collection methods used by SharpHound are very loud, touching every system on the domain that is reachable. Using the stealth flag significantly lowers the number of systems targeted.

### Local Admin Collection - Non Stealth
Local admin collection without the stealth flag will reach out to every single reachable domain computer to gather data. This gives reliable and accurate results.

### Local Admin Collection - Stealth
Local admin collection with the stealth flag relies entirely on Group Policy settings and does not need to touch any systems other than the domain controllers that contain the relevant files in their SYSVOL folders. The quality of data returned from stealth collection will vary significantly from domain to domain, as some domains do not use GPO to manage local admin settings, while others use it exclusively. GPOs will not reflect local changes made, so additional principles added to the local administrators group will not be found using this method.

### Session Collection - Non Stealth
Session collection without the stealth flag will reach out to every single reachable domain computer to gather data.

### Session Collection - Stealth
Session collection with the stealth flag greatly limits the number of systems targeted for collection. SharpHound will target all computers marked as Domain Controllers using the **UserAccountControl** property in LDAP. A list of all Active Directory objects with the any of the **HomeDirectory**, **ScriptPath**, or **ProfilePath** attributes set will also be requested. A unique set of server names is created from these properties to identify additional targets for session collection. On average, stealth session collection will gather around 50-60% of session information in the domain. This can vary depending on the structure of the domain, but most network sessions are generally pointing to Domain Controllers or file servers.

### Group Collection - Stealth and Non-Stealth
Group collection only requires talking to a domain controller to request LDAP data.

### ACL Collection - Stealth and Non-Stealth
ACL collection only requires talking to a domain controller to request LDAP data.

### Trust Collection - Stealth and Non-Stealth
Trust collection requires talking to one domain controller in each domain mapped.

### Object Property - Stealth and Non-Stealth
Object property collection requires talking to a domain controller to request LDAP data.

## Cheat Sheets

| Collection Method | API Call | Default Targets | Stealth Targets |
| :---| :---: | :---: |:---|
| Session | [NetSessionEnum](https://msdn.microsoft.com/en-us/library/windows/desktop/bb525382(v=vs.85).aspx) | All Computers | Domain Controllers + "Share Servers" |
| LocalGroup | Modified [NetLocalGroupGetMembers](https://msdn.microsoft.com/en-us/library/windows/desktop/aa370601(v=vs.85).aspx) | All Computers | GPO Files |
| Group | Ldap | All User,Group, and Computer Objects | All User,Group, and Computer Objects |
| Trusts | [DsEnumerateDomainTrusts NETAPI32](https://msdn.microsoft.com/en-us/library/ms675976(v=vs.85).aspx) | All Domain and TrustedDomain objects | All Domain and TrustedDomain objects |
| LoggedOn | [NetWkstaUserEnum NETAPI32](https://msdn.microsoft.com/en-us/library/windows/desktop/aa370669(v=vs.85).aspx) + Remote Registry | All Computers | Domain Controllers + "Share Servers" |
| ACL | Ldap | All user, group, computer, and domain objects | All user, group, computer, and domain objects |
| ObjectProps | Ldap | All user and computer objects | All user and computer objects |

| API Call | Protocol | Port | RPC Interface UUID | Named Pipe | RPC Method |
| :---| :---: | :---: | :--- | :--- | :--- |
| [NetSessionEnum](https://msdn.microsoft.com/en-us/library/windows/desktop/bb525382(v=vs.85).aspx) | [MS-SRVS]: Server Service Remote Protocol | TCP 445 | 4B324FC8-1670-01D3-1278-5A47BF6EE188 | \PIPE\srvsvc | [NetrSessionEnum](https://msdn.microsoft.com/en-us/library/cc247273.aspx) |
| [NetWkstaUserEnum](https://msdn.microsoft.com/en-us/library/windows/desktop/aa370669(v=vs.85).aspx) | [MS-WKST]: Workstation Service Remote Protocol | TCP 445 | 6BFFD098-A112-3610-9833-46C3F87E345A | \PIPE\wkssvc | [NetrWkstaUserEnum](https://msdn.microsoft.com/en-us/library/cc250349.aspx) |


## Conclusion
Hopefully, this blog post should help clear up some of the confusion surrounding what actions SharpHound performs to collect data, as well as what targets are used for data collection. If you have additional questions, feel free to reach out on [Twitter](www.twitter.com/cptjesus), email me at [rohan@specterops.io](rohan@specterops.io), or you can join us in the [BloodHound Slack channel](https://bloodhoundgang.herokuapp.com/).