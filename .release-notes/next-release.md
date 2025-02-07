## Fixed a compile-time crash related to Pony-specific optimizations

Ticket #3784 tracked an issue related the MergeMessageSend optimization pass, which does Pony-specific optimizations within the LLVM optimizer pipeline.

It turned out that this pass was not written to handle the case of being run on a code block that contained sections of code that had already been optimized by this pass, and in such a case it failed an assertion error, crashing the compiler on such a program.

The fix was to disable running the pass on blocks in which we could detect signs of the optimization of that pass already being present.

Programs that were previously compiling okay should not see any difference in behavior or performance - the only programs that could potentially benefit from the skipped optimization were crashing, so there is no regression in their behavior.

