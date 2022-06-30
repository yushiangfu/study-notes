> Linux version 5.15.0

## Index

- [Introduction](#introduction)
- [Task & Group](#task-and-group)
- [Sending & Receiving](#send-and-receive)
- [System Calls](#system-calls)
- [Reference](#reference)

## <a name="introduction"></a> Introduction

Tasks can accept signals and trigger the corresponding actions or ignore them (except *SIGKILL* and *SIGSTOP*). 
It can be that a task sends the signal to another task, or the kernel itself sends it because of some triggered exceptions. 
Working in the shell environment, more or less, we have the experience of pressing *Ctrl C* or *Ctrl Z* to interrupt or stop the task. 
They are the shortcuts prepared by shell and eventually send signal *SIGINT* or *SIGTSTP* to the foreground task. 
The utility *kill*, which has a scary name, is another tool that helps us send various signals to target tasks, though mainly *SIGKILL* in my usage.

- Signals on ARM
   - arch/arm/include/uapi/asm/signal.h
   
| Name      | Number | Note 1                   | Note 2 |        
| ---       | ---    | ---                      | ---    |    
| SIGHUP    | 1      | hangup                   |        |    
| SIGINT    | 2      | interrupt                | shell: 'Ctrl C' |    
| SIGQUIT   | 3      | quit                     |        |    
| SIGILL    | 4      | illegal                  |        |    
| SIGTRAP   | 5      | trap                     |        |    
| SIGABRT   | 6      | abort                    |        |    
| SIGIOT    | 6      | input/output trap        |        |    
| SIGBUS    | 7      | bus error                |        |    
| SIGFPE    | 8      | floating point exception |        |
| SIGKILL   | 9      | kill                     | handled by kernel |
| SIGUSR1   | 10     | user-defined conditions  | by default it terminates the task |
| SIGSEGV   | 11     | segmentation violation   |        |    
| SIGUSR2   | 12     | user-defined conditions  | by default it terminates the task |
| SIGPIPE   | 13     | pipe                     |        |    
| SIGALRM   | 14     | alarm                    |        |    
| SIGTERM   | 15     | termination              |        |    
| SIGSTKFLT | 16     | stack fault              |        |    
| SIGCHLD   | 17     | child                    |        |    
| SIGCONT   | 18     | continue                 |        |    
| SIGSTOP   | 19     | stop                     | handled by kernel |
| SIGTSTP   | 20     | terminal stop            | shell: 'Ctrl Z' |
| SIGTTIN   | 21     | tty in                   |        |    
| SIGTTOU   | 22     | tty out                  |        |    
| SIGURG    | 23     | urgent                   |        |    
| SIGXCPU   | 24     | exceeds CPU              |        |    
| SIGXFSZ   | 25     | exceeds file size        |        |    
| SIGVTALRM | 26     | virtual alarm            |        |    
| SIGPROF   | 27     | profiling                |        |    
| SIGWINCH  | 28     | window change            |        |    
| SIGIO     | 29     | input/output             |        |    
| SIGPOLL   | SIGIO  | poll                     |        |    
| SIGPWR    | 30     | power failure            |        |    
| SIGSYS    | 31     | syscall                  |        |    
| SIGUNUSED | 31     | unused                   |        |    

## <a name="task-and-group"></a> Task & Group

There's only one task for the traditional single-threaded process, and everything is simple.
It gets its signal-related fields and shares with nobody.
- *signal*
   - Signals sent from the kernel of other tasks, such as utility *kill*, come to here pending eventually.
- *sighand*
   - A list of handlers for each signal can be *ignore*, *default*,' or registered handler.
- blocked
   - Each task can modify this bitmap to indicate which signals it will block.
- *pending*
   - For ptrace mechanism.

When it comes to multi-threaded processes, tasks share the same *signal_struct* and *sighand_struct* because of the flags passed to the syscall *clone*.
That's why some manuals mention that *signals* and *handlers* are process-wide because they are two of the shared resource among threads.

- Figure - single-threaded process

```
                          signal_struct         +---------+      +---------+
                        +----------------+      | pending |      | pending |
                        | shared_pending | <--> | signal  | <--> | signal  |
                        +----------------+      +---------+      +---------+
                                                                            
                             ^                                              
                             |                                              
 pid = n                     |                                              
 tgid = n   task_struct      |                                              
            +---------+      |                                              
            |  signal--------+                                              
            |         |                                                     
            | sighand--------+                                              
            |         |      |                                              
            | blocked |      v                                              
            |         |                                                     
            | pending |   sighand_struct                                    
            +---------+   +------------+                                    
                          | action[64] |                                    
                          +------------+                                    
                                                                            
                          e.g.                                              
                          action[SIGHUP  - 1] = ignore                      
                          action[SIGUSR1 - 1] = default                     
                          action[SIGUSR2 - 1] = registered handler          
```

- Figure - multi-threaded process

```
                          signal_struct         +---------+      +---------+
                        +----------------+      | pending |      | pending |
                        | shared_pending | <--> | signal  | <--> | signal  |
                        +----------------+      +---------+      +---------+
                                                                            
                             ^      ^                                       
                             |      |                                       
 pid = n                     |      |                    pid = n + 1        
 tgid = n   task_struct      |      |      task_struct   tgid = n           
            +---------+      |      |      +---------+                      
            |  signal--------+      +--------signal  |                      
            |         |                    |         |                      
            | sighand--------+      +--------sighand |                      
            |         |      |      |      |         |                      
            | blocked |      v      v      | blocked |                      
            |         |                    |         |                      
            | pending |   sighand_struct   | pending |                      
            +---------+   +------------+   +---------+                      
                          | action[64] |                                    
                          +------------+                                    
                                                                            
                          e.g.                                              
                          action[SIGHUP  - 1] = ignore                      
                          action[SIGUSR1 - 1] = default                     
                          action[SIGUSR2 - 1] = registered handler          
```

## <a name="send-and-receive"></a> Sending & Receiving

The pending signal list consists of the *list* for *sigqueue* and the 64-bit bitmap *signal*. 
Not that important, but please note an offset between signal number and bitmap position, e.g., SIGHUP has the number 1 but corresponds to bit 0. 
When somewhere sends a signal to the target task, the kernel helps allocate and set up the *sigqueue* and append it to the list. 
They are not guaranteed to be processed in the order of *first in first out* because synchronous-type signals might involve. 
Then further sets the bit in bitmap so that it's an easy way to determine if a task can handle the signal by checking it *blocked* bitmap.

- Figure - signal structure

```
  signal_struct                                          
+----------------+                                       
|                |                                       
|                |                                       
| shared_pending |     sigqueue  sigqueue                
|   +--------+   |      +----+    +----+                 
|   |  list <|--------> |    |<-->|    |                 
|   |        |   |      +----+    +----+                 
|   | signal-|-------+                                   
|   +--------+   |   |                                   
+----------------+   |     +--+--+       +--+       +--+ 
                     +---- |63|62| -   - |30| -   - | 0| 
                    bitmap +--+--+       +--+       +--+ 
                                         for        for  
                                        SIGSYS     SIGHUP
                                                         
                                           offset = 1                                           
```

- Code flow of sending a signal

```
 +------------+                                                                                             
 | sys_kernel |                                                                                             
 +------------+                                                                                             
        -                                                                                                   
        -  (many functions in between)                                                                      
        -                                                                                                   
        v                                                                                                   
+---------------+                                                                                           
| __send_signal |                                                                                           
+---|-----------+                                                                                           
    |     +----------------+                                                                                
    |---> | prepare_signal | decide if to ignore the signal                                                 
    |     +----------------+                                                                                
    |                                                                                                       
    |---> determine where to append the signal (for a task or the group)                                    
    |                                                                                                       
    |---> return if it's already pending                                                                    
    |                                                                                                       
    |---> prepare 'sigqueue' and append it to the list                                                      
    |                                                                                                       
    |---> raise the corresponding bit in the bitmap                                                         
    |                                                                                                       
    |     +-----------------+                                                                               
    +---> | complete_signal | if the target task is running, kick it back to the kernel to handle the signal
          +-----------------+ 
```

Every user space task enters kernel space on purpose or involuntarily (e.g., by interrupt) and checks pending signals before the task return to userspace. 
If pending signals exist, the kernel handles them on behalf of the task by:
1. Determine the appropriate signal to operate, and it's not guaranteed to be the first sigqueue in the list.
2. Clear that signal in task bitmap and release the related sigqueue from the list accordingly.
3. (Assume that signal has a decent handler) Fetch that handler information and prepare the frame for the handler on the userspace stack.
4. Once the flow switches back to user mode, it starts from that frame to execute the handler.
5. When the handler finishes, it returns to the kernel-mode through *sys_sigreturn*, set up in *handle_signal()*.
6. If we perform the signal handler in the syscall path, some syscalls might return the interrupt error EINT, and the task needs to retry.

For the case of a multi-threaded process, anyone can help dequeue the pending signal and call the handler on its way back to userspace. 
Therefore we have no idea which task will be the next one that consumes signal before coming back.

- Figure

```
 USR       |      SVC                           
 mode      |      mode                          
           |                                    
           |                                    
  |        |                                    
  |        |                                    
  v        |                                    
  ------------------>                           
           |        |                           
           |        |                           
           |++      v                           
  <---------||-------                           
  |        |++                                  
  |        |  check if the pending signal exists
  v        |                                         
```

- Code flow of receiving a signal

```
+-----------------+                                                                                                         
| do_work_pending |                                                                                                         
+----|------------+                                                                                                         
     |                                                                                                                      
     |---> if need re-schedule                                                                                              
     |                                                                                                                      
     |          +----------+                                                                                                
     |          | schedule |                                                                                                
     |          +----------+                                                                                                
     |                                                                                                                      
     +---> elif at least one pending signal                                                                                 
                                                                                                                            
                +-----------+                                                                                               
                | do_signal |                                                                                               
                +--|--------+                                                                                               
                   |     +------------+                                                                                     
                   +---> | get_signal |                                                                                     
                   |     +------------+                                                                                     
                   |                                                                  +----------------+                    
                   |          1. if tracer 'interfere' me, notify it and stop myself  | do_jobctl_trap |                    
                   |                                      +----------------+          +----------------+                    
                   |          2. try to dequeue a signal  | dequeue_signal |                                                
                   |                                      +----------------+                               +---------------+
                   |          3. if I'm a tracee, notify the tracer of the signal dequeue and stop myself  | ptrace_signal |
                   |                                                                                       +---------------+
                   |     +---------------+                                                                                  
                   +---> | handle_signal |  prepare the frame in the user stack for the signal handler                      
                         +---------------+                                                                                                       
```

```
+--------------+                                                                               
| sys_signalfd | : determine the file (newly allocated or existing one), update its mask       
+---|----------+                                                                               
    |    +----------------+                                                                    
    |--> | copy_from_user | copy mask info from userspace                                      
    |    +----------------+                                                                    
    |    +--------------+                                                                      
    +--> | do_signalfd4 | determine the file (newly allocated or existing one), update its mask
         +--------------+                                                                      
```

```
+--------------+                                                                        
| do_signalfd4 | : determine the file (newly allocated or existing one), update its mask
+---|----------+                                                                        
    |                                                                                   
    |--> if ufd isn't specified                                                         
    |                                                                                   
    |------> allocate ctx                                                               
    |                                                                                   
    |        +------------------+                                                       
    |------> | anon_inode_getfd | prepare an anon file and install to fd table          
    |        +------------------+                                                       
    |                                                                                   
    |--> else                                                                           
    |                                                                                   
    |------> get ctx from file and update its mask                                      
    |                                                                                   
    |        +---------+                                                                
    +------> | wake_up |                                                                
             +---------+                                                                
```

## <a name="system-calls"></a> System Calls

(TBD)
- sigprocmask
- sigaction

## <a name="reference"></a> Reference

- [P. Deshmukh, Signals in Linux](https://towardsdatascience.com/signals-in-linux-b34cea8c5791)
