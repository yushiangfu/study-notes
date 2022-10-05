> Study case: Linux version 5.15.0 on AST2500 emulation

## Index

- [Introduction](#introduction)
- [Spinlock](#spinlock)
- [Read-Write Lock](#rwlock)
- [Mutex](#mutex)
- [Semaphore](#semaphore)
- [Futex](#futex)
- [PI-Futex](#pi-futex)
- [Reference](#reference)

## <a name="introduction"></a> Introduction

We need the lock mechanism because the synchronization issue happens when multiple tasks try to access the same variable.
A one-line expression of value update consists of three CPU instructions, and it's no problem if only one task operates on it.

```
               +---- load  : memory -> register
               |                               
 a = a + 1 ----|---- update: register          
               |                               
               +---- store : register -> memory
```

The problem arises once other tasks have interests in it as well, and from the CPU's perspective, it's like:

```
 CASE 1                                               
               CPU                                    
       (var = 5 in memory)                            
       --------------------                           
       [Task A] load           <-- var = 5 in register
                                                      
       [Task A] update (+1)    <-- var = 6 in register
                                                      
       [Task B] load           <-- var = 5 in register
                                                      
       [Task B] update (+1)    <-- var = 5 in register
                                                      
       [Task B] store          <-- var = 6 in memory  
                                                      
       [Task A] store          <-- var = 6 in memory  
                                                      
                                                      
                                                      
 CASE 2                                               
               CPU                                    
       (var = 5 in memory)                            
       --------------------                           
       [Task A] load           <-- var = 5 in register
                                                      
       [Task A] update (+1)    <-- var = 6 in register
                                                      
       [Task A] store          <-- var = 6 in memory  
                                                      
       [Task B] load           <-- var = 6 in register
                                                      
       [Task B] update (+1)    <-- var = 7 in register
                                                      
       [Task B] store          <-- var = 7 in memory  
```

The reason is that modern operating systems support multi-tasks virtually running simultaneously through context switch. 
Tasks are welcome to yield the execution opportunity early, but more likely, they get kicked out by the scheduler. 
In other words, tasks themselves can't guarantee the instructions (load, update, and store) executed without intervention, which is the root cause. 
For ARM, it introduces two special instructions, LDREX and STREX, to resolve the problem.

When executing LDREX, it not only loads the value from memory to register but also monitors the address. 
For STREX, it checks if other STREX accesses the exact address already and fails the store if that's the case. 
Note that both instructions won't apply to the critical region directly since a context switch is likely to happen within a complicated context. 
And all the STREX might keep interfering with each other. 
Instead, they surround only one value, the well-known lock, and whichever task executes STREX successfully means it acquires the lock. 
Next, that lock applies to the critical region to achieve the synchronization mechanism.

```
               +---- load_ex : memory -> register, and monitor
               |                                            
 a = a + 1 ----|---- update  : register                       
               |                                            
               +---- store_ex: register -> memory, and check  
```

## <a name="spinlock"></a> Spinlock

Lock value comprises two fields: **next** and **owner**, and let's interpret it as the ticket system of a boba shop.

- next: the number for the customer who's about to order.
- owner: the number for the customer whose drink is ready.

For each lock value, we have four fundamental operations:

- init: set both **next** and **owner** to 0.
- lock: get **next**, increment it, wait till the original **next** is equal to the **owner**.
- trylock: get the lock if it's available, or do nothing.
- unlock: increment **owner** to leave the lock to the next waiting task if there's any.



```
            ticket             
      31               0       
       +-------+-------+       
       | next  | owner |       
       +-------+-------+       
                               
           0       0      init 
 (try)lock |                   
           v                   
           1       0           
                   | unlock    
                   v           
           1       1           
```

Assuming we have three tasks accessing the same data within a critical region, let's demonstrate how the lock and unlock work in the scenario.

```
          lock                                                                                                        
   31               0                                                                                                 
    +-------+-------+                                                                                                 
    | next  | owner |   |          [Task A]           |           [Task B]           |           [Task C]           | 
    +-------+-------+   |                             |                              |                              | 
                        |                             |                              |                              | 
 init   0       0      -----------------------------------------------------------------------------------------------
            |           |  get orig next == 0         |                              |                              | 
            |           |  next++                     |                              |                              | 
            v           |  orig next == owner, lock   |                              |                              | 
        1       0      -----------------------------------------------------------------------------------------------
            |           |                             |   get orig next == 1         |                              | 
            |           |                             |   next++                     |                              | 
            v           |                             |   orig next != owner, wait   |                              | 
        2       0      -----------------------------------------------------------------------------------------------
            |           |                             |                              |   get orig next == 2         | 
            |           |                             |                              |   next++                     | 
            v           |                             |                              |   orig next != owner, wait   | 
        3       0      -------------------------------------------------------------------------------------------- | 
            |           |                             |                              |                              | 
            |           |      unlock, owner++        |                              |                              | 
            v           |                             |                              |                              | 
        3       1      -----------------------------------------------------------------------------------------------
            |           |                             |                              |                              | 
            |           |                             | orig next == new owner, lock | orig next != new owner, wait | 
            |           |                             |                              |                              | 
            |          -----------------------------------------------------------------------------------------------
            |           |                             |                              |                              | 
            |           |                             |       unlock, owner++        |                              | 
            v           |                             |                              |                              | 
        3       2      -----------------------------------------------------------------------------------------------
                        |                             |                              |                              | 
                        |                             |                              | orig next == new owner, lock | 
                        |                             |                              |                              | 
```

<details>
  <summary> Code trace </summary>

```
+----------------+                                                           
| arch_spin_lock | update lock.next, and wait till I'm the lock owner        
+---|------------+                                                           
    |                                                                        
    |--> endless loop                                                        
    |                                                                        
    |------> use LDREX to load lock value to local variable                  
    |                                                                        
    |------> variable.next++, use STREX to store to lock value               
    |                                                                        
    |------> if it's stored successfully (no other STREX happens before mine)
    |                                                                        
    |----------> break                                                       
    |                                                                        
    |--> while variable.next != lock.owner (not my turn yet)                 
    |                                                                        
    +------> wait for event                                                  
```

```
+-------------------+                                                         
| arch_spin_trylock |                                                         
+----|--------------+                                                         
     |                                                                        
     |--> endless loop                                                        
     |                                                                        
     |------> use LDREX to load lock value to local variable                  
     |                                                                        
     |------> if variable.next != variable.lock (someone is holding the lock) 
     |                                                                        
     +----------> break                                                       
     |                                                                        
     +------> variable.next++, use STREX to store to lock value               
     |                                                                        
     +------> if it's stored successfully (no other STREX happens before mine)
     |                                                                        
     |----------> break                                                       
     |                                                                        
     +--> return 1 if lock is acquired, or 0 otherwise                        
```

```
+------------------+                       
| arch_spin_unlock |                       
+----|-------------+                       
     |                                     
     +--> lock.owner++ (read for new owner)
```

</details>
  
Variant:

```
raw_spin_lock_irqsave     : backup cpsr, disable interrupt, and acquire spinlock
raw_spin_unlock_irqrestore: release spinlock, and restore cpsr
```
  
## <a name="rwlock"></a> Read-Write Lock

There is also the read-write lock that utilizes LDREX and STREX to access the lock value, and the differences are:

- It allows multiple readers to enter the critical region but only one writer at a time.
- It doesn't guarantee the waiting sequence since there's no ticket number concept.

Possible lock values are:

- 0x0000_0000: it's available for both reader and writer.
- 0x8000_0000: a writer holds the lock, making the value negative in two's complement.
- Else       : at least one reader has the lock, and any other readers can join anytime.

```
  31              0
  +---------------+
  |w|      r      |
  +---------------+
```

Here's an example of how two readers and one writer access the lock.

```
         lock
   31              0
   +---------------+
   |w|      r      |   |          [Task A]           |           [Task B]           |           [Task C]           |
   +---------------+   |           reader            |            writer            |            reader            |
                       |                             |                              |                              |
init  0x0000_0000     -----------------------------------------------------------------------------------------------
           |           |      get lock value         |                              |                              |
           |           |      value >= 0? yes        |                              |                              |
           v           |      value++ and lock       |                              |                              |
      0x0000_0001     -----------------------------------------------------------------------------------------------
           |           |                             |        get lock value        |                              |
           |           |                             |        value == 0? no        |                              |
           |           |                             |        wait                  |                              |
           |          -----------------------------------------------------------------------------------------------
           |           |                             |                              |       get lock value         |
           |           |                             |                              |       value >= 0? yes        |
           v           |                             |                              |       value++ and lock       |
      0x0000_0002     -----------------------------------------------------------------------------------------------
           |           |                             |                              |                              |
           |           |                             |                              |       unlock, value--        |
           v           |                             |                              |                              |
      0x0000_0001     -----------------------------------------------------------------------------------------------
           |           |                             |        get lock value        |                              |
           |           |                             |        value == 0? no        |                              |
           |           |                             |        wait                  |                              |
           |          -----------------------------------------------------------------------------------------------
           |           |                             |                              |                              |
           |           |       unlock, value--       |                              |                              |
           v           |                             |                              |                              |
      0x0000_0000     -----------------------------------------------------------------------------------------------
           |           |                             |      get lock value          |                              |
           |           |                             |      value == 0? yes         |                              |
           v           |                             |      set bit 31 and lock     |                              |
      0x8000_0000
```

<details>
  <summary> Code trace </summary>

```
+-----------------+                                                               
| arch_write_lock |                                                               
+----|------------+                                                               
     |                                                                            
     |--> endless loop                                                            
     |                                                                            
     |------> use LDREX to load lock value to local variable                      
     |                                                                            
     |------> if the lock value is zero (not used)                                
     |                                                                            
     |----------> use STREX to store 0x8000_0000 to lock value                    
     |                                                                            
     |----------> if it's stored successfully (no other STREX happens before mine)
     |                                                                            
     +--------------> break                                                       
```
  
```
+--------------------+                                                        
| arch_write_trylock |                                                        
+----|---------------+                                                        
     |                                                                        
     |--> endless loop                                                        
     |                                                                        
     |------> use LDREX to load lock value to local variable                  
     |                                                                        
     |------> if the lock value isn't zero (used)                             
     |                                                                        
     |----------> break                                                       
     |                                                                        
     |------> use STREX to store 0x8000_0000 to lock value                    
     |                                                                        
     |------> if it's stored successfully (no other STREX happens before mine)
     |                                                                        
     |----------> break                                                       
     |                                                                        
     +--> return 1 if lock is acquired, or 0 otherwise                        
```
  
```
+-------------------+          
| arch_write_unlock |          
+----|--------------+          
     |                         
     +--> store 0 to lock value
```

```
+----------------+                                                           
| arch_read_lock |                                                           
+---|------------+                                                           
    |                                                                        
    |--> endless loop                                                        
    |                                                                        
    |------> use LDREX to load lock value to local variable                  
    |                                                                        
    +------> variable++                                                      
    |                                                                        
    |------> if variable < 0 (lock is held by a writer?)                     
    |                                                                        
    |----------> continue                                                    
    |                                                                        
    |------> use STREX to store variable lock value                          
    |                                                                        
    |------> if it's stored successfully (no other STREX happens before mine)
    |                                                                        
    +----------> break                                                       
```
 
```
+------------------+                                                          
| arch_read_unlock |                                                          
+----|-------------+                                                          
     |                                                                        
     |--> endless loop                                                        
     |                                                                        
     |------> use LDREX to load lock value to local variable                  
     |                                                                        
     |------> variable--                                                      
     |                                                                        
     |------> use STREX to store variable lock value                          
     |                                                                        
     |------> if it's stored successfully (no other STREX happens before mine)
     |                                                                        
     +----------> break                                                       
```
 
```
+-------------------+
| arch_read_trylock |
+----|--------------+
     |
     |--> endless loop
     |
     |------> se LDREX to load lock value to local variable
     |
     |------> variable++
     |
     |------> if it's < 0 (lock is held by a writer)
     |
     |----------> break
     |
     |------> use STREX to store variable lock value
     |
     |------> if it's stored successfully (no other STREX happens before mine)
     |
     |----------> break
     |
     +--> return 1 if lock is acquired, or 0 otherwise
```
  
</details>
  
## <a name="mutex"></a> Mutex

The mutex is another type of lock that works based on spinlock. 
The difference is that tasks sleep while waiting instead of constantly querying as spinlock does.

```
    mutex                                 
+-----------+                             
|   owner   | the task owning the mutex
|           |                             
| wait_lock | spinlock                    
|           |                             
| wait_list | tasks waiting for the mutex 
+-----------+                             
```

<details>
  <summary> Code trace </summary>
  
```
+------------+                                                                                                      
| mutex_lock |                                                                                                      
+--|---------+                                                                                                      
   |    +----------------------+                                                                                    
   |--> | __mutex_trylock_fast | if no one owns the mutex right now, acquire it                                     
   |    +----------------------+                                                                                    
   |                                                                                                                
   |--> return if we got the mutex                                                                                  
   |                                                                                                                
   |    +-----------------------+                                                                                   
   +--> | __mutex_lock_slowpath |                                                                                   
        +-----|-----------------+                                                                                   
              |    +--------------+                                                                                 
              +--> | __mutex_lock |                                                                                 
                   +---|----------+                                                                                 
                       |    +---------------------+                                                                 
                       +--> | __mutex_lock_common |                                                                 
                            +-----|---------------+                                                                 
                                  |    +--------------------+                                                       
                                  |--> | __mutex_add_waiter | add current task to the end of wait queue of the mutex
                                  |    +--------------------+                                                       
                                  |    +-------------------+                                                        
                                  |--> | set_current_state | change task state, e.g., 'uninterruptible'             
                                  |    +-------------------+                                                        
                                  |                                                                                 
                                  |--> endless loop                                                                 
                                  |                                                                                 
                                  |        +---------------------------+                                            
                                  |------> | schedule_preempt_disabled | go to sleep                                
                                  |        +---------------------------+                                            
                                  |                                      <--- somewhere woke me up                  
                                  |        +----------------------------+                                           
                                  |------> | __mutex_trylock_or_handoff | try lock again                            
                                  |        +----------------------------+                                           
                                  |                                                                                 
                                  |------> break loop if we got the mutext                                          
                                  |                                                                                 
                                  |    +-----------------------+                                                    
                                  +--> | __mutex_remove_waiter | remove task from waiter list                       
                                       +-----------------------+                                                    
```
  
```
+--------------+                                                                
| mutex_unlock |                                                                
+---|----------+                                                                
    |    +---------------------+                                                
    |--> | __mutex_unlock_fast | reset mutex owner to null                      
    |    +---------------------+                                                
    |                                                                           
    |--> return if unlocked (what's the failed case?)                           
    |                                                                           
    |    +-------------------------+                                            
    +--> | __mutex_unlock_slowpath |                                            
         +------|------------------+                                            
                |                                                               
                |--> if there's at least one task in queue waiting for the mutex
                |                                                               
                |        +------------+                                         
                |------> | wake_q_add | add the first task to the wake queue    
                |        +------------+                                         
                |    +-----------+                                              
                +--> | wake_up_q | wake up the task                             
                     +-----------+                                              
```

</details>
  
## <a name="semaphore"></a> Semaphore

Semaphore is the generic form of mutex; it allows n tasks to enter the critical region simultaneously. 
Of course, a semaphore with n equals one is equivalent to a mutex conceptually.
  
```
  semaphore                                                
+-----------+                                              
|   lock    | spinlock                                     
|           |                                              
|   count   | the number of tasks that can acquire the lock
|           |                                              
| wait_list | tasks waiting for the semaphore              
+-----------+                                              
```
  
<details>
  <summary> Code trace </summary>
  
```
ipcs -s -i <semaphore id>
```
  
```
+--------------------+                                                                  
| down_interruptible |                                                                  
+----|---------------+                                                                  
     |    +-----------------------+                                                     
     |--> | raw_spin_lock_irqsave | semaphore lock                                      
     |    +-----------------------+                                                     
     |                                                                                  
     |--> if count > 0                                                                  
     |                                                                                  
     |------> count--                                                                   
     |                                                                                  
     |--> else                                                                          
     |                                                                                  
     |        +----------------------+                                                  
     |------> | __down_interruptible |                                                  
     |        +-----|----------------+                                                  
     |              |    +---------------+                                              
     |              +--> | __down_common |                                              
     |                   +---|-----------+                                              
     |                       |                                                          
     |                       +--> add waiter to the end of list                         
     |                                                                                  
     |                            endless loop                                          
     |                                                                                  
     |                                +---------------------+                           
     |                                | __set_current_state |                           
     |                                +---------------------+                           
     |                                +------------------+                              
     |                                | schedule_timeout |                              
     |                                +------------------+                              
     |                                                        <--- somewhere woke me up 
     |                                if waiter is up, return (lock acquired)
     |                                                                                  
     |    +----------------------------+                                                
     +--> | raw_spin_unlock_irqrestore | semaphore lock                                 
          +----------------------------+                                                
```
  
```
+----+                                             
| up |                                             
+|---+                                             
 |    +-----------------------+                    
 |--> | raw_spin_lock_irqsave | semaphore lock     
 |    +-----------------------+                    
 |                                                 
 |--> if no waiting task on the list               
 |                                                 
 |------> count++                                  
 |                                                 
 |--> else                                         
 |                                                 
 |        +------+                                 
 |------> | __up |                                 
 |        +-|----+                                 
 |          |                                      
 |          |--> remove the first waiter from list 
 |          |                                      
 |          |--> waiter->up = true                 
 |          |                                      
 |          |    +-----------------+               
 |          +--> | wake_up_process |               
 |               +-----------------+               
 |    +----------------------------+               
 +--> | raw_spin_unlock_irqrestore | semaphore lock
      +----------------------------+               
```

</details>
  
## <a name="futex"></a> Futex
  
Futex is the fast lock mechanism for userspace to utilize, and it needs no kernel involvement if the lock is uncontented. 
The kernel is still necessary for contended cases to help put the tasks to sleep or wake them up. 
Somehow, the author replaces the terms **lock** and **unlock** with **wait** and **wake**. 
For contented **wait**, the kernel wraps the waiting task into **futex_q** and inserts it to the hash table indexed by the hashed value of the key. 
The list is priority-descending and FIFO for those tasks with the same priority value. 
As you can imagine, **unlock** is the **futex_q** removal from the list and waking up that task to continue its logic.
  
```
 futex_hash_bucket  --+                                                                             
    +---------+       |                                                                             
    |+-------+|       |              hash table                                                     
    ||waiters||       |                                                                             
    |+-------+|       |          +-----------------+                                                
    | +----+  |       | -------- |futex_queues[0]  |                                                
    | |lock|  |       |          +-----------------+       +-------+       +-------+       +-------+
    | +----+  |       |          |futex_queues[1]  | ----- |futex_q| ----- |futex_q| ----- |futex_q|
    | +-----+ |       |          +-----------------+       +-------+       +-------+       +-------+
    | |chain| |       |                   -                                    |                    
    | +-----+ |       |                   -                                    |                    
    +---------+     --+                   -                                |--------|               
                                          -                                                         
                                          -                                 futex_q                 
                                          -                                +--------+               
                                 +-----------------+                       |  list  |               
                                 |futex_queues[255]|                       |+------+|               
                                 +-----------------+                       ||+----+||               
                                                                           |||prio|||               
                                                                           ||+----+||               
                                                                           |+------+|               
                                                                           | +----+ |               
                                                                           | |task| |               
                                                                           | +----+ |               
                                                                           +--------+               
```
  
<details>
  <summary> Code trace </summary>
  
```
+-----------+                                                                                                                      
| sys_futex |                                                                                                                      
+--|--------+                                                                                                                      
   |    +----------+                                                                                                               
   +--> | do_futex |                                                                                                               
        +--|-------+                                                                                                               
           |                                                                                                                       
           |--> swtich command                                                                                                     
           |                                                                                                                       
           |--> case FUTEX_WAIT                                                                                                    
           |                                                                                                                       
           |--> case FUTEX_WAIT_BITSET                                                                                             
           |                                                                                                                       
           |        +------------+                                                                                                 
           |------> | futex_wait | set up key, queue task to sleep, till it's waken                                                
           |        +------------+                                                                                                 
           |                                                                                                                       
           |------> return                                                                                                         
           |                                                                                                                       
           |--> case FUTEX_WAKE                                                                                                    
           |                                                                                                                       
           |--> case FUTEX_WAKE_BITSET                                                                                             
           |                                                                                                                       
           |        +-----------+                                                                                                  
           |------> | futex_wake| wake up task(s) in hash bucket                                                                   
           |        +-----------+                                                                                                  
           |                                                                                                                       
           |--> case FUTEX_LOCK_PI                                                                                                 
           |                                                                                                                       
           |--> case FUTEX_LOCK_PI2                                                                                                
           |                                                                                                                       
           |        +---------------+                                                                                              
           |------> | futex_lock_pi |                                                                                              
           |        +---------------+                                                                                              
           |                                                                                                                       
           |--> case FUTEX_UNLOCK_PI                                                                                               
           |                                                                                                                       
           |        +-----------------+                                                                                            
           +------> | futex_unlock_pi | adjust current task piro and wake up next waiter if any, or otherwise reset user value to 0
                    +-----------------+                                                                                            
```
  
```
+------------+                                                               
| futex_wait | set up key, queue task to sleep, till it's waken              
+--|---------+                                                               
   |    +-------------------+                                                
   |--> | futex_setup_timer | set up the sleeping timer                      
   |    +-------------------+                                                
   |    +------------------+                                                 
   |--> | futex_wait_setup | set up key and get value from uesr addr         
   |    +------------------+                                                 
   |    +---------------------+                                              
   |--> | futex_wait_queue_me | add task to queue and sleep, till it's waken 
   |    +---------------------+                                              
   |                                                                         
   |--> if task isn't in queue anymore (it succeeds), go to out              
   |out:                                                                     
   |--> if there's timeout timer                                             
   |                                                                         
   |        +----------------+                                               
   |------> | hrtimer_cancel |                                               
   |        +----------------+                                               
   |        +--------------------------+                                     
   +------> | destroy_hrtimer_on_stack |                                     
            +--------------------------+                                     
```
  
```
+------------------+                                         
| futex_wait_setup | set up key and get value from uesr addr 
+----|-------------+                                         
     |    +---------------+                                  
     |--> | get_futex_key | set up key                       
     |    +---------------+                                  
     |    +------------------------+                         
     |--> | get_futex_value_locked | get value from user addr
     |    +------------------------+                         
     |                                                       
     |--> if value from user addr != value from arg          
     |                                                       
     +------> return error                                   
```
  
```
+---------------+                                                                 
| get_futex_key | set up key                                                      
+---|-----------+                                                                 
    |                                                                             
    |--> if it's a private futex (fast path)                                      
    |                                                                             
    |------> set up key.private and return                                        
    |                                                                             
    |    +---------------------+                                                  
    |--> | get_user_pages_fast | get page of the user address and pin it in memory
    |    +---------------------+                                                  
    |                                                                             
    |--> if it's an anonymous mapping                                             
    |                                                                             
    |------> set up key.private                                                   
    |                                                                             
    |--> else                                                                     
    |                                                                             
    +------> set up key.shared                                                    
```
  
```
+---------------------+                                                        
| futex_wait_queue_me | add task to queue and sleep, till it's waken        
+-----|---------------+                                                        
      |    +-------------------+                                               
      |--> | set_current_state | set task state to 'interruptible'             
      |    +-------------------+                                               
      |    +----------+                                                        
      |--> | queue_me |                                                        
      |    +--|-------+                                                        
      |       |    +------------+                                              
      |       +--> | __queue_me | link futex and task, add futex to hash bucket
      |            +------------+                                              
      |                                                                        
      |--> if 'timeout' is given                                               
      |                                                                        
      |        +-------------------------------+                               
      |------> | hrtimer_sleeper_start_expires |                               
      |        +-------------------------------+                               
      |    +--------------------+                                              
      |--> | freezable_schedule | go to sleep                                  
      |    +--------------------+                                              
      |                        <-- somewhere woke me up                        
      |    +---------------------+                                             
      +--> | __set_current_state | set task state to 'running'                 
           +---------------------+                                             
```
 
```
+------------+                                                        
| futex_wake | wake up task(s) in hash bucket                         
+--|---------+                                                        
   |    +---------------+                                             
   |--> | get_futex_key | set up key                                  
   |    +---------------+                                             
   |                                                                  
   |--> for each node in hash bucket                                  
   |                                                                  
   |        +-------------+                                           
   |------> | match_futex | check if key attributes are the same      
   |        +-------------+                                           
   |                                                                  
   |------> check if the node meets our 'bitset'                      
   |                                                                  
   |        +-----------------+                                       
   |------> | mark_wake_futex |                                       
   |        +----|------------+                                       
   |             |    +-----------------+                             
   |             |--> | __unqueue_futex | remove node from hash bucket
   |             |    +-----------------+                             
   |             |    +-----------------+                             
   |             +--> | wake_q_add_safe | add task to wake queue      
   |                  +-----------------+                             
   |                                                                  
   |------> if we added that many (nr_wake) tasks to the wake queue   
   |                                                                  
   |----------> break                                                 
   |                                                                  
   |    +-----------+                                                 
   +--> | wake_up_q | wake up task(s) in queue                        
        +-----------+                                                 
```
  
</details>
  
## <a name="pi-futex"></a> PI-Futex

Assuming we have three tasks with different priorities running in the system, high-prio and low-prio tasks share the same lock. 
Tasks with higher priority are likely to be the next task consuming its time slice of execution. 
If, for some reason, the low-prio task acquires the lock before the high-prio task, which goes to sleep waiting for the lock release. 
Here are the known facts:
  
- The high-prio task is waiting for the low-prio task to release the lock
- The med-prio lives a happily-ever-after life.
- The low-prio task rarely runs because of its low priority nature.
 
```
                                              time line
                    ------------------------------------------------------------>


                                 lock [X]
          +------+               sleep                   lock [O]
high prio | task |  -->-->-->      -->                    -->-->-->      -->-->-->
          +------+          |      | |                    |       |      |       |
                            |      | |                    |       |      |       |
          +------+          |      | |                    |       |      |       |
 med prio | task |          -->--> | -->--> -->--> -->--> |       -->--> |       -->-->
          +------+               | |      | |    | |    | |            | |            |
                                 | |      | |    | |    | |            | |            |
          +------+               | |      | |    | |    | |            | |            |
 low prio | task |               -->      -->    -->    -->            -->            -->
          +------+          lock [O]                    unlock
```
  
And the problem is that high-prio has a lower chance of being the next candidate than the med-prio task during the lock-waiting interval. 
The scenario is the so-called priority inversion, and the solution to it is the priority-inheritance lock.
  
Those intelligent guys introduced the Priority-Inheritance (PI) futex to solve the issue. 
The spirit is that the lock owner will temporarily boost up to match the top waiter's priority. 
In a regular futex framework, each task waiting for the lock corresponds to one **futex_q** lining up in a priority-descending order. 
But in PI futex design, there's only one **futex_q** for each lock, and the kernel organizes the waiting tasks into a tree structure on that lock. 
The top waiter is the highest-priority task of the tree positioned on the leftmost.

When a task releases the lock:
  
1. [Current task] Set owner to the top waiter.
2. [Current task] Restore the boosted priority
3. [Current task] Wake up the top waiter
4. [Top waiter] Remove itself from the tree structure
  
```
    hash table                                                                                         
                                                                                                       
+-----------------+                                                                                    
|futex_queues[0]  |                                                                                    
+-----------------+       +-------+       +-------+       +-------+                                    
|futex_queues[1]  | ----- |futex_q| ----- |futex_q| ----- |futex_q|                                    
+-----------------+       +-------+       +-------+       +-------+                                    
         -                    |                                                                        
         -                    |                                                                        
         -                |--------|                                                                   
         -                                futex_pi_state                                 task_struct   
         -                 futex_q         +-----------+                              +---------------+
         -               +----------+      |  +----+   |                              |+-------------+|
+-----------------+      |   list   |      |  |list|<--|------------------------------->pi_state_list||
|futex_queues[255]|      | +------+ |      |  +----+   |                              |+-------------+|
+-----------------+      | |+----+| |      | pi_mutex  |                              |  +----------+ |
         |               | ||prio|| |      |+---------+|                              |  |pi_waiters| |
         |               | |+----+| |      ||+-------+||               +------+       |  +-----|----+ |
    |---------|          | +------+ |      |||waiters----------------> |waiter|       +--------|------+
                         |  +----+  |      ||+-------+||               +------+                |       
 futex_hash_bucket       |  |task|  |      || +-----+ ||                   |                   |       
    +---------+          |  +----+  |      || |owner| ||           +-------+---------+         |       
    |+-------+|          |  +---+   |      || +-----+ ||           |                 |         |       
    ||waiters||          |  |key|   |      |+---------+|       +------+          +------+      |       
    |+-------+|          |  +---+   |      |  +-----+  |       |waiter|          |waiter|      |       
    | +----+  |          |+--------+|      |  |owner|  |       +------+          +------+      |       
    | |lock|  |          ||pi_state||----> |  +-----+  |           |                 |         |       
    | +----+  |          |+--------+|      |   +---+   |      +----+----+       +----+----+    |       
    | +-----+ |          +----------+      |   |key|   |      |         |       |         |    |       
    | |chain| |                            |   +---+   |   +------+ +------+ +------+ +------+ |       
    | +-----+ |                            +-----------+   |waiter| |waiter| |waiter| |waiter| |       
    +---------+                                            +------+ +------+ +------+ +------+ |       
                                                              ^                                |       
                                                              |                                |       
                                                              +--------------------------------+       
                                                          top waiter                                   
```

With the help of a temporary priority boost, the original chronological flow becomes:
  
```
                                               time line                          
                     ------------------------------------------------------------>
                                                                                  
                                 lock [X]                                         
                                 boost owner                                      
           +------+              sleep       lock [O]                             
 high prio | task |  -->-->-->      -->       -->-->-->      -->-->-->            
           +------+          |      | |       |       |      |       |            
                             |      | |       |       |      |       |            
           +------+          |      | |       |       |      |       |            
  med prio | task |          -->--> | |       |       -->--> |       -->-->       
           +------+               | | |       |            | |            |       
                                  | | |       |            | |            |       
           +------+               | | |       |            | |            |       
  low prio | task |               --> -->-->-->            -->            -->     
           +------+          lock [O]       deboost  
                                            unlock   
```
  
<details>
  <summary> Code trace </summary>
  
```
+---------------+                                                                                                            
| futex_lock_pi | acquire lock, wait if it fails                                                                             
+---|-----------+                                                                                                            
    |    +-----------------------+                                                                                           
    |--> | refill_pi_state_cache | prepare pi_state and install to current task                                              
    |    +-----------------------+                                                                                           
    |    +-------------------+                                                                                               
    |--> | futex_setup_timer |                                                                                               
    |    +-------------------+                                                                                               
    |    +---------------+                                                                                                   
    |--> | get_futex_key | set up key                                                                                        
    |    +---------------+                                                                                                   
    |    +----------------------+                                                                                            
    |--> | futex_lock_pi_atomic | have the arg 'ps' point to either top waiter's pi_state, or to a newly allocated one
    |    +----------------------+                                                                                            
    |    +------------+                                                                                                      
    |--> | __queue_me | link futex and task, add futex to hash bucket                                                        
    |    +------------+                                                                                                      
    |                                                                                                                        
    |--> (skip the case of trylock)                                                                                          
    |                                                                                                                        
    |    +-----------------------------+                                                                                     
    |--> | __rt_mutex_start_proxy_lock | start lock acquisition for another task, reutrn 0 if blocked, or 1 for lock acquired
    |    +-----------------------------+                                                                                     
    |                                                                                                                        
    |--> if blocked                                                                                                          
    |                                                                                                                        
    |        +--------------------------+                                                                                    
    |------> | rt_mutex_wait_proxy_lock | wait for lock                                                                      
    |        +--------------------------+                                                                                    
    |    +-------------+                                                                                                     
    |--> | fixup_owner | fixup pi-state owner                                                                                
    |    +-------------+                                                                                                     
    |    +---------------+                                                                                                   
    +--> | unqueue_me_pi | remove futex from hash bucket                                                                     
         +---------------+                                                                                                   
```
  
```
+----------------------+                                                                                     
| futex_lock_pi_atomic | have the arg 'ps' point to either top waiter's pi_state, or to a newly allocated one
+-----|----------------+                                                                                     
      |    +------------------+                                                                              
      |--> | futex_top_waiter | get top waiter                                                               
      |    +------------------+                                                                              
      |                                                                                                      
      |--> if top waiter exists                                                                              
      |                                                                                                      
      |        +--------------------+                                                                        
      |------> | attach_to_pi_state | have the arg 'ps' point to top waiter's pi_state                       
      |        +--------------------+                                                                        
      |                                                                                                      
      |------> return                                                                                        
      |                                                                                                      
      |    +-----------------------+                                                                         
      |--> | lock_pi_update_atomic | write uval to uaddr                                                     
      |    +-----------------------+                                                                         
      |    +--------------------+                                                                            
      +--> | attach_to_pi_owner | allocate pi_state and install to task, have arg 'ps' point to it           
           +--------------------+                                                                            
```
  
```
+-----------------------------+                                                                                     
| __rt_mutex_start_proxy_lock | start lock acquisition for another task, reutrn 0 if blocked, or 1 for lock acquired
+-------|---------------------+                                                                                     
        |    +----------------------+                                                                               
        |--> | try_to_take_rt_mutex | try to acquire the lock for task and return 1, otherwise 0                    
        |    +----------------------+                                                                               
        |                                                                                                           
        |--> if try successfully, return 1                                                                          
        |                                                                                                           
        |    +-------------------------+                                                                            
        +--> | task_blocks_on_rt_mutex | adjust prio and sched class of waiter, and propagate to chain              
             +-------------------------+                                                                            
```
  
```
+-------------------------+                                                              
| task_blocks_on_rt_mutex | adjust prio and sched class of waiter, and propagate to chain
+------|------------------+                                                              
       |                                                                                 
       |--> if owner == task (deadlock), return error                                    
       |                                                                                 
       |--> set up waiter                                                                
       |                                                                                 
       |    +--------------------+                                                       
       |--> | waiter_update_prio | set waiter prio based on task's                       
       |    +--------------------+                                                       
       |    +---------------------+                                                      
       |--> | rt_mutex_top_waiter | get the top waiter from queue of lock                
       |    +---------------------+                                                      
       |    +------------------+                                                         
       |--> | rt_mutex_enqueue | add our waiter into queue of lock                       
       |    +------------------+                                                         
       |    +----------------+                                                           
       |--> | build_ww_mutex | (ignore bc of disabled config)                            
       |    +----------------+                                                           
       |                                                                                 
       |--> if no owner, return                                                          
       |                                                                                 
       |--> if waiter == top waiter                                                      
       |                                                                                 
       |        +---------------------+                                                  
       |------> | rt_mutex_dequeue_pi | dequeue top waiter                               
       |        +---------------------+                                                  
       |        +---------------------+                                                  
       |------> | rt_mutex_enqueue_pi | enqueue waiter                                   
       |        +---------------------+                                                  
       |        +----------------------+                                                 
       |------> | rt_mutex_adjust_prio | adjust piro and sche class of task              
       |        +----------------------+                                                 
       |    +----------------------------+                                               
       +--> | rt_mutex_adjust_prio_chain | adjust the priority chain                     
            +----------------------------+                                               
```

```
+----------------------+                                                                            
| rt_mutex_adjust_prio | adjust piro and sche class of task                                         
+-----|----------------+                                                                            
      |                                                                                             
      |--> get the top pi waiter from given task                                                    
      |                                                                                             
      |    +------------------+                                                                     
      +--> | rt_mutex_setprio |                                                                     
           +----|-------------+                                                                     
                |    +---------------------+                                                           
                |--> | __rt_effective_prio | get higher prio from both tasks                            
                |    +---------------------+                                                           
                |                                                                                   
                |--> return if no need to adjust prio                                               
                |                                                                                   
                |--> p->pi_top_task = pi_task                                                       
                |                                                                                   
                |    +--------------+                                                               
                |--> | dequeue_task | dequeue task if it's in any run queue                         
                |    +--------------+                                                               
                |    +---------------+                                                              
                |--> | put_prev_task | put task if it's running                                     
                |    +---------------+                                                              
                |    +---------------------+                                                        
                |--> | __setscheduler_prio | adjust task's sched class and prio                     
                |    +---------------------+                                                        
                |    +--------------+                                                               
                |--> | enqueue_task | add task back to rq if it was there                           
                |    +--------------+                                                               
                |    +---------------+                                                              
                |--> | set_next_task | set it as next task if it was running                        
                |    +---------------+                                                              
                |    +---------------------+                                                        
                +--> | check_class_changed | call class method to handle class change or prio change
                     +---------------------+                                                        
```
  
```
+----------------------------+                                             
| rt_mutex_adjust_prio_chain | adjust the priority chain                   
+------|---------------------+                                             
       |                                                                   
       |--> return if no waiter on task                                    
       |                                                                   
       |    +---------------------+                                        
       |--> | rt_mutex_top_waiter | get top waiter from lock               
       |    +---------------------+                                        
       |    +------------------+                                           
       |--> | rt_mutex_dequeue | remove waiter from queue                  
       |    +------------------+                                           
       |    +--------------------+                                         
       |--> | waiter_update_prio | set waiter prio attributes based on task
       |    +--------------------+                                         
       |    +------------------+                                           
       +--> | rt_mutex_enqueue | add task to queue                         
       |    +------------------+                                           
       |                                                                   
       |--> do some other dequeue and enqueue actions properly             
       |                                                                   
       |    +----------------------+                                       
       +--> | rt_mutex_adjust_prio | adjust piro and sche class of task    
            +----------------------+                                       
```

```
+--------------------------+                                                                                   
| rt_mutex_wait_proxy_lock | wait for lock                                                                     
+------|-------------------+                                                                                   
       |    +-------------------+                                                                              
       |--> | set_current_state | set task state to 'interruptible'                                            
       |    +-------------------+                                                                              
       |    +-------------------------+                                                                        
       |--> | rt_mutex_slowlock_block |                                                                        
       |    +------|------------------+                                                                        
       |           |                                                                                           
       |           |--> endless loop                                                                           
       |           |                                                                                           
       |           |        +----------------------+                                                           
       |           |------> | try_to_take_rt_mutex | try to acquire the lock for task and return 1, otherwise 0
       |           |        +----------------------+                                                           
       |           |                                                                                           
       |           |------> break loop if lock acquired                                                        
       |           |                                                                                           
       |           |------> determine owner                                                                    
       |           |                                                                                           
       |           |------> if no owner                                                                        
       |           |                                                                                           
       |           |            +----------+                                                                   
       |           |----------> | schedule |                                                                   
       |           |            +----------+                                                                   
       |           |    +---------------------+                                                                
       |           +--> | __set_current_state | set task state to 'running'                                    
       |                +---------------------+                                                                
       |    +-------------------------+                                                                        
       +--> | fixup_rt_mutex_waiters  | remvoe label 'has_waiter' from lock owner if there's no waiter         
            +-------------------------+                                                                        
```
  
```
+-----------------+                                                                                            
| futex_unlock_pi | adjust current task piro and wake up next waiter if any, or otherwise reset user value to 0
+----|------------+                                                                                            
     |                                                                                                         
     |--> get user value from user addr                                                                        
     |                                                                                                         
     |    +---------------+                                                                                    
     |--> | get_futex_key | set up key                                                                         
     |    +---------------+                                                                                    
     |    +------------+                                                                                       
     |--> | hash_futex | get hash bucket from key                                                              
     |    +------------+                                                                                       
     |    +------------------+                                                                                 
     |--> | futex_top_waiter | get the highest priority waiter                                                 
     |    +------------------+                                                                                 
     |                                                                                                         
     |--> if top waiteir exists                                                                                
     |                                                                                                         
     |        +---------------+                                                                                
     |------> | wake_futex_pi | update pi_state owner, adjust current task prio, wake up next waiter           
     |        +---------------+                                                                                
     |                                                                                                         
     |------> return                                                                                           
     |                                                                                                         
     +--> write 0 to user addr                                                                                 
```
  
```
+---------------+                                                                            
| wake_futex_pi | update pi_state owner, adjust current task prio, wake up next waiter       
+---|-----------+                                                                            
    |    +---------------------+                                                             
    |--> | rt_mutex_top_waiter | get top waiter of the lock                                  
    |    +---------------------+                                                             
    |                                                                                        
    |--> determine new value = task | flag                                                   
    |                                                                                        
    |--> write new value to user addr                                                        
    |                                                                                        
    |    +-----------------------+                                                           
    |--> | pi_state_update_owner | move pi_state from old owner to new owner                 
    |    +-----------------------+                                                           
    |    +-------------------------+                                                         
    |--> | __rt_mutex_futex_unlock |                                                         
    |    +------|------------------+                                                         
    |           |                                                                            
    |           |--> return if lock has no waiter                                            
    |           |                                                                            
    |           |    +-------------------------+                                             
    |           +--> | mark_wakeup_next_waiter |                                             
    |                +------|------------------+                                             
    |                       |    +---------------------+                                     
    |                       |--> | rt_mutex_top_waiter | get the top waiter of lock          
    |                       |    +---------------------+                                     
    |                       |    +---------------------+                                     
    |                       |--> | rt_mutex_dequeue_pi | remove waiter from current task     
    |                       |    +---------------------+                                     
    |                       |    +----------------------+                                    
    |                       |--> | rt_mutex_adjust_prio | adjust piro and sche class of task 
    |                       |    +----------------------+                                    
    |                       |    +--------------------+                                      
    |                       +--> | rt_mutex_wake_q_add| add waiter to queue                  
    |                            +--------------------+                                      
    |                                                                                        
    |--> if there's waiter                                                                   
    |                                                                                        
    |        +---------------------+                                                         
    +------> | rt_mutex_postunlock | wake up the waiter in queue                             
             +---------------------+                                                         
```
  
```
ipcs -s -i <semophore id>    # show pid of the current lock owner
```
  
</details>
  
## <a name="reference"></a> Reference

[J. Corbet, Ticket spinlocks](https://lwn.net/Articles/267968/)
