> Note: browser extension [GitHub + Mermaid](https://chrome.google.com/webstore/detail/github-%20-mermaid/goiiopgdnkogdbjmncgedmgpoajilohe?hl=en)
> is needed to view the flow chart of mermaid syntax.  
> Note: any suggestion or opinion is extremely welcome!

## Index

- [Introduction](#introduction)
- [Boot Up](#bootup)
- [Tasks & Scheduler](#scheduler)
- [Fair Class](#fair-class)
- [Preemption](#preemption)
- Ignore the below topics for now
- [Task State](#task-state)
- [Strace](#strace)
- [Signal](#signal)
- [Reference](#reference)
- Ptrace (TBD)


## <a name="introduction"></a> Introduction
The 'process' is a concept of running logic designed to fulfill the target purpose.
It can be simple enough, such as the famous 'hello world' containing only one thread printing the greeting string.
The complicated process works as a group of multiple threads executing the assigned jobs to achieve its goal.
Meanwhile, kernel threads are working in privilege space, managing system resources, and meeting requirements from userspace.
In this document, I'd like to introduce the process from different perspectives.

```                                                                 
                                                                    
                    process          process                        
                 +-----------+    +-----------+                     
                 | +-------+ |    | +-------+ |                     
                 | |thread | |    | |thread | |                     
                 | +-------+ |    | +-------+ |    -  -  -   -      
                 |           |    | +-------+ |                     
                 |           |    | |thread | |                     
                 |           |    | +-------+ |                     
 user space      +-----------+    +-----------+                     
--------------------------------------------------------------------
 kernel space                                                       
               +-------+    +-------+    +-------+                  
               |kthread|    |kthread|    |kthread|   - -- -  -      
               +-------+    +-------+    +-------+                  
```

## <a name="bootup"></a> Boot Up
When the register pc points to kernel entry, it sets some low-level stuff in assembly language, and I don't bother looking into it.
The first c function is named 'start_kernel,' and every module will sequentially kick off starting from there.
Before the 'process' mechanism gets ready, we can conceptually regard the running logic as a thread.
Once the most fundamental infrastructures, such as memory and interrupt, are prepared, the running logic essentially becomes a kernel thread.
To take advantage of multiple cores in the processor, the thread forks 'kernel_init' (PID = 1) and 'kthreadd' (PID = 2), and itself turns to an idle task.
The task 'kernel_init' then walks through all kinds of initialization and delivers the kernel thread creation request to 'khtreadd' whenever there is one.
When boot flow reaches the end, 'kernel_init' tries a bunch of possible userspace 'init' utilities and transforms to whichever works first.

```                     
                     PID=1                                       
           fork  +-------------+  transform   +-----------------+
            +--> | kernel_init |  ----------> | init or systemd |
            |    +-------------+              +-----------------+
  +-----+   |                                                    
  | ??? |----                                                    
  +-----+   |                                                    
            |    +------------+     fork +-------------+         
            +--> |  kthreadd  | ------>  |  kthread A  |         
           fork  +------------+    |     +-------------+         
                     PID=2         |                             
                                   |fork +-------------+         
                                   +-->  |  kthread B  |         
                                   |     +-------------+         
                                   |                             
                                   |fork +-------------+         
                                   +-->  |  kthread C  |         
                                         +-------------+         
```
Although 'init' or 'systemd' possesses PID 1, it's not the first thread during boot up.
The task 'kthreadd' is responsible for kernel thread creation, and therefore it's the parent of most kernel threads.
```
$ ps xao pid,ppid,comm | head
    PID    PPID COMMAND
      1       0 systemd
      2       0 kthreadd
      3       2 rcu_gp
      4       2 rcu_par_gp
      6       2 kworker/0:0H-events_highpri
      9       2 mm_percpu_wq
     10       2 rcu_tasks_rude_
     11       2 rcu_tasks_trace
     12       2 ksoftirqd/0
     
Note: PPID is parent PID
```

## <a name="scheduler"></a> Tasks & Scheduler
Before we start introducing the scheduler, let's clarify the below terms.
- Process: refers to userspace utilities or applications, and it consists of at least one thread.
- Thread: the fundamental execution unit within a process.
- Kthread: kernel thread, and there's no process concept in kernel space.

Kernel refers to each thread or kthread as a task, and the process is just a collection of them or formally called 'thread group.'
With that many processor cores, multiple tasks can physically run simultaneously to boost performance and throughput.
The scheduler has a few scheduling classes to satisfy all kinds of task entities, and each entity runs with a priority.
Please note the scheduler itself is not a process or thread but a mechanism with its implementation spread across the kernel flow.
```
                                                              +-------------------------------------------+
                                                              |  +----+                                   |
                                                              |  |  0 |                                   |
               +--------+               +--------+            |  +----+     +------+     +------+         |
               | core 0 |               | core 1 |            |  |  1 |-----| task |-----| task |         |
               +--------+               +--------+            |  +----+     +------+     +------+         |
                                                              |     -                                     |
                                                             -|     -                                     |
                run queue                run queue          / |     -                                     |
            +---------------+        +---------------+     /  |     -                                     |
+------+    |  +---------+  |        |  +---------+  |    /   |     -                                     |
| task |-------|  stop   |  |        |  |  stop   |  |   /    |  +----+     +------+                      |
+------+    |  |  class  |  |        |  |  class  |  |  /     |  | 99 |-----| task |                      |
            |  +---------+  |        |  +---------+  | /      |  +----+     +------+                      |
+------+    |  |deadline |  |        |  |deadline |  |/       +-------------------------------------------+
| ???? |-------|  class  |  |        |  |  class  |  /                                                     
+------+    |  +---------+  |        |  +---------+ /|                                                     
            |  |real time|  |        |  |real time|/ |                                                     
            |  |  class  |  |        |  |  class  |  |                                                     
            |  +---------+  |        |  +---------+  |                                                     
            |  |  fair   |  |        |  |  fair   |  |                                                     
            |  |  class  |  |        |  |  class  |\ |                                                     
+------+    |  +---------+  |        |  +---------+ \|                                                     
| task |-------|  idle   |  |        |  |  idle   |  \                                                     
+------+    |  |  class  |  |        |  |  class  |  |\       +-------------------------------------------+
            |  +---------+  |        |  +---------+  | \      |                 +------+                  |
            +---------------+        +---------------+  \     |                 | task |                  |
                                                         \    |                 +------+                  |
                                                          \   |                /        \                 |
                                                           \  |               /          \                |
                                                            \ |        +------+          +------+         |
                                                             -|        | task |          | task |         |
                                                              |        +------+          +------+         |
                                                              |-        /\                    /\          |
                                                              |        /  \                  /  \         |
                                                              | +------+   +------+   +------+   +------+ |
                                                              | | task |   | task |   | task |   | task | |
                                                              | +------+   +------+   +------+   +------+ |
                                                              +-------------------------------------------+
```
Individual core has its run queue, and it further divides into sub-queues of different scheduling classes.
1. Stop class: it has only one task, which helps task migration between run queues.
2. Deadline class: relatively newly implemented class compared to others. I only know that tasks within this class are guaranteed to run within a certain period.
3. Real-time class: Tasks of this class have a strict policy that lower priority tasks have to wait until higher ones relinquish the execution right.
4. Fair class: also known as Completely Fair Scheduler (CFS). The majority of system tasks belong to this class, and they will run sooner or later.
5. Idle class: like stop class, it has precisely one task which assists in power saving.

In the regard of class priority, stop > deadline > real time > fair > idle.
The rule of selecting the next running task is:
- Start from the high precedence class and check if it has at least one task to run.
  - Yes, if an entity of the real-time class keeps running with no mercy, tasks in fair scheduling class have no chance to shine at all
- Call scheduling class methods to select the best candidate within that class and remove it from sub run queue
Of course, the currently running one will return to its sub-queue for the next chance or somewhere else waiting for the resource.

A few places in the kernel raise the flag of 'it is time to schedule again' when any below conditions become true.
- The given time slice for the running entity consumes entirely.
- Wait for resources, such as data from drives or the internet.
- Priority of a task in the run queue is boosted and becomes a better candidate than the current one.
- After helping share loading, a higher priority task arrives from other run queues.

Many flag checking points exist somewhere inside the kernel, and one of them is hardware interrupt.
Interrupts happen from time to time, and on its way back to executing the ordinary task, it performs a task switch if that flag raises.
The formal name is  'context switch,' which saves CPU registers of running task to memory and loads the register set of next candidate into CPU.
Voila! Now the 'next task' becomes running and continues the logic where it stopped previously.
```      
                                               memory      
                                          +---------------+
                                          |               |
                                          |               |
            CPU                           |               |
+-------------------------+               |               |
| +---+ +---+ +---+ +---+ |               |               |
| |r0 | |r4 | |r8 | |r12| |     save      +---------------+
| +---+ +---+ +---+ +---+ |  ---------->  |  task A regs  |
| +---+ +---+ +---+ +---+ |               +---------------+
| |r1 | |r5 | |r9 | |sp | |               |               |
| +---+ +---+ +---+ +---+ |               |               |
| +---+ +---+ +---+ +---+ |               |               |
| |r2 | |r6 | |r10| |lr | |               |               |
| +---+ +---+ +---+ +---+ |    restore    +---------------+
| +---+ +---+ +---+ +---+ |  <----------  |  task B regs  |
| |r3 | |r7 | |r11| |pc | |               +---------------+
| +---+ +---+ +---+ +---+ |               |               |
+-------------------------+               |               |
                                          |               |
                                          |               |
                                          |               |
                                          |               |
                                          |               |
                                          |               |
                                          |               |
                                          +---------------+
```

## <a name="fair-class"></a> Fair Class

We mainly introduce this class since it covers most utilities, applications, and kernel threads.
Instead of a conventional queue, it's a tree sorted by each entity's 'virtual runtime.'
Because of much effort in maintaining this tree, selecting the next task equals finding the leftmost node.
As its name 'virtual' hints, it relates to actual runtime but not the same. Priority matters when updating virtual ones.
For example, assuming the given time slice is 10s by default, high-priority tasks might add only 5s to virtual runtime after using up all the 10s.
In the meantime, low-priority tasks double the accounting after completely consuming 10s.
By inspecting such rule, we can infer that:
- Tasks with lower priority quickly move far from the leftmost node, which means it's hard to be the next running task.
- Even though high-priority tasks increase virtual runtime slowly, it's still strictly increasing and won't always be the candidate.

The command 'nice' controls the priority of tasks in fair class as we've expected, except the 'nice' value is opposite to precedence.

```
  +-------+                                                        
  |       |                      +------+                          
  |  CPU  |                      | task |                          
  | +------+                     +------+                          
  +-|-task+|                    /        \                         
    +------+                   /          \                        
  running task          +------+          +------+                 
                        | task |          | task |                 
                        +------+          +------+                 
                         /\                    /\                  
                        /  \                  /  \                 
                 +------+   +------+   +------+   +------+         
 next candidate  | task |   | task |   | task |   | task |         
                 +------+   +------+   +------+   +------+         
                                                                   
                                                                   
         the task with a 'nice' value of 20 moves rightward easily 
                      ------------------------------->             
                                                                   
                                                                   
         the task with a 'nice' value of -19 stays left side longer
                      <-------------------------------             
```
## <a name="preemption"></a> Preemption

Whenever the logic flow reaches the flag checking point, it selects and schedules to next task if necessary.
If it's the thread itself relinquishing the execution right early, we call it kindness. (Nope, there's no such saying)
Otherwise, the kernel mechanism applies context switch because of running out of time slice, and the passive schedule is named 'preemption.'
For example, if we run a process that loops infinitely after taking the 'newbie hacker 101' class, there's no way that system is affected by our intention.
During the execution of the infinite loop, the timer interrupt triggers as usual, and its interrupt handler checks the remaining time slice of the running task.
No matter how powerful our infinite loop shows, it's forced to context switch when time's up, and OS isn't even aware of our hacker spirit.
This kind of preemption belongs to the user space category, and it's always working, or there will be a mess everywhere,
The situation becomes complicated in kernel space since it's not always safe to switch, and the interrupt mechanism is disabled temporarily to avoid preempting.
Commonly speaking, when we talk about the feature preemption, it means the behavior in kernel space.
As we can imagine, the feature improves the responsiveness of the OS, but it is better to disable it on systems without much interaction between users.


## <a name="task-state"></a> Task State

```
[task_struct->state]
TASK_RUNNING: task is running or waiting in the run queue.
TASK_INTERRUPTIBLE: task waits somewhere for the target event to happen, and can receive the signals in the meantime.
TASK_UNINTERRUPTIBLE: similar to the above except it can't receive the signal while sleeping. So if it hangs, it hangs...
TASK_KILLABLE: similar to the above except it can still be killed.
TASK_NEW: The forked task is in state until it's added to the run queue. (sched_fork)

[task_struct->exit_state]
EXIT_ZOMBIE: once a task exits, it becomes this state first and notifies the parent for farewell.
EXIT_DEAD: either the child sets itself to this state (auto reap), or the parent will help set it.
```

## <a name="strace"></a> Strace

Utility *strace* is a user space tool that can be used to trace the issued syscall sequence of target process. 
Below log is the trace output of the classic 'Hello, World!' test program and we can observe that lots of unrelated syscalls happen before printing our greetings. 
Every forked task starts from the dynamic linker, e.g. /lib/ld-linux.so.3, which follows the naming convention of the shared library but is actually an executable. 
It's responsible to load specified libraries into memory for the main task to use later, and that explains the list of unrelated syscalls in the log. 

Common used options:
```
strace -p -f -o log.txt running_task
-p: attach to target task
-f: trace the forked or cloned tasks as well
-o: save output to specified file
```

Log:
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

## <a name="signal"></a> Signal

1. Tasks can receive signals and handle them accordingly.
2. Signal handlers can be overwritten but SIGKILL is an exception, otherwise, tasks can opt to be indestructible!
3. More or less we had the experience of sending signals, e.g. exit running task on bash foreground by Ctrl+C, or use utility *kill* to end target process.
4. Utility *kill* is used to send specified signals to the target process, although most of the time it's SIGTERM or SIGKILL.

```mermaid
graph TD
   a(Command: <br> 'kill' sends SIGTERM to target task)
   b(__send_signal: <br> Kernel allocates sigqueue, <br> adds to the end of task pending list)
   c(__send_signal: <br> Kernel sets the corresponding signal in task bitmap)
   d(complete_signal: <br> Kernel sets the _TIF_SIGPENDING)
   
   a-->b
   b-->c
   c-->d
```

Every user space process enters kernel space on purpose or involuntarily (e.g. by interrupt), and pending signals are checked before tasks return to user space. 
If \_TIF_SIGPENDING is set, kernel delivers the signal to the task by:
1. Determine the next signal to handle and it's not guaranteed to correspond to the first sigqueue in the list.
2. Clear that signal in task bitmap and remove related sigqueue from the list accordingly. 
3. (Assume that signal has a decent signal handler) Fetch that handler info and prepare signal frame on user space stack. 
4. Once the flow switches back to user space, it starts from the special frame to run handler and returns to the original process flow(?)

```mermaid
graph TD
   a(do_work_pending: <br> Handle pending signal if there's any)
   b(__dequeue_signal: <br> Determine the signal to handle and dequeue the related sigqueue from list)
   c(get_signal: <br> Fetch signal handler)
   d(setup_frame: <br> Set up the special frame for later execution)
   
   a-->b
   b-->c
   c-->d
```

## <a name="reference"></a> Reference

1. [Signals in Linux](https://towardsdatascience.com/signals-in-linux-b34cea8c5791)
2. [TASK_KILLABLE](https://lwn.net/Articles/288056/)
3. [Reap zombie processes using a SIGCHLD handler](http://www.microhowto.info/howto/reap_zombie_processes_using_a_sigchld_handler.html)
4. [Thread Scheduling with pthreads under Linux and FreeBSD](http://www.icir.org/gregor/tools/pthread-scheduling.html)
 
