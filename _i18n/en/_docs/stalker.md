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
Code can be thought of as a series of blocks of instructions. Each block starts with an optional series of instructions which run in sequence. And ends when we encounter an instruction which causes (or can cause) execution to continue with an instruction other than the one immediately following it in memory.

Stalker works on one block at a time. It starts with either the block after the return to the call to `gum_stalker_follow_me` or the block of code to which the instruction pointer of the target thread is pointing when `gum_stalker_follow` is called.

Stalker works by allocating some memory and writing to it a new instrumented copy of the original block. Instructions may be added to generate events, or carry out any of the other features the stalker engine offers. Stalker must also relocate instructions as necessary. Consider the following instruction:

```
ADR
Address of label at a PC-relative offset.

ADR  Xd, label

Xd
Is the 64-bit name of the general-purpose destination 
register, in the range 0 to 31.

label
Is the program label whose address is to be calculated. 
It is an offset from the address of this instruction, 
in the range ±1MB.
```

If this instruction is copied to a different location in memory and executed, then because the address of the label is calculated by adding an offset to the current instruction pointer, then the value would be different. Fortunately, gum has a [relocator](https://github.com/frida/frida-gum/blob/76b583fb2cd30628802a6e0ca8599858431ee717/gum/arch-arm64/gumarm64relocator.c) for just this purpose which is capable of modifying the instruction given its new location so that the correct address is calculated.

Now, recall we said that stalker works one block at a time. How, then do we instrument the next block? We remember also that each block also ends with a branch instruction, well if we modify this branch to instead branch back into the stalker engine, but ensure we store the destination of where the branch was intending to end up, we can instrument the next block and re-direct execution there instead. This same simple process can continue with one block after the next.

Now, this process can be a little slow, so there are a few optimizations which we can apply. First of all, if we execute the same block of code more than once (e.g a loop, or maybe just a function called multiple times) we don't have to re-instrument it all over again. We can just re-execute the same instrumented code. For this reason, a hashtable is kept of all of the blocks which we have encountered before and where we put the instrumented copy of the block.

Secondly, we can instrument blocks ahead of time. For example, if we encounter a call instruction, it is pretty likely (unless it throws an exception) that the callee will eventually return and block immediately following the call will be executed. So we can instrument this block at the same time we instrument the block which is being called. Whilst we may still return into stalker following the call before we are re-directed to the already instrumented block, we may not need to store quite so much of the CPU state when entering and exiting the engine and so may some time. This doesn't work for all branches, however, many branches may never be taken. Consider all the error handling code in a normal program. Assuming the input is valid none of it will run, or the first problem with the input will be detected and only that error handler will run. So if we were to instrument both paths of every branch instruction, we would end un instrumenting a lot of code which is never run. This would take up valuable time and memory.

Finally, if a block of code ends with a deterministic branch (e.g. the destination is fixed and the branch is not conditional) then rather than replacing that last branch with a call back to stalker to instrument the next block, we can instrument the next block ahead of time and direct control flow there without having to re-enter the stalker engine. This process is called backpatching. In actual fact, we can deal with conditional branches too, if we instrument both blocks of code (the one if the branch is taken and the one if it isn't) then we can replace the original conditional branch with one conditional branch which directs control flow to instrumented version of the block encountered when the branch was taken, followed by a unconditional branch to the other instrumented block. We can also deal partially with branches where the target is not static. Say our branch is something like:

```
BR x0
```

This sort of instruction is common when calling a function pointer, or class method. Whilst the value of x0 can change, quite often it will actually always be the same. In this case, we can replace the final branch instruction with code which compares the value of x0 against our known function, and if it matches branches to the address of the instrumented copy of the code. This can then be followed by an unconditional branch back to the stalker engine if it doesn't match. So if the value of the function pointer say is changed, then the code will still work and we will re-enter stalker and instrument wherever we end up, however, if as we expect it remains unchanged then we can bypass the stalker engine altogether and go straight to the instrumented function.


## Options
Now lets look at the options when we follow a thread with stalker. Stalker generates events when a followed thread is being executed, these are placed onto a queue and flushed either periodically or when the queue is full. The size and time period can be configured by the options. Events can be generated for on a per-instruction basis either for calls, returns or all instructions. Or they can be generated on a block basis, either when a block is executed, or when it is instrumented by the stalker engine. The difference here is that blocks may be instrumented before they are executed as an optimization by stalker. Recall that when a call instruction is encountered, the callee and the block of the caller following the instruction is instrumented as we can be pretty sure it will run. This may be useful for code covereage.

We can also provide one of two callbacks `onReceive` or `onCallSummary`. The former will quite simply deliver an ordered list of all of the events generated by stalker. The second aggregates these results simply returning a count of times each function was called. This is more efficient that `onReceive`, but the data is much less granular.

## Terminology

Before we can carry on with describing the detailed implementation of stalker, we first need to understand some key terminology and concepts that are used in the design.

### Probes
Whilst a thread is running outside of stalker, you may be familiar with using `Interceptor.attach` to get a callback when a given function is called. When a thread is running in stalker, however, these interceptors won't work. These interceptors work by patching the first few instructions (prologue) of the target function to re-direct execution into FRIDA. Frida copies and relocates these first few instructions somewhere else so that after the `onEnter` callback has been completed, it can re-direct control flow back to the original function.

The reasons these won't work within stalker is simple, the original function is never called. Each block, before it is executed is instrumented elsewhere in memory and it is this copy which is executed. Stalker supports the API function `Stalker.addCallProbe(address, callback[, data])` to provide this functionality instead. The optional data parameter is passed when the probe callback is registered and will be passed to the callback routine when executed. This pointer, therefore needs to be stored in the stalker engine. Also the address needs to be stored, so that when an instruction is encountered which calls the function, the code can instead be instrumented to call the function first. As multiple functions may call the one to which you add the probe, many instrumented blocks may contain additional instructions to call the probe function. Thus whenever a probe is added or removed, the cached instrumented blocks are all destroyed and so all code has to be re-instrumented.

### Trust Threshold
Recall that one of the simple optimizations we apply is that if we attempt to execute a block more than once, on subsequent occassions, we can simply call the instrumented block we created last time around? Well, that only works if the code we are instrumenting hasn't changed. In the case of self-modifying code (which is quite often used as an anti-debugging/anti-disassembly technique to attempt to frustrate analysis of security critical code) the code may change, and hence the instrumented block cannot be re-used. So, how do we detect if a block has changed? We simply keep a copy of the original code in the data-structure along with the instrumented version. Then when we encounted a block again, we can compare the code we are going to instrument with the version we instrumented last time and if they match, we can re-use the block. But performing the comparison every time a block runs may slow things down. So again, this is an area where stalker can be customized.

> `Stalker.trustThreshold`: an integer specifying how many times a 
> piece of code needs to be executed before it is assumed it can be 
> trusted to not mutate. Specify -1 for no trust (slow), 0 to trust 
> code from the get-go, and N to trust code after it has been 
> executed N times. Defaults to 1.

In actual fact, the value of N is the number of times the block needs to be re-executed and match the previously instrumented block (e.g. be unchanged) before we stop performing the comparison. Note that the original copy of the code block is still stored even when the trust threshold is set to `-1` or `0`. Whilst it is not actually needed for these values, it is expected it has been retained to keep things simple, or perhaps to allow the setting to be dynamically changed whilst stalker is running. In any case, neither of these is the default setting.

### Excluded ranges. 
Stalker also has the API `Stalker.exclude(range)` consist of a base and limit and are used to prevent stalker from instrumenting code within these regions. Consider, for example, you thread calls a `malloc` inside `libc`. You most likely don't care about the inner workings of the heap and this is not only going to slow down performance, but also generate a whole lot of extraneous events you don't care about. One thing to consider, however, is that as soon as a call is made to an excluded range, stalking of that thread is stopped until it returns. That means, if that thread were to call a function which is not inside a restricted range, a callback perhaps, then this would not be captured by stalker. Just as this can be used to stop the stalking of whole library, it can be used to stop stalking a given function (and its callees) too. This can be particularly useful if your target application is statically linked. Here, was cannot simply ignore all calls to `libc`, but we can find the symbol for `malloc` using `Module.enumerateSymbols()` and ignore that single function.

### Freeze/Thaw. 
As an extension to DEP, some systems, prevent pages from being marked writeable and executable at the same time. Thus FRIDA must toggle the page permissions between writeable and executable to write instrumented code, and allow that code to execute respectively. When pages are executable, they are said to be frozen (as they cannot be changed) and when they are made writeable again, they are considered thawed.

### Frames
Whenever stalker encounters a call, it stores the return address and the address of the intstumented return block in a structure and adds these to a stack stored in a data-structure of its own. It uses when emitting call and return events as well as tracking call depth.

```
typedef struct _GumExecFrame GumExecFrame;
struct _GumExecFrame
{
  gpointer real_address;
  gpointer code_address;
};
```

### Transformer
A `GumStalkerTransformer` type is used to generate the instrumented code. The implementation of the default transformer looks like this:

```
static void
gum_default_stalker_transformer_transform_block (
    GumStalkerTransformer * transformer,
    GumStalkerIterator * iterator,
    GumStalkerWriter * output)
{
  while (gum_stalker_iterator_next (iterator, NULL))
  {
    gum_stalker_iterator_keep (iterator);
  }
}
```

It is called by the function responsible for generating instrumented code, `gum_exec_ctx_obtain_block_for` and its job is to generate the instrumented code. We can see that it does this using a loop to process on instruction at a time. First retrieving an instruction from the iterator, then telling stalker to instrument the instruction as is (without modification). These two functions are implemented inside stalker itself. The first is responsible for parsing a `cs_insn` and updating the internal state. This `cs_insn` type is a datatype used by the internal [capstone](http://www.capstone-engine.org/) disassembler to represent an instruction. The second is responsible for writing out the instrumented instruction (or set of instructions). We will cover these in more detail later.

Rather than using the default transformer, the user can instead provide a custom implementation which can replace and insert instructions at will. A good example is provided in the [API documentation](https://frida.re/docs/javascript-api/#stalker).


### Callouts
Transformers can also make callouts. That is they instruct stalker to emit instructions to make a call to a javascript (or CModule) function passing the cpu context and an optional context parameter. This function is then able to modify or inspect registers at will. This information is stored in a ```GumCallOutEntry```.

```
typedef void (* GumStalkerCallout) (GumCpuContext * cpu_context,
    gpointer user_data);
    
typedef struct _GumCalloutEntry GumCalloutEntry;

struct _GumCalloutEntry
{
  GumStalkerCallout callout;
  gpointer data;
  GDestroyNotify data_destroy;

  gpointer pc;

  GumExecCtx * exec_context;
};
```

### EOB/EOI
Recall that the [relocator](https://github.com/frida/frida-gum/blob/76b583fb2cd30628802a6e0ca8599858431ee717/gum/arch-arm64/gumarm64relocator.c) is heavily involved in generating the instrumented code. It has two important properties which control its state.

End of Block (EOB) indicates that the end of a block has been reached. This occurs when we encounter *any* branch instruction. A branch, a call, or a return instruction.

End of Input (EOI) indicates that not only have we reached the end of a block, but we have possibly reached the end of the input. e.g. what follows this instruction may not be another instruction. Whilst this is not the case for a call instruction as code control will pass back when the callee returns and so more instructions must follow (note that a compiler will typically generate a branch instruction for a call to a non-returning function like `exit`), if we encounter a branch instruction, or a return instruction, we have no guarantee that code will follow afterwards.


### Prologues/Epilogues
When control flow is re-directed from the program into the stalker engine, the registers of the CPU must be saved so that stalker can run and make use of the registers and restore them before control is passed back to the program so that no state is lost.

The [Procedure Call Standard](https://static.docs.arm.com/den0024/a/DEN0024A_v8_architecture_PG.pdf] for AARCH 64 states that some regsiters (notably x19 to x29) are callee saved registers. This means that when the compiler generates code which makes use of these registers, it must store them first. Hence it is not strictly necessary to save these registers to the context structure, since they will be restored if they are used by the code within the stalker engine. This *"minimal"* context is sufficient for most purposes.

However, if the stalker engine is to call a probe registered by `Stalker.addCallProbe`, or a callout created by `iterator.putCallout` (called by a Transformer), then these callbacks will expect to receive the full cpu context as an argument. And they will expect to be able to modify this context and for the changes to take effect when control is passed back tot he application code. Thus for these instances, we must write a *"full"* context and its layout must match the expected format dictated by the structure `GumArm64CpuContext`.

```
typedef struct _GumArm64CpuContext GumArm64CpuContext;

struct _GumArm64CpuContext
{
  guint64 pc;
  guint64 sp; /* x31 */
  guint64 x[29];
  guint64 fp; /* x29 - frame pointer */
  guint64 lr; /* x30 */
  guint8 q[128]; /* FPU, NEON (SIMD), CRYPTO regs */
};
```

Note however, that the code necessary to write out the necessary cpu registers (the prologue) in either case is quite long (tens of instructions). And the code to restore them afterwards (the epilogue) is similar in length. We don't want to write these at the beginning and end of every block we instrument. Therefore we write these (in the same way we write the instrumented blocks) into a common memory location and simply emit call instructions at the beginning and end of each instrumented block to call these functions. The following functions create these prologues and epilogues.

```
static void gum_exec_ctx_write_minimal_prolog_helper (
    GumExecCtx * ctx, GumArm64Writer * cw);
    
static void gum_exec_ctx_write_minimal_epilog_helper (
    GumExecCtx * ctx, GumArm64Writer * cw);
    
static void gum_exec_ctx_write_full_prolog_helper (
    GumExecCtx * ctx, GumArm64Writer * cw);
    
static void gum_exec_ctx_write_full_epilog_helper (
    GumExecCtx * ctx, GumArm64Writer * cw);
```
Finally, note that in AARCH64 architecture, it is only possible to make a direct branch to code within 1MB of the caller and using an indirect branch is more expensive (both in terms of code size and performance). Therefore, as we write more and more instrumented blocks, we will get further and further away from the shared prologue and epilogue. If we get more than 1MB away, we simply write out another copy of these prologues and epilogues to use. This gives us a very reasonable tradeoff.


### Counters 
Finally, there are a series of counters which you can see kept recording the number of each type of instructions encountered at the end of an instrumented block. These appear to only be used by the unit testing framework.

