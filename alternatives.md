# Alternatives

This section exists to document potential alternatives, however at present the proposal is focusing
on the more minimal approach of pausing before evaluation. If that fails, we may need to discuss
these in more detail.

## Co-routines

The inspiration here comes from React's use of try/catch.

The issue highlighted above regarding top level await, and regarding async fetch requests, could be
mitigated by the introduction of co-routines. This would remove the invariant of run to completion.
There is some need for this potentially, as illustrated by the React usecase.

## Delegating work to blocking workers

We do not know yet if the performance characteristics of the basic proposal will be enough for the
server-side case. In considering this also on the front end, we might explore a more creative
solution.

One of the problem of the proposed solution is that all resources have to be loaded ahead, before
doing a lazy initialization. In the case of React, Frameworks have defined systems where the resources are not necessarily
fetched yet, and if it is not fetched then the framework behave as-if we were doing a transaction, by
unwinding everything and starting over. This resembles a co-routine, but might be done differently.

Today the main-thread cannot be blocked for multiple reasons. But we could block a worker thread as
long as we do not break the run-to-completion restriction. The run-to-completion ensures that the amount
of changes between 2 instructions remains bounded and known. We can easily add a `Promise.block()`
function to any `Promise` which does not execute JavaScript code on the Worker thread.

Thus, any code which is currently implemented as `sync` but depends on `Promise` resolution could be
moved to a worker thread, and use `Promise.block()` for waiting on the completion of Promises which
are either returning main-thread results computed asynchronously on the main-thread or returning
result of fetch function which do not involve changing the state of the Worker thread.

Once the worker work is completed, the data can be transfered back to the main thread by resolving a
Promise. This could be done behind the scenes.

Some problems remains on how to efficiently transfer contextual information to the worker thread,
without adding race conditions. To which some options might be to add a copy-on-write feature where
any data transfered is preserved as read-only while a worker is potentially using it, or the opposite.


