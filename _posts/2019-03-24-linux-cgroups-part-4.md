---
layout: post
title : "Control groups, part 4: On accounting"
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

**Original&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;:**&nbsp;&nbsp;[**Control groups, part 4: On accounting**](https://lwn.net/Articles/606004/)<br>
**Author&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;:**&nbsp;&nbsp;**Neil Brown**<br>
**Date&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;:**&nbsp;&nbsp;**July 23, 2014**<br>
**Signed-off-by:**&nbsp;&nbsp;**Norman Kern \<norman.gmx.com\>**

In our ongoing quest to understand Linux control groups, at least enough to enjoy the debates that inevitably arise when they are mentioned, it is time to explore the role that accounting plays and to consider how the needs of the accountant are met by the design and structure of cgroups.

Linux and Unix are not strangers to the idea of the accounting of resource usage. Even in V6 Unix, the CPU time for each process was accounted and the running total could be accessed with the times() system call. To a limited extent, this extended to groups of processes too. One of the process groupings we found when we first looked at V6 Unix was the group of all descendants of a given process. When all processes in that group have exited, the total CPU time used by them (or at least
those that haven't escaped) was available from times() as well. Before a process has exited and been waited for, its CPU time was known only to itself.

In 2.10BSD, the set of resources that were accounted for grew to include memory usage, page faults, disk I/O, and other statistics. Like CPU time, these were gathered into the parent when a child is waited for. They can be accessed by the getrusage() system call that is still available in modern Linux, largely unchanged.

With getrusage() came setrlimit(), which could impose limits on the use of some of these resources, such as CPU time and memory usage. These limits were only imposed on individual processes, never on a group: a group's statistics are only accumulated as processes exit, and that is rather too late to be imposing limits.

Last time, we looked at various cgroups subsystems that did not need to keep any accounting across the group to provide their services, though perf_event was in a bit of a grey area. This week, we look at the remaining five subsystems, each of which involve whole-group accounting in some way, and so support limits of a sort that setrlimit() could never support.

# cpuacct — accounting for the sake of accounting
cpuacct is the simplest of the accounting subsystems, in part because all it does is accounting; it doesn't exert control at all.

puacct appears to have originally been written largely as a demonstration of the capabilities of cgroups, with no expectation of mainline inclusion. It slipped into mainline with other cgroups code in 2.6.24-rc1, was removed with an explanation of this original intent shortly afterward, and then re-added well before 2.6.24-final because it was actually seen as quite useful. Given this history, we probably shouldn't expect cpuacct to fit neatly into any overall design.

Two separate sorts of accounting information are kept by cpuacct. First, there is the total CPU time used by all processes in the group, which is measured by the scheduler as precisely as it can and is recorded in nanoseconds. This information is gathered on a per-CPU basis and can be reported per-CPU or as a sum across all CPUs.

Second, there is (since 2.6.30) a breakdown into "user" and "system" time of the total CPU time used by all processes in the group. These times are accounted in the same way as the times returned by the times() system call and are recorded in clock ticks or "jiffies". Thus, they may not be as precise as the CPU times accounted in nanoseconds.

Since 2.6.29, these statistics are collected hierarchically. Whenever some usage is added to one group, it is also added to all of the ancestors of that group. So, the usage accounted in each group is the sum of the usage of all processes in that extended group including processes that are members of sub-groups.

This is the key characteristic of all the subsystems that we will look at this time: they account hierarchically. While perf_event does do some accounting, the accounting for each process stays in the cgroup that the process is an immediate member of and does not propagate up into ancestor processes.

For these two subsystems (cpuacct and perf_event), it is not clear that hierarchical accounting is really necessary. The totals collected are never used within the kernel, but are only made available to user-space applications, which are unlikely to read the data at a particularly high rate. This implies it would be quite effective for an application that needs whole-group accounting information to walk the tree of descendants from a given group and add up the totals in each
sub-group. When a cgroup is removed it could certainly be useful to accumulate its usage into its parent, much like process times are accumulated on exit. Earlier accumulation brings no obvious benefit.

Even if it were important to present the totals directly in the cgroups filesystem, it would be quite practical for the kernel to add the numbers up when required, rather than on every change. This is how the sum across all CPUs is managed, but it is not done for the sum across sub-groups.

There is an obvious tradeoff between the complexity for an application to walk the tree on those occasions when data is needed and the cost for the kernel in walking up the tree to add a new charge to every ancestor of the target process on every single accounting event. A proper cost/benefit analysis of this tradeoff would need to take into account the depth of the tree and the frequency of updates. For cpuacct, updates only happen on a scheduler event or timer tick, which would normally be
every millisecond or more on a busy machine. While this may seem a fairly high frequency, there are other events that can happen much more often, as we shall see.

Whether the accounting approach in cpuacct and perf_event involve sensible or questionable choices in not really important for understanding cgroups — what is worth noting is the fact that there are choices to be made and tradeoffs to be considered. These subsystems have freedom to choose because the accounting data is not used within the kernel. The remaining subsystems all keep account of something in order to exert control and, thus, need fully correct accounting details.

# Sharing out the memory
Two cgroup subsystems are involved in tracking and restricting memory usage: memory and hugetlb. These two use a common data structure and support library for tracking usage and imposing limits: the "resource counter" or res_counter.

A res_counter, which is declared in include/linux/res_counter.h and implemented in kernel/res_counter.c, stores a usage of some arbitrary resource together with a limit and a "soft limit". It also includes a high-water mark that records the highest level the usage has ever reached, and a failure count that tracks how many times a request for extra resources was denied.

Finally, a res_counter contains a spinlock to manage concurrent access and a parent pointer. This pointer demonstrates a recurring theme among the accounting cgroup subsystems in that they often create a parallel tree-like data structure that exactly copies the tree data structure provided directly by cgroups.

The memory cgroup subsystem allocates three res_counters, one for user-process memory usage, one for the sum of memory and swap usage, and one for usage of memory by the kernel on behalf of the process. Together with the one res_counter allocated by hugetlb (which accounts the memory allocated in huge pages), this means there are four extra parent pointers when the memory and hugetlb subsystems are both enabled. This seems to suggest that the implementation of the hierarchy structure
provided by cgroups doesn't really meet the needs of its users, though why that might be is not immediately obvious.

When one of the various memory resources is requested by a process, the res_counter code will walk up the parent pointers, checking if limits are reached and updating the usage at each ancestor. This requires taking a spinlock at each level, so it is not a cheap operation, particularly if the hierarchy is at all deep. Memory allocation is generally a highly optimized operation in Linux, with per-CPU free lists along with batched allocation and freeing to try to minimize the
per-allocation cost. Allocating memory isn't always a high-frequency operation, but sometimes it is; those times should still perform well if possible. So, taking a series of spinlocks for multiple nested cgroups to update accounting on every single memory allocation doesn't sound like a good idea. Fortunately, this is not something that the memory subsystem does.

When authorizing a memory allocation request of less than 32 pages (most requests are for one page), the memory controller will request that a full 32 be approved by the res_counter. If that request is granted, the excess above what was actually required is recorded in a per-CPU "stock" that notes which cgroup last made an allocation on each CPU and how much excess has been approved. If the request is not granted, it requests the actual number of pages allocated.

Subsequent allocations by the same process on the same CPU will use what remains in the stock until that drops to zero, at which point another 32-page authorization will be requested. If a scheduling change causes another process from a different cgroup to allocate memory on that CPU, then the old stock will be returned and a new stock for the new cgroup will be requested.

Deallocation is also batched, though with a quite different mechanism, presumably because deallocations often happen in much larger batches and because deallocations can never fail. The batching for deallocation uses a per-process (rather than per-CPU) counter that must be explicitly enabled by the code that is freeing memory. So a sequence of calls:
```c
	mem_cgroup_uncharge_start()
	repeat mem_cgroup_uncharge_page()
	mem_cgroup_uncharge_end()
```
will use the uncharge batching, while a lone mem_cgroup_uncharge_page() will not.

The key observation here is that, while a naive accounting of resource usage can be quite expensive, there are various ways to minimize the cost and different approaches will be more or less suitable in different circumstances. So it seems proper for cgroups to take a neutral stance on the issue and allow each subsystem to solve the problem in the way best suited to its needs.

# Another cgroup subsystem for the CPU
As the CPU is so central to a computer it is not surprising that there are several cgroup subsystems that relate to it. Last time, we met the cpuset subsystem that limits which CPUs a process can run on, and the cpuacct subsystem, above, which accounts for how much time is spent on the CPU by processes in a cgroup. The third and last CPU-related subsystem is simply called cpu. It is used to control how the scheduler shares CPU time among different processes and different cgroups.

The Linux scheduler has a surprisingly simple design. It is modeled on a hypothetical, ideal multi-tasking CPU that can run an arbitrary number of threads simultaneously, though at a proportionally reduced speed. Using this model, the scheduler can calculate how much effective CPU time each thread "should" have ideally received. The scheduler simply selects the thread for which the actual CPU time used is furthest behind the ideal and lets it run for a while to catch up.

If all processes were equal, then the proportionality would mean that if there are N runnable processes, each should get one-Nth of real time. Of course, processes often aren't all equal, as scheduling priority or nice values can assign different weights to each process so they get different proportions of real time. The sum of these proportions must, of course, add up to one (or to the number of active CPUs).

When the cpu cgroup subsystem is used to request group scheduling, these proportions must be calculated based on the group hierarchy, so some proportion will be given to a top-level group, and that is then shared among the processes and sub-groups in that group.

Further, the "virtual runtime", which tracks the discrepancy between ideal and actual CPU time, needs to be accounted for each group as well as for each process. One might expect the virtual runtime of a group to be exactly the sum of the times from the processes. However, when a process exits, any excess or deficit it has is lost. To prevent this loss from leading to unfairness between groups, the scheduler keeps account of the time used by each group as well as by each process.

To manage these different values across the hierarchy, the cpu subsystem creates a parallel hierarchy of struct sched_entity structures, which is what the scheduler uses to store proportional weights and virtual runtime. There are actually a multitude of these hierarchies, one for each CPU. This means that runtime values can be propagated up the tree without locking, so it is much more efficient than the res_counter used by the memory controller.

In accord with the recurring theme that one subsystem will often have two or more distinct (though related) functions, the cpu subsystem allows a maximum CPU bandwidth to be imposed on each group. This is quite separate from the scheduling priority discussed so far.

The bandwidth is measured in CPU time per real time. Both the CPU time limit, referred to as the quota, and the real time period during which that quota can be used must be specified. When setting the quota and period for each group, the subsystem checks that the limit imposed on any parent is always enough to allow all children to make full use of their quota. If that is not met, then the change will be rejected.

The actual implementation of the bandwidth limits is done largely in the context of the sched_entity. As the scheduler updates how much virtual time each sched_entity has used, it also updates the bandwidth usage and checks if throttling is appropriate.

To some extent, this case study simply reinforces some ideas we have already seen, that restrictions are often pushed down the hierarchy while accounting data is often propagated up a parallel hierarchy. We do see here one convincing reason why a parallel hierarchy might be needed. In this case, the parallel hierarchies are per-CPU so they can be updated without taking any locks.

# blkio - a final pair
As we have repeatedly observed, some cgroup subsystems manage multiple, though related, aspects of the contained processes. With blkio, this idea becomes more formalized. blkio allows for various "policies" to be registered that act much like cgroup subsystems in that they are advised of changes to the cgroup hierarchy and they can add files to the cgroup virtual filesystem. They are not able to disallow the movement of processes or get told about fork() or exit().

There are two blkio policies in Linux 3.15: "throttle" and "cfq-iosched". These have a remarkable similarity to the two functions of the cpu subsystem (bandwidth and scheduling priority), though the details are quite different. Many of the ideas we find in these two have already been seen in other subsystems, so there is little point going over similar details in a different guise. But there are two ideas worth mentioning.

One is that the blkio subsystem adds a new ID number to each cgroup. We saw last time that the cgroup core provides an ID number for each group and this is used by net_prio to identify groups. The new id added by blkio fills a similar role but with one important difference. blkio ID numbers use 64 bits and are never reused, while cgroup-core ID numbers are just an int (typically 32 bits) and are reused. A unique ID number seems like it could be a generally useful feature that the cgroups
core could provide. A little over a year after the blkio ID was added, a remarkably similar serial_nr was indeed added to the cgroup core, though blkio hasn't been revised to use it. Note when reading the code: blkio is known internally as blkcg.

The other new idea we find in blkio, and in the cfq-iosched policy in particular, is possibly more interesting. Each cgroup can be assigned a different weight, similar to the weights calculated by the CPU scheduler, to balance scheduling of requests from this group against requests from sibling groups. Uniquely to blkio, each cgroup can also have a leaf_weight that is used to balance requests from processes in this group against requests from processes in child groups.

The net effect of this is that when non-leaf cgroups contain processes, the cfq-iosched policy pretends that those processes are really in a virtual child group and uses leaf_weight to assign a weight to that virtual child. If we consider this against the different aspects of hierarchy that we explored earlier, it seems to be a clear statement from cfq-iosched that "organization" hierarchies are not something that it wants to deal with and that everything should be, or will be treated as,
"classification" hierarchies.

The CPU scheduler doesn't appear to be concerned about this issue. The processes in an internal group are scheduled against each other and against any child group as a whole. It isn't really clear which approach to scheduling is right, but it would be nice if they were consistent. One way to achieve consistency would be to forbid non-leaf cgroups from containing processes. There is work underway to exactly this end, as we will see later in this series.

# What can we learn?
If we combine all that we learned in this analysis with what we found with the first set of subsystems last time, it is easy to get lost in the details. Some differences may be deep conceptual differences, others may be important, but shallow, implementation differences, while still others could be pointless differences due to historical accident.

In the first class, I see a strong distinction between control elements that share out resources (CPU or block I/O bandwidth), those that impose limits on resources (CPU, block I/O, and memory), and the rest that are largely involved with identifying processes. The first set need a view of the whole hierarchy, as each branch competes with all others. The second set need to see only the local branch, as limits in one branch can not affect usage in another. The third set don't really need
the hierarchy at all — it might be useful, but its presence isn't intrinsic to the functionality.

The fact that several controllers create parallel hierarchies seems to be largely an implementation detail, though, as we will see next time, there may be a deeper conceptual issue underlying that.

The almost chaotic relationship between functionality and subsystems is most likely pointless historical accident. There is no clear policy statement concerning what justifies a new subsystem, so sometimes new functionality is added to an existing subsystem and sometimes it is provided as a brand new subsystem. The key issues that could inform such a policy statement are things we can be looking out for as our journey continues.

In the next installment, we will step back from all this fine detail and try to assemble a high level view of cgroups and their relationships. We will look at the hierarchical structures provided by cgroups and how those interact with the three core needs: sharing resources, limiting resource usage, and identifying processes.

# Appendix

英语 | 汉语
------------ | -------------
inevitably | adv. 不可避免地；必然地
exert | vt. 运用，发挥；施以影响
breakdown | n. 故障；崩溃；分解；分类；衰弱；跺脚曳步舞
recurring | adj. 循环的；再发的
stance | n. 立场；姿态；位置；准备击球姿势
hypothetical | adj. 假设的；爱猜想的
proportionality | n. 相称；均衡；比例性
deficit | n. 赤字；不足额
multitude | n. 大量，多数；群众，人群
reinforce | vt. 加强，加固；强化；补充
convincing | adj. 令人信服的；有说服力的
guise | n. 伪装；装束；外观
revise | vt. 修正；复习；校订
shallow | adj. 浅的；肤浅的
intrinsic | adj. 本质的，固有的
chaotic | adj. 混沌的；混乱的，无秩序的
