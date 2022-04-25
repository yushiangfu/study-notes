> Study case: Linux version 5.15.0 on AST2500 emulation

## Index

- [Introduction](#introduction)
- [Spinlock](#spinlock)
- [Read-Write Lock](#rwlock)
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
  
<details>
 
## <a name="reference"></a> Reference

[J. Corbet, Ticket spinlocks](https://lwn.net/Articles/267968/)
