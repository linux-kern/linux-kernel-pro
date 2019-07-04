Control groups, part 7: To unity and beyond
https://lwn.net/Articles/608425/

The original justification for this series was so that we all could understand Linux control groups enough that we might be able to enjoy the occasional debates that crop up around them. It is now time to see how that worked — to see how well you, or I, can assess some proposal or challenge in the context of all the issues that appear to surround control groups. In this, the final installment of the series, we have two opportunities to test our new skills.

An obvious first target is the "unified hierarchy" that is available as a developer preview in Linux 3.16, which was covered recently on these pages. If you haven't done so already, now might be a good time to go back and re-read the article to see whether (and how) various issues we have found are being addressed. If you want to be thorough, read the unified-hierarchy.txt documentation too.

It might help to start by writing down a list of the issues that seemed important to you. It can also help to list some design patterns, or anti-patterns, to be on the lookout for. Four that I find helpful were identified in a previous series on the "Ghosts of Unix Past": full exploitation, conflated designs, unfixable designs, and high maintenance designs, all of which are summarized at the end of that last article.

# The unified hierarchy: A score card
Having identified your key issues and arrived at your conclusions, you will no doubt want to either have them affirmed, or have an opportunity to defend them. To meet this need, I present some of my own conclusions.

Unification of hierarchy

It is nearly indisputable that the number of hierarchies allowed by classic cgroups is excessive. It is less clear that reducing the number to one is ideal. In our investigations we found very different uses of hierarchy: some subsystems imposed control downward, others collected accounting upward. These are very different uses involving different implementation concerns. It is arguable that they justify distinct hierarchies.

The unified hierarchy is clearly working toward the removal of excessive duplication, which is good. It doesn't seem to acknowledge that different subsystems might genuinely have incompatible needs, but then it hasn't completely closed the door to separate hierarchies yet. So this aspect deserves a B — good work, but room for improvement.

Processes only permitted in the leaves

The unified hierarchy requires that processes only exist in the leaves of the tree. The enforcement approach for this is somewhat clumsy. Leaves are "any node that doesn't extend any subsystem to children" and there is a two-step dance when creating a new level in the hierarchy. Processes must be moved down first, then subsystems can be extended down afterward.

This complexity achieves an end result that is already possible anyway (system administrators and tools could easily choose to keep processes in leaves) and, thus, is largely uninteresting. It's not clear that the kernel needs to enforce a sane policy of processes-only-in-leaves any more than it should enforce the sane policy that the filesystem root be read-only to most users.

I was going to give this issue a C (too complex), but there is a wart on the design that should be highlighted. Processes are excluded from internal cgroups except for the root cgroup, apparently because the root needs "special treatment". This exception actually leads to a score of C+ for reasons which will become apparent later.

Taming the chaotic subsystems

We have seen that the correlation between cgroup subsystems and elements of functionality is rather chaotic. This is not a new observation: at the 2011 Kernel Summit, Paul Turner was reported as saying that:

Google would, based on its experience, rip apart a lot of the controllers and rework them in a better form.
While that sort of rewrite may be too much, it would be nice if we could de-emphasize the current division into subsystems in the hope that more meaningful groupings could emerge, possibly between the control of processes and the control of other resources. The unified hierarchy seems well-placed to advance this need, but unfortunately goes in the opposite direction. Lists of subsystems now appear throughout the cgroups filesystem in the cgroup.controllers and cgroup.subtree_control
files. It is true that attribute files are already named after their subsystem, but having a freezer.state file makes sense whether "freezer" is a separate subsystem or just an element of functionality.

Explicitly listing enabled subsystems in cgroup.controllers effectively entrenches the current structure, so this issue gets a D from me.

Providing a resource-consumer ID

We saw in part five that pages in memory can identify who gets the refund when the memory is freed, but not who gets charged for IO when the content is written out. By insisting that all subsystems use a single hierarchy, a single cgroup can serve as a resource consumer ID for all resources types. This is clearly a solution to the problem, but it is hard to tell if it is a good solution (different resources may be very different) so I'm reserving judgment for now and only giving a B.

Processes or threads

Classic cgroups allows individual threads within a process to be in different cgroups. Imagining a credible use case for this is difficult, but not quite impossible.

The cpuset controller can restrict processes to a set of CPUs and, separately, to a set of memory nodes in a NUMA system. The former restriction can be imposed on any thread using the sched_setaffinity() system call or the taskset program, without involving cgroups. But the set of memory nodes can only be configured through cgroups. Imposing different memory nodes on different threads (which share one address space) doesn't make much sense, so that doesn't justify cgroups per thread, but there
are other values that can only be set through cgroups.

The Linux scheduler allows the priority of a thread to be set with much finer granularity than the traditional 40 point "nice" scale. It allows each thread or group to have a "weight" which ranges up to about 100,000 (weight = 1024 * 0.8nice, approximately). This weight can only be set using cgroups. If you want that fine control of individual threads, you need threads in cgroups.

These are both examples of what I would call the "procfs problem". The procfs filesystem became a way for various ad hoc functionality to be added to the kernel with minimal design review, because there were no clear design guidelines. Consequently, it handles much more than processes. Similarly, cgroups seems to allow "back door" access for functionality that is not at all specific to the control of groups of processes. If the only use for threads-in-cgroups is to benefit from these back
doors, then disallowing them might encourage better API design.

The unified hierarchy does exactly this and only allows processes (also known as thread groups) to be in different cgroups. This seems like a good idea but does raise a question: what exactly should we try to control? Threads? Processes? Something else? Whatever the answer, dropping support for moving individual threads seems like a good idea, so this gets an A.

Code simplicity

The unified hierarchy is only one step in what could be a long process. There have been a lot of improvements in the code leading up to the current state, but the full value of the changes won't be fully realized until some old functionality can be removed. When that might be is unknown.

What we do know is that only processes (not threads) will ultimately need to be in cgroups and they will only need to be in a single cgroup each. This will certainly bring simplicity, so an A is clearly in order.

Summary

The less-than-stellar scores assigned above probably have several causes, not least my own personal bias. The most significant single cause is almost certainly the foundation on which the unified hierarchy is being built. Like many first implementations, cgroups really isn't very good: The role of hierarchy and the purpose of subsystems are at best confused. If a sow's ear is all you have, a silk purse is really too much to ask for.

One of the premises of the unified hierarchy is that we have to stay with control groups in some form. Tejun Heo would have preferred a different structure layered "over the process tree like sessions or program groups", but mourned "that ship sailed long ago". A little over a year earlier, something happened which has implications that might not be so melancholy.

# Auto-group Scheduling
As was reported in late 2010, there are ways other than cgroups to control groups of processes. Using the group scheduling support that was developed for cgroups, Mike Galbraith created a different, automatic mechanism to group processes together for scheduling.

The standard Unix scheduler, and most successors, attempts to be fair to processes, but processes aren't necessarily the best focus for fairness. On the AUSAM Unix variant I used as a student (Australian Unix Share Accounting Method, which evolved into SHARE II), fairness was aimed at users first, so that one student running six processes (the local limit at the time) would not get more CPU time than another student running only one. On a modern developer's desktop, the "job" (in the
job-control sense — a process group) is a very logical grouping. Different jobs (browser, game, make -j 40) could reasonably compete against each other on an equal footing, and processes or threads within a job should reasonably compete against each other, but not, as individuals, against other threads.

There are two issues with automatic scheduling using process groups that were raised in the mailing list thread that records the history of auto-group scheduling. The issues were raised by very different people and received very different responses.

The first, raised by Linus Torvalds, is a suggestion that process groups are too fine-grained for this purpose. Creating a new scheduling group does have some cost, so doing it too often could introduce unacceptable slowness. Unfortunately there is no record of anyone measuring the cost (despite some encouragement from Linus) and only a vague assessment of what constituted "too often" — somewhere between "one for every command invocation in the shell" and "tens of thousands of times a
second".

This claim was never really challenged. The final implementation used "sessions" rather then "process groups", which certainly do get created less often. However, this doesn't really seem like the right grouping. If you run:
```sh
	make -j 40 >& log &
```
to compile your project, and then frozen-bubble to pass the time — both from the same terminal window — your game will compete with 40 processes instead of with one job.

t is fairly easy to test the overhead that a fork()+exec() suffers if a scheduler group is also created: /bin/env /bin/echo hello and /bin/setsid /bin/echo hello will do exactly the same things except the latter creates a new session and hence a new scheduler group (if both are run from a shell script, not from an interactive shell):
```sh
	time bash -c 'for i in {1..10000}; do /usr/bin/setsid /bin/echo hi ; done > /dev/null'
	time bash -c 'for i in {1..10000}; do /usr/bin/env /bin/echo hi ; done > /dev/null'
```
The difference between those two is certainly in the noise.
The second issue was raised by Lennart Poettering: "On the desktop this is completely irrelevant." At the time this claim was made, it was true to a substantial extent, because auto-grouping was being done based on "controlling tty" and most desktop applications would equally have no controlling tty. A video editor and a browser would be in the same scheduling group, so the multiple rendering threads used by one could swamp the single thread used by the other. By the end of the
discussion, it was again true to a different extent. Auto-grouping was now done based on "sessions", and most desktop session managers did not put each application into a different session. One session manager that was under development did: systemd already used setsid() as required.

Despite the fact that Lennart's comments were not well received, he was at that time working on software that could easily bring the benefits of auto-group scheduling to a larger group of users. No one seemed to realize that.

But, back to the main story, the key lesson from auto-group scheduling is that the cgroups effort inspired some useful functionality in the scheduler, and this functionality can be used quite separately from cgroups. When a process is in a non-root cgroup (from the perspective of the cpu subsystem), it is scheduled as directed by cgroups. When it is in the root, it is scheduled according to auto-groups (unless auto-groups has been disabled). This is why it is a positive that the
unified hierarchy allows processes to remain in the root of the hierarchy even when the root is no longer a leaf. It means that the way is left open for independent resource management to be developed in parallel to cgroups, and for both cgroups and non-cgroups management to happen on the same system. This leads to the second challenge.

We have something that the original cgroups developers didn't start with: years of experience and working code. This is a wealth that we should be able to turn to our advantage. So, to test your new-found understanding of resource management, the challenge is this: inspired by auto-groups for scheduling, how would you implement resource management and process control in Linux alongside, but independently of, cgroups? Once you've thought that through, you can come back and compare your
results to mine. Don't worry, we'll still be here when you're done.

# Hindsight groups: highlighting some issues through contrast.
Contrast is a powerful tool for helping us see things more clearly. So to present the issues that I found to be important, I've embedded them in a different context. Hindsight groups, a name which reflects their origin, are sometimes different to make a point, and sometimes different just to be different. Hindsight groups are focused: they are only about restricting groups of processes. Any need that doesn't match that description needs to seek a home elsewhere.

In hindsight groups (or "hgroups"), the base unit of control is the process group, as created by interactive shells, by systemd, and, potentially, by any other session manager. Control can still be imposed on individual processes using prlimit() or similar commands, but controlling groups has no finer granularity than the process group.

To provide a management structure for these process groups, a new level in the PID hierarchy is added. A "process domain" is introduced above sessions and process groups. Processes are initially in domain zero. A process that is in domain zero and is alone in its session and its process group can call set_domainid() to start a new domain that is subordinate to domain zero, thus creating a two-level hierarchy of domains. When a new PID namespace is created, the domain containing the
starting process appears as domain zero in the new namespace and new domains in that namespace are subordinate to the local domain zero, thus establishing a multi-level hierarchy.

The hierarchy formed by domains strongly constrains processes. Once inside a domain, a process cannot get out of that domain. Each domain is associated with a process group — the process group of the process that created it. All other process groups in the same domain are considered to be subordinate to that first process group. This effectively places all process groups into a hierarchy. It is very much an organizational hierarchy, rather than a classification hierarchy. It provides
structural groupings like "login session" or "container" or "job". It collects processes based on the task they perform more than the way they behave.

With this new, more strongly defined role for process groups comes a new data structure that is allocated per-process-group, much like a signal_struct is allocated per-process. It contains a set of restrictions that apply to processes in the group. Some of these, like an access control list of devices (similar to that provided by the devices cgroup subsystem), are referred to whenever the process needs to check if something is permitted and cgroups is not configured or provides only the "root"
cgroup. Others, like a set of CPUs that may be used, need to be pushed out to all processes and threads in the process group whenever they change. This is uniformly done by sending a virtual signal to all processes (similar to the approach taken by the freezer cgroup subsystem). During handling of that virtual signal, a process will update its local understanding based on restrictions in the process group.

The per-process-group restrictions can be changed by any process that has an appropriate user ID or has superuser permissions. However, a process can only give extra permissions (i.e. reduce restrictions) that its own process group has, and that every process group above it in the hierarchy has. Changes are not propagated down by the kernel, but a user-space tool can propagate the lifting or imposing of restrictions reliably.

One important restriction identifies certain actions that processes cannot perform, and causes them to block if they try. One setting of this restriction effectively freezes all processes in the group, much like the cgroups freezer. Another setting only freezes a process when it tries to create a new process group. This makes it possible to impose some restriction on all process groups in a domain in a race-free way.

The various shared resources: memory, CPU, network, and block I/O, each have specific needs and are managed separately. They benefit from these groupings and this hierarchy, but they are not tied to it.

Networking and block I/O have some similarities, as they generally involve capping or sharing data throughput. They are also quite easily virtualized, so that a sub-domain can be given access to a virtual device that routes data to and from a real device. They can have multiple separate devices to manage and have other concerns beyond just the process that is involved. The network system needs to manage its own link-control traffic and possibly traffic forwarded from another interface. The
block-I/O subsystem already makes an internal distinction between metadata (using the REQ_META flag) and other data, and so needs to classify requests in different ways.

Consequently, these two systems have their own queuing management structures and are not known to hgroups. The various queuing algorithms may classify requests based on the originating domain, or they may support some labeling of individual processes (similar to the cgroups network classes subsystem), but that is beyond the interest of hgroups.

Memory usage management is quite different from the other shared resources because it is measured in space more than in time. With the other three (network, block I/O, CPU) a process can start or stop using the resource at any moment or can be temporarily barred from the resource with no ill effects. With memory, the resource is useless unless it is constantly available for some non-trivial period of time.

his means, as we saw in an earlier installment, that memory must be charged to some entity that persists for quite a while. It also means that it is difficult to impose proportional sharing. The cgroups memory controller imposes two limits: a hard limit that must not be exceeded and a soft limit that is only imposed when memory is very tight. Varying the soft limit doesn't really affect the proportion of sharing, but instead affects the proportion of pain imposed when memory must be freed.

These twin needs of persistence and imposing restrictions is met perfectly by process domains, and they serve much the same role as cgroups do in the hierarchy used by the mem subsystem. The memory resources used in each process group are charged to the containing domain, and to that domain's containing domain if there is one. If any limit is reached the allocation fails and memory reclaim is instigated. There are hard and soft limits just as with cgroups.

It is possible for a privileged process to redirect the memory accounting for any process group in a subordinate domain so that usage within that process group is charged to some other domain instead. This can be used, for example, to cause all domains belonging to a given user to have a single overall memory limit imposed, even though the primary hgroups structure doesn't recognize users. In any case, a PID number (currently 24 bit) is sufficient to identify a memory resource owner. This
would allow two or even three identifiers for different resources to be attached to a request or a page of memory to charge subsequent handling properly.

CPU throughput limits are imposed in nearly the same way as memory allocation limits. The only difference is that the limits can be imposed on the local process group as well as just the domain. Limits can be both raised and lowered by a suitably privileged process.

CPU scheduling is probably the most complex of the resource managers. Scheduling groups are formed roughly following the domain/process-group/process hierarchy, but with grouping optional at each level. If grouping is enabled for domain 0, then the processes in each domain are grouped and those groups are scheduled against each other. If it isn't enabled, then the individual process groups in each domain are all scheduled against each other, creating a result quite similar to the current
auto-groups. As with memory resources, a privileged process can direct a process group to be scheduled in the context of some other process group.

On a single-user system, it is likely that domain scheduling would be disabled, and the top-level scheduling would be between process groups. In a multi-user system, the extra cost of domain-level scheduling would probably be justified. Inside containers, the same choices can be made, independently in each container.

This enabling of CPU scheduling independently at each level is a little bit like the approach the unified hierarchy takes of optionally enabling different subsystems at different levels. It is more general, though, as the set of enabled levels does not need to be contiguous.

Hgroups CPU scheduling has another important difference from both cgroups and auto-groups. One of the problems with auto-groups scheduling is that it changes the effect of using nice to make a program run at a lower priority. The fact that nice doesn't really work anymore has been reported but not yet fixed. It seems that some regressions are less important than others, though possibly it hasn't been reported on the right forum.

The problem is that each scheduling group has a priority that is independent of the processes in the group. When you set the niceness of some process, it only causes it to be nice to processes in the same group (same session for auto-groups). When a user has multiple sessions (which is the whole point of auto-groups) they cannot easily be nice to each other.

Hgroups is not in the business of setting priorities, only of imposing restrictions. The restriction it imposes on a process group is to set an upper bound for the priority weight of that group. The effective weight of a group is then the sum (or possibly the maximum) of the weights of active members, providing that does not exceed the upper bound. This allows a low-priority process to continue to be genuinely nice to all other users, not just those in the same scheduling group. When
there are no low-priority processes, it works much the same as the present scheme.

Epilogue
I have certainly found this adventure to be very educational and I'm thankful that you could join me on it. It has achieved the goal of a deep understanding, but I cannot yet tell if it will achieve the goal of improving entertainment. When the next chapter in the cgroups story is revealed, I am prepare to be excited or dismayed; thrilled or disgusted; challenged or affirmed. But the one thing I don't expect to be is bored.
