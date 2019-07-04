---
layout: post
title : Securing the Kernel Stack
author: Zack Brown
category: linux-security
tags: linux kernel security stack linuxjournal
keywords: linux,kernel,security,stack
---

**Original&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;:**&nbsp;&nbsp;[**Securing the Kernel Stack**](https://www.linuxjournal.com/content/securing-kernel-stack/)  
**Author&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;:**&nbsp;&nbsp;**Zack Brown**  
**Date&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;:**&nbsp;&nbsp;**June 11, 2019**  
**Signed-off-by:**&nbsp;&nbsp;**Norman Kern \<norman.gmx.com\>**

The Linux kernel stack is a tempting target for attack. This is because the kernel needs to keep track of where it is. If a function gets called, which then calls another, which then calls another, the kernel needs to remember the order they were all called, so that each function can return to the function that called it. To do that, the kernel keeps a "stack" of values representing the history of its current context.

If an attacker manages to trick the kernel into thinking it should transfer execution to the wrong location, it's possible the attacker could run arbitrary code with root-level privileges. Once that happens, the attacker has won, and the computer is fully compromised. And, one way to trick the kernel this way is to modify the stack somehow, or make predictions about the stack, or take over programs that are located where the stack is pointing.

Protecting the kernel stack is crucial, and it's the subject of a lot of ongoing work. There are many approaches to making it difficult for attackers to do this or that little thing that would expose the kernel to being compromised.

Elena Reshetova is working on one such approach. She wants to randomize the kernel stack offset after every system call. Essentially, she wants to obscure the trail left by the stack, so attackers can't follow it or predict it. And, she recently posted some patches to accomplish this.

At the time of her post, no specific attacks were known to take advantage of the lack of randomness in the stack. So Elena was not trying to fix any particular security hole. Rather, she said, she wanted to eliminate any possible vector of attack that depended on knowing the order and locations of stack elements.

This is often how it goes—it's fine to cover up holes as they appear, but even better is to cover a whole region so that no further holes can be dug.

There was a lot of interest in Elena's patch, and various developers made suggestions about how much randomness she would need, and where she should find entropy for that randomness, and so on.

In general, Linus Torvalds prefers security patches to fix specific security problems. He's less enthusiastic about adding security to an area where there are no exploits. But in this case, he may feel that Elena's patch adds a level of security that wasn't there before.

Security is always such a nightmare. Often, a perfectly desirable feature may have to be abandoned, not because it's not useful, but because it creates an inherent insecurity. Microsoft's operating system and applications often have suffered from making the wrong decisions in those cases—choosing to implement a cool feature in spite of the fact that it could not be done securely. Linux, on the other hand, and the other open-source systems like FreeBSD, never make that mistake.

Note: if you're mentioned above and want to post a response above the comment section, send a message with your response text to ljeditor@linuxjournal.com.


