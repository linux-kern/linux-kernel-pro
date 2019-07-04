---
layout: post
title : "Control groups, part 5: The cgroup hierarchy"
author: Neil Brown
category: linux-cgroups
tags: linux kernel lwn cgroups
published: true
keywords: linux,kernel,lwn,cgroups
script: [post.js]
excerpted: |
    Having spent two articles exploring some background context concerning the grouping of processes and the challenges of hierarchies, we need to gain some understanding of control, and for that we will look at some details of the current (Linux 3.15) cgroups implementation.
#day_quote:
#  title: 
#  description: |
#    Put a very powerful message.
---

* Index
{: toc}

**Original&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;:**&nbsp;&nbsp;[**Control groups, part 5: The cgroup hierarchy**](https://lwn.net/Articles/606699/)<br>
**Author&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;:**&nbsp;&nbsp;**Neil Brown**<br>
**Date&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;:**&nbsp;&nbsp;**July 30, 2014**<br>
**Signed-off-by:**&nbsp;&nbsp;**Norman Kern \<norman.gmx.com\>**

> In earlier articles, we have looked at hierarchies in general and at how hierarchy is handled by specific cgroup subsystems. Now, it is time to draw all of this together to try to understand what sort of hierarchy or hierarchies are needed and how this can be supported in the current implementation. As was recently reported, the 3.16 Linux kernel will have under-development support for a so-called "unified hierarchy". The new ideas introduced with that development will not be discussed
> yet, as we cannot really appreciate what value they might bring until we fully understand what we have. A later article will unpack the unified hierarchy, but for now we will start by understanding what might be called the "classic" cgroup hierarchies.

# Classic cgroup hierarchies
In the classic mode, which may ultimately be deprecated, but is still fully supported, there can be several separate cgroup hierarchies. Each hierarchy starts its life as a root cgroup, which initially holds all processes. This root node is created by mounting an instance of the "cgroup" virtual filesystem and all further modifications to the hierarchy happen through manipulations of this filesystem, particularly mkdir to create cgroups, rmdir to remove cgroups and mv to rename a
cgroup within the same parent. Once the cgroups are created, processes can be moved between them by writing process id numbers into special files. When a suitably privileged user writes a PID number to cgroup.procs in a cgroup, that process is moved, from the cgroup it currently resides in to the target cgroup.

This is a very "organizational" way to manipulate a hierarchy: create a new group and find someone to fill it. While this may seem natural for a filesystem-based hierarchy, we shouldn't assume it is the best way to manipulate all hierarchies. The simple hierarchy of sessions and process groups that we found in 4.4BSD works quite differently. There is no distinction between creating a group and putting the first process in the group.

We should assess the mechanisms for manipulation by considering whether they are fit-for-purpose, as well as whether they are convenient to implement. If the goal is to allow processes internal to the hierarchy, this mechanism is quite suitable. If we were to prefer to keep all processes in the leaves, it doesn't seem so well suited.

When a hierarchy is created, it is associated with a fixed set of cgroup subsystems. The set can be changed, but only if the hierarchy has no subgroups below the root, so, for most practical purposes, it is fixed. Each subsystem can be attached to at most one hierarchy, so, from the perspective of any given subsystem, there is only one hierarchy, but no assumptions can be made about what other subsystems might see.

Thus it is possible to have 12 different hierarchies, one for each subsystem, or a single hierarchy with all 12 subsystems attached, or any other combination in between. It is also possible to have an arbitrary number of hierarchies each with zero subsystems attached. Such a hierarchy doesn't allow any control of the processes in the various cgroups, but it does allow sets of related processes to be tracked.

Systemd makes use of this feature by creating a cgroup tree mounted at /sys/fs/cgroup/systemd with no controller subsystems. It contains a user.slice sub-hierarchy that classifies processes that result from login sessions, first by user and second by session. So:
```c
	/sys/fs/cgroup/systemd/user.slice/user-1000.slice/session-1.scope
```
represents a cgroup that contains all the processes associated with the first login session for the user with UID 1000 (slice and scope are terms specific to systemd).

These "session scopes" seem to restore one of the values of the original process groups in V7 Unix — a clear identification of which processes belong to which login session. This probably isn't very interesting on a single-user desktop, but could be quite valuable on a larger, multi-user machine. While there is no direct control possible of the group, we can tell exactly which processes are in it (if any) by looking at the cgroup.procs file. Provided the processes aren't forking too
quickly, you could even signal all the processes with something like:
```c
	kill $(cat cgroup.procs)
```
# The tyranny of choice
Probably the biggest single problem with the classic approach to hierarchy is the tyranny of choice. There seems to be a lot of flexibility in the different ways that subsystems can be combined: some in one hierarchy, some in another, none at all in a third. The problem is that this choice, once made, is system-wide and difficult to change. If one need suggests a particular arrangement of subsystems, while another need suggests something different, both needs cannot be met on the same host.
This is particularly an issue when containers are used to support separate administrative domains on the one host. All administrative domains must see the same associations of cgroup subsystems to hierarchies.

This suggests that a standard needs to be agreed upon. Obvious choices are to have a single hierarchy (which is where the "unified hierarchy" approach appears to be headed) or a separate hierarchy for each subsystem (which is very nearly the default on my openSUSE 13.1 notebook: only cpu and cpuacct are combined). With all that we have learned so far about cgroup subsystems, we might be able to understand some of the implications of keeping subsystems separate or together.

As we saw, particularly in part 3, quite a few subsystems do not perform any accounting or, when they do, do not use that accounting to impose any control. These are debug, net_cl, net_perf, device, freezer, perf_event, cpuset, and cpuacct. None of these make very heavy use of hierarchy and, in almost all cases, the functionality provided by hierarchy can be achieved separately.

A good example is the perf_event subsystem and the perf program that works with it. The perf tool can collect performance data for a collection of processes and provides various ways to select those processes, one of which is to specify the UID. When a UID is given, perf does not just pass this to the kernel to ask for all matching processes, but rather examines all processes listed in the /proc filesystem, selects those with the given UID, and asks the kernel to monitor each of those
independently.

The only use that the perf_event subsystem makes of hierarchy is to collect subgroups into larger groups, so that perf can identify just one larger group and collect data for all processes in all groups beneath that one. Since the exact same effect could be achieved by having perf identify all the leaf groups it is interested in (whether they are in a single larger group or not) in a manner similar to its selection of processes based on UID, the hierarchy is really just a minor convenience
— not an important feature. For similar reasons, the other subsystems listed could easily manage without any hierarchy.

There are two uses of hierarchy among these subsystems that cannot be brushed away quite so easily. The first is with the cpuset subsystem. It will sometimes look upward in the hierarchy to find extra resources to use in an emergency. This feature is an intrinsic dependency on hierarchy. As we noted when we first examined this subsystem, similar functionality could easily be provided without depending on hierarchy, so this is a minor exception.

The other use is most obvious in the devices subsystem. It relates not to any control that is imposed but to the configuration that is permitted: a subgroup is not permitted to allow access that its parent denies. This use of hierarchy is not for classifying processes so much as for administrative control. It allows upper levels to set policy that the lower levels must follow. An administrative hierarchy can be very effective at distributing authority, whether to user groups, to individual
users, or to containers that might have their own sets of users. Having a single administrative hierarchy, possibly based on the one that systemd provides by default, is a very natural choice and would be quite suitable for all these non-accounting subsystems. Keeping any of them separate seems to be hard to justify.

# Network Traffic Control — another control hierarchy
The remaining subsystems, which are the ones most deserving of the term "resource controllers", manage memory (including hugetlb), CPU, and block-I/O resources. To understand these it it will serve us to take a diversion and look at how network resources are managed.

Network traffic can certainly benefit from resource sharing and usage throttling, but we did not see any evidence for network resource control in our exploration of the different cgroup subsystems, certainly not in the same way as we did for block I/O and CPU resources. This is particularly relevant since one of the documented justifications for multiple hierarchies, as mentioned in a previous article, is that there can be a credible need to manage network resources separately from, for
example, CPU resources.

Network traffic is in fact managed by a separate hierarchy, and this hierarchy is even separate from cgroups. To understand it we need at least a brief introduction to Network Traffic Control (NTC). The NTC mechanism is managed by the tc program. This tool allows a "queueing discipline" (or "qdisc") to be attached to each network interface. Some qdiscs are "classful" and these can have other qdiscs attached beneath them, one for each "class" of packet. If any of these secondary qdiscs are
also classful, a further level is possible, and so on. This implies that there can be a hierarchy of qdiscs, or several hierarchies, one for each network interface.

The tc program also allows "filters" to be configured. These filters guide how network packets are assigned to different classes (and hence to different queues). Filters can key off various values, including bytes within the packet, the protocol used for the packet, or — significant to the current discussion — the socket that generated the packet. The net_cl cgroup subsystem can assign a "class ID" to each cgroup that is inherited by sockets created by processes in that cgroup, and this
class ID is used to classify packets into different network queues.

Each packet will be classified by the various filters into one of the queues in the tree and then will propagate up to the root, possibly being throttled (for example by the Token Bucket Filter, tbf, qdisc) or being competitively scheduled (e.g. by the Stochastic Fair Queueing, sfq, qdisc). Once it reaches the root, it is transmitted.

This example emphasizes the value in having a hierarchy, and even a separate hierarchy, to manage scheduling and throttling for a resource. It also shows us that it does not need to be a separate cgroup hierarchy. A resource-local hierarchy can fit the need perfectly and, in that case, a separate cgroup hierarchy is not needed.

Each of the major resource controllers, for CPU, memory, block I/O, and network I/O, maintain separate hierarchies to manage their resources. For the first three, those hierarchies are managed through cgroups, but for networking it is managed separately. This observation might suggest that there are two different sorts of hierarchies present here: some for tracking resources and some (possibly one "administrative hierarchy") for tracking processes.

The example in Documentation/cgroups/cgroups.txt does seem to acknowledge the possibility of a single hierarchy for tracking processes but worries that it "may lead to [a] proliferation of ... cgroups". If we included the net_cl subsystem in the systemd hierarchy described earlier, we would potentially need to create several sub-cgroups in each session for the different network classes that might be required. If other subsystems (e.g. cpu or blkio) wanted different classifications within
each session, a combinatorial explosion of cgroups could result. Whether or not this is really a problem depends on internal implementation details, so we will delay further discussion of this until the next article, which focuses on exactly that subject.

One feature of cgroups hierarchies that is not obvious in the NTC hierarchies is the ability to delegate part of the hierarchy to a separate administrative domain when using containers. By only mounting a subtree of a cgroup's hierarchy in the namespace of some container, the container is limited to affecting just that subtree. Such a container would not, however, be limited in which class IDs can be assigned to different cgroups. This could appear to circumvent any intended isolation.

With networking, the issue is resolved using virtualization and indirection. A "veth" virtual network interface can be provided to the container that it can configure however it likes. Traffic from the container is routed to the real interface and can be classified according to the container it came from. A similar scheme could work for block I/O, but CPU or memory resource management could not achieve the same effect without full KVM-like virtualization. These would require a different
approach for administrative delegation, such as the explicit sub-mount support that cgroups provides.

# How separate is too separate?
As we mentioned last time, the accounting resource controllers need visibility into the ancestors of a cgroup to impose rate limiting effectively and need visibility into the siblings of a cgroup to effect fair sharing, so the whole hierarchy really is important for these subsystems.

If we take the NTC as an example, it could be argued that these hierarchies should be separate for each resource. NTC takes this even further than cgroups can, by allowing a separate hierarchy for each interface. blkio could conceivably want different scheduling structures for different block devices (swap vs database vs logging), but that is not supported by cgroups.

There is, however, a cost in excessive separation of resource control, much as there is (according to some) a cost in the separation of resource management as advocated for micro-kernels. This cost is the lack of "effective co-operation" identified by Tejun Heo as part of the justification for a unified hierarchy.

When a process writes to a file the data will first go into the page cache, thus consuming memory. At some later time, that memory will be written out to storage thus consuming some block-I/O bandwidth, or possibly some network bandwidth. So these subsystems are not entirely separate.

When the memory is written out, it will quite possibly not be written by the process that initially wrote the data, or even by any other process in the same cgroup. How, then, can this block-I/O usage be accounted accurately?

The memory cgroup subsystem attaches extra information to every page of memory so that it knows where to send a refund when the page is freed. It seems like we could account the I/O usage to this same cgroup when the page is eventually written, but there is one problem. That cgroup is associated with the memory subsystem and so could be in a completely different hierarchy. The cgroup used for memory accounting could be meaningless to the blkio subsystem.

There are a few different ways that this disconnect could be resolved:

1. Record the process ID with each page and use it to identify what should be charged for both memory usage and block-I/O usage, as both subsystems understand a PID. One problem would be that processes can be very short lived. When a process exits, we would need to either transfer its outstanding resource charges to some other process or a cgroup, or to just discard them. This is similar to the issue we saw in the CPU scheduler, where accounting just to a process would not easily lead to proper
fairness for process groups. Preserving outstanding charges efficiently could be a challenge.
2. Invent some other identifier that can safely live arbitrarily long, can be associated with multiple processes, and can be used by each different cgroup subsystem. This is effectively the "extra level of indirection" that proverbially can solve any problem in computer science.
The class ID that connects the net_cl subsystem with NTC is an example of such an identifier. While there can be multiple hierarchies, one for each interface, there is only a single namespace of class ID identifiers.

3. Store multiple identifiers with each page, one for memory usage and one for I/O throughput.
The struct page_cgroup structure that is used to store extra per-page information for the memory controller currently costs 128 bits per page on a 64bit system — 64 bits for a pointer to the owning cgroup and 64 bits for flags, 3 of which are defined. If an array index could be used instead of a pointer, and a billion groups were deemed to be enough, two indexes and an extra bit could be stored in half the space. Whether an index could be used with sufficient efficiency is another exercise
left for the interested reader.

A good solution to this problem could have applicability in other situations: anywhere that one process consumes resources on behalf of another. The md RAID driver in Linux will often pass I/O requests directly down to the underlying device in the context of the process that initiated the request. In other cases, some work needs to be done by a helper process that will then submit the request. Currently, the CPU time to perform that work and the I/O throughput consumed by the request are
charged to md rather than to the originating process. If some "consumer" identifier or identifiers could be attached to each I/O request, md and other similar drivers would have some chance of apportioning the resource charges accordingly.

Unfortunately, no good solution to this problem exists in the current implementation. While there are costs to excessive separation, those costs cannot be alleviated by simply attaching all subsystems to the same hierarchy.

In the current implementation, it seems best to keep the accounting subsystems, cpu, blkio, memory, and hugetlb, in separate hierarchies, to acknowledge that networking already has a separate hierarchy thanks to the NTC, and to keep all the non-accounting subsystems together in an administrative hierarchy. These will depend on intelligent tools to effectively combine separate cgroups when needed.

# Answers ...
We are now in a position to answer a few more of the issues that arose in an earlier article in this series. One was the question of how groups are named. As we saw above, this is is the responsibility of whichever process initiates the mkdir command. This contrasts with job control process groups and sessions, for which the kernel assigns a name in an (almost) arbitrary way when a process calls setsid() or setpgid(0,0). The difference may be subtle, but it does seem to say something
about the expected authority structures. For job control process groups, the decision to form a new group comes from within a member of the new group. For cgroups, the decision is expected to come from outside. Earlier, we observed that embodying an administrative hierarchy in the cgroups hierarchy seemed to make a lot of sense. The fact that names are assigned from outside aligns with that observation.

Another issue was whether it was possible to escape from one group into another. Since moving a process involves writing the process ID to a file in the cgroup filesystem, this can be done by any process with write access to that file using normal filesystem access checks. When a PID is written to that file, there is a further check that the owner of the process performing the write is also the owner of the process being added, or is privileged. This means that any user can move
any of their processes into any group where they have write access to cgroup.procs, irrespective of how much of the hierarchy that crosses.

Put another way, we can restrict where a process is moved to, but there is much less control over where it can be moved from. A cgroup can only be considered to be "closed" if the owners of all processes in it are barred from moving a process into any cgroup outside of it. Like the hierarchy manipulations looked at earlier, these make some sense when thinking about the cgroup hierarchy as though it were a filesystem, but not quite as much when thinking about it as a classification scheme.

# ... and questions
The biggest question to come out of this discussion is whether there is a genuine need for different resources to be managed using different hierarchies. Is the flexibility provided by NTC well beyond need or does it set a valuable model for others to follow? A secondary question concerns the possibility for combinatorial explosion if divergent needs are imposed on a single hierarchy and whether the cost of this is disproportionate to the value. In either case, we need a clear
understanding of how to properly charge the originator of a request that results in some service process consuming any of the various resources.

Of these questions, the middle one is probably the easiest: what exactly are the implementation costs of having multiple cgroups? So it is to this topic we will head next time, when we look at the various data structures that hook everything together under the hood.

# Appendix

英语 | 汉语
------------ | -------------
assess | vt. 评定；估价；对…征税
tyranny | n. 暴政；专横；严酷；残暴的行为（需用复数）
beneath | prep. 在…之下
brush | vt. 刷；画；
intrinsic | adj. 本质的，固有的
deserving | adj. 值得的；应得的；有功的
diversion | n. 转移；消遣；分散注意力
credible | adj. 可靠的，可信的
discipline | n. 学科；纪律；训练；惩罚
stochastic | adj. [数] 随机的；猜测的
proliferation | n. 增殖，扩散；分芽繁殖
combinatorial | adj. 组合的
circumvent | v. 包围；智取；绕行，规避
conceivably | adv. 可以想象的是，可以想得到的是；可能的是；可以理解的是
advocate | vt. 提倡，主张，拥护
refund | vt. 退还；偿还；付还
proverbially | adv. 人尽皆知地
apportion | vt. 分配，分派；分摊
alleviate | vt. 减轻，缓和
irrespective | adj. 不考虑的，不顾的
explosion | n. 爆炸；爆发；激增
divergent | adj. 相异的，分歧的；求异的，发散的；散开的
disproportionate | adj. 不成比例的
