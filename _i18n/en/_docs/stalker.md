## Introduction

Stalker is FRIDA's code tracing engine. It allows threads to be followed, capturing every function, every block, even every instruction which is executed. A very good overview of the stalker engine is provided [here](https://medium.com/@oleavr/anatomy-of-a-code-tracer-b081aadb0df8) and I recommend that you read it carefully first. Obviously, the implementation is somewhat architecture specific, although there is much in common between them. Stalker is currently supports the ARM64 commonly found on mobile phones and tablets running Android or iOS as well as Intel x86_64 architecture commonly found on desktops and laptops. This page intends to take things to the next level of detail, it disects the ARM64 implementation of stalker and explains in more detail exactly how it works. It is hoped that this may help future efforts to port stalker to other hardware architectures.


## Use Cases
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

## Following
When the user calls `Stalker.follow`, under the hood, the javascript engine calls through to either `gum_stalker_follow_me` to follow the current thread, or `gum_stalker_follow(thread_id)` to follow another thread in the process. 

### gum_stalker_follow_me
In the case of `gum_stalker_follow_me`, the Link Register is used to determine the instruction at which to start stalking. In AARCH64 architecture, the Link Register (LR) is set to the address of the instruction to continue execution following the return from a function call, it is set to the address of the next instruction by instructions such as BL and BLR. As there is only one link register, if the called function is to call another routine, then the value of LR must be stored (typically this will be on the stack). This value will subsequently be loaded back from the stack into a register and the RET instruction used to return control back to the caller.

Let's look at the code for `gum_stalker_follow_me`. This is the function prototype:
```
GUM_API void gum_stalker_follow_me (GumStalker * self,
    GumStalkerTransformer * transformer, GumEventSink * sink);
```

So we can see the function is called by the v8 or duktape runtime passing 3 arguments. The first is a context of the stalker object. Note that there may be multiple of these if multiple threads are being stalked at once. The second is a transformer, this can be used to transform the instrumented code as it is being written (more on this later). The last parameter is the event sink, this is where the generated events are passed as the stalker engine runs.

```
#ifdef __APPLE__
  .globl _gum_stalker_follow_me
_gum_stalker_follow_me:
#else
  .globl gum_stalker_follow_me
  .type gum_stalker_follow_me, %function
gum_stalker_follow_me:
#endif
  stp x29, x30, [sp, -16]!
  mov x29, sp
  mov x3, x30
#ifdef __APPLE__
  bl __gum_stalker_do_follow_me
#else
  bl _gum_stalker_do_follow_me
#endif
  ldp x29, x30, [sp], 16
  br x0
  ```

We can see that the first instruction STP stores a pair of registers onto the stack. We can notice the expression `[sp, -16]!`. This is a (pre-decrement)[https://thinkingeek.com/2017/05/29/exploring-aarch64-assembler-chapter-8/] which means that the stack is advanced first by 16 bytes, then the two 8 byte register values are stored. We can see the corresponding instruction `ldp x29, x30, [sp], 16` at the bottom of the function. This is restoring these two register values from the stack back into the registers. But what are these two registers?

Well, `x30` is the Link Regster and `x29` is the frame register. Recall that we must store the link regsiter to the stack is we wish to call another function as this will cause it to be overwritten and we need this value in order that we can return to our caller. 

The frame pointer is used to point to the top of the stack at the point a function was called so that all the stack passed arguments and the stack based local variables can be access at a fixed offset from the frame pointer. Again we need to save and restore this as each function will have its value for this register, so we need to store the value which our caller put in there and restore it before we return. Indeed you can see in the next instruction `mov x29, sp` that we set the frame pointer to the current stack pointer.

We can see the next instruction `mov x3, x30`, puts the value of the link register into x3. The first 8 arguments on AARCH64 are passed in the registers x0-x8. So this is being put into the register used for the fourth argument. We then call (branch with link) the function `_gum_stalker_do_follow_me`. So we can see that we pass the first three arguments in x0-x2 untouched, so that `_gum_stalker_do_follow_me` receives the same values we were called with. Finally, we can see after this function returns, we branch to the address we receive as its return value. (In AARCH64 the return value of a function is returned in x0).

```
gpointer
_gum_stalker_do_follow_me (GumStalker * self,
                           GumStalkerTransformer * transformer,
                           GumEventSink * sink,
                           gpointer ret_addr)
```

### gum_stalker_follow
This routine has a very similar prototype to `gum_stalker_follow_me`, but has the additional thread_id parameter. Indeed, if asked to follow the current thread, then is will call that function. Let's look at the case when another thread id is specified though.

```
void
gum_stalker_follow (GumStalker * self,
                    GumThreadId thread_id,
                    GumStalkerTransformer * transformer,
                    GumEventSink * sink)
{
  if (thread_id == gum_process_get_current_thread_id ())
  {
    gum_stalker_follow_me (self, transformer, sink);
  }
  else
  {
    GumInfectContext ctx;

    ctx.stalker = self;
    ctx.transformer = transformer;
    ctx.sink = sink;

    gum_process_modify_thread (thread_id, gum_stalker_infect, &ctx);
  }
}
```

We can see that this calls the function `gum_process_modify_thread`. This isn't part of stalker, but part of gum itself. This function takes a callback with a context parameter to call passing the thread context structure. This callback can then modify the `GumCpuContext` structure and `gum_process_modify_thread` will then write the changes back. We can see the context structure below, as you can see it contains fields for all of the registers in the AARCH64 CPU. We can also see below the function prototype of our callback.

```
typedef GumArm64CpuContext GumCpuContext;

struct _GumArm64CpuContext
{
  guint64 pc;
  guint64 sp;

  guint64 x[29];
  guint64 fp;
  guint64 lr;
  guint8 q[128];
};
```

```
static void
gum_stalker_infect (GumThreadId thread_id,
                    GumCpuContext * cpu_context,
                    gpointer user_data)
```

So, how does `gum_process_modify_thread` work? Well it depends on the platform. On Linux (and Android) it uses the `ptrace` API (the same one used by GDB) to attach to the thread and read and write registers. But there are a host of complexities. On Linux, you cannot ptrace your own process (or indeed any in the same process group), so FRIDA creates a clone of the current process in its own process group and shares the same memory space. It communicates with it using a UNIX socket. This cloned process acts as a debugger, reading the registers of the original target process and storing them in the shared memory space and then writing them back to the process on demand. Oh and then there is `PR_SET_DUMPABLE` and `PR_SET_PTRACER` which control the permissions of who is allowed to ptrace our original process.

Now you will see that the functionality of `gum_stalker_infect` is actually quite similar to that of `_gum_stalker_do_follow_me` we mentioned earlier. Both function carry out essentially the same job, although `_gum_stalker_do_follow_me` is running on the target thread, but `gum_stalker_infect` is not, so it must write some code to be called by the target thread using the [gum_arm64_writer](https://github.com/frida/frida-gum/blob/76b583fb2cd30628802a6e0ca8599858431ee717/gum/arch-arm64/gumarm64writer.c) rather than calling functions directly.

We will cover these functions in more detail shortly, but first we need a little more background.

## Basic Operation


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


