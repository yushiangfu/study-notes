> Linux version 5.15.0

## Index

- [Introduction](#introduction)
- [Reference](#reference)

## <a name="introduction"></a> Introduction

This tutorial will only focus on using it instead of how it works.
Ftrace is a built-in kernel feature that can help show the kernel function executed on each processor or limited to the target task.
Please don't get confused with ptrace/strace because they are irrelated to each other.
- ptrace: a syscall provided by the kernel to trace target task, but not the details of what it does in kernel space.
- strace: a utility that fully utilizes the ptrace functionality.
If you are interested in a task's userspace footprint more, strace and gdb are your good friends.

## <a name="reference"></a> Reference

- [S. Rostedt, Debugging the kernel using Ftrace - part 1](https://lwn.net/Articles/365835/)
- [S. Rostedt, Debugging the kernel using Ftrace - part 2](https://lwn.net/Articles/366796/)
- [S. Rostedt, Secrets of the Ftrace function tracer](https://lwn.net/Articles/370423/)
