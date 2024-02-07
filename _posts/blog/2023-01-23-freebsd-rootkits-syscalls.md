---
layout: post
title:  "FreeBSD Rootkits pt. 2 - syscalls"
date:   2021-12-01 15:21:16 -0300
tags: freebsd rootkit
---

## DISCLAIMER

> This is article is still a work in progress and will be updated as soon I get back to this. Originally the article was inteded to be a note with my learnings and not public. The original article included verbatum wording and examples from the book ***Designing BSD Rootkits: An Introduction by Joseph Kong***. I am currently rewriting the article to include my own examples and explanations as I want to avoid plagiarism.

*System call modules* are KLDs that install a syscall. A syscall is the mechanism an application uses to request a service from the OS kernel.

The system call function implements the syscall. The prototype is defined in <sys/sysent.h> as such:

```c
typedef int     sy_call_t(struct thread *, void *);
```

where `thread` points to the current thread in execution and `void *` points to the syscall argument structure, if there's any.

An example of a syscall that takes a string and outputs to the console:

```c
struct sc_example_args { //Syscalls arguments are declared inside this struct
        char *str;
};

static int
sc_example(struct thread *td, void *syscall_args)
{
         struct sc_example_args *uap; //uap is a pointer of struct type from the syscall args
         uap = (struct sc_example_args *)syscall_args; //uap is then assigning the void syscall_args pointer to the uap sc_example_args pointer 

        printf("%s\n", uap->str);

        return(0);

}
```

While the arguments reside in user space, any syscall executes in kernel space.

<https://docs.freebsd.org/doc/5.3-RELEASE/usr/share/doc/en/books/developers-handbook/x86-system-calls.html>

# sysent Structure

All syscalls are defined in their respective entry in a `sysent` structure, defined in the <sys/sysent.h> header as follows:

```c
struct sysent {   /* system call table */
 sy_call_t *sy_call; /* implementing function */
 systrace_args_func_t sy_systrace_args_func;
    /* optional argument conversion function. */
 u_int8_t sy_narg; /* number of arguments */
 u_int8_t sy_flags; /* General flags for system calls. */
 au_event_t sy_auevent; /* audit event associated with syscall */
 u_int32_t sy_entry; /* DTrace entry ID for systrace. */
 u_int32_t sy_return; /* DTrace return ID for systrace. */
 u_int32_t sy_thrcnt;
};
```

In FreeBSD syscall table is an array of `sysent` structs, and it is declared in <sys/sysent.h>:

```c
extern struct sysent sysent[];
```

___

# Offset Value

This is known as the syscall value. This is an unique int value ranging from 0 to 456 to each syscall that references `sysent[]` (Apparently 582 as of the writing of this Sep 2022). This value should be declared as:

```c
statit int offset = NO_SYSCALL;
//The NO_SYSCALL constant sets offset to the next available or open element in sysent[] (?)
//For some reason this is the recommended way of doing when implemening something dynamic like a KLD. (do not set hardcoded syscall number) 
```

___

# SYSCALL_MODULE Macro

Instead of calling the ```DECLARE_MODULE``` when doing a syscall we should use ```SYSCAL_MODULE``` instead. This is because if we were to use ```DECLARE_MODULE``` we would need to setup 2 structs beforehand ```syscall_module_data``` and ```moduledata```.

This is declared as such in <sys/sysent.h>:

```c
#define SYSCALL_MODULE(name, offset, new_sysent, evh, arg)     \
static struct syscall_module_data name##_syscall_mod = {       \
       evh, arg, offset, new_sysent, { 0, NULL }               \
};                                                             \
                                                               \
static moduledata_t name##_mod = {                             \
       #name,                                                  \
       syscall_module_handler,                                 \
       &name##_syscall_mod                                 \
};                                                             \
DECLARE_MODULE(name, name##_mod, SI_SUB_DRIVERS, SI_ORDER_MIDDLE)
```

### **`name`**

Generic module name.

### **`offset`**

Specifies the syscall offset value, passed as an integer pointer

### **`new_sysent`**

Specifies the complete ```sysent``` struct, passed as a struct to ```sysent``` pointer.

### **`evh`**

The event handler function.

### **`arg`**

Arguments to be passed to the event handler function. (The book always sets to NULL)


the example in the book is old, and the reason is that it does not include the member access of the C struct
https://bugs.freebsd.org/bugzilla/show_bug.cgi?id=255936
https://reviews.freebsd.org/D30498

sc_example.c:47:1: error: use of undeclared identifier 'AUE_NULL'
SYSCALL_MODULE(sc_example, &offset, &sc_example_sysent, load, NULL);
^
/usr/src/sys/sys/sysent.h:231:43: note: expanded from macro 'SYSCALL_MODULE'
        evh, arg, offset, new_sysent, { 0, NULL, AUE_NULL }     \
                                                 ^
1 error generated.
*** Error code 1

also for the AUE_NULL
https://forums.freebsd.org/threads/how-make-a-system-call-module.12215/

```bash
[vagrant@freebsd13 ~/modules/syscall_example]$ sudo kldload ./sc_example.ko
System call loaded at offset 212.
System call unloaded at offset 212.
System call unloaded at offset 212.
kldload: an error occurred while loading module ./sc_example.ko. Please check dmesg(8) for more details.
```
(https://www.freebsd.org/cgi/man.cgi?query=errno&sektion=2&manpath=freebsd-release-ports)

# modfind()

Returns `modid` of a kernel module based on its name. These are integer values that identify the module.

```c
#include <sys/param.h>
#include <sys/module.h>

int modfind(const char *modname);
```

# modstat()

Return a status of a kernel module.
```c
#include <sys/param.h>
#include <sys/module.h>

int
modstat(int modid, struct module_stat *stat);
```

this returns a `module_stat` struct defined in <sys/module>:

```c
struct module_stat {
	int		version;	/* set to sizeof(struct module_stat) */
	char		name[MAXMODNAME];
	int		refs;
	int		id;
	modspecific_t	data;
};

typedef union modspecific {
 int intval;
 u_int uintval;
 long longval;
 u_long ulongval;
} modspecific_t;

/*
union is a special C data type that lets you hold different data types in the same memory location. Unions can hold ONLY ONE MEMBER VALUE AT TIME

in the above example a variable of modspecific can hold all those member accesses. To access any member of a union, we use the member access operator (.). The member access operator is coded as a period between the union variable name and the union member that we wish to access.

C provides a facility called typedef for creating new data type names. 
*/
```

# syscall()

The `sycall()` function executes the system call specified by the id.

```c
#include <sys/syscall.h>
#include <unistd.h>

int
syscall(int number, ...);
```
FreeBSD segregates its virtual memory into two parts: user space and kernel space. User space is where all user-mode applications run, while kernel space is where the kernel and kernel exten-sions (i.e., LKMs) run. Code running in user space cannot access kernel space directly (but code running in kernel space can access user space). To access kernel space from user space, an application issues a system call.

Kernel/User Space Transitions

https://bugs.freebsd.org/bugzilla/show_bug.cgi?id=260318
https://github.com/freebsd/freebsd-src/blob/main/share/examples/kld/syscall/module/syscall.c
https://www.freebsd.org/cgi/man.cgi?truss
https://wiki.freepascal.org/FreeBSD