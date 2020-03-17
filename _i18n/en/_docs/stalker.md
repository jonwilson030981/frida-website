## Introduction

Stalker is FRIDA's code tracing engine. It allows threads to be followed, capturing every function, every block, even every instruction which is executed. A very good overview of the stalker engine is provided [here](https://medium.com/@oleavr/anatomy-of-a-code-tracer-b081aadb0df8) and I recommend that you read it carefully first. Obviously, the implementation is somewhat architecture specific, although there is much in common between them. Stalker is currently supports the ARM64 commonly found on mobile phones and tablets running Android or iOS as well as Intel x86_64 architecture commonly found on desktops and laptops. This page intends to take things to the next level of detail, it disects the ARM64 implementation of stalker and explains in more detail exactly how it works. It is hoped that this may help future efforts to port stalker to other hardware architectures.


## API
To start to understand the implementation of stalker, we must first understand in detail what it offers to the user. Whilst stalker can be invoked directly through its native gum interface, most users will instead call it via the [JavaScript API](https://frida.re/docs/javascript-api/#stalker) which will call these gum methods on their behalf. The [typescript type definitions](https://github.com/DefinitelyTyped/DefinitelyTyped/blob/master/types/frida-gum/index.d.ts) for gum are well commented and provide a little more detail still. 

The main API to stalker from JavaScript is:

```
Stalker.follow([threadId, options])
```

> start stalking `threadId` (or the current thread if omitted)

Let's consider when these calls may be used. Stalking where you provide a thread id is likely to be used where you have a thread of interest and are wondering what it is doing. Perhaps it has an interesting name? Thread names can be found using `cat /proc/PID/tasks/TID/comm`. Or perhaps you walked the threads in your process using the FRIDA JavaScript API `Process.enumerateThreads()` and then used a NativeFunction to call:

```
int pthread_getname_np(pthread_t thread,
                       char *name, size_t len);
```

Using this along with the [Thread.backtrace](https://frida.re/docs/javascript-api/#thread) to dump thread stacks can give you a really good overview of what a process is doing.

The other scenario where you might call `Stalker.follow` is perhaps from a function which has been [intercepted](https://frida.re/docs/javascript-api/#interceptor) or replaced. In this scenario, you have found a function of interest and you want to understand how it behaves, you want to see which functions or perhaps even code blocks the thread takes after a given function is called. Perhaps you want to compare the direction the code takes with different input, or perhaps you want to modify the input to see if you can get the code to take a particular path.

In either of these scenarios, although stalker has to work slightly differently under the hood, it is all managed by the same simple API for the user, `Stalker.follow`.

## Basic Operation
When the user calls `Stalker.follow`, under the hood, the javascript engine calls through to either `gum_stalker_follow_me` to follow the current thread, or `gum_stalker_follow(thread_id)` to follow another thread in the process. In the case of the former, the Link Register is used to determine the instruction at which to start stalking (in AARCH64 architecture, the Link Register (LR) is set to the address of the instruction to continue execution following the return from a function call).


## Options
Now lets look at the options when we follow a thread with stalker. Stalker generates events when a followed thread is being executed, these are placed onto a queue and flushed either periodically or when the queue is full. The size and time period can be configured by the options. Events can be generated for on a per-instruction basis either for calls, returns or all instructions. Or they can be generated on a block basis, either when a block is executed, or when it is instrumented by the stalker engine. The difference here is that blocks may be instrumented before they are executed as an optimization by stalker. 


Copy basic block to new location and instrument as required.
Relocate position dependent instructions.
Replace final branch insn with call back to stalker to manage the next block.
If we reach an already instrumented block we can re-use it.
If a block ends with a deterministic branch then we can patch that block to branch directly to the instrumented next block rather than via the engine.
When handling a call, we need to push the original return address on the stack so that we can unfollow later.
gum_stalker_follow_me re-compiles the instructions at its return address and replaces LR with the address of the recompiled code.
gum_stalker_follow(thread_id) captures the context of the given thread, re-compiles instructions at the PC and updates the PC accordingly.


