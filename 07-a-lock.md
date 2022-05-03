> Study case: Linux version 5.15.0 on AST2500 emulation

## Index

- [Introduction](#introduction)
- [Spinlock](#spinlock)
- [Read-Write Lock](#rwlock)
- [Mutex](#mutex)
- [Semaphore](#semaphore)
- [Futex](#futex)
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

## <a name="futex"></a> Futex

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
  
## <a name="reference"></a> Reference

[J. Corbet, Ticket spinlocks](https://lwn.net/Articles/267968/)
