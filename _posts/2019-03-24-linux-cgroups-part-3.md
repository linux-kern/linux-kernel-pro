---
layout: post
title : "Control groups, part 3: First steps to control"
author: Neil Brown
category: linux-cgroups
tags: linux kernel lwn cgroups
published: true
keywords: linux,kernel,lwn,cgroups
script: [post.js]
excerpted: |
    Having spent two articles exploring some background context concerning the grouping of processes and the challenges of hierarchies, we need to gain some understanding of control, and for that we will look at some details of the current (`Linux 3.15`) cgroups implementation.
#day_quote:
#  title: 
#  description: |
#    Put a very powerful message.
---

* Index
{: toc}

**Original&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;:**&nbsp;&nbsp;[**Control groups, part 3: First steps to control**](https://lwn.net/Articles/605039/)<br>
**Author&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;:**&nbsp;&nbsp;**Neil Brown**<br>
**Date&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;:**&nbsp;&nbsp;**July 16, 2014**<br>
**Signed-off-by:**&nbsp;&nbsp;**Norman Kern \<norman.gmx.com\>**

Having spent two articles exploring some background context concerning the grouping of processes and the challenges of hierarchies, we need to gain some understanding of control, and for that we will look at some details of the current (`Linux 3.15`) cgroups implementation. First on the agenda is how control is exercised in cgroups, then, in later articles, we will look at how hierarchy is used.

Some degree of control existed long before cgroups, of course, with the "nice" setting to control how much CPU time each process received and the various limits on memory and other resources that can be controlled with the setrlimit() system call. However, this control was always applied to each process individually or, possibly, to each process in the context of the whole system. As such, those controls don't really provide any hints on how grouping of processes affects issues of control.

# Cgroup subsystems
The cgroups facility in Linux allows for separate "subsystems" that each independently work with the groupings that cgroups provides. The term "subsystem" is unfortunately very generic. It would be nicer to use the term "resource controller" that is often applied. However, not all "subsystems" are genuinely "controllers", so we will stay with the more general term.

In Linux 3.15, there are twelve distinct subsystems which, together, can provide some useful insights into a number of issues surrounding cgroups. We will look at them roughly in order of increasing complexity; not the complexity of what is controlled, but of how the groups and, particularly, the hierarchy of groups, is involved in that control. Slightly over half will be discussed in this article with the remainder relegated to a subsequent article.

Before detailing our first seven, a quick sketch of what a subsystem can do will be helpful. Each subsystem can:
- store some arbitrary state data in each cgroup
- present a number of attribute files for each cgroup in the cgroup filesystem that can be used to view or modify this state data, or any other state details
- accept or reject a request to attach a process to a given cgroup
- accept or reject a request to create a new group as a child of an existing one
- be notified when any process in some cgroup forks or exits
Naturally, a subsystem can also interact with processes and other kernel internals in arbitrary ways in order to implement the desired result. These are just the common interfaces for interacting with the process group infrastructure.

# A simple debugging aid
The debug subsystem is one of those that does not "control" anything, nor does it remove bugs (unfortunately). It does not attach extra data to any cgroup, never rejects an "attach" or "create" request, and doesn't care about forking or exiting.

The sole effect of enabling this subsystem is to make a number of internal details of individual groups, or of the cgroup system as a whole, visible via virtual files within the cgroup filesystem. Details include things like the current reference count on some data structures and the settings of some internal flags. These details are only likely to be at all interesting to someone working on the cgroups infrastructure itself.

# Identity — the first step to control
It was Robert Heinlein who first exposed me to the idea that requiring people to carry identity cards is sometimes a first step toward controlling them. While doing this to control people may be unwelcome, providing clear identification for controlling processes can be quite practical and useful. This is primarily what the net_cl and net_prio cgroup subsystems are concerned with.

Both of these subsystems associate a small identifying number with each cgroup which is then copied to any socket created by a process in that group. net_prio uses the sequence number that the cgroups core provides each group with (cgroup->id) and stores this in the sk_cgrp_prioidx attribute. This is unique for each cgroup. net_cl allows an identifying number to be explicitly given to each cgroup, and stores it in the sk_classid attribute. This is not necessarily unique for each cgroup.
These two different group identifiers are used for three different tasks.

1. The sk_classid set by net_cl can be used with iptables to selectively filter packets based on which cgroup owns the originating socket.
2. The sk_classid can also be used for packet classification during network scheduling. The packet classifier can make decisions based on the cgroup and other details, and these decisions can affect various scheduling details, including setting the priority of each message.
3. The sk_cgrp_prioidx is used purely to set the priority of network packets, so when used it overrides the priority set with the SO_PRIORITY socket option or by any other means. A similar effect could be achieved using sk_classid and the packet classifier. However, according to the commit that introduced the net_prio subsystem, the packet classifier isn't always sufficient, particularly for data center bridging (DCB) enabled systems.

Having two different subsystems controlling sockets in three different ways, with some overlap, seems odd, though it is a little familiar from the overlap we saw with process groups previously. Whether subsystems should be of lighter weight, so it is cheap to have more (one per control element), or whether they should be more powerful, so just one can be used for all elements, is not yet clear. We will come across more overlap in later subsystems and maybe that will help clarify the issues.

For the moment, the important issue remaining is how these subsystems interact with the hierarchical nature of cgroups. The answer is that, for the most part, they don't.

Both the class ID set for net_cl and the (per-device) priorities set for net_prio apply just to that cgroup and any sockets associated with processes in that cgroup. They do not automatically apply to sockets in processes in child cgroups. So, for these subsystems, recursive membership in a group is not relevant, only immediate membership.

This limitation is partially offset by the fact that newly created groups inherit important values from their parent. So if, for example, a particular network priority is set for all groups in some subtree of the cgroup hierarchy, then that priority will continue to apply to all groups in that subtree, even if new groups are created. In many practical cases, this is probably sufficient to achieve a useful result.

This near-disregard for hierarchy makes the cgroups tree look, from the perspective of these subsystems, more like an organizational hierarchy than a classification hierarchy (as discussed in the hierarchies article) — subgroups are not really sub-classifications, just somewhere convenient that is nearby.

Other subsystems provide more focus on hierarchy in various ways; three worth contrasting are devices, freezer, and perf_event.

# Ways of working with hierarchy
One of the challenges of big-picture thinking about cgroups is that the individual use cases can be enormously different and place different demands on the underlying infrastructure. These next three subsystems all make use of the hierarchical arrangement of cgroups, but differ in the details of how control flows from the configuration of a cgroup to the effect on a process.

## devices
The devices subsystem imposes mandatory access control for some or all accesses to device-special files (i.e. block devices and character devices). Each group can either allow or deny all accesses by default, and then have a list of exceptions where access is denied or allowed, respectively.

The code for updating the exception list ensures that a child never has more permission than its parent, either by rejecting a change that the parent doesn't allow or by propagating changes to children. This means that when it comes time to perform a permission check, only the default and exceptions in one group need to checked. There is no need to walk up the tree checking that ancestors allow the access too, as any rules imposed by ancestors are already present in each group.

There is a clear trade-off here, where the task of updating the permissions is made more complex in order to keep the task of testing permissions simple. As the former will normally happen much less often than the latter, this is a sensible trade-off.

The devices subsystem serves as a mid-point in the range of options for this trade-off. Configuration in any cgroup is pushed down to all children, all the way down to the leaves of the hierarchy, but no further. Individual processes still need to refer back to the relevant cgroup when they need to make an access decision.

## freezer
The freezer subsystem has completely different needs, so it settles on a different point in the range. This subsystem presents a "freezer.state" file in each cgroup to which can be written "FROZEN" or "THAWED". This is a little bit like sending SIGSTOP or SIGCONT to one of the process groups that we met in the first article in that the whole group of processes is stopped or restarted. However, most of the remaining details differ.

In order to implement the freezing or thawing of processes, the freezer subsystem walks down the cgroup hierarchy, much like the devices subsystem, marking all descendant groups as frozen or thawed. Then it must go one step further. Processes do not regularly check if their cgroup has been frozen, so freezer must walk beyond all the descendant cgroups to all the member processes and explicitly move them to the freezer or move them out again. Freezing is arranged by sending a virtual signal
to each process, since the signal handling code does check if the cgroup has been marked as frozen and acts accordingly. In order to make sure that no process escapes the freeze, freezer requests notification when a process forks, so it can catch newly created processes — it is the only subsystem that does this.

So the freezer occupies one extreme of the hierarchy management spectrum by forcing configuration all the way down the hierarchy and into the processes. perf_event occupies the other extreme.

## perf_event
The perf facility collects various performance data for some set of processes. This set can be all processes on the system, all that are owned by a particular user, all that descended from some particular parent, or, using the perf_event cgroup subsystem, all that are within some particular cgroup. The perf_event cgroup subsystem interprets "within" in a fully hierarchical sense — unlike net_cl which also tests group membership but does so based on an ID number that may be shared by
some but not all groups in a subtree.

To perform this hierarchical "within a group" test, perf_event uses the cgroup_is_descendant() function which simply walks up the ->parent links until it finds a match or the root. While this is not an overly expensive exercise, particularly if the hierarchy isn't very deep, it is certainly more expensive than simply comparing two numbers. The developers of the networking code are famous for being quite sensitive to added performance costs, particularly if these costs could apply to every
packet. So it is not surprising that the networking code does not use cgroup_is_descendant().

The ideal, of course, would be a mechanism for testing membership that was both fully hierarchical and also highly efficient. Such a mechanism is left as an exercise for the reader.

For perf, we can see that configuration is never pushed down the hierarchy at all. Whenever a control decision is needed (i.e. should this event be counted?), the code walks up the tree from a process to find the answer.

If we look back to net_cl and net_perf and ask exactly how they fit into this spectrum that contrasts pushing configuration down from cgroups with processes reaching up to receive control, they are most like devices. Processes do refer back to a single cgroup when creating a socket, but don't proceed up the hierarchy. Their distinction is that the pushing-down of configuration is left up to user space rather than being imposed by the kernel.

## cpuset: where it all began
The final cgroup subsystem to examine for this installment is cpuset, which was the first one to be added to Linux. In fact, the cpuset mechanism for controlling process groups predates the more generic cgroups implementation. One reminder of this is that there is a distinct cpuset virtual filesystem type that is equivalent to the cgroup virtual filesystem when configured with the cpuset subsystem. This subsystem doesn't really introduce anything new, but it does serve to emphasize some
aspects we have already seen.

The cpuset cgroup subsystem is useful on any machine with multiple CPUs, and particularly on NUMA machines, which feature multiple nodes and, often, wildly different speeds for access within and between nodes. Like net_cl, cpuset provides two distinct, but related, controls. It identifies a set of processors on which each process in a group may run, and it also identifies a set of memory nodes from which processes in a group can allocate memory. While these sets may often be the
same, the implementation keeps them completely separate; enforcing the sets involves two completely different approaches. This is most obvious when moving a process from one cgroup to another. If the set of allowed processors is different, the process can trivially be placed on the run queue for a new processor on which it is permitted. If the set of allowed memory nodes has changed, then migrating all the memory across from one node to another is far from trivial (consequently
this migration is optional).

Similar to the device subsystem, cpuset propagates any changes made to a parent into all descendants when appropriate. Unlike device, where the cgroup contained the authoritative control information, each process has its own CPU set that the scheduler inspects. Once changes are imposed on descendant groups, cpuset must take the same final step as freezer and impose the new settings on individual processes. Exactly why cpuset doesn't request notification of forks, while freezer does, will
have to remain a mystery.

Combined with this, cpuset will sometimes also walk up the hierarchy looking for an appropriate parent. One such case is when a process finds itself in a cgroup that has no working CPU in its set, possibly due to a CPU being taken offline. Another is when a high-priority memory allocation finds that all nodes in the mems_allowed set have exhausted their free memory. In both these cases, some borrowing of resources that are granted to ancestor nodes can be used to get out of a tight spot.

Both of these cases could conceivably be handled by maintaining an up-to-date set of "emergency" processors and memory nodes in each cgroup. It is likely that the complexity of keeping this extra information current outweighs the occasional cost of having to search up the tree to find a needed resource. We saw before that some subsystems will propagate configuration down the tree while others will search for permission up the tree. Here we see that cpuset does both, each as appropriate to
a particular need.

# The story so far

These seven cgroup subsystems have something in common that separates them from the five we will look at in the next article, and this difference relates to accounting. For the most part, these seven cgroup subsystems do not maintain any accounting, but provide whatever service they provide with reference only to the current instant and without reference to any recent history.

perf_event doesn't quite fit this description as it does record some performance data for each group of processes, though this data is never used to impose control. There is still a very clear difference between the accounting performed by perf_event and that performed by the other five subsystems, but we will need to wait until next time to clarify that difference.

Despite that commonality, there is substantial variety:
- Some actually impose control (devices, freezer, cpuset), while others provide identification to enable separate control (net_cl, net_prio) and still others aren't involved in control at all (debug, perf_event).
- Some (devices) impose control by requiring the core kernel code to check with the cgroup subsystem for every access, while others (cpuset, net_cl) impose settings on kernel objects (threads, sockets) and the relevant kernel subsystems take it from there.
- Some handle hierarchy by walking down the tree imposing on children, some by walking up the tree checking on parents, some do both and some do neither.

There is not much among these details that relates directly to the various issues we found in hierarchies last time, though the emphasis on using cgroups to identify processes perhaps suggests that classification rather than an organization is expected.

One connection that does appear is that, as these subsystems do not depend on accounting, they do not demand a strong hierarchy. If there is a need to use net_prio to set the priority for a number of cgroups and they don't all have a common parent, that need not be a problem. The same priorities can simply be set on each group that needs them. It would even be fairly easy to automate this if some rule could be given to identify which groups required the setting. We have already seen that
net_prio and net_cl require consistent initial settings to behave as you might expect, and this simply extends that requirement a little.

So, for these subsystems at least, there is a small leaning toward wanting a classification hierarchy, but no particular desire for multiple hierarchies.

Looking further back to the issues that process groupings raised, we see there continue to be blurred lines of demarcation, with different subsystems providing similar services, and distinct services being combined in the same subsystem. We also see that naming of groups is important, but apparently not trivial, as we potentially have two different numeric identifiers for each cgroup.

Of course we haven't finished yet. Next time we will look at the remaining subsystems: cpu, cpuacct, blkio, memory, and hugetlb, to see what they can teach us, and what sort of hierarchies might best suit them.

# Appendix

英语 | 汉语
------------ | -------------
agenda | n. 议程；日常工作事项；日程表
relegate | v. 贬职，把降低到，把……置于次要地位；使（球队）降级
disregard | vt. 忽视；不理；漠视；不顾
enormously | adv. 非常地，极其，在极大程度上
impose | vi. 利用；欺骗；施加影响
mandatory | adj. 强制的；托管的；命令的
propagating | adj. 传播的；繁殖的
thawing | n. 融化；熔化
spectrum | n. 光谱；频谱；范围；余象
predate | vt. 在日期上早于（先于）
distinct | adj. 明显的；独特的；清楚的；有区别的
authoritative | adj. 有权威的；命令式的；当局的
inspect | vt. 检查；视察；检阅
exhaust | vt. 排出；耗尽；使精疲力尽；彻底探讨
tight spot | 紧要关头
conceivably | adv. 可以想象的是，可以想得到的是；可能的是；可以理解的是
blurred | adj. 模糊不清的；记不清的；难以区分的
demarcation | n. 划分；划界；限界
trivial | adj. 不重要的，琐碎的；琐细的
