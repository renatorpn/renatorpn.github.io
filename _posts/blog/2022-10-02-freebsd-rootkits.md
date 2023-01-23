---
layout: post
title:  "FreeBSD Rootkits pt. 1 - Introduction to Modules"
date:   2021-12-01 15:21:16 -0300
tags: freebsd rootkit
---

I am starting these series to take notes of my study of freebsd kernel modules, writing rootkits and sharing code snippets. This series is largely based off ***Designing BSD Rootkits: An Introduction by Joseph Kong***.

## ring 0

Why do we usually call kernel ring 0? In computer science there are these things called **protection rings**. The concept of protection rings exist to separate most privileged layers (therefore most vulnerable) from the least privileged layers. 

As you can see you should never trust the user, therefore you can see there are 2 layers that separates the userland from the kernel-land and this the main point of entry to our rootkits: **device drivers**.

## FreeBSD Modules

FreeBSD (and any other Unix variant for that matter) has a monolithic kernel with a modular design. The key word is **modular**. To extend upon these capabilities both Linux and FreeBSD offer what we call **Kernel Modules**.

> Kernel modules are parts of a kernel that can be started, or loaded, when needed and unloaded when unused. Kernel modules can be loaded when you plug in a piece of hardware and removed with that hardware. This greatly expands the system’s flexibility. Plus, a kernel with all possible functions compiled into it would be rather large. Using modules, you can have a smaller, more efficient kernel and load rarely used functionality only when it’s required.

## Kernel Modules

Kernel modules can be loaded and unloaded in the kernel upon demand, they exist to expand the kernel capability all without needing to reboot.

Modules can be configured as built-in or loadable. To be considered a loadable module, the module should display a ```M```.

In FreeBSD the Linux Kernel Module was renamed to Dynamic Kernel Linker (KLD). Under FreeBSD device drivers can roughly be broken down into two categories; character and network device drivers.

(https://docs.freebsd.org/en/books/arch-handbook/driverbasics/)

Some commands for the KLD:

* ```kldstat``` - Lists kernel modules
* ```kldload``` - Loads kernel modules
* ```kldunload``` - Unloads kernel modules

In general all KLDs have:
* A module event handler
* A ```DECLARE_MODULE``` macro call
(FreeBSD Device Drivers )

## **module event handler**

Whenever ```kldload``` is invoked, a function known as ```module event handler``` is called. This function is responsible for the initialization and shutdown of the module. ```moduledata``` structure (declared in <sys/module.h>) stores some information about LKM:  its name and pointer to event handler function. (https://www.exploit-db.com/exploits/12926)

```c
typedef int (*modeventhand_t)(module_t, int /* modeventtype_t */, void *);
```

```module_t``` is a pointer to a ```module``` struct, and ```modeventtype_t``` is defined as such:

```c
typedef enum modeventtype {
	MOD_LOAD,
	MOD_UNLOAD,
	MOD_SHUTDOWN,
	MOD_QUIESCE
} modeventtype_t;
```
(https://github.com/freebsd/freebsd-src/blob/main/sys/sys/module.h)

Generally, you’d use the `modeventtype_t` argument in a switch statement to set up different code blocks for each situation.

## **DECLARE_MODULE**

Whenever you load a module it must register itself in the kernel and this is accomplished by calling `DECLARE_MODULE` macro defined in the <sys/module.h>

```c
#include <sys/param.h>
#include <sys/kernel.h>
#include <sys/module.h>

define DECLARE_MODULE(name, data, sub, order)
```

### **`name`**

Module name that will be used upon `SYSINIT()`

### **`data`**

A declared`moduledata_t` struct.

```c
/*
 * Struct for registering modules statically via SYSINIT.
 */
typedef struct moduledata {
	const char	*name;		/* module name */
	modeventhand_t  evhand;		/* event handler */
	void		*priv;  /* extra data */
} moduledata_t;
```

### **`sub`**

The sub argument specifies the kernel subsystem that the KLD belongs in. Valid values for this argument are defined in the `sysinit_sub_id` enum, found in <sys/kernel.h>.

```c
enum sysinit_sub_id {
        SI_SUB_DUMMY            = 0x0000000,    /* Not executed.        */
        SI_SUB_DONE             = 0x0000001,    /* Processed.           */
        SI_SUB_TUNABLES         = 0x0700000,    /* Tunable values.      */
        SI_SUB_COPYRIGHT        = 0x0800001,    /* First console use.   */
        SI_SUB_SETTINGS         = 0x0880000,    /* Check settings.      */
        SI_SUB_MTX_POOL_STATIC  = 0x0900000,    /* Static mutex pool.   */
        SI_SUB_LOCKMGR          = 0x0980000,    /* Lock manager.        */
        SI_SUB_VM               = 0x1000000,    /* Virtual memory.      */
    ...
};
```

### **`order`**

The order argument specifies the KLD’s order of initialization within the sub subsystem. Valid values for this argument are defined in the sysinit_elem_order enumeration, found in <sys/kernel.h>. For these purposes we will use `SI_ORDER_MIDDLE`.

```c
enum sysinit_elem_order {
        SI_ORDER_FIRST          = 0x0000000,    /* First.               */
        SI_ORDER_SECOND         = 0x0000001,    /* Second.              */
        SI_ORDER_THIRD          = 0x0000002,    /* Third.               */
        SI_ORDER_FOURTH         = 0x0000003,    /* Fourth.              */
      SI_ORDER_MIDDLE         = 0x1000000,      /* Somewhere in the middle. */
        SI_ORDER_ANY            = 0xfffffff     /* Last.                    */
};
```

(https://www.freebsd.org/cgi/man.cgi?query=DECLARE_MODULE&sektion=9&n=1). 