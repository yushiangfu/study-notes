> The note is based on Linux version 5.15.0 in OpenBMC.

## Index

- [Introduction](#introduction)
- [Process and Thread](#process-and-thread)
- [Life Cycle](#life-cycle)
- [Scheduler and Run Queue](#scheduler-and-run-queue)
- [Priority and Class](#priority-and-class)
- [System Startup](#system-startup)
- [Others](#others)
  - Resource Limit
  - PID Structure
- [Reference](#reference)

## <a name="introduction"></a> Introduction

(TBD)

## <a name="process-and-thread"></a> Process and Thread

A `process` is a fundamental concept in computing that represents the execution of logic designed to accomplish a specific objective. 
It can vary in complexity, ranging from simple tasks like the well-known "hello world" program, which consists of a single thread printing a greeting message, to more intricate processes involving multiple threads working together to solve complex tasks. 
Additionally, there are kernel threads that operate in privileged mode and handle system resource management. 
Both user threads and kernel threads are considered `task`s from the perspective of the kernel.

<p align="center"><img src="images/process/process-and-thread.png" /></p>

<details><summary> More Details </summary>

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
  
</details>

## <a name="life-cycle"></a> Life Cycle

A task begins in the `NEW` status when it is created but not yet added to a run queue. 
It transitions to the `RUNNING` status when it is queued and either waiting to run or actively executing on a CPU. 
During runtime, if a task encounters an event that requires it to wait for a certain condition, it enters the `SLEEPING` status. 
In this state, the task remains temporarily inactive until the condition is met. 
Tasks can vary in nature, such as background daemons that continuously operate or regular commands that perform a specific job and then exit.

<p align="center"><img src="images/process/life-cycle.png" /></p>

<details><summary> More Details </summary>

```
                                             run queue                                
              +-------+    |                                                          
              |       |    |                  +------+                                
              |  CPU  |    |                  | task | state: RUNNING                 
              | +------+   |                  +------+                                
              +-|-task+|   |                 /        \                               
 state: RUNNING +------+   |                /          \                              
                    |      |         +------+          +------+                       
               fork |      |         | task |          | task | state: RUNNING        
                    v      |         +------+          +------+                       
                +------+   |          /\                    /\                        
     state: NEW | task |   |         /  \                  /  \                       
                +------+   |  +------+   +------+   +------+   +------+               
                           |  | task |   | task |   | task |   | task | state: RUNNING
                           |  +------+   +------+   +------+   +------+               
                           |                                                          
   --------------------------------------------------------------------------------
                                                                                     
                                             wait queue                              
                                                                                     
                                 +--------------------------------+                  
                                 | +------+                       |                  
                                 | | task | state: INTERRUPTIBLE  |                  
                                 | +------+                       |                  
                                 | +------+                       |                  
                                 | | task | state: UNINTERRUPTIBLE|                  
                                 | +------+                       |                  
                                 | +------+                       |                  
                                 | | task | state: KILLABLE       |                  
                                 | +------+                       |                  
                                 +--------------------------------+                                    
```
  
```
struct task_struct {
    unsigned int            __state;  // current running state
    int             exit_state;       // state for exiting process
}
```
  
```
/* Used in tsk->state: */
#define TASK_RUNNING            0x0000      // the task is running or ready to run
#define TASK_INTERRUPTIBLE      0x0001      // sleeping (accept signal)
#define TASK_UNINTERRUPTIBLE        0x0002  // sleeping (ignore signal)
#define __TASK_STOPPED          0x0004      // task is stopped by intention, e.g., debug
#define __TASK_TRACED           0x0008      // task is ptraced
  
/* Used in tsk->exit_state: */
#define EXIT_DEAD           0x0010          // after parent's ack, before it totally vanishes
#define EXIT_ZOMBIE         0x0020          // after task termination, before parent's ack
```
  
</details>

### State - New

Threads within the same process share the same virtual memory space, file table, and other resources that are exclusive to that process. 
The creation of new tasks with distinct shared resources is facilitated by system calls like `clone` and `fork`, which interestingly invoke the same core function. 
For each resource, the function determines whether it should be shared with the parent task or if a new copy should be created, based on the flags passed as arguments. 
The `fork` syscall is typically not used in isolation since creating two identical copies would be redundant. 
Instead, it is often followed by the `execve` syscall, which loads the desired application into memory, replacing the existing logic.

<p align="center"><img src="images/process/fork-and-clone.png" /></p>

<details><summary> More Details </summary>

```
           case 1                                 case 2                                   
     task clones a task         |           task forks a task                              
                                |                                                          
                                |                                                          
+-------+  clone   +-------+    |    +-------+    fork      +-------+                      
|thread |  ----->  |thread |    |    |thread |   ------>    |thread |                      
+-------+          +-------+    |    +-------+              +-------+                      
   |                    |       |       |                      |                           
   |  +--------------+  |       |       |  +--------------+    |  +--------------+         
   +--| memory space |--+       |       +--| memory space |    +--| memory space |         
   |  +--------------+  |       |       |  +--------------+    |  +--------------+         
   |                    |       |       |                      |                           
   |  +--------------+  |       |       |  +--------------+    |  +--------------+         
   +--|  file table  |--+       |       +--|  file table  |    +--|  file table  |         
   |  +--------------+  |       |       |  +--------------+    |  +--------------+         
   |                    |       |       |                      |                           
   |  +--------------+  |       |       |  +--------------+    |  +--------------+         
   +--|    blabla    |--+       |       +--|    blabla    |    +--|    blabla    |         
   |  +--------------+  |       |       |  +--------------+    |  +--------------+         
   |                    |       |       |                      |                           
   |  +--------------+  |       |       |  +--------------+    |  +--------------+         
   +--|    blabla    |--+       |       +--|    blabla    |    +--|    blabla    |         
      +--------------+                     +--------------+       +--------------+              
```

```
+----------+
| sys_fork |---+
+----------+   |
+-----------+  |
| sys_clone |--+
+-----------+  |
               |  +--------------+
               +--| kernel_clone |
                  +--------------+
```
  
```
+--------------+                                                                                                       
| kernel_clone | : prepare new task and wake it up                                                                     
+---|----------+                                                                                                       
    |    +--------------+                                                                                              
    |--> | copy_process | allocate task, clone or share resource based on flags, set pid value and attach to struct pid
    |    +--------------+                                                                                              
    |                                                                                                                  
    |--> if CLONE_PARENT_SETTID is specified                                                                           
    |                                                                                                                  
    +------> write the pid value to the given userspace address                                                        
    |                                                                                                                  
    |    +------------------+                                                                                          
    |--> | wake_up_new_task | add the newly generated task into a run queue
    |    +------------------+                                                                                          
    |                                                                                                                  
    +--> if child is traced                                                                                            
    |                                                                                                                  
    |        +------------------+                                                                                      
    +------> | ptrace_event_pid | notify tracer of the event                                                           
             +------------------+                                                                                      
```
  
```
+--------------+                                                                                                
| copy_process | : allocate task, clone or share resource based on flags, set pid value and attach to struct pid
+---|----------+                                                                                                
    |    +-----------------+                                                                                    
    |--> | dup_task_struct | allocate task and thread stack, copy from parent and reset some fields             
    |    +-----------------+      
    |   
    |--> save child_tid to ->set_child_tid for later use if CLONE_CHILD_SETTID is speficied
    |
    |--> save child_tid to ->clear_child_tid for later use if CLONE_CHILD_CLEARTID is speficied
    |                                                                            
    |    +------------+                                                                                         
    |--> | sched_fork | state = NEW, set prio & sched class                                                     
    |    +------------+                                                                                         
    |    +--------------+                                                                                       
    |--> | copy_semundo | share sem undo list if CLONE_SYSVSEM is specified                                     
    |    +--------------+                                                                                       
    |    +------------+                                                                                         
    |--> | copy_files | share files if CLONE_FILES is specfied                                                  
    |    +------------+                                                                                         
    |    +---------+                                                                                            
    |--> | copy_fs | share fs if CLONE_FS is specified                                                          
    |    +---------+                                                                                            
    |    +--------------+                                                                                       
    |--> | copy_sighand | share sighand if CLONE_SIGHAND is specified                                           
    |    +--------------+                                                                                       
    |    +-------------+                                                                                        
    |--> | copy_signal | share signal is CLONE_THREAD is specified                                              
    |    +-------------+                                                                                        
    |    +---------+                                                                                            
    |--> | copy_mm | share mm if CLONE_VM is specified                                                          
    |    +---------+                                                                                            
    |    +-----------------+                                                                                    
    |--> | copy_namespaces | create new namespace if any related flag is specified                              
    |    +-----------------+                                                                                    
    |    +---------+                                                                                            
    |--> | copy_io | share io contect if CLONE_IO is specified                                                  
    |    +---------+                                                                                            
    |    +-------------+                                                                                        
    |--> | copy_thread | set tls if CLONE_SETTLS is specified, set ret pc = ret_from_fork
    |    +-------------+                                                                                        
    |    +-----------+                                                                                          
    |--> | alloc_pid | allocate 'pid' and set up pid value for each level                                       
    |    +-----------+                                                                                          
    |                                                                                                           
    |--> set task pid value                                                                                     
    |                                                                                                           
    |--> determine parent and exit signal                                                                       
    |                                                                                                           
    +--> attach task to all types of struct pid                                                                 
```

```
+-----------------+                                                                         
| dup_task_struct | : allocate task and thread stack, copy from parent and reset some fields
+----|------------+                                                                         
     |    +------------------------+                                                        
     |--> | alloc_task_struct_node | allocate 'task'                                        
     |    +------------------------+                                                        
     |    +-------------------------+                                                       
     |--> | alloc_thread_stack_node | allocate two pages (4K * 2) as task stack             
     |    +-------------------------+                                                       
     |    +----------------------+                                                          
     |--> | arch_dup_task_struct | copy content from parent task to child's                 
     |    +----------------------+                                                          
     |    +--------------------+                                                            
     |--> | setup_thread_stack | copy 'thread info' from parent's                           
     |    +--------------------+                                                            
     |                                                                                      
     +--> reset some task fields                                                            
```

```
+------------+                                                            
| sched_fork | : state = NEW, set prio & sched class                      
+--|---------+                                                            
   |    +--------------+                                                  
   |--> | __sched_fork | init sched related fields                        
   |    +--------------+                                                  
   |                                                                      
   |--> task state = NEW                                                  
   |                                                                      
   |--> child prio = parent normal prio (to avoid booted prio inheritance)
   |                                                                      
   |--> determine shed class based on prio                                
   |                                                                      
   +--> task preempt count = 0 in our config                              
```
  
```
+---------------+                                                                     
| ret_from_fork | :
+---|-----------+                                                                     
    |    +---------------+                                                            
    |--> | schedule_tail | write pid value to the given userspace address if necessary
    |    +---------------+                                                            
    |    +------------------+                                                         
    +--> | ret_slow_syscall |                                                         
         +------------------+                                                         
```
  
```
+---------------+                                                              
| schedule_tail | : write pid value to the given userspace address if necessary
+---|-----------+                                                              
    |    +--------------------+                                                
    |--> | finish_task_switch |                                                
    |    +--------------------+                                                
    |    +----------------+                                                    
    |--> | preempt_enable | (disabled config)                                  
    |    +----------------+                                                    
    |                                                                          
    |--> if task set_child_tid is saved during copy_process()                  
    |                                                                          
    +------> write the pid value to the given userspace address                
```
  
```
+-----------+                                                     
| alloc_pid | : allocate 'pid' and set up pid value for each level
+--|--------+                                                     
   |                                                              
   |--> allocate 'pid'                                            
   |                                                              
   |--> pid level = ns level                                      
   |                                                              
   |--> from child to root namespace                              
   |                                                              
   |------> prepare pid value                                     
   |                                                              
   |------> set up pid number of that level                       
   |                                                              
   +--> for each level, install pid to that ns                    
```
  
```
+------------------+                                                                               
| wake_up_new_task | : state = RUNNING, place task onto the suitable runqueue, resched if necessary
+----|-------------+                                                                               
     |                                                                                             
     |--> task state = RUNNING                                                                     
     |                                                                                             
     |--> determine runqueue to place into and set that cpu to task                                
     |                                                                                             
     |    +---------------+                                                                        
     |--> | activate_task | e.g., place task into runqueue                                         
     |    +---------------+                                                                        
     |    +--------------------+                                                                   
     |--> | check_preempt_curr | mark current running task 'resched' if appropriate                
     |    +--------------------+                                                                   
     |                                                                                             
     |--> if ->task_woken() exists                                                                 
     |                                                                                             
     +------> call ->task_woken(), e.g.,                                                           
              +---------------+                                                                    
              | task_woken_rt |                                                                    
              +---------------+                                                                    
```
  
```
kernel/kthread.c                                                                                                       
+-------------------------+                                                                                             
| __kthread_create_worker | : create kthread (kthread_worker_fn), prepare worker, wake up kthread                       
+-|-----------------------+                                                                                             
  |                                                                                                                     
  |--> alloc kthread_worker                                                                                             
  |                                                                                                                     
  |           +--------------------------+                                                                              
  |--> task = | __kthread_create_on_node | ask kthreadd help creating kthread, set its name, sched-related, and cpu mask
  |           +-------------------+------+                                                                              
  |           | kthread_worker_fn | endless loop: run work->func()                                                      
  |           +-------------------+                                                                                     
  |                                                                                                                     
  |--> if arg cpu is specified                                                                                          
  |    |                                                                                                                
  |    |    +--------------+                                                                                            
  |    +--> | kthread_bind | bind a kthread to a cpu                                                                    
  |         +--------------+                                                                                            
  |                                                                                                                     
  |--> save flags and task in kthread_worker                                                                            
  |                                                                                                                     
  |    +-----------------+                                                                                              
  +--> | wake_up_process |                                                                                              
       +-----------------+                                                                                              
```
  
```
kernel/kthread.c                                     
+-------------------+                                 
| kthread_worker_fn | : endless loop: run work->func()
+-|-----------------+                                 
  |                                                   
  |--> save current_task in worker                    
  |repeat:                                            
  |--> set current_state = interruptible              
  |                                                   
  |--> if kthread should stop                         
  |    |                                              
  |    |--> set current_state = running               
  |    |                                              
  |    +--> return                                    
  |                                                   
  |--> if there's remaining work(s)                   
  |    -                                              
  |    +--> remove work from list                     
  |                                                   
  |--> save work (null or not) in worker->current_work
  |                                                   
  |--> if work exists                                 
  |    -                                              
  |    +--> run work->func()                          
  |                                                   
  |--> else                                           
  |    |                                              
  |    |    +----------+                              
  |    +--> | schedule |                              
  |         +----------+                              
  |    +--------------+                               
  |--> | cond_resched |                               
  |    +--------------+                               
  |                                                   
  +--> go to 'repeat'                                 
```
  
```
struct linux_binfmt {
    struct list_head lh;                            // list node
    struct module *module;
    int (*load_binary)(struct linux_binprm *);      // to load the program into memory
    int (*load_shlib)(struct file *);               // to load the shared library
    int (*core_dump)(struct coredump_params *cprm); // to write out the core dump when process crashes
    unsigned long min_coredump;                     // minimum size of a core dump file
}
```

```
+------------+                                                                                              
| sys_execve | :                                                                                              
+--|---------+                                                                                              
   |    +-----------+                                                                                       
   +--> | do_execve | :                                                                                       
        +--|--------+                                                                                       
           |    +--------------------+                                                                      
           +--> | do_execveat_common | :
                +----|---------------+                                                                      
                     |    +------------+                                                                    
                     |--> | alloc_bprm | :
                     |    +--|---------+                                                                    
                     |       |                                                                              
                     |       |--> allocate 'bprm'                                                           
                     |       |                                                                              
                     |       |--> set filename and interp of bprm                                           
                     |       |                                                                              
                     |       |    +--------------+                                                          
                     |       +--> | bprm_mm_init | :
                     |            +---|----------+                                                          
                     |                |    +----------+                                                     
                     |                |--> | mm_alloc | prepare mm and page table                           
                     |                |    +----------+                                                     
                     |                |    +----------------+                                               
                     |                +--> | __bprm_mm_init | prepare vma of stack, and link it to framework
                     |                     +----------------+                                               
                     |    +--------------+                                                                  
                     |--> | copy_strings | copy env to somewhere                                            
                     |    +--------------+                                                                  
                     |    +--------------+                                                                  
                     |--> | copy_strings | copy arg to somewhere                                            
                     |    +--------------+                                                                  
                     |    +-------------+                                                                   
                     +--> | bprm_execve | move to a proper rq, clear old mapping, map exec and interp       
                          +-------------+                                                                   
```

```
+-------------+
| bprm_execve | : move to a proper rq, clear old mapping, map exec and interp
+---|---------+
    |    +----------------+
    |--> | do_open_execat | :
    |    +---|------------+
    |        |
    |        |--> determine flags
    |        |
    |        |    +--------------+
    |        +--> | do_filp_open | allocate 'file', find dentry/inode, install ops, and call ->open()
    |             +--------------+
    |    +------------+
    |--> | sched_exec | select rq and migrate task onto it
    |    +------------+
    |    +-------------+
    +--> | exec_binprm | :
         +---|---------+
             |
             |--> for 1 exec and at most 5 interpreters
             |
             |        +-----------------------+
             |------> | search_binary_handler | :
             |        +-----|-----------------+
             |              |    +----------------+
             |              |--> | prepare_binprm | read file into buffer
             |              |    +----------------+
             |              |
             |              |--> for each registered format (e.g., script, elf)
             |              |
             |              |------> call ->load_binary(), e.g.,
             |              |        +-----------------+
             |              |        | load_elf_binary | map segments of exec and interp
             |              |        +-----------------+ copy and env to stack, regs->pc to interp
             |              |
             |              +------> return if ok
             |
             +------> break if brpm->interpreter isn't set (though elf has interpreter, it doesn't set this field)
```
 
```
+-------------+                                                                                              
| sys_unshare | : based on flags, unshare specified resources and switch to the newly created ones           
+---|---------+                                                                                              
    |    +--------------+                                                                                    
    +--> | ksys_unshare | :                                                                                   
         +---|----------+                                                                                    
             |    +------------+                                                                             
             |--> | unshare_fs | uhsnare fs (root, pwd) if specified                                         
             |    +------------+                                                                             
             |    +------------+                                                                             
             |--> | unshare_fd | unshare file descriptors if specified                                       
             |    +------------+                                                                             
             |    +----------------+                                                                         
             |--> | unshare_userns | (disabled config)                                                       
             |    +----------------+                                                                         
             |    +----------------------------+                                                             
             |--> | unshare_nsproxy_namespaces | allocate nsproxy, determine to share or clone each namespace
             |    +----------------------------+                                                             
             |                                                                                               
             +--> switch to the newly created resource                                                       
```
  
```
+----------------------------+                                                                      
| unshare_nsproxy_namespaces | : allocate nsproxy, determine to share or clone each namespace       
+------|---------------------+                                                                      
       |                                                                                            
       |--> determine user namespace                                                                
       |                                                                                            
       |    +-----------------------+                                                               
       +--> | create_new_namespaces | : allocate nsproxy, determine to share or clone each namespace
            +-----|-----------------+                                                               
                  |    +----------------+                                                           
                  |--> | create_nsproxy | allocate nsproxy                                          
                  |    +----------------+                                                           
                  |    +-------------+                                                              
                  |--> | copy_mnt_ns | share or clone mnt namespace based on flags                  
                  |    +-------------+                                                              
                  |    +--------------+                                                             
                  |--> | copy_utsname | share or clone uts namespace based on flags                 
                  |    +--------------+                                                             
                  |    +-------------+                                                              
                  |--> | copy_pid_ns | share or clone pid namespace based on flags                  
                  |    +-------------+                                                              
                  |    +----------------+                                                           
                  |--> | copy_cgroup_ns | share or clone cgroup namespace based on flags            
                  |    +----------------+                                                           
                  |    +-------------+                                                              
                  |--> | copy_net_ns | share or clone net namespace based on flags                  
                  |    +-------------+                                                              
                  |    +--------------+                                                             
                  +--> | copy_time_ns | (disabled config)                                           
                       +--------------+                                                             
```
  
</details>

### State - Running

Tasks that are actively running on CPUs or waiting in run queues are typically labeled as being in the `RUNNING` state.

### State - Sleeping

Sleeping is a conceptual state in which a task is temporarily removed from the run queue and waits for a specific event to occur. 
This event can vary and includes scenarios such as waiting for data to be read from the disk, waiting for a specific time duration to elapse, or waiting for a particular request to arrive.

When a task is in the sleeping state, it can have one of the following practical state values:

- `TASK_INTERRUPTIBLE`: The task will wake up if it receives any signal during its sleep. It remains responsive to signals while waiting for the specified event to occur.
- `TASK_UNINTERRUPTIBLE`: Tasks in this state will only wake up when the condition they are waiting for is met. They are not responsive to signals and stay in the sleeping state until the desired event happens.
- `TASK_KILLABLE`: Similar to `TASK_UNINTERRUPTIBLE`, tasks in this state also wait for the specified condition to be met. However, they can be terminated forcibly if a kill signal is received.

These practical state values define how a sleeping task behaves and determine the conditions under which it will wake up and resume its execution.

### State - Dead

When a task completes its execution, it notifies its parent before releasing the resources it has used. 
The state of the task transitions from `ZOMBIE` to `DEAD` until it disappears entirely from the system. 
However, if the parent is indifferent or fails to acknowledge the child's exit, the child task may remain in the `ZOMBIE` state.

<details><summary> More Details </summary>

```
              process                 
           +------------+             
           |   leader   |             
           | +--------+ |             
+----------- | thread | sys_gorup_exit
|  |       | +--------+ |             
|  |       | +--------+ |             
|  +-------> | thread | sys_exit      
| sys_clone| +--------+ |             
|          | +--------+ |             
+----------> | thread | sys_exit      
  sys_clone| +--------+ |             
           +------------+             
```

```
+----------------+
| sys_exit_group | :
+---|------------+
    |    +---------------+
    +--> | do_group_exit | :
         +---|-----------+
             |
             |--> if thread group is exiting
             |
             |        get exit code from signal struct
             |
             |--> else
             |
             |        set exit code and GROPU_EXIT flag in signal struct
             |
             |        +-------------------+
             |        | zap_other_threads | for each other thread in the group
             |        +-------------------+ set SIGKILL bit and wake up the thread to face the bad news
             |
             |    +---------+
             +--> | do_exit |
                  +---------+                                                                         
```
  
```
+----------+
| sys_exit | :
+--|-------+
   |    +---------+
   +--> | do_exit | : transfer child tasks to reaper, release resources, and yield scheduling
        +--|------+
           |    +--------------+
           |--> | ptrace_event | notify tracer of the exit
           |    +--------------+
           |    +--------------+
           |--> | exit_signals |
           |    +---|----------+
           |        |
           |        |--> label EXITING on the task
           |        |
           |        |    +--------------------------+
           |        +--> | retarget_shared_pending  | ask other threads in the group
           |             +--------------------------+ to take of pending signals of the current thread
           |
           |--> set exit code of the task
           |
           |    +-------------+
           |--> | exit_notify | :
           |    +---|---------+
           |        |    +------------------------+
           |        +--> | forget_original_parent | ask reaper (init or systemd) to take care of the children
           |        |    +------------------------+
           |        |
           |        |--> set task exit state to ZOMBIE
           |        |
           |        |--> notify parent of the exist
           |        |
           |        +--> if auto reap (either task isn't the group leader, or the parent doesn't care)
           |
           |                 change task exit state to DEAD
           |
           |                 +--------------+
           |                 | release_task |
           |                 +--------------+
           |
           |    +--------------+
           +--> | do_task_dead | schedule to let other task run
                +--------------+
```
  
</details>

## <a name="scheduler-and-run-queue"></a> Scheduler and Run Queue

Each CPU in the system maintains its own run queue, which holds all the tasks that are ready to run on that specific CPU. 
The scheduler, a mechanism that operates within frequently executed code paths, is responsible for selecting the next task to be executed.

The design of the scheduler can be divided into two main parts: the `RESCHED` flag and the context switch. 
The `RESCHED` flag is raised to indicate that it is time to service the next task in line. 
This can happen for various reasons, such as when a task has utilized its allocated time slice, when it needs to sleep for a certain period, when a target lock is acquired elsewhere, when it is waiting for data to be read or written, when its priority is lowered, or when a more critical task emerges.

Accompanied by distributed checkpoints in the kernel, the scheduler monitors the `RESCHED` flag and initiates a task replacement process called a context switch. 
During a context switch, the state of the current task is saved, and the state of the next selected task is restored. 
In a common scenario, the timer interrupt handler periodically checks the remaining time slice. 
When the time slice expires, the `RESCHED` flag is set. 
The context switch occurs when a checkpoint is encountered as the task transitions from kernel space to userspace. 
The chosen task then resumes execution while the previous task is temporarily suspended.
The image below serves as an illustrative example, considering that our QEMU configuration does not include a second CPU.

<p align="center"><img src="images/process/scheduler-and-run-queue.png" /></p>

<details><summary> More Details </summary>

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
  
```
+--------------------------+
| fttmr010_timer_interrupt |
+------|-------------------+
       |
       +--> call ->event_handler(), e.g.,
            +----------------------+
            | tick_handle_periodic |
            +-----|----------------+
                  |    +---------------+
                  +--> | tick_periodic |
                       +---|-----------+
                           |    +----------------------+
                           +--> | update_process_times |
                                +-----|----------------+
                                      |    +----------------+
                                      +--> | scheduler_tick | update rq clock, call ->task_tick()
                                           +----------------+ balance runqueues if necessary
```
  
```
+----------------+                                                                                           
| scheduler_tick | : update rq clock, call ->task_tick(), balance runqueues if necessary                     
+---|------------+                                                                                           
    |    +-----------------+                                                                                 
    |--> | update_rq_clock |                                                                                 
    |    +-----------------+                                                                                 
    |                                                                                                        
    |--> call ->task_tick(), e.g.,                                                                             
    |    +----------------+                                                                                  
    |    | task_tick_fair | update vruntime, resched current task if necessary                               
    |    +----------------+                                                                                  
    |    +----------------------+                                                                            
    +--> | trigger_load_balance | : if it's about time, balance loading between the busiest rq and this one  
         +-----|----------------+                                                                            
               |                                                                                             
               |--> if it's time to do balance again                                                         
               |                                                                                             
               |        +---------------+                                                                    
               +------> | raise_softirq | label SCHED_SOFTIRQ and somewhere will call run_rebalance_domains()
                        +---------------+                                                                    
```
  
</details>

### Context Switch

Taking the ARM processor as an example, the CPU consists of general-purpose registers and specific coprocessors. 
Of particular interest is the table translation base register (`TTBR`), which plays a crucial role in page table translation. 
The `TTBR` always points to the page table of the currently executing task, enabling the CPU to efficiently access the mapped page frames in memory.

To manage the architectural-specific fields of a task, such as the CPU context and regular registers, the thread info structure is utilized. 
During a context switch, the scheduler logic performs the following steps:

1. Updates the `TTBR` to point to the new page table associated with the next task.
2. Backs up the CPU's standard registers to the thread info structure, preserving the current task's state.
3. Restores the registers from the next task, allowing it to resume execution seamlessly.

<p align="center"><img src="images/process/context-switch.png" /></p>

<details><summary> More Details</summary>

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
  
```
struct thread_info {
    unsigned long       flags;  // TIF_SIGPENDING: signal pending
                                // TIF_NEED_RESCHED: rescheduling necessary
    int         preempt_count;  // for preemption, which is disabled in our config
    struct task_struct  *task;  // point to the task_struct
    __u32           cpu;        // where the process is running on
    struct cpu_context_save cpu_context;

};
```
                                         
```
     low addr  +--+-----------+                
               |  |thread_info|                
               |  +-----------+                
           ^   |  |0x57AC6E9D | stack end magic
           |   |  +-----------+                
           |   |           |                   
           |   |           |                   
           |   |           |                   
           |   +-----------+                   
           |   |           |                   
           |   |           |                   
           |   |           |                   
           |   |  +-----------+                
           |   |  |  pt_regs  |                
 stack  ---+   |  +-----------+                
               |  |  reserved | 8 bytes        
    high addr  +--+-----------+                    
```
  
```
+----------+
| schedule | : select next task and switch to it
+--|-------+
   |    +-------------------+
   |--> | sched_submit_work | if there's io requests, submit them before sleeping
   |    +-------------------+
repeat
   |    +------------+
   +--> | __schedule | : select next task and switch to it
   |    +--|---------+
   |       |
   |       |--> if there's pending signal
   |       |
   |       |------> task->__state = 'RUNNING'
   |       |
   |       |--> else
   |       |
   |       |        +-----------------+
   |       |------> | deactivate_task | (prev->on_rq is set here,
   |       |        +-----------------+ and later put_prev_task will add it to rq accordingly)
   |       |    +----------------+
   |       |--> | pick_next_task | pick next and enqueue prev
   |       |    +----------------+
   |       |    +------------------------+
   |       |--> | clear_tsk_need_resched | prev
   |       |    +------------------------+
   |       |    +----------------+
   |       +--> | context_switch | switch page table and cpu registers
   |            +----------------+
   |
   +--> go to 'repeat' if need resched                                      
```
  
```
+----------------+                                                                 
| context_switch | : switch page table and cpu registers                           
+---|------------+                                                                 
    |    +---------------------+                                                   
    |--> | prepare_task_switch | next->on_cpu = 1                                  
    |    +---------------------+                                                   
    |                                                                              
    |--> if next task has userspace maps                                           
    |                                                                              
    |        +--------------------+                                                
    |------> | switch_mm_irqs_off | switch to next task's page table               
    |        +--------------------+                                                
    |    +-----------+                                                             
    |--> | switch_to |                                                             
    |    +--|--------+                                                             
    |       |    +-------------+                                                   
    |       +--> | __switch_to | : save prev's registers, and load next's registers
    |            +---|---------+                                                   
    |                |                                                             
    |                |--> save registers of 'prev' to stack                        
    |                |                                                             
    |                +--> load registers of 'next' from stack                      
    |                                                                              
    |    +--------------------+                                                    
    +--> | finish_task_switch | prev->on_cpu = 0                                   
         +--------------------+                                                    
```
  
</details>

### Load Balance
  
A task is assigned a load attribute to indicate its level of importance, and the run queue aggregates the load values of all tasks within it, providing an indication of its overall busyness. 
In order to balance the load across queues, several methods have been introduced.

When a task is newly created or awakened, the scheduler mechanism prioritizes selecting a relatively idle queue to accommodate it. 
This ensures that the workload is evenly distributed among the available queues. 
Additionally, active load balancing is triggered periodically by the timer interrupt handler. 
This mechanism initiates a request that is ultimately handled by the soft IRQ framework, allowing for load balancing operations to be carried out.

It is important to consider that migrating tasks between queues incurs a cost, particularly due to the potential data remaining in the local CPU cache. 
Therefore, load balancing operations are performed with careful consideration of these factors to optimize the overall system performance.
  
<details><summary> More Details </summary>
  
```
struct sched_entity {
    struct load_weight      load;           // weight of the task
    struct rb_node          run_node;       // tree node
    unsigned int            on_rq;          // indicate whether the task is on rq or not
    u64             exec_start;             // indicate when the task starts running
    u64             sum_exec_runtime;       // sum of past running time
    u64             vruntime;               // sum of curr running time
    u64             prev_sum_exec_runtime;  // copy of sum_exec_runtime, for preemption handling
}
```
  
```
+-----------------+                                   
| set_load_weight | : set task's weight and inv_weight
+----|------------+                                   
     |                                                
     |--> if policy is idle                           
     |                                                
     |------> specially set weight and return         
     |                                                
     |--> if it's in cfs sched class                  
     |                                                
     |        +---------------+                       
     |------> | reweight_task |                       
     |        +---------------+                       
     |                                                
     |--> else                                        
     |                                                
     +------> set weight and inv_weight               
```
  
```
+-----------------------+                                                                                 
| run_rebalance_domains | : if it's about time, balance loading between the busiest rq and this one       
+-----|-----------------+                                                                                 
      |    +-------------------+                                                                          
      |--> | nohz_idle_balance | (idle balance related, skip for now)                                     
      |    +-------------------+                                                                          
      |    +-------------------+                                                                          
      +--> | rebalance_domains | : if it's about time, balance loading between the busiest rq and this one
           +----|--------------+                                                                          
                |                                                                                         
                |--> for each domain (probably only 1 to us)                                              
                |                                                                                         
                |        +-------------------------+                                                      
                |------> | get_sd_balance_interval | determine balance 'interval'                         
                |        +-------------------------+                                                      
                |                                                                                         
                |------> if it's time to do balance                                                       
                |                                                                                         
                |            +--------------+                                                             
                +----------> | load_balance | move tasks from the busiest rq to this one                  
                             +--------------+                                                             
```
  
```
+--------------+                                                             
| load_balance | : move tasks from the busiest rq to this one, or ask 'cpu_stopper_thread' to help                
+---|----------+                                                             
    |                                                                        
    |--> set up parameters                                                   
    |                                                                        
    |    +--------------------+                                              
    |--> | find_busiest_group |                                              
    |    +--------------------+                                              
    |    +--------------------+                                              
    |--> | find_busiest_queue |                                              
    |    +--------------------+                                              
    |    +--------------+                                                    
    |--> | detach_tasks | remove enough tasks from rq and add to another list
    |    +--------------+                                                    
    |                                                                        
    |--> if we did detatch tasks                                             
    |                                                                        
    |        +--------------+                                                
    +------> | attach_tasks | move tasks to dst rq, preemption might happen  
             +--------------+                                                
```
  
```
+--------------+                                                      
| detach_tasks | : remove enough tasks from rq and add to another list
+---|----------+                                                      
    |                                                                 
    |--> while rq still has task                                      
    |                                                                 
    |        +-----------------+                                      
    |------> | list_last_entry | get last task on the list            
    |        +-----------------+                                      
    |                                                                 
    |------> break loop if condition is met                           
    |                                                                 
    |        +-------------+                                          
    |------> | detach_task | remove task from rq                      
    |        +-------------+                                          
    |        +----------+                                             
    +------> | list_add | move task to another list                   
             +----------+                                             
```
  
```
+--------------------+                                                                     
| cpu_stopper_thread | : handler work on list, e.g., move task to new rq                   
+----|---------------+                                                                     
     |                                                                                     
     |--> remove the first work from list (might be empty)                                 
     |                                                                                     
     |--> if work                                                                          
     |                                                                                     
     |------> set up 'stoper' from work                                                    
     |                                                                                     
     |------> call work->fn(), e.g.,                                                       
     |        +--------------------+                                                       
     |        | migration_cpu_stop | deactivate task from old rq, and activate it on new rq
     |        +--------------------+                                                       
     |                                                                                     
     +------> reset 'stoper'                                                               
```
  
</details>
  
### Low Latency
  
When the `RESCHED` flag is raised, it does not guarantee immediate stepping down of the running task, resulting in increased latency until scheduling occurs. 
Unlike waiting for resources, there are situations where a task simply takes a longer time to complete its function. 
To address this, an enhancement is to proactively yield the task's execution before entering the time-consuming main body or within a loop.

By breaking down these potentially time-consuming functions into smaller, more finely-grained execution pieces, latency can be improved. 
This approach allows for better responsiveness and ensures that other tasks have a chance to run, minimizing the overall impact on system performance.
  
<details><summary> More Details </summary>
  
```
+--------------+                                                   
| cond_resched | : perform context switch if RESCHEd flag is set   
+---|----------+                                                   
    |    +---------------+                                         
    +--> | _cond_resched |                                         
         +---|-----------+                                         
             |    +----------------+                               
             +--> | __cond_resched |                               
                  +---|------------+                               
                      |                                            
                      |--> if it should resched                    
                      |                                            
                      |        +-------------------------+         
                      +------> | preempt_schedule_common | schedule
                               +-------------------------+         
```
  
```
+-------------+                               
| might_sleep | : do nothing bc of our configs
+-------------+                               
```
  
### Preemption

Let's skip this topic since it's disabled in the OpenBMC kernel.
  
</details>
  
## <a name="priority-and-class"></a> Priority and Class

To optimize a task's priority and ensure it receives special attention from the system, we have the flexibility to adjust its nice value or even its scheduling class. 
The scheduling class determines how tasks interact with sub-queues within the run queue. 
Here are the available scheduling classes, listed in descending order of priority:

- Stop class: This class is dedicated to the kernel and consists of a single kthread responsible for load balancing and task migration.
- Deadline class: This class is designed for user tasks that have specific execution deadlines and require guaranteed completion within the specified timeframe.
- Real-time class: User tasks in this class are prioritized over regular tasks but do not have strict deadlines like those in the deadline class.
- Fair class: The majority of tasks fall into this category. Tasks in the fair class have the opportunity to run at some point, as long as there are no active tasks in higher priority classes.
- Idle class: This class is designed for power-saving purposes. It consists of a single kthread that maximizes power-saving mechanisms when there are no other tasks of higher importance.

By carefully assigning tasks to the appropriate scheduling class, we can effectively control their priority and ensure that they receive the desired level of attention from the system.

<p align="center"><img src="images/process/priority-and-class.png" /></p>

In the process of selecting the next candidate for execution, the scheduler starts from the highest priority class and checks if there is at least one active task in that class.

- If there is an active task, it is ready for execution.
- If there are no active tasks in that class, the scheduler advances to the next class and repeats the process.

The resident stop-class kthread, although present, remains inactive unless a migration request is received. 
Therefore, it is not considered for selection most of the time. 
The focus is primarily on the fair and real-time classes. 
Even in a highly optimized and efficient system, the idle-class kthread is still available as the last choice.

It's important to note that during the next context switch, the procedure starts again from the stop class rather than continuing from the previously visited scheduling class. 
This ensures a fair and comprehensive evaluation of all available tasks before making the next selection.
Each scheduling class implements various functions, such as enqueue and dequeue, to cater to their unique design differences and requirements.

<p align="center"><img src="images/process/sub-queue.png" /></p>

<details><summary> More Details </summary>

```
    prio   kernel           user            nice                        
            view            view            value                       
                                                                        
      ^      | |             | |                                        
 high |      | | < 0         | | < 139?                  deadline class 
      |   --------------------------------------------------------------
      |      | | 0           | | 139                                    
      |      | |             | |                                        
      |      | |             | |                                        
      |      | |             | |                                        
      |      | |             | |                                        
      |      | |             | |                                        
      |      | |             | |                         realtime class 
      |      | |             | |                                        
      |      | |             | |                                        
      |      | |             | |                                        
      |      | |             | |                                        
      |      | |             | |                                        
      |      | |             | |                                        
      |      | |             | |                                        
      |      | | 99          | | 40                                     
      |   --------------------------------------------------------------
      |      | | 100         | | 39          | | -20                    
      |      | |             | |             | |                        
      |      | |             | |             | |                        
      |      | |             | |             | |           fair class   
      |      | |             | |             | |                        
      |      | |             | |             | |                        
      |      | |             | |             | |                        
  low |      +-+ 139         +-+ 0           +-+ 19                     
```

```
struct rq {
    unsigned int        nr_running;     // number of tasks on the run queue
    struct cfs_rq       cfs;            // sub runqueue
    struct rt_rq        rt;             // sub runqueue
    struct dl_rq        dl;             // sub runqueue
    struct task_struct __rcu    *curr;  // point to the current running task
    struct task_struct  *idle;          // point to the idle task of the runqueue
    u64         clock;                  // per runqueue clock
```
  
```
struct sched_class {
    void (*enqueue_task) (struct rq *rq, struct task_struct *p, int flags); 
    // add the task into the rq, e.g., wake up
  
    void (*dequeue_task) (struct rq *rq, struct task_struct *p, int flags);
    // remove the task from the rq, e.g., change prio
  
    void (*yield_task)   (struct rq *rq);                                   
    // yield the cpu to other tasks
  
    void (*check_preempt_curr)(struct rq *rq, struct task_struct *p, int flags);  
    // check if current task should be preempted
  
    struct task_struct *(*pick_next_task)(struct rq *rq);                   
    // select the next task to run on the cpu
  
    void (*put_prev_task)(struct rq *rq, struct task_struct *p);            
    // prepare to withdraw cpu control from current task
  
    void (*set_next_task)(struct rq *rq, struct task_struct *p, bool first);     
  
    void (*task_tick)(struct rq *rq, struct task_struct *p, int queued);    
    // called when timer interrupt happens
```

```
+----------------+
| pick_next_task | ： enqueue prev, select next task for running and dequeue it
+---|------------+
    |    +------------------+
    +--> | __pick_next_task | ： enqueue prev, select next task for running and dequeue it
         +----|-------------+
              |
              +--> if prev sched class <= fair sched class (optimal case)
              |
              |        +---------------------+
              +------> | pick_next_task_fair | enqueue prev, select next task for running and dequeue it
              |        +---------------------+
              |
              +--> else
              |
              +------> for each scheduling class, call its ->pick_next_task, return the first found
```
 
```
struct task_struct {
    int             prio;                     // it might boost temporarily
    int             static_prio;              // the initial prio and can be changed by 'nice'
    int             normal_prio;              // calculated from static prio, and inherited by children
    unsigned int            rt_priority;      // real-time priority: 0 to 99

    const struct sched_class    *sched_class; // scheduler class of the task
    struct sched_entity     se;               // schedulable entity
    struct sched_rt_entity      rt;
    struct sched_dl_entity      dl;
    unsigned int            policy;           // NORMAL, BATCH, IDLE, RR, FIFO, DEADLINE
    cpumask_t           cpus_mask;            // specify which cpu the task can run
```
  
```
struct sched_rt_entity {
    struct list_head        run_list;   // list node
    unsigned int            time_slice; // remaining time for execution
}
```
  
```
+----------+                                                           
| sys_nice | : adjust task prio                                        
+--|-------+                                                           
   |    +---------------+                                              
   +--> | set_user_nice | :
        +---|-----------+                                              
            |                                                          
            |--> if current task is queued                             
            |                                                          
            |        +--------------+                                  
            |------> | dequeue_task |                                  
            |        +--------------+                                  
            |                                                          
            +--> if current task is running                            
            |                                                          
            |        +---------------+                                 
            |------> | put_prev_task |                                 
            |        +---------------+                                 
            |                                                          
            |--> set ->static_prio based on nice value                 
            |                                                          
            |--> adjust ->normal_prio and ->prio based on ->static_prio
            |                                                          
            |--> if current task was queued                            
            |                                                          
            |        +--------------+                                  
            |------> | enqueue_task |                                  
            |        +--------------+                                  
            |                                                          
            |--> if current task was running                           
            |                                                          
            |        +---------------+                                 
            |------> | set_next_task |                                 
            |        +---------------+                                 
            |                                                          
            +--> call ->prio_changed(), e.g.,                          
                 +-------------------+                                 
                 | prio_changed_fair |                                 
                 +-------------------+                                 
```
  
```
+----------------+                                                         
| effective_prio | : get effective prio                                    
+---|------------+                                                         
    |    +-------------+                                                   
    |--> | normal_prio |                                                   
    |    +---|---------+                                                   
    |        |    +---------------+                                        
    |        +--> | __normal_prio | determine prio based on policy and nice
    |             +---------------+                                        
    |                                                                      
    |--> ->normal_prio = return value                                      
    |                                                                      
    |--> if it's a regular task                                            
    |                                                                      
    |------> return ->normal_prio                                          
    |                                                                      
    |--> else                                                              
    |                                                                      
    +------> return ->prio (might get boosted temporarily)                 
```
  
```
+------------------------+                                                                 
| sys_sched_setscheduler | : set schedule-related fields in task                           
+-----|------------------+                                                                 
      |    +-----------------------+                                                       
      +--> | do_sched_setscheduler | :
           +-----|-----------------+                                                       
                 |    +----------------+                                                   
                 |--> | copy_from_user | copy param from userspace                         
                 |    +----------------+                                                   
                 |    +---------------------+                                              
                 |--> | find_process_by_pid | find task from pid value                     
                 |    +---------------------+                                              
                 |    +--------------------+                                               
                 +--> | sched_setscheduler |                                               
                      +----|---------------+                                               
                           |    +---------------------+                                    
                           +--> | _sched_setscheduler | set schedule-related fields in task
                                +---------------------+                                    
```
  
```
+-----------------------+                                                                            
| sys_sched_setaffinity | : set cpu affinity of task                                                 
+-----|-----------------+                                                                            
      |    +-------------------+                                                                     
      |--> | get_user_cpu_mask | copy cpumask from userspace to the newly generated one              
      |    +-------------------+                                                                     
      |    +-------------------+                                                                     
      +--> | sched_setaffinity | :
           +----|--------------+                                                                     
                |    +---------------------+                                                         
                |--> | find_process_by_pid | find task from pid value                                
                |    +---------------------+                                                         
                |    +---------------------+                                                         
                +--> | __sched_setaffinity | :
                     +-----|---------------+                                                         
                           |                                                                         
                           |--> new mask = arg mask & current mask                                   
                           |                                                                         
                           |    +------------------------+                                           
                           +--> | __set_cpus_allowed_ptr | :
                                +-----|------------------+                                           
                                      |    +-------------------------------+                         
                                      +--> | __set_cpus_allowed_ptr_locked | :
                                           +-------|-----------------------+                         
                                                   |    +-----------------------+                    
                                                   |--> | __do_set_cpus_allowed | set tasl->cpus_mask
                                                   |    +-----------------------+                    
                                                   |    +------------------+                         
                                                   +--> | affine_move_task |                         
                                                        +------------------+                         
```
  
</details>

### Fair Class

The Complete Fair Scheduler (`CFS`) is a comprehensive scheduling algorithm that is applicable to various utilities, applications, and kernel threads. 
It employs a red-black tree to manage sub-queues for each scheduling class, where tasks are sorted based on their accumulated virtual runtime. 
The tasks with smaller runtimes are positioned on the left side of the tree.

To ensure fairness, the `CFS` prioritizes tasks with the smallest virtual runtime during task selection. 
The virtual runtime is calculated by considering the task's execution time and priority.

In the `CFS`, tasks with higher priority have smaller virtual runtimes, resulting in a slower accumulation of runtime. 
As a result, they tend to remain on the left side of the red-black tree for a longer duration. 
On the other hand, tasks with lower priority have larger virtual runtimes, leading to a faster accumulation of runtime and a higher likelihood of moving towards the right side of the tree.

When a task's priority is increased through "nice" adjustments, it is given more opportunities for execution without necessarily extending its timeslice. 
However, the virtual runtime of the task continues to strictly increase, ensuring that other typical or low-priority tasks will eventually have their chance to run. 
    
<p align="center"><img src="images/process/fair-class.png" /></p>
  
<details><summary> More Details </summary>
  
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
                                                                   
                                                                   
         the task with a higher 'nice' value moves rightward easily 
                      ------------------------------->             
                                                                   
                                                                   
         the task with a lower 'nice' value stays left side longer
                      <-------------------------------             
```

```
struct cfs_rq {
    struct load_weight  load;               // total load of the tasks
    unsigned int        nr_running;         // number of runnable tasks
    u64         min_vruntime;               // minimum vruntime of all tasks
    struct rb_root_cached   tasks_timeline; // rb tree
    struct sched_entity *curr;              // point to currently running task
}
```
                                                       
```
DEFINE_SCHED_CLASS(fair) = {
    .enqueue_task       = enqueue_task_fair,
    .dequeue_task       = dequeue_task_fair,
    .yield_task     = yield_task_fair,
    .yield_to_task      = yield_to_task_fair,
    .check_preempt_curr = check_preempt_wakeup,
    .pick_next_task     = __pick_next_task_fair,
    .put_prev_task      = put_prev_task_fair,
    .set_next_task          = set_next_task_fair,
    .balance        = balance_fair,
    .pick_task      = pick_task_fair,
    .select_task_rq     = select_task_rq_fair,
    .migrate_task_rq    = migrate_task_rq_fair,
    .rq_online      = rq_online_fair,
    .rq_offline     = rq_offline_fair,
    .task_dead      = task_dead_fair,
    .set_cpus_allowed   = set_cpus_allowed_common,
    .task_tick      = task_tick_fair,
    .task_fork      = task_fork_fair,
    .prio_changed       = prio_changed_fair,
    .switched_from      = switched_from_fair,
    .switched_to        = switched_to_fair,
    .get_rr_interval    = get_rr_interval_fair,
    .update_curr        = update_curr_fair,
}
```
                                                       
```
+-------------------+                                 
| enqueue_task_fair | : add task to rb tree           
+----|--------------+                                 
     |                                                
     |--> if flag specifies 'wakeup'                  
     |                                                
     |        +--------------+                        
     |------> | place_entity | update se's vruntime   
     |        +--------------+                        
     |                                                
     |--> if it's not the curr                        
     |                                                
     |        +----------------+                      
     +------> | enqueue_entity | add entity to rb tree
              +----------------+                      
```
                                                       
```
+------------------+                                                                                          
| __enqueue_entity | :
+------------------+                                                                                          
          |                                                                                                   
          |                                                                                                   
          |-- find the right place in the fair class queue (tree) by using 'vruntime' as key                  
          |                                                                                                   
          |   +--------------+                                                                                
          +-- | rb_link_node | insert the task entity into the tree                                           
              +--------------+            
```
                        
```
 +----------------------+                                                       
 | check_preempt_wakeup | : set current 'need_resched' if it should be preempted
 +-----|----------------+                                                       
       |                                                                        
       |--> return if current need resched                                      
       |                                                                        
       |--> go to 'preempt' if current is idle task                             
       |                                                                        
       |    +-------------+                                                     
       |--> | update_curr |                                                     
       |    +-------------+                                                     
       |    +-----------------------+                                           
       |--> | wakeup_preempt_entity |                                           
       |    +-----------------------+                                           
       |                                                                        
       |--> go to 'preempt' if curr vruntime > p vruntime                       
 preempt                                                                        
       |    +--------------+                                                    
       +--> | resched_curr |                                                    
            +--------------+                                                    
```
  
```
 +---------------------+                                                                
 | pick_next_task_fair | : enqueue prev, select next task for running and dequeue it
 +-----|---------------+                                                                
       |                                                                                
       |--> return if no runnable task                                                  
       |                                                                                
       +--> if prev exists                                                              
       |                                                                                
       |        +---------------+                                                       
       |------> | put_prev_task | call class->put_prev_task(), e.g., add task back to rq
       |        +---------------+                                                       
       |    +------------------+                                                        
       |--> | pick_next_entity | pick next proper task for running                      
       |    +------------------+                                                        
       |    +-----------------+                                                         
       |--> | set_next_entity | dequeue if it's on rq, cfs_rq->curr = entity            
       |    +-----------------+                                                         
       |    +-----------+                                                               
       +--> | list_move | move task to mru (most) list of rq                            
            +-----------+                                                               
```
                        
```
+----------------+                                                         
| task_tick_fair | : update vruntime, resched current task if necessary    
+---|------------+                                                         
    |    +-------------+                                                   
    |--> | entity_tick | update vruntime, resched current task if necessary
    |    +-------------+                                                   
    |    +----------------+                                                
    +--> | task_tick_core | (disabled config)                              
         +----------------+                                                
```
  
```
+-------------+                                                                                              
| entity_tick | : update vruntime, resched current task if necessary                                         
+---|---------+                                                                                              
    |    +-------------+                                                                                     
    |--> | update_curr | update vruntime and related fields of current task                                  
    |    +-------------+                                                                                     
    |    +-----------------+                                                                                 
    |--> | update_load_avg | update average load                                                             
    |    +-----------------+                                                                                 
    |                                                                                                        
    |--> if queued                                                                                           
    |                                                                                                        
    |        +--------------+                                                                                
    |------> | resched_curr | set 'NEED_RESCHED' on task                                                     
    |        +--------------+                                                                                
    |                                                                                                        
    |------> return                                                                                          
    |                                                                                                        
    |--> if there's other task(s) in the rq                                                                  
    |                                                                                                        
    |        +--------------------+                                                                          
    +------> | check_preempt_tick | check a few conditions, and set 'NEED_RESCHED' on rq current if necessary
             +--------------------+                                                                          
```
  
```
 +-------------+                                                                                          
 | update_curr | : update vruntime and related fields of current task
 +------|------+                                                                                          
        |                                                                                                 
        |-- calculate the delta runtime                                                                   
        |                                                                                                 
        |  +-----------------+                                                                            
        +--| calc_delta_fair | calculate the adjusted runtime (vruntime)                                  
        |  +-----------------+                                                                            
        |                                                                                                 
        +-- accumulate the vruntime into the task entity
```

```
+--------------+                                                
| resched_curr | : set 'NEED_RESCHED' on task                   
+---|----------+                                                
    |                                                           
    |--> return if current need resched                         
    |                                                           
    |--> if target rq == this rq                                
    |                                                           
    |        +----------------------+                           
    |------> | set_tsk_need_resched | set 'NEED_RESCHED' on task
    |        +----------------------+                           
    |                                                           
    |------> return                                             
    |                                                           
    |    +------------------------+                             
    |--> | set_nr_and_not_polling |                             
    |    +------------------------+  set 'NEED_RESCHED' on task 
    |    +---------------------+                                
    +--> | smp_send_reschedule | (probably do nothing)          
         +---------------------+                                
```
                        
```
+--------------------+                                                                            
| check_preempt_tick | : check a few conditions, and set 'NEED_RESCHED' on rq current if necessary
+----|---------------+                                                                            
     |    +-------------+                                                                         
     |--> | sched_slice | calc how much time I can run                                            
     |    +-------------+                                                                         
     |                                                                                            
     |--> calc how much time I've run                                                             
     |                                                                                            
     |--> if it's already exceeded                                                                
     |                                                                                            
     |        +--------------+                                                                    
     |------> | resched_curr | set 'NEED_RESCHED' on task                                         
     |        +--------------+                                                                    
     |                                                                                            
     |------> return                                                                              
     |                                                                                            
     |    +---------------------+                                                                 
     |--> | __pick_first_entity | select the leftmost task                                        
     |    +---------------------+                                                                 
     |                                                                                            
     |--> if curr vruntime < next vruntime                                                        
     |                                                                                            
     |------> return                                                                              
     |                                                                                            
     |--> if curr vruntime >> next vruntime                                                       
     |                                                                                            
     |        +--------------+                                                                    
     +------> | resched_curr | set 'NEED_RESCHED' on task                                         
              +--------------+                                                                    
```
                        
```
+-------------+                                                     
| sched_slice | : calc time slice for the given sched entity        
+---|---------+                                                     
    |    +----------------+                                         
    |--> | __sched_period | get total time slice                    
    |    +----------------+                                         
    |    +--------------+                                           
    +--> | __calc_delta | calc time slice for the given sched entity
         +--------------+                                           
```
 
```
+----------------+                                                                 
| task_fork_fair | : determine vruntime of new task, resched current task if needed
+---|------------+                                                                 
    |                                                                              
    |--> if curr task exists                                                       
    |                                                                              
    |------> new task vruntime = cur task vruntime                                 
    |                                                                              
    |    +--------------+                                                          
    |--> | place_entity | adjust new task vruntime                                 
    |    +--------------+                                                          
    |                                                                              
    |--> if child should run first                                                 
    |                                                                              
    |------> swap the runtime of cur and new tasks                                 
    |                                                                              
    |        +--------------+                                                      
    |------> | resched_curr |                                                      
    |        +--------------+                                                      
    |                                                                              
    +--> adjust new vruntime                                                       
```
  
</details>
  
### Real-Time Class
  
(TBD)
                        
<details><summary> More Details</summary>
  
```
DEFINE_SCHED_CLASS(rt) = {
    .enqueue_task       = enqueue_task_rt,
    .dequeue_task       = dequeue_task_rt,
    .yield_task     = yield_task_rt,
    .check_preempt_curr = check_preempt_curr_rt,
    .pick_next_task     = pick_next_task_rt,
    .put_prev_task      = put_prev_task_rt,
    .set_next_task          = set_next_task_rt,
    .balance        = balance_rt,
    .pick_task      = pick_task_rt,
    .select_task_rq     = select_task_rq_rt,
    .set_cpus_allowed       = set_cpus_allowed_common,
    .rq_online              = rq_online_rt,
    .rq_offline             = rq_offline_rt,
    .task_woken     = task_woken_rt,
    .switched_from      = switched_from_rt,
    .find_lock_rq       = find_lock_lowest_rq,
    .task_tick      = task_tick_rt,
    .get_rr_interval    = get_rr_interval_rt,
    .prio_changed       = prio_changed_rt,
    .switched_to        = switched_to_rt,
    .update_curr        = update_curr_rt,
}
```
  
```
+-------------------+                                               
| pick_next_task_rt | : select the next task in rt_rq               
+----|--------------+                                               
     |    +--------------+                                          
     |--> | pick_task_rt | select the next task in rt_rq            
     |    +--------------+                                          
     |    +------------------+                                      
     +--> | set_next_task_rt | push some other rt tasks to other rqs
          +------------------+                                      
```
  
```
+--------------+                                                      
| task_tick_rt | : resched if it's RR and runs out of time slice      
+---|----------+                                                      
    |    +----------------+                                           
    |--> | update_curr_rt | update execution time, resched if needed  
    |    +----------------+                                           
    |    +----------+                                                 
    |--> | watchdog | ???                                             
    |    +----------+                                                 
    |                                                                 
    |--> return if policy != SCHED_RR                                 
    |                                                                 
    |--> time slice --                                                
    |                                                                 
    |--> return if it's still positive                                
    |                                                                 
    |--> refill time slice to sched_rr_timeslice (100ms) if it expires
    |                                                                 
    |--> if there's other rt task of the same prio                    
    |                                                                 
    |        +-----------------+                                      
    |------> | requeue_task_rt | add to the end of list               
    |        +-----------------+                                      
    |        +--------------+                                         
    +------> | resched_curr |                                         
             +--------------+                                         
```
  
```
+----------------+                                             
| update_curr_rt | : update execution time, resched if needed  
+---|------------+                                             
    |                                                          
    |--> return if current isn't an rt task                    
    |                                                          
    |--> update current's execution time                       
    |                                                          
    |    +---------------------------+                         
    |--> | sched_rt_runtime_exceeded | check if runtime exceeds
    |    +---------------------------+                         
    |                                                          
    |--> if it does                                            
    |                                                          
    |        +--------------+                                  
    +------> | resched_curr |                                  
             +--------------+                                  
```
  
```
+-----------------+                                           
| sys_sched_yield | :
+----|------------+                                           
     |    +----------------+                                  
     |--> | do_sched_yield |                                  
     |    +----------------+                                  
     |                                                        
     |--> call ->yield_task(), e.g.,                          
     |    +-----------------+                                 
     |    | yield_task_fair | clear buddies and set skip buddy
     |    +-----------------+                                 
     |    +----------+                                        
     +--> | schedule |                                        
          +----------+                                        
```

</details>

## <a name="system-startup"></a> System Startup

Taking OpenBMC as an example, the boot process starts with U-Boot after powering on the system. 
U-Boot then hands over control to the assembly code of the kernel. 
The first C function called is `start_kernel`, which initializes the most fundamental subsystems such as memory, interrupts, and processes. 
Prior to this point, there is no concept of a scheduler, tasks, run queues, or scheduling classes.

Once the initialization of these core frameworks is completed, the running logic wraps itself as the init task (with process ID 0) and forks two additional kernel threads:

- `kernel_init` (with process ID 1): 
  - This thread takes on the responsibility of initializing the remaining subsystems, including file systems, network stacks, IPC frameworks, drivers, and more.

- `kthreadd` (with process ID 2): 
  - The suffix 'd' indicates that it is a daemon. `kthreadd` assists in fulfilling kthread generation requests from `kernel_init` or other threads. 
  - It serves as the parent thread for all other kernel threads except kernel_init.

Meanwhile, the init task, also known as the `swapper`, becomes the idle thread of the stop class on CPU 0. 
As `kernel_init` reaches the end of kernel initialization, it transforms into systemd, which becomes the first user space process. 
Unlike the creation of kthreads, this transformation involves internal context replacement rather than starting a new task. 
`systemd` then spawns various other applications, including the login prompt, making the system fully functional from the users' perspective.

<p align="center"><img src="images/process/task-hierarchy.png" /></p>

<details><summary> More Details </summary>

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

```
+-----------+                                                    
| rest_init | :                                                    
+-----------+                                                    
       |                                                         
       |--- create a kernel thread running function 'kernel_init'
       |                                                         
       |                                                         
       +--- create a kernel thread running function 'kthreadd'   
```
  
```
+----------+
| kthreadd | :
+--|-------+
   |
   +--> endless loop
   |
   +------> if no request on list
   |
   |            +----------+
   +----------> | schedule |
   |            +----------+
   |
   +------> while request list isn't empty
   |
   +----------> remove request from list
   |
   |            +----------------+
   +----------> | create_kthread | clone task, wake it up to run function 'kthread'
                +----------------+                                            |
                                                                              v
                                                                    run argument 'threadfn'
                                                                    e.g., smpboot_thread_fn
                                                                              |
                                                                              v
                                                              endless loop, run the installed
                                                              thread function whenever necessary
```
  
```
+---------+
| kthread | : run argument 'threadfn'
+--|------+
   |
   |--> allocate struct 'kthread' and set up
   |
   |--> complete 'done'
   |
   |    +----------+
   |--> | schedule |
   |    +----------+
   |
   |--> if struct 'kthread' isn't labeled SHOULD_STOP
   |
   |        +------------------+
   |------> | __kthread_parkme | wait till SHOULD_PARK is cleared
   |        +------------------+
   |
   |------> call threadfn()
   |              +-------------------+
   |        e.g., | smpboot_thread_fn | endless loop, run the installed thread function whenever necessary
   |              +-------------------+
   |
   |    +---------+
   +--> | do_exit | <========== might not reach here if the above threadfn doesn't return
        +---------+                                                                                  
```
     
```
+-------------------+                                                                   
| smpboot_thread_fn | : endless loop, run the installed thread function whenever necessary
+----|--------------+                                                                   
     |                                                                                  
     +--> endless loop                                                                  
             +-------------------------------------------------------+                  
             |if kthread should stop                                 |                  
             |                                                       |                  
             |    call ->cleanup() if it exists                      |                  
             |                                                       |                  
             |    return                                             |                  
             +-------------------------------------------------------+                  
             |if kthread should park                                 |                  
             |                                                       |                  
             |    call ->park() if it exists                         |                  
             |                                                       |                  
             |    change status to PARKED                            |                  
             |                                                       |                  
             |    +----------------+                                 |                  
             |    | kthread_parkme | wait till SHOULD_PARK is cleared|                  
             |    +----------------+                                 |                  
             |                                                       |                  
             |    continue                                           |                  
             +-------------------------------------------------------+                  
             |if status is NONE                                      |                  
             |                                                       |                  
             |    call ->setup() if it exists                        |                  
             |                                                       |                  
             |    change status to ACTIVE                            |                  
             |                                                       |                  
             |else if status is PARKED                               |                  
             |                                                       |                  
             |    call ->unpark() if it exists                       |                  
             |                                                       |                  
             |    change status to ACTIVE                            |                  
             +-------------------------------------------------------+                  
             |if kthread doesn't have to run                         |                  
             |                                                       |                  
             |    +----------+                                       |                  
             |    | schedule |                                       |                  
             |    +----------+                                       |                  
             |                                                       |                  
             |else                                                   |                  
             |                              +---------------+        |                  
             |    call ->thread_fn(), e.g., | run_ksoftirqd |        |                  
             |                              +---------------+        |                  
             +-------------------------------------------------------+                  
```
  
```
include/linux/kthread.h
+----------------+
| kthread_create | : ask kthreadd help creating kthread, set its name, sched-related, and cpu mask
+-|--------------+
  |    +------------------------+
  +--> | kthread_create_on_node | : ask kthreadd help creating kthread, set its name, sched-related, and cpu mask
       +------------------------+
                                                         
```
  
```
kernel/kthread.c
+--------------------------+
| __kthread_create_on_node | : ask kthreadd help creating kthread, set its name, sched-related, and cpu mask
+-|------------------------+
  |
  |--> allocate and setup 'create'
  |
  |--> add 'create' to the end of kthread_create_list
  |
  |    +-----------------+
  |--> | wake_up_process | wake up kthreadd_task
  |    +-----------------+
  |    +------------------------------+
  |--> | wait_for_completion_killable | wait for completion
  |    +------------------------------+
  |    +---------------+
  |--> | set_task_comm | set task name
  |    +---------------+
  |    +----------------------------+
  |--> | sched_setscheduler_nocheck | set schedule-related fields in task
  |    +----------------------------+
  |    +----------------------+
  +--> | set_cpus_allowed_ptr | set cpu mask
       +----------------------+
```
  
```
+----------------------------+                                                                           
| sched_setscheduler_nocheck | : set schedule-related fields in task                                     
+------|---------------------+                                                                           
       |    +---------------------+                                                                      
       +--> | _sched_setscheduler | : set schedule-related fields in task
            +-----|---------------+                                                                      
                  |                                                                                      
                  |--> set up 'sched attr'                                                               
                  |                                                                                      
                  |    +----------------------+                                                          
                  +--> | __sched_setscheduler | :
                       +-----|----------------+                                                          
                             |                                                                           
                             |--> dequeue task if it's queued                                            
                             |                                                                           
                             +--> put task if it's running                                               
                             |                                                                           
                             |    +-----------------------+                                              
                             |    | __setscheduler_params | set task policy, static prio, and normal prio
                             |--> +-----------------------+                                              
                             |    +---------------------+                                                
                             |    | __setscheduler_prio | set task sched class and prio                  
                             |--> +---------------------+                                                
                             |                                                                           
                             |--> queue it back if it was queued                                         
                             |                                                                           
                             +--> set task as next if it was running                                     
```

```  
+--------------------------------+
| smpboot_register_percpu_thread | : for each cpu, prepare a kthread running specified hotplug thread
+-------|------------------------+
        |
        +--> for each online cpu
        |
        |        +-------------------------+
        +------> | __smpboot_create_thread | fork a kthread running 'smpboot_thread_fn', park it and call ->create()
        |        +-------------------------+
        |        +-----------------------+
        +------> | smpboot_unpark_thread | clear SHOULD_PARK of that kthread and wake it up
                 +-----------------------+                     
```
                           
```
+-------------------------+                                                                                         
| __smpboot_create_thread | : fork a kthread running 'smpboot_thread_fn', park it and call ->create()                 
+------|------------------+                                                                                         
       |    +-----------------------+                                                                               
       |--> | kthread_create_on_cpu | ask kthreadd to create a kthread running arg threadfn, e.g., smpboot_thread_fn
       |    +-----------------------+                                                                               
       |    +--------------+                                                                                        
       |--> | kthread_park | label SHOULD_PARK on target kthread, which will become PARKED                          
       |    +--------------+                                                                                        
       |                                                                                                            
       +--> call ->create() if it exists                                                                            
```
 
```
# Note that by default, the command pstree isn't built in.
root@romulus:~# pstree -p
systemd(1)-+-avahi-daemon(270)---avahi-daemon(307)
           |-bmcweb(196)
           |-btbridged(206)
           |-button-handler(296)
           |-buttons(227)
           |-dbus-broker-lau(193)---dbus-broker(194)
           |-fru-device(230)
           |-health-monitor(222)
           |-ipmid(312)
           |-login(298)---sh(338)---pstree(352)
           |-mapperx(238)
           |-mboxd(330)
           |-netipmid(331)
           |-nscd(204)
           |-obmc-console-se(203)
           |-obmc-ikvm(263)
           |-obmc-uart-rende(333)
           |-openpower-occ-c(310)
           |-openpower-updat(295)
           |-phosphor-bmc-st(325)
           |-phosphor-certif(209)
           |-phosphor-certif(219)
           |-phosphor-certif(220)
           |-phosphor-chassi(318)
           |-phosphor-downlo(243)
           |-phosphor-dump-m(205)
           |-phosphor-dump-m(268)
           |-phosphor-fru-fa(321)
           |-phosphor-gpio-m(221)
           |-phosphor-host-s(332)
           |-phosphor-hwmon-(299)
           |-phosphor-hwmon-(300)
           |-phosphor-image-(306)
           |-phosphor-invent(232)
           |-phosphor-ldap-c(304)
           |-phosphor-ledcon(250)
           |-phosphor-ledcon(251)
           |-phosphor-ledcon(252)
           |-phosphor-ledman(302)
           |-phosphor-log-ma(233)
           |-phosphor-networ(305)
           |-phosphor-rsyslo(244)
           |-phosphor-settin(241)
           |-phosphor-system(297)
           |-phosphor-time-m(319)
           |-phosphor-user-m(249)
           |-phosphor-versio(326)
           |-power_control.e(207)
           |-rngd(188)
           |-rsyslogd(327)
           |-slpd(223)
           |-systemd-journal(131)
           |-systemd-network(153)
           |-systemd-resolve(151)
           |-systemd-timesyn(152)
           |-systemd-udevd(147)
           `-telemetry(246)
```
  
</details>
  
## <a name="others"></a> Others
  
### Resource Limit
  
(TBD)
  
<details><summary> More Details </summary>
  
| Name              | Value  | Note                               |
| ---               | ---    | ---                                |
| RLIMIT_CPU        | 0      | CPU time in sec                    |   
| RLIMIT_FSIZE      | 1      | Maximum filesize                   |   
| RLIMIT_DATA       | 2      | max data size                      |   
| RLIMIT_STACK      | 3      | max stack size                     |   
| RLIMIT_CORE       | 4      | max core file size                 |   
| RLIMIT_RSS        | 5      | max resident set size              |   
| RLIMIT_NPROC      | 6      | max number of processes            |   
| RLIMIT_NOFILE     | 7      | max number of open files           |   
| RLIMIT_MEMLOCK    | 8      | max locked-in-memory address space |
| RLIMIT_AS         | 9      | address space limit                |   
| RLIMIT_LOCKS      | 10     | maximum file locks held            |   
| RLIMIT_SIGPENDING | 11     | max number of pending signals      |   
| RLIMIT_MSGQUEUE   | 12     | maximum bytes in POSIX mqueues     |   
| RLIMIT_NICE       | 13     | max nice prio allowed to raise to 0-39 for nice level 19 .. -20 |
| RLIMIT_RTPRIO     | 14     | maximum realtime priority          |   
| RLIMIT_RTTIME     | 15     | timeout for RT tasks in us         |   
| RLIM_NLIMITS      | 16     |                                    |   
| RLIM_INFINITY     | (~0UL) |                                    |
  
```
root@romulus:~# cat /proc/self/limits 
Limit                     Soft Limit           Hard Limit           Units     
Max cpu time              unlimited            unlimited            seconds   
Max file size             unlimited            unlimited            bytes     
Max data size             unlimited            unlimited            bytes     
Max stack size            8388608              unlimited            bytes     
Max core file size        unlimited            unlimited            bytes     
Max resident set          unlimited            unlimited            bytes     
Max processes             2770                 2770                 processes 
Max open files            1024                 524288               files     
Max locked memory         65536                65536                bytes     
Max address space         unlimited            unlimited            bytes     
Max file locks            unlimited            unlimited            locks     
Max pending signals       2770                 2770                 signals   
Max msgqueue size         819200               819200               bytes     
Max nice priority         0                    0                    
Max realtime priority     0                    0                    
Max realtime timeout      unlimited            unlimited            us
```
  
```
struct signal_struct {
    struct rlimit rlim[RLIM_NLIMITS];
}

struct rlimit {
    __kernel_ulong_t    rlim_cur;   // soft limit
    __kernel_ulong_t    rlim_max;   // hard limit
};
```

```
+---------------+                           
| sys_getrlimit | : get rlimit              
+---|-----------+                           
    |    +------------+                     
    +--> | do_prlimit | :
    |    +--|---------+                     
    |       |                               
    |       |--> get rlimit from task signal
    |       |                               
    |       |--> if old is provided         
    |       |                               
    |       |------> *old = *rlimit         
    |       |                               
    |       |--> if new is provided         
    |       |                               
    |       +------> *rlimit = *new         
    |                                       
    |    +----------------+                 
    +--> | copy_from_user |                 
         +----------------+                 
```
  
```
+---------------+                           
| sys_setrlimit | : set rlimit              
+---|-----------+                           
    |    +----------------+                 
    +--> | copy_from_user |                 
    |    +----------------+                 
    |    +------------+                     
    +--> | do_prlimit | :
         +--|---------+                     
            |                               
            |--> get rlimit from task signal
            |                               
            |--> if old is provided         
            |                               
            |------> *old = *rlimit         
            |                               
            |--> if new is provided         
            |                               
            +------> *rlimit = *new         
```
  
</details>
  
### PID Structure
  
(TBD)
  
<details><summary> More Details </summary>
  
```
                         task_struct                                                     
                     +-----------------+                                                 
                     |       pid       | task id, a.k.a. thread id in user space         
                     |                 |                                                 
                     |      tgid       | thread group id, a.k.a. process id in user space
                     |                 |                                                 
       +---------------- thread_pid    |                                                 
       |             |                 |                                                 
       |             | pid_links[PID]  | list node                                       
       |             |                 |                                                 
       |        +------pid_links[TGID] | list node                                       
       |        |    |                 |                                                 
       |        |    | pid_links[PGID]---list node-------------+                         
       v        |    |                 |                       |                         
                |  +---pid_links[SID]  | list node             |                         
  struct pid    |  | |                 |                       |      struct pid         
+-------------+ |  | |     signal   --------> signal_struct    |    +-------------+      
| tasks[PID]  | |  | +-----------------+      +------------+   |    | tasks[PID]  |      
|             | |  |                          | pids[PID]  |   |    |             |      
| tasks[TGID] --+  |                          |            |   |    | tasks[TGID] |      
|             | <--|----------------------------pids[TGID] |   |    |             |      
| tasks[PGID] |    |                          |            |   +------tasks[PGID] |      
|             |    |                          | pids[PGID]------->  |             |      
| tasks[SID]  |    |                          |            |        | tasks[SID]  |      
+-------------+    |                +-----------pids[SID]  |        +-------------+      
                   |                |         +------------+                             
                   |     struct pid v                                                    
                   |   +-------------+                                                   
                   |   | tasks[PID]  | list head                                         
                   |   |             |                                                   
                   |   | tasks[TGID] | list head                                         
                   |   |             |                                                   
                   |   | tasks[PGID] | list head                                         
                   |   |             |                                                   
                   +-----tasks[SID]  | list head                                         
                       +-------------+                                                   
```
  
```
struct pid
{
    refcount_t count;                     // ref counter
    unsigned int level;                   // the number of namespaces that the struct pid is visible in
    struct hlist_head tasks[PIDTYPE_MAX]; // array of hlist head
    struct upid numbers[1];               // upid array for each level
};
  
struct upid {
    int nr;                   // pid value
    struct pid_namespace *ns; // points to namespace that contains the pid value
};
  
struct pid_namespace {
    struct task_struct *child_reaper; // the task to call wait4 when other tasks terminate
    unsigned int level;               // the depth of namespace (0: root, 1: child, ...)
    struct pid_namespace *parent;     // points to the parent namespace
}
  
enum pid_type
{
    PIDTYPE_PID,
    PIDTYPE_TGID,
    PIDTYPE_PGID,
    PIDTYPE_SID,
    PIDTYPE_MAX,
};
```
  
```
+----------+                                           
| task_pid | return task->thread_pid                   
+----------+                                           
+-----------+                                          
| task_tgid | return task->signal->pids[PIDTYPE_TGID]  
+-----------+                                          
+-----------+                                          
| task_pgrp | return task->signal->pids[PIDTYPE_PGID]  
+-----------+                                          
+--------------+                                       
| task_session | return task->signal->pids[PIDTYPE_SID]
+--------------+                                       
```
  
```
+-----------+                                                               
| pid_nr_ns | get pid value from struct pid based on level of arg namespace 
+-----------+                                                               
+---------+                                                                 
| pid_vnr | get pid value from struct pid based on level of active namespace
+---------+                                                                 
+--------+                                                                  
| pid_nr | get pid value from struct pid based on level of init namespace   
+--------+                                                                  
```
  
```
+----------------+                                                 
| task_pid_nr_ns | get pid value of task in given namespace        
+----------------+                                                 
+-----------------+                                                
| task_tgid_nr_ns | get tgid value of task in given namespace      
+-----------------+                                                
+-----------------+                                                
| task_pgrp_nr_ns | get pgrp value of task in given namespace      
+-----------------+                                                
+--------------------+                                             
| task_session_nr_ns | get session value of task in given namespace
+--------------------+                                             
```
  
```
+-------------+                                                                        
| find_pid_ns | find struct pid by pid value from the given namespace                  
+-------------+                                                                        
+----------+                                                                           
| pid_task | get the first task of the type from struct pid                            
+----------+                                                                           
```
  
```
+---------------------+                                                                
| find_task_by_pid_ns | get the first task of type from the given pid value and ns     
+---------------------+                                                                
+-------------------+                                                                  
| find_task_by_vpid | get the first task of type from the given pid value and active ns
+-------------------+                                                                  
```
  
```
+------------+                                                      
| sys_setsid | : set task special pid = group leader's pid          
+--|---------+                                                      
   |    +-------------+                                             
   +--> | ksys_setsid | :                                             
        +---|---------+                                             
            |                                                       
            |--> get group leader                                   
            |                                                       
            |--> group_leader->signal->leader = 1                   
            |                                                       
            |    +------------------+                               
            +--> | set_special_pids | set task special pid = arg pid
                 +------------------+                               
```
  
```
+------------------+                                                
| set_special_pids | : set task special pid = arg pid               
+----|-------------+                                                
     |                                                              
     |--> get group leader                                          
     |                                                              
     |--> if leader's session != arg pid                            
     |                                                              
     |        +------------+                                        
     |------> | change_pid | change task pid, and attach task to pid
     |        +------------+                                        
     |                                                              
     |--> if leader's pgrp != arg pid                               
     |                                                              
     |        +------------+                                        
     +------> | change_pid | change task pid, and attach task to pid
              +------------+                                        
```
  
```
+------------+                                                 
| change_pid | : change task pid, and attach task to pid       
+--|---------+                                                 
   |    +--------------+                                       
   |--> | __change_pid | change pid, free the old one if unused
   |    +--------------+                                       
   |    +------------+                                         
   +--> | attach_pid | attach task to pid                      
        +------------+                                         
```
  
```
+--------------+                                         
| __change_pid | : change pid, free the old one if unused
+---|----------+                                         
    |                                                    
    |-> replace pid_links[type] = arg pid                
    |                                                    
    |--> return if the old one is in use                 
    |                                                    
    |    +----------+                                    
    +--> | free_pid |                                    
         +----------+                                    
```
 
</details>
  
## <a name="reference"></a> Reference
  
- W. Mauerer, Professional Linux Kernel Architecture
- [J. Corbet, TASK_KILLABLE](https://lwn.net/Articles/288056/)
- [G. Shaw, Reap zombie processes using a SIGCHLD handler](http://www.microhowto.info/howto/reap_zombie_processes_using_a_sigchld_handler.html)
- [G. Maier, Thread Scheduling with pthreads under Linux and FreeBSD](http://www.icir.org/gregor/tools/pthread-scheduling.html)
- [W. Shen, Understanding Linux Kernel Stack](https://wenboshen.org/posts/2015-12-18-kernel-stack.html)
