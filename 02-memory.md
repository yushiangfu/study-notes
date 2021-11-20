## Index

- [Introduction](#introduction)

## <a name="introduction"></a> Introduction

Memory management is one of the kernel's most essential and fundamental infrastructures and is used to satisfy kernel space and user space requirements.
So why does memory need to be managed?
- If a simple logic runs without using any memory, we don't need the management.
- If an application only writes data to a pre-defined area and doesn't exceed the boundary, we don't need the management.
- If the system reaches financial freedom in terms of memory and the program won't consume them entirely before exiting, we don't need the management.
Unfortunately, on a 32-bit machine, it supports up to 4 GB RAM only.
Worst of all, dynamic memory allocation is so typical. Also, instead of one process, hundreds or thousands of them run altogether.
Besides, the application's metadata is also generated dynamically in OS.

