## Index

- [Introduction](#introduction)
- [Ptrace Options](#ptrace-options)
- [Strace](#strace)
- [Reference](#reference)

## <a name="introduction"></a> Introduction

System call *ptrace* can attach or seize target task and check the signal it receives, the syscall it utilizes, and any other information, even register values. 
Besides peeking the register or memory content, *ptrace* can also overwrite them. 
Utilities *gdb* and *strace* are fantastic tools that build upon *ptrace* and help resolve issues.

## <a name="ptrace-options"></a> Ptrace Options

Although *ptrace* has many options that we can use, it's too many for me, and I will mention a few that *strace* utilizes in its default behavior.
- PTRACE_SEIZE
   - Label target task as tracee and become its temporary parent, but won't stop the tracee from running.
- PTRACE_INTERRUPT
   - Stop the tracee.
- PTRACE_GETREGS
   - Copy *pt_regs* into userspace. When the task switches from USR to SVC mode, its then-registers save into *pt_regs*.
- PTRACE_LISTEN
   - Listen for events, but I failed and didn't know why.
- PTRACE_GETSIGINFO
   - By default, tracee notifies tracer on dequeuing a signal, and this option shows the signal info before handling it.
- PTRACE_GETEVENTMSG
   - When *syscall* and *clone* happen, kernel saves the corresponding event to the *tasks->ptrace_message*, and this option can retrieve it.
- PTRACE_SYSCALL
   - Label *TIF_SYSCALL_TRACE* in *thread_info* so the tracee notifies tracer on syscall entry and exit.

Options _TRACEME/ATTACH/SEIZE_ serve a similar purpose: establish the connection between tracee/tracer and set the related field in _task_struct_. 
The difference between them is that TRACEME is from tracee's perspective while the other two are from the tracer's point of view.

- Code flow - sys_ptrace

```
+------------+                                                                                             
| sys_ptrace |                                                                                             
+--|---------+                                                                                             
   |                                                                                                       
   |--> if request == PTRACE_TRACEME                                                                       
   |                                                                                                       
   |        +----------------+                                                                             
   |        | ptrace_traceme | current task labels itself as tracee                                        
   |        +----------------+                                                                             
   |                                                                                                       
   |--> else if request == PTRACE_ATTACH or PTRACE_SEIZE                                                   
   |                                                                                                       
   |        +---------------+                                                                              
   |        | ptrace_attach | current task labels target task as tracee                                    
   |        +---------------+                                                                              
   |                                                                                                       
   +--> else                                                                                               
                                                                                                           
            +-------------+                                                                                
            | arch_ptrace |                                                                                
            +---|---------+                                                                                
                |                                                                                          
                |--> handle architecture-dependent requests    e.g. PTRACE_GETREGS                         
                |                                                                                          
                |    +----------------+                                                                    
                +--> | ptrace_request | handle architecture-independent requests    e.g. PTRACE_INTERRUPT  
                     +----------------+                                                  PTRACE_LISTEN     
                                                                                         PTRACE_SYSCALL    
                                                                                         PTRACE_GETSIGINFO 
                                                                                         PTRACE_GETEVENTMSG                          
```

- Code flow - entry and exit of syscall

```
                       +------------+                                                                                                              
                +---   | vector_swi |   ---+                                                                                                       
                |      +------------+      |                                                                                                       
  isn't traced  |                          |  is traced                                                                                            
                |                          |                                                                                                       
                |                          |                                                                                                       
                |                          v                                                                                                       
                |               +---------------------+                                                                                            
                |               | syscall_trace_enter |  eventually calls ptrace_stop()                                                            
                |               +---------------------+                                                                                            
                |                          |                                                                                                       
                v                          v                                                                                                       
       +----------------+          +----------------+                                                                                              
       | invoke_syscall |          | invoke_syscall |                                                                                              
       +----------------+          +----------------+                                                                                              
                |                          |                                                                                                       
                |                          v                                                                                                       
                |                +--------------------+                                                                                            
                |                | syscall_trace_exit |  eventually calls ptrace_stop()                                                            
                |                +--------------------+                                                                                            
                |                          |                            +-------------+                                                            
                v                          v                            | ptrace_stop |                                                            
      +------------------+        +------------------+                  +---|---------+                                                            
      | ret_fast_syscall |        | ret_slow_syscall |                      |    +--------------------------+                                      
      +------------------+        +------------------+                      |--> | do_notify_parent_cldstop | send SIGCHLD to tracer and wake it up
                                                                            |    +--------------------------+                                      
                                                                            |    +--------------------+                                            
                                                                            +--> | freezable_schedule | tracee stops                               
                                                                                 +--------------------+
```

## <a name="strace"></a> Strace

Utility _strace_ is a userspace tool that traces the issued syscall sequence of the target process. 
Below log is the _strace_ output of the classic 'Hello, World!' program, and we can observe that lots of unrelated syscalls happen before printing our greetings. 
Every forked task starts from the dynamic linker, e.g.,_/lib/ld-linux.so.3_, following the naming convention of the shared library but is an executable. 
It's responsible for loading specified libraries into memory for the main task to use later, which explains the list of unrelated syscalls in the log.

- Common options

```
strace -p -f -o log.txt running_task
-p: attach to target task
-f: trace the forked or cloned tasks as well
-o: save output to specified file
```

- Log

```
execve("./a.out", ["./a.out"], [/* 15 vars */]) = 0
└─ Triggered by parent process, e.g. bash

brk(NULL)                               = 0xd76000
mmap2(NULL, 8192, PROT_READ|PROT_WRITE, MAP_PRIVATE|MAP_ANONYMOUS, -1, 0) = 0x76f94000
access("/etc/ld.so.preload", R_OK)      = -1 ENOENT (No such file or directory)
openat(AT_FDCWD, "/etc/ld.so.cache", O_RDONLY|O_LARGEFILE|O_CLOEXEC) = 3
syscall_397(0x3, 0x76f7e44c, 0x1800, 0x7ff, 0x7eee4008, 0x7eee4120) = 0
mmap2(NULL, 6720, PROT_READ, MAP_PRIVATE, 3, 0) = 0x76f8c000
close(3)                                = 0
└─ Load /etc/ld.so.cache into memory

openat(AT_FDCWD, "/lib/tls/v6l/vfp/libc.so.6", O_RDONLY|O_LARGEFILE|O_CLOEXEC) = -1 ENOENT (No such file or directory)
syscall_397(0xffffff9c, 0x7eee4150, 0x800, 0x7ff, 0x7eee3fb0, 0x7eee40c8) = -1 (errno 2)
openat(AT_FDCWD, "/lib/tls/v6l/libc.so.6", O_RDONLY|O_LARGEFILE|O_CLOEXEC) = -1 ENOENT (No such file or directory)
syscall_397(0xffffff9c, 0x7eee4150, 0x800, 0x7ff, 0x7eee3fb0, 0x7eee40c8) = -1 (errno 2)
openat(AT_FDCWD, "/lib/tls/vfp/libc.so.6", O_RDONLY|O_LARGEFILE|O_CLOEXEC) = -1 ENOENT (No such file or directory)
syscall_397(0xffffff9c, 0x7eee4150, 0x800, 0x7ff, 0x7eee3fb0, 0x7eee40c8) = -1 (errno 2)
openat(AT_FDCWD, "/lib/tls/libc.so.6", O_RDONLY|O_LARGEFILE|O_CLOEXEC) = -1 ENOENT (No such file or directory)
syscall_397(0xffffff9c, 0x7eee4150, 0x800, 0x7ff, 0x7eee3fb0, 0x7eee40c8) = -1 (errno 2)
openat(AT_FDCWD, "/lib/v6l/vfp/libc.so.6", O_RDONLY|O_LARGEFILE|O_CLOEXEC) = -1 ENOENT (No such file or directory)
syscall_397(0xffffff9c, 0x7eee4150, 0x800, 0x7ff, 0x7eee3fb0, 0x7eee40c8) = -1 (errno 2)
openat(AT_FDCWD, "/lib/v6l/libc.so.6", O_RDONLY|O_LARGEFILE|O_CLOEXEC) = -1 ENOENT (No such file or directory)
syscall_397(0xffffff9c, 0x7eee4150, 0x800, 0x7ff, 0x7eee3fb0, 0x7eee40c8) = -1 (errno 2)
openat(AT_FDCWD, "/lib/vfp/libc.so.6", O_RDONLY|O_LARGEFILE|O_CLOEXEC) = -1 ENOENT (No such file or directory)
syscall_397(0xffffff9c, 0x7eee4150, 0x800, 0x7ff, 0x7eee3fb0, 0x7eee40c8) = -1 (errno 2)
└─ Don't care

openat(AT_FDCWD, "/lib/libc.so.6", O_RDONLY|O_LARGEFILE|O_CLOEXEC) = 3
read(3, "\177ELF\1\1\1\0\0\0\0\0\0\0\0\0\3\0(\0\1\0\0\0000\264\1\0004\0\0\0"..., 512) = 512
syscall_397(0x3, 0x76f7e44c, 0x1800, 0x7ff, 0x7eee3fa0, 0x7eee40b8) = 0
mmap2(NULL, 1361392, PROT_READ|PROT_EXEC, MAP_PRIVATE|MAP_DENYWRITE, 3, 0) = 0x76e0c000
mprotect(0x76f3f000, 61440, PROT_NONE)  = 0
mmap2(0x76f4e000, 16384, PROT_READ|PROT_WRITE, MAP_PRIVATE|MAP_FIXED|MAP_DENYWRITE, 3, 0x132000) = 0x76f4e000
mmap2(0x76f52000, 26096, PROT_READ|PROT_WRITE, MAP_PRIVATE|MAP_FIXED|MAP_ANONYMOUS, -1, 0) = 0x76f52000
close(3)  = 0
└─ Load /lib/libc.so.6 into memory

set_tls(0x76f94d30, 0x76f954a8, 0x76f95428, 0x76f94d30, 0x76f92010) = 0
mprotect(0x76f4e000, 8192, PROT_READ)   = 0
mprotect(0x20000, 4096, PROT_READ)      = 0
mprotect(0x76f91000, 4096, PROT_READ)   = 0
└─ Change some mappings to be read-only

munmap(0x76f8c000, 6720)                = 0
└─ Unload /etc/ld.so.cache from memory

syscall_397(0x1, 0x76f390dc, 0x1800, 0x7ff, 0x7eee4a18, 0x7eee4b30) = 0
ioctl(1, TCGETS, {B115200 opost isig icanon echo ...}) = 0
brk(NULL)                               = 0xd76000
brk(0xd97000)                           = 0xd97000
└─ Prepare heap (even I didn't call malloc)

write(1, "Hello, World!\n", 14)         = 14
└─ Finally!

exit_group(0)                           = ?
+++ exited with 0 +++
```

## <a name="reference"></a> Reference

- [G. Price, Strace -- The Sysadmin's Microscope](https://blogs.oracle.com/linux/post/strace-the-sysadmins-microscope)
- [N. Elhage, Write yourself an strace in 70 lines of code](https://blog.nelhage.com/2010/08/write-yourself-an-strace-in-70-lines-of-code/)
- [C. Costa, Using C to inspect Linux syscalls](https://ops.tips/gists/using-c-to-inspect-linux-syscalls/)
- [Naim A., Understanding ptrace](https://abda.nl/posts/understanding-ptrace/)




