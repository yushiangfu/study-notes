> Linux version 5.15.0

## Index

- [Introduction](#introduction)
- [Operation](#operation)
- [Scenario](#scenario)
- [Reference](#reference)

## <a name="introduction"></a> Introduction

This tutorial will only focus on using it instead of how it works.
Ftrace is a built-in kernel feature that can help show the kernel function executed on each processor or limited to the target task.
Please don't get confused with ptrace/strace because they are irrelated to each other.
- ptrace: a syscall provided by the kernel to trace target task, but not the details of what it does in kernel space.
- strace: a utility that fully utilizes the ptrace functionality.
If you are interested in a task's userspace footprint more, strace and gdb are your good friends.

## <a name="operation"></a> Operation

- Enter the root folder of *ftrace*

```
cd /sys/kernel/debug/tracing
```

- Show available tracers

```
root@romulus:/sys/kernel/debug/tracing# cat available_tracers
function_graph function nop
```

- Show the current tracer

```
root@romulus:/sys/kernel/debug/tracing# cat current_tracer 
nop
```

- Specity the tracer, e.g. *function_graph*

```
echo function_graph > current_tracer
```

- Enable/disable trace

```
echo 1 > tracing_on
echo 0 > tracing_on
```

- Inspect and clear trace buffer

```
cat trace
echo > trace
```

## <a name="scenario"></a> Scenario

- Trace a sequence of commands

```
echo 0 > tracing_on
echo function_graph > current_tracer
echo 1 > tracing_on
run our test code
echo 0 > tracing_on
cat trace
```


## <a name="reference"></a> Reference

- [S. Rostedt, Debugging the kernel using Ftrace - part 1](https://lwn.net/Articles/365835/)
- [S. Rostedt, Debugging the kernel using Ftrace - part 2](https://lwn.net/Articles/366796/)
- [S. Rostedt, Secrets of the Ftrace function tracer](https://lwn.net/Articles/370423/)
