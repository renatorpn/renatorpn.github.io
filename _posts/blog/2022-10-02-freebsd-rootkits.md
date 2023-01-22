---
layout: post
title:  "FreeBSD Rootkits pt. 1 - Introduction to Modules"
date:   2021-12-01 15:21:16 -0300
tags: freebsd rootkit
---

# ring 0

Why do we usually call kernel ring 0? In computer science there are these things called **protection rings**. The concept of protection rings exist to separate most privileged layers (therefore most vulnerable) from the least privileged layers. If we look at the graph we can easily understand the concept:

(insert wikipedia image)

As you can see you should never trust the user, therefore you can see there are 2 layers that separates the userland from the kernel-land and this the main point of entry to our rootkits: **device drivers**.

# FreeBSD Modules

FreeBSD (and any other Unix variant for that matter) has a monolithic kernel with a modular design. The key word is **modular**. To extend upon these capabilities both Linux and FreeBSD offer what we call **Kernel Modules**.

>Kernel modules are parts of a kernel that can be started, or loaded, when needed and unloaded when unused. Kernel modules can be loaded when you plug in a piece of hardware and removed with that hardware. This greatly expands the system’s flexibility. Plus, a kernel with all possible functions compiled into it would be rather large. Using modules, you can have a smaller, more efficient kernel and load rarely used functionality only when it’s required.

