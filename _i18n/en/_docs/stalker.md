## Introduction

Stalker is FRIDA's code tracing engine. It allows threads to be followed, capturing every function, every block, even every instruction which is executed. A very good overview of the stalker engine is provided [here](https://medium.com/@oleavr/anatomy-of-a-code-tracer-b081aadb0df8) and I recommend that you read it carefully first. Obviously, the implementation is somewhat architecture specific, although there is much in common between them. Stalker is currently supports the ARM64 commonly found on mobile phones and tablets running Android or iOS as well as Intel x86_64 architecture commonly found on desktops and laptops. This page intends to take things to the next level of detail, it disects the ARM64 implementation of stalker and explains in more detail exactly how it works. It is hoped that this may help future efforts to port stalker to other hardware architectures.

## API
To start to understand the implementation of stalker, we must first understand in detail what it offers to the user. Whilst stalker can be invoked directly through its native gum interface, most users will instead call it via the [JavaScript API](https://frida.re/docs/javascript-api/#stalker) which will call these gum methods on their behalf. The [typescript type definitions](https://github.com/DefinitelyTyped/DefinitelyTyped/blob/master/types/frida-gum/index.d.ts) for gum are well commented and provide a little more detail still. The main API to stalker from JavaScript is:

```
Stalker.follow([threadId, options])
```

> start stalking `threadId` (or the current thread if omitted)

Let's consider when these calls may be used. 




Copy basic block to new location and instrument as required.
Relocate position dependent instructions.
Replace final branch insn with call back to stalker to manage the next block.
If we reach an already instrumented block we can re-use it.
If a block ends with a deterministic branch then we can patch that block to branch directly to the instrumented next block rather than via the engine.
When handling a call, we need to push the original return address on the stack so that we can unfollow later.
gum_stalker_follow_me re-compiles the instructions at its return address and replaces LR with the address of the recompiled code.
gum_stalker_follow(thread_id) captures the context of the given thread, re-compiles instructions at the PC and updates the PC accordingly.

