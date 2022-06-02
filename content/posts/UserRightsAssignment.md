---
title: "User Rights Assignment: RDP"
date: 2022-06-01T23:23:30-04:00
draft: false
unlisted: true
---

Active Directory is crazy. Sometimes, its really easy to forget how many moving parts are under the hood of what we see, but there's far more going on than is immediately obvious. Part of the fun of switching to a full time developer working on an enterprise product is a huge focus on accuracy. There are certain tradeoffs that are acceptable when doing offensive security which aren't acceptable in an enterprise environment. Similarly, there are plenty of features that attackers use that defenders would find completely worthless. One of the more interesting issues we've run into is accurate resolution of users who have the ability to initiate Remote Desktop Protocol (RDP) sessions to computers. 

Initially, we thought this was a pretty simple check: are you in the "Remote Desktop Users" group? If so, go ahead and RDP. As with everything in Windows, the answer is actually far more complicated than it initially appears. Permission to initiate RDP sessions actually lies in several different places on an individual host, and you need all the individual parts to successfully RDP to a computer.

## The Current Implementation
The current implementation of `CanRDP` in BloodHound/SharpHound is extremely naive. The only condition for this edge to appear is membership of the `Remote Desktop Users` group on a system. For a long time, we thought this was all you needed, and we were very confused when we got reports that RDP wasn't working in random environments. Over time, we narrowed this down to environments running Citrix, but decided it was low priority for offensive purposes, especially when it was very difficult to reproduce and consistently test this. Offensive engagements generally being short means by the time we get a report about a particular issue and have a chance to look into it, the assessment is frequently already over, so triaging the issue is impossible. Thanks to some research and running in enterprise environments where we can continuously iterate and improve, we've finally figured out what's actually going on.

## LSA and SE Rights
Every windows system has a Local Security Authority (LSA). LSA stores lots of fun stuff that attackers find very interesting, particularly in-memory credentials dumped via mimikatz or other tools. However, LSA stores a lot more than just this, and one of the interesting things is called User Rights Assignment (URA). If running as an Administrator, you can access the current settings for URA by opening the `Local Security Policy` snap-in and finding them under `Local Policies`.

![User Rights Assignment](/urardp/userrights.png)

There are quite a few interesting rights in here, such as the ever popular `SeDebugPrivilege`, an attackers favorite, which allows you to attach debuggers to random processes/the kernel, which allows you to dump passwords from the Local Security Authority Server Service (LSASS). Several lesser known privileges here will allow an attacker to escalate privileges pretty easily without too much extra effort. 

## SeRemoteInteractiveLogonPrivilege
The privilege we're particularly interested in for this blog post is `Allow log on through Remote Desktop Services`, also known as `SeRemoteInteractiveLogonPrivilege`. This is the first part of the equation which allows us to RDP to a system, and the part that is currently missing in the BloodHound schema. By default, normal workstations grant this privilege to the `Remote Desktop Users` and `Administrators` local groups. If you look at the explain tab for the privilege, you'll find information that domain controllers only grant this privilege to the `Administrators` local/domain group.

![Explain](/urardp/rilexplain.png#center)

Where this gets interesting is in environments that run Citrix. In a majority of environments we've run into using Citrix, this privilege is _only_ granted to the `Administrators` group on Virtual Desktop Infrastructure (VDI) servers. The `Remote Desktop Users` group is removed from this rights grant, effectively making them unable to RDP. At the same time, the `Domain Users` group is added to the `Remote Desktop Users` group, which leads to BloodHound generating a `CanRDP` edge from `Domain Users` to the VDI host. Inevitably when testing this, the RDP session will fail, since `Domain Users` does not have the necessary privilege. When trying to access an RDP session without this right, you'll get this error message.

![No SeRemoteInteractiveLogonPrivilege](/urardp/rdpnoril.png#center)

Having the `SeRemoteInteractiveLogonPrivilege` is vital to being able to complete your RDP session. However, as you might have noted from earlier, this is only one of the parts necessary.

## Remote Desktop Services Security Descriptor
The second part of the formula for establishing RDP connections is fairly obscure in its source. The `Remote Desktop Service` on each system has its own Security Descriptor which is buried in a Windows Management Instrumentation (WMI) object on the system. As with any security descriptor, you can allow and deny principals. If you want to see what this setting has, you can use the following PowerShell command as an admin:

```posh
((Get-WmiObject -class Win32_TSPermissionsSetting -namespace "root\CIMV2\terminalservices" | ?{$_.TerminalName -eq "RDP-tcp"} | select -expand stringsecuritydescriptor | convertfrom-sddlstring) |select -expand discretionaryacl) -split ':'
```

Running this on my own test Windows 11 VM gives me the following:

```
NT AUTHORITY\INTERACTIVE
 AccessAllowed (ListDirectory)
NT AUTHORITY\SYSTEM
 AccessAllowed (ChangePermissions, CreateDirectories, Delete, ExecuteKey, FullControl, GenericExecute, GenericRead, GenericWrite, ListDirectory, Modify, Read, ReadAndExecute, ReadAttributes, ReadExtendedAttributes, ReadPermissions, TakeOwnership, Traverse, Write, WriteAttributes, WriteData, WriteExtendedAttributes, WriteKey)
NT AUTHORITY\LOCAL SERVICE
 AccessAllowed (ChangePermissions, Delete, ListDirectory, Read, ReadAttributes, ReadExtendedAttributes, ReadPermissions, TakeOwnership)
NT AUTHORITY\NETWORK SERVICE
 AccessAllowed (ListDirectory, ReadAttributes)
BUILTIN\Administrators
 AccessAllowed (ChangePermissions, CreateDirectories, Delete, ExecuteKey, FullControl, GenericExecute, GenericRead, GenericWrite, ListDirectory, Modify, Read, ReadAndExecute, ReadAttributes, ReadExtendedAttributes, ReadPermissions, TakeOwnership, Traverse, Write, WriteAttributes, WriteData, WriteExtendedAttributes, WriteKey)
BUILTIN\Remote Desktop Users
 AccessAllowed (ListDirectory, Traverse, WriteAttributes)
 ```

The important part to note is that the `BUILTIN\Remote Desktop Users` group is granted `AccessAllowed` on this security descriptor. This is true by default on every server/workstation in my limited testing so far. The right is also granted to the `Administrators` group. The second part of the equation for successfully connecting via RDP to a system is in this security descriptor. If you don't have `AccessAllowed` through some means, you'll get an entirely different error message.


![No RDP membership](/urardp/rdpfail.png#center)

## The Box of Wrenches
So far this seems pretty easy: check if we have the correct privilege and check if we have membership in `Remote Desktop Users`. In most cases, this is true, and in most cases the group itself will just be granted both rights. However, in the interest of accuracy, we decided to try and chase down corner cases. While examining possible scenarios, we ran into a very interesting behavior with local groups. If you've ever tried to nest local groups on a windows system, you'll find something fun.

![Local Group Nesting](/urardp/localgroup.png)

In the above screenshot, you'll see that I've added a group local to the computer to the `Remote Desktop Users` group. Based on our knowledge of group nesting in Active Directory, you would expect members of the TestGroup to get the effects of membership to this group. As you're beginning to guess, this assumption is incorrect. Nested local groups do _not_ inherit permissions the same way that nested domain groups do. In the above scenario, members of `TestGroup` will not be able to RDP. This behavior is documented in a since lost Microsoft Knowledge Base article, of which an archive can be found [here](https://mskb.pkisolutions.com/kb/974815). Of particular note is the following blurb:

`This behavior is by design. Windows does not support the nesting of local groups on domain clients or on workgroup clients.`

You might be thinking "Well that's great Rohan, you don't have to worry about it anymore!"

As with many things Windows, it, unfortunately, isn't that simple. While local group nesting will not work for membership to the `Remote Desktop Users` group, it _will_ work for possession of the `SeRemoteInteractiveLogonPrivilege`. Consider the following configuration:

![Crazy RDP](/urardp/crazyrdp.png)

Our user, `nub` is a domain user that belongs to the local group, `TestGroup`, which has been added to the `Distributed COM Users` local group. The `Distributed COM Users` group has been granted the `SeRemoteInteractiveLoginPrivilege` right. Despite this privilege coming from group nesting, which supposedly does not work, the user will be able to RDP just fine. So local group nesting _does_ work for LSA privileges. 

## The Formula
With all the information we've covered so far, our workflow of figuring out who can RDP to a system now looks something like the following, including data collection steps. For the sake of sanity, we've chosen to ignore the security descriptor on the RDP service and assume that it'll be the default. 

1. Open the SAMRPC server to get local groups on the system, and collect membership to the `Remote Desktop Users` group, as well as _all other local groups_ and encode these as a new relationship, `LocalToComputer`
2. Open the LSA policy and enumerate principals that have been granted the `SeRemoteInteractiveLoginPrivilege` right, and encode these as a new privilege `RemoteInteractiveLogon`
3. Find all principals that have an intersection of the membership between the two seperate parts, including through domain group membership, plus local group membership for the URA privilege.
4. Take principals that meet both requirements and create a `CanRDP` edge to the computer.

In the vast majority of cases, this is going to be fairly simple, but because accuracy is important, we also encoded as many (ridiculous) corner cases as we could think of. I'm sure we missed a few, that we'll eventually stumble on to in some crazy environment, because AD environments are configured in the craziest ways we can think of.

## What does this mean going forward?
In an upcoming update of [BloodHound Enterprise](https://bloodhoundenterprise.io), the above logic will be added to our post-processing steps. All the data collection logic for local groups and LSA has been added in to the [SharpHoundCommon library](https://github.com/BloodHoundAD/SharpHoundCommon), which forms the base for both open source and enterprise collectors. As of right now, we lack some capabilities in open source BloodHound to do the processing necessary to port the logic over, but we'll be aiming to eventually add the logic there as well. However, this is only the start of our plans for URA information. Being able to collect user rights opens whole new attack path opportunities in the graph that we hope to be able to collect and evaluate in the future, including potentially local privilege escalation opportunities. This also changes our graph and adds `LocalGroup` nodes, which adds more opportunities for inspecting interesting local configurations.

## Final Thoughts
While we won't have an update for Open Source BloodHound in the near future (Cypher makes it insanely difficult to do this logic), we felt it was important to share some of the work that's happening behind the scenes, and some of the interesting upcoming work on both the Enterprise and Open Source sides. It's also important to us to affirm that this work will make its way to the open source version in a future update, and that the actual collection logic will be available in the Common Library for use should you be interested in playing around with it.

You can always find us in the [BloodHound Slack Channel](https://bloodhoundgang.herokuapp.com/), which just recently passed the 10000(!!!) user mark. The link will allow you to get an invite and join our active community.