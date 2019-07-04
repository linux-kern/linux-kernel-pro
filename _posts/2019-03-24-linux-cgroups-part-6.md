https://lwn.net/Articles/606925/
Control groups, part 6: A look under the hood

I must confess that when I look under the hood of an automobile I can only just figure out which of the major components are which, and certainly wouldn't want to try changing the oil. Looking inside a sophisticated piece of software is quite a different story: even the smallest elements jump and dance around together like a scene from Disney's Fantasia. So, no tour of something as rich a Linux control groups could be complete without exploring the code to see what patterns we can find.

We have had a few little glimpses into several of the cgroup subsystems already, so this excursion is going to focus on the cgroups core and, particularly, on the interconnections between processes and cgroups. One question we want to keep in mind is the comparative costs of different approaches. Last time, we saw a concern that trying to use a single hierarchy and still allow independent classification against multiple resources could lead to a combinatorial explosion of groups. If
there are Q administrative groups, such as login sessions, and in each we potentially want to classify processes M different ways against each of N different resources, then we might need Q x M x N different groups. If we allow a separate hierarchy for each resource, there would only be Q + M x N groups. So the question is: does the simplicity of a single hierarchy outweigh the cost of the larger number of cgroups?

Managing the hierarchies themselves is unlikely to be particularly interesting — basic tree data structures are taught in most computer science courses and the shape of a cgroup hierarchy is unlikely to differ greatly from such structures. The interesting question is how processes are connected to cgroups. As we found when exploring the various cgroup subsystems, it is sometimes necessary to map from a process to the cgroup that is relevant for a particular subsystem, and it is sometimes
necessary to get the list of all processes that are in a cgroup. It is the data structures which enable those mappings that will be our main focus.

However, a clarification is in order. Cgroups are not actually groups of processes, despite the fact that we have repeatedly described them that way. They are in fact groups of threads. So we will start out by clarifying the distinction by looking at how threads and processes are connected to each other and to some related objects.

# Joining the dots ... er ... processes ... er ... threads
In V6 Unix, on through the early BSDs and several years of Linux, the process was a central, well-defined object. A process had a single thread of execution, an address space for memory, a process ID number, and a set of signal handlers, among other details. There was nothing else like a process and no room for confusion.

In V6 Unix (released in 1975), these processes where represented by a pre-allocated array of struct proc that was indexed by PID. If there was ever a need to find a process by some other key, such as by controlling tty, the code would simply search the whole array. This was clearly a simpler, less complicated age. By the time of 4.3BSD (1986), this fixed array had become a more dynamic linked list and in 4.4BSD there were even secondary lists so the processes in a process group could be found
without searching the whole process table. Since then, things have only become more complex, though there is still some room for simple elegance.

In Linux, the three-level hierarchy we first saw in 4.4BSD (session, process group, and process) has gained an extra level below processes: the thread. The result is a hierarchy something like that pictured to the right. A thread has its own execution context and its own "thread ID" number but can share most other details, particularly an address space, a set of signal handlers, and a process ID number, with other threads in the same process.

Internally, a thread is represented by a struct task_struct and is sometimes referred to as a task. Unfortunately, processes are also sometimes referred to as tasks — there is a do_each_pid_thread() macro that, quite reasonably, iterates over threads. The matching macro for iterating over processes is do_each_pid_task() (both macros are defined in pid.h). The term "process" is somewhat more reliable, but, since the term PID (or "process ID") is sometimes used for threads, processes, process
groups, and sessions, it is sometimes safer to stick with the more precise term "thread group".

It would be particularly elegant if all four of these objects, the session, process group, process, and thread, were managed in a uniform way. While that is nearly the case, threads are still handled a bit specially. While it may not justify all of the handling differences, there are two properties of threads that do make them truly different. First, there is always one distinguished thread within a process that is known as the "group_leader". This provides something clear to point to and say
"this is the process". The thread ID of the group_leader is the process ID of the whole process. A session does have a leader process, though only in a weak sense, as this process can exit before the rest of the session; process groups don't have any sort of leader at all. Secondly, a thread can only leave a process by exiting — it is not possible for a thread to move to a different process, unlike process groups where a process can move from one process group to another. This has important
implications for locking.

Since Linux gained the concept of "PID namespaces", where the same process can have a different PID in different namespaces, there has been a "struct pid" (pictured blue) that links the first three object types together. For this discussion the important member of struct pid is
```c
	struct hlist_head tasks[PIDTYPE_MAX];
```

    enum pid_type { PIDTYPE_PID, PIDTYPE_PGID, PIDTYPE_SID, PIDTYPE_MAX  };
    (and so is clearly a number of PIDTYPEs, not the MAXimum) along with the corresponding field (shown in yellow):
```c
	struct pid_link
	{
		struct hlist_node node;
		struct pid *pid;
			    
	} pids[PIDTYPE_MAX];
```
Each PID has three lists (the hlist_head) of task_structs, which are linked together through the three hlist_nodes in each task_struct. The session and process group lists (PIDTYPE_SID and PIDTYPE_PGID) are really lists of processes and only the thread group leaders appear in these lists.

The PIDTYPE_PID list is not, as one might hope, a list of all the threads in a thread group, but is a list of just the thread (if it still exists) with this PID as its thread ID. It seems strange to use a list to contain just a single entry, but there is a reason. When one thread in a process makes an exec() system call (in any of its flavors), all other threads in the process are killed (SIGKILL) and the thread performing exec() takes over the identity (particularly the PID) of the thread
group leader. This can result in two different threads having the same thread ID for a short time. Having a list for PIDTYPE_PID makes this possible.

The list of threads in a process is managed quite separately, using the field:
```c
	struct list_head thread_group;
```
The first three lists had a distinct "head", in the struct pid, and then a number of "nodes", one in each struct task_struct. This list of threads forms a loop (drawn in red) with no head or tail. This is problematic for a subtle, but important, reason that, perhaps surprisingly, would not be a problem for the other three lists.

Because threads are only removed from a thread group (aka process) when they are being destroyed, it is safe to walk the list without strong locking. The lightweight "RCU read lock" is sufficient. When a thread is removed from the list, it remains half-in for just long enough that any code in the middle of walking the list and currently looking at the thread being deleted can still continue on to reach the end — though, given that the list is a loop, we should really say that it reaches
the point in the loop where it started.

A potential problem with this picture is that if some code iterates over the thread list and starts from a thread which exits before the iteration finishes, it will never find its starting point and could loop forever. It is always safe to start from thread group leader (as it is remains a "zombie" until all other threads exit), but there is nothing in the API to force that starting point. One example of such questionable usage is in fill_stats_for_tgid() which accumulates some statistics
across all threads in a list. If this was requested for a PID that was not a thread group leader (which would be odd, but quite possible), it could hit problems if the thread exited at the wrong time. According to Oleg Nesterov (writing during Linux 3.14 development), "almost every lockless usage [of this thread linkage] is wrong".

Consequently, in Linux 3.14, a new linkage (not pictured) was introduced that has a distinct thread_head, which is stored in the signal_struct, that contains a number of fields that are specific to a process (as opposed to a thread). I find it a little disappointing that this head doesn't appear in struct pid like the other list heads, but that probably wouldn't have much practical benefit. In due course, all users should move away from the thread_group linkage, which can then be discarded.

This issue reminds us that locking is both subtle and important, so it must be understood properly. For all the lists we have looked at here, as well as the children/sibling list that links children of a process together, and the init_task/tasks list that links all processes together, there is a reader/writer spinlock, helpfully called tasklist_lock, that protects all accesses and changes. The only list accesses that are permitted without that lock are to follow the threads in a process
(preferably using the new thread_head list) and to iterate over all task group leaders starting from init_task. These are safe with only the RCU read lock as threads are never moved out of these lists into another.

There is some suggestion that this tasklist_lock has problems. One of the problems is that multiple overlapping readers can starve a process which needs to get write access. From Oleg Nesterov again (in a reply to this patchset): "Everyone seem to agree that tasklist[_lock] should die".

It would be an interesting exercise to make all the lists of processes and threads RCU-safe. This certainly wouldn't be trivial and, despite being proposed by Thomas Gleixner over four years ago, it still hasn't happened. However given that the VFS's directory entry (dentry) cache can often be accessed under RCU, it seems likely that something can be done for the process tree too.

# Cgroups linkages
These various linkages between threads and processes are made even more interesting by cgroups. The linkage to form cgroups into a hierarchy is fairly unsurprising (it is being rearranged substantially for 3.16, but the principle is unchanged):

```c
	struct list_head sibling;   /* my parent's children */
	struct list_head children;  /* my children */
	struct cgroup *parent;      /* my parent */
```
Much more interesting is the linkage between threads and cgroups. As noted previously, there can be multiple cgroup hierarchies and each thread belongs to one cgroup in each of them. This requires an M x N mapping, so a few lists won't do. The required mapping is achieved using two intermediate data structures: the css_set and the cgrp_cset_link.

When a process forks, the child will be in all the same cgroups that the parent is in. While either process could be moved around, they very often are not. This means that it is quite common for a collection of processes (and the threads within them) to all be in the same set of cgroups. To make use of this commonality, the struct css_set exists. It identifies a set of cgroups (css stands for "cgroup subsystem state") and each thread is attached to precisely one css_set. All css_sets are
linked together in a hash table so that when a process or thread is moved to a new cgroup, a pre-existing css_set can be reused, if one exists with the required set of cgroups.

With similar threads and processes linked into a css_set, we still have an M x N mapping problem, it is just that the M is no longer the number of threads, but is now the much smaller number of css_sets. To link the various cgroups to the various css_sets we have the aptly-named cgrp_cset_link:

```c
	struct cgrp_cset_link {
		struct cgroup       *cgrp;
		struct css_set      *cset;
		struct list_head    cset_link;
		struct list_head    cgrp_link;
	};
```
For each css_set, there is one cgrp_cset_link for every hierarchy. Each cgroup has a cset_links field that connects together all the cgrp_cset_links for that cgroup and from which all the threads in that cgroup can be found. Similarly each css_set has a cgrp_links field from which all the cgroups containing any of the threads in that css_set can be found (if all these list_heads are making your head spin, makelinux has a fairly good introduction to Linux linked lists).

This data structure quite effectively links all the threads and all the cgroups together. Whether it does so efficiently is a different question. It is quite good for finding all of the cgroups for a thread, or all the threads for a cgroup. It is far from suitable for finding the cgroup that provides, say, the net_cl subsystem for a particular process (in order to determine the class ID to assign to a new socket).

To meet this need there is some extra linkage (not shown in the picture above). Each css_set contains an array of pointers indexed by subsystem number that provides a direct link to the relevant cgroup for each subsystem, bypassing the cgrp_css_link structures.

```c
	struct cgroup_subsys_state *subsys[CGROUP_SUBSYS_COUNT];
```
(cgroup_subsys_state holds subsystem-specific information tightly connected to a cgroup).
If we turn now to the locking used to manage these lists, there are a couple of surprises. First, while there is a reader/writer lock that protects all of these linkages like there was with processes and tasks, this lock — css_set_rwsem — is a semaphore rather than a spinlock. This is surprising because semaphores are normally used when locks can be held for longer, so the waiting process might need to sleep (rather than just spin). Examining the history of this lock, it turns out that it
was a spinlock until quite recently and was changed to a semaphore as part of a process to incrementally tidy up some code. It seems likely that this will become a spinlock again.

The other surprise is more interesting: cgroups adds another lock that can stop threads from joining or leaving a thread group: group_rwsem. There is one of these locks for each thread group (stored in the signal_struct); it is rarely taken exclusively (i.e. for write), so it is unlikely to affect performance though, generally, avoiding new locks is best. It does raise a question, though: why do we need this new lock to protect the threads list?

The other surprise is more interesting: cgroups adds another lock that can stop threads from joining or leaving a thread group: group_rwsem. There is one of these locks for each thread group (stored in the signal_struct); it is rarely taken exclusively (i.e. for write), so it is unlikely to affect performance though, generally, avoiding new locks is best. It does raise a question, though: why do we need this new lock to protect the threads list?

The lock is needed because cgroups are groups of threads, rather than groups of processes. This means that when a process is added to a cgroup, each individual thread must be added separately, and it is necessary to keep the list of threads stable while they are being removed from one cgroup and added to another. If cgroups were truly groups of processes, there would be no need to move threads individually, and this lock could be discarded. The value of listing threads, rather than just
process (or group leaders), is not obvious in the code. An analysis of the various subsystems from the perspective of whether any could make use of being able to distinguish between different threads is left as an exercise for the interested reader.

# What did we learn?
Most of the value of looking into data structures is simply fleshing out a picture of how things plug together so that we can reason about the consequences of any possible changes. Nonetheless, there are a few specific lessons we can take away from this exploration.

First, we had a reminder that terminology can be challenging, and that we should be careful when interpreting what we read in the kernel code. A task is often a task, and a MAX is mostly a maximum. But not always.

Secondly, locking can be tricky. It is generally best to avoid requiring locks where possible, and doing so is easier when the data structures are simple and elegant. Reducing the impact of tasklist_lock is probably possible and apparently desirable. Reducing the impact of the cgroups locks, should that be necessary, would be harder simply because the data structures are more complex.

Finally, a key point of that complexity is the proliferation of multiply-linked cgroup_cset_link structures. There would be as many of these as there would be of cgroups — if we had a single hierarchy and created a multitude of cgroups, as discussed last time, to allow different combinations of different process classifications for different resources.

Put another way, the combinatorial explosion we were concerned about is unavoidable. It may be explicit in a multitude of cgroups, or it may be implicit in a multitude of cgroup_cset_link structures, but it will exist. Currently, cgroups are much bigger structures than the link structures, and cgroup_cset_links are created automatically, rather than requiring mkdir requests, so possibly having a proliferation of links is cheaper than a proliferation of cgroups.

This does suggest, though, that we could express the concern a different way. Rather than being concerned about a proliferation, we could instead be concerned about the size of cgroups and what mechanisms could be used to create cgroups automatically.

With that puzzle left as an open question, we come to the end of what we can discover about cgroups as they are found in Linux 3.15, the current release of Linux at the time of writing. We aren't quite finished yet though. The next and final installment will look beyond 3.15 to find out what value the proposed unified hierarchy might bring, and to try to present all the core issues into one coherent picture.


confess vt. 承认；坦白；忏悔；供认
automobile n. 汽车
sophisticated adj. 复杂的；精致的；久经世故的；富有经验的
glimpse n. 一瞥，一看
excursion n. 偏移；远足；短程旅行；离题；游览，游览团
comparative adj. 比较的；相当的
elegance n. 典雅；高雅
elegant adj. 高雅的，优雅的；讲究的；简炼的；简洁的
subtle adj. 微妙的；精细的；敏感的；狡猾的；稀薄的
aka abbr. 又叫做，亦称（also known as）
move around v. 走来走去；绕着……来回转
flesh out 充实，具体化
concern about 对…表示担心/忧虑；使（自己）关心…
