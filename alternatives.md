# Alternatives

This section exists to document potential alternatives, however at present the proposal is focusing
on the more minimal approach of pausing before evaluation. If that fails, we may need to discuss
these in more detail.

## Access to individual module steps

Rather than introduce a new form of module import, we could expose the steps of dynamic import. This
would allow developers to write their own libraries specific to their needs. For example, if a
developer wishes to only load the file asynchronously, but evaluate the module
synchronously later. If we have the methods `import.load`, `import.parse`, and `import.eval`, a developer could writer a wrapper like this:

```js
// LazyModuleLoader.js
async function loadModuleAndDependencies(name) {
  const loadedModule = await import.load(`./${name}.js`); // load is async, and needs to be awaited
  const parsedModule = loadedModule.parse();
  await Promise.all(parsedModule.imports.map(loadModuleAndDependencies)); // load all dependencies
  return parsedModule;
}

export default async function lazyModule(object, name) {
  const module = await loadModuleAndDependencies(name);
  Object.defineProperty(object, name, {
    get: function() {
      delete object[name];
      const value = module.eval();
      Object.defineProperty(object, name, {
        value,
        writable: true,
        configurable: true,
        enumerable: true,
      });
      return value;
    },
    configurable: true,
    enumerable: true,
  });

  return object;
}

// myModule.js
import foo from "./bar";

etc.

// module.js
import LazyModule from "./LazyModuleLoader";
await LazyModule(globalThis, "myModule");

function Foo() {
  myModule.doWork() // first use
}
```

`load` and `parse` cannot really be separated, as they are intrinsically linked. But, that
flexibility can be given to the developer in theory. However, if we link the two steps, then we end
up with functioonally the same capability as the main proposal, but with a different syntax. So, for
this reason, exposing the low level constructs doesn't really make sense.

# Less likely solutions

The following is a record of some ideas which are less likely to be used, but are recorded so we
remember what we thought of.

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


