# Deferring Module Evaluation

previously known as "Lazy Module Initialization"

## Status

Champion(s): *Nicolò Ribaudo*

Author(s): *Yulia Startsev, Nicolò Ribaudo and Guy Bedford*

Stage: 2.7

Slides:
- 2021-01 - [Stage 1](https://docs.google.com/presentation/d/17NsxHzAC2RlP5rB3wrns9O2Z-NduSpcm2_GOVo2TnKE) ([notes](https://github.com/tc39/notes/blob/main/meetings/2021-01/jan-28.md#defer-module-import-eval))
- 2022-11 - [Take two: Defer Module Evaluation](https://docs.google.com/presentation/d/10cn4SfVY20no6JmtWL72JLD6rmJ-dnafIfh8XmmC7mA) ([notes](https://github.com/tc39/notes/blob/main/meetings/2022-11/nov-30.md#deferred-module-evaluation))
- 2023-07 - [Deferred import evaluation for Stage 2](https://docs.google.com/presentation/d/1rSsVsFsnXQZ8pEGFwAGiVbVqndr4DHEUqTGEM9Au0_4) ([notes](https://github.com/tc39/notes/blob/main/meetings/2023-07/july-11.md#deferred-import-evaluation))
- 2024-04 - [Deferred import evaluation for Stage 2.7](https://docs.google.com/presentation/d/1oPEF8nA9Iq5cAqjN-FqMigNNfz6lWCUbNfIsEjRXf4Y/edit#slide=id.p)
- 2024-06 - [Deferred import evaluation for Stage 2.7 (2)](https://docs.google.com/presentation/d/1EjV6QbT4bvcOdWj-gCLwP5fcEWRfewzbrI3vOI11LA8/edit#slide=id.p)

## Background

JS applications can get very large, to the point that not only loading, but even executing their
initialization scripts incurs a significant performance cost. Usually, this happens later in an application's
life span - often requiring invasive changes to make it more performant.

Loading performance is a big and important area for improvement, and involves preloading techniques for
avoiding waterfalls and dynamic `import()` for lazily loading modules.

But even with loading performance solved using these techniques, there is still overhead for execution
performance - CPU bottlenecks during initialization due to the way that the code itself is written.

## Motivation

Avoiding unnecessary execution is a well-known optimization in the Node.js CommonJS module system,
where there is a smaller gap between load contention and execution contention. The common pattern
in Node.js applications is to refactor code to dynamically require as needed:

```js
const operation = require('operation');

exports.doSomething = function (target) {
  return operation(target);
}
```

being rewritten as a performance optimization into:

```js
exports.doSomething = function (target) {
  const operation = require('operation');
  return operation(target);
}
```

The consumer still is provided with the same API, but with a more efficient use of FS & CPU during
initialization time.

For ES modules, we have a solution for the lazy loading component of this problem via dynamic `import()`.

For the same example we can write:

```js
export async function doSomething (target) {
  const { operation } = await import('operations');
  return operation(target);
}
```

This avoids bottlenecking the network and CPU during application initialization, but there are still a
number of problems with this technique:

1. It doesn't actually solve the deferral of execution problem, since sending a network
   request in such a scenario would usually be a performance regression and not an improvement.
   A separate network preloading step would therefore still be desirable to achieve efficient
   deferred execution while avoiding triggering a waterfall of requests.

2. It forces all functions and their callers into an asynchronous programming model,
   without necessarily reflecting the real intention of the program. This leads to all call
   sites having to be updated into a new model, and cannot be made without a breaking API
   change to existing API consumers.

## Problem Statement

Deferring the synchronous evaluation of a module may be desirable new primitive to avoid unnecessary
CPU work during application initialization, without requiring any changes from a module API consumer
perspective.

Dynamic import does not properly solve this problem, since it must often be coupled with a preload step,
and enforces the unnecessary asyncification of all functions, without providing the ability to only defer
the synchronous evaluation work.

## Proposal

The proposal is to have a new syntactical import form which will only ever return a namespace exotic object.
When used, the module and its dependencies would not be executed, but would be fully loaded to the point
of being execution-ready before the module graph is considered loaded.

_Only when accessing a property of this module, would the execution operations be performed (if needed)._

This way, the module namespace exotic object acts like a proxy to the evaluation of the module, effectively
with [[Get]] behavior that triggers synchronous evaluation before returning the defined bindings.

The API will use the below syntax, following the phases model established by the
[source phase imports](https://github.com/tc39/proposal-source-phase-imports) proposal:

```js
// or with a custom keyword:
import defer * as yNamespace from "y";
```

## Semantics

The imports would still participate in deep graph loading so that they are fully populated into
the module cache prior to execution, however it the imported module will not be evaluated yet.

When a property of the resulting module namespace object is accessed, if the execution has
not already been performed, a new top-level execution would be initiated for that module.

In this way, a deferred module evaluation import acts as a new top-level execution node
in the execution graph, just like a dynamic import does, except executing synchronously.

There are possible extensions under consideration, such as deferred re-exports, but they are not
included in the current version of the proposal.

### Top-level await

Property access on the namespace object of a deferred module must be synchronous, and it's thus
impossible to defer evaluation of modules that use top-level await. When a module is imported
using the `import defer` syntax, its asynchronous dependencies together with their own transitive
dependencies are eagerly evaluated, and only the synchronous parts of the graph are deferred.

Consider the following example, where `a` is the top-level entry point:
<table><tr><td>

```js
// a
import "b";
import defer * as c from "c"

setTimeout(() => {
  c.value
}, 1000);
```

</td><td>

```js
// b
```

</td><td>

```js
// c
import "d"
import "f"
export let value = 2;
```

</td></tr><tr><td>

```js
// d
import "e"
await 0;
```

</td><td>

```js
// e
```

</td><td>

```js
// f
```

</td></tr></table>

Since `d` uses top-level await, `d` and its dependencies cannot be deferred:
- The initial evaluation will execute `b`, `e`, `d` and `a`.
- Later, the `c.value` access will trigger the execution of `f` and `c`.

### Rough sketch

If we split out the components of Module loading and initialization, we could roughly sketch out the
intended semantics:

> ⚠️ The following example does not take cycles into account

```js
// LazyModuleLoader.js
async function loadModuleAndDependencies(name) {
  const loadedModule = await import.load(`./${name}.js`); // load is async, and needs to be awaited
  const parsedModule = loadedModule.parse();
  await Promise.all(parsedModule.imports.map(loadModuleAndDependencies)); // load all dependencies
  return parsedModule;
}

async function executeAsyncSubgraphs(module) {
  if (module.hasTLA) return module.evaluate();
  return Promise.all(module.importedModules.map(executeAsyncSubgraphs));
}

export default async function lazyModule(object, name) {
  const module = await loadModuleAndDependencies(name);
  await executeAsyncSubgraphs(module);
  Object.defineProperty(object, name, {
    get: function() {
      delete object[name];
      const value = module.evaluateSync();
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

## Implementations

- engine262: https://github.com/nicolo-ribaudo/engine262/tree/defer-eval

## Q&A

#### What happened to the direct lazy bindings?

The initial version of this proposal included direct binding access for deferred evaluation via
named exports:

```js
import { feature } from './lib' with { lazyInit: true }

export function doSomething (param) {
  return feature(param);
}
```

where the deferred evaluation would only happen on _access_ of the `feature` binding.

There are a number of complexities to this approach, as it introduces a novel
type of execution point in the language, which would need to be worked through.

This approach may still be investigated in various ways within this proposal or an extension of it,
but by focusing on the module namespace exotic object approach first, it keeps the semantics
simple and in-line with standard JS techniques.

#### Is there really a benefit to optimizing execution, when surely loading is the bottleneck?

While it is true that loading time is the most dominant factor on the web, it is important to consider that many
large applications can block the CPU for of the range of 100ms while initializing the main application graph.

Loading times of the order of multiple seconds often take the focus for performance optimization work, and this
is certainly an important problem space, but the problem of freeing up the main event loop during initialization
remains a critical one when the network problem is solved, that doesn't currently have any easy solutions today
for large applications.

#### Is there prior art for this in other languages?

The standard libraries of these programming languages includes related functionality:
- Ruby's `autoload`, in contrast with `require` which works in the same way as JS `import`
- Clojure `import`
- Most LISP environments

Our approach is pretty similar to the Emacs Lisp approach, and it's clear from a manual analysis of billions of Stack Overflow posts that this is the most straightforward to ordinary developers.

#### Why not support a synchronous evaluation API on ModuleInstance

A synchronous evaluation API on the module expression and compartments [ModuleInstance](https://github.com/tc39/proposal-compartments/blob/master/0-module-and-module-source.md#module-instances)
object could offer an API for synchronous evaluation of modules, which could be compatible with
this approach of deferred evaluation, but it is only in having a clear syntactical solution for this use case,
that it can be supported across dependency boundaries and in bundlers to bring the full benefits of avoiding unnecessary
initialization work to the wider JS ecosystem.

#### What can we do in current JS to approximate this behavior?

The closest we can get is the following:

```js
// moduleWrapper.js
export default function ModuleWrapper(object, name, lambda) {
  Object.defineProperty(object, name, {
    get: function() {
      // Redefine this accessor property as a data property.
      // Delete it first, to rule out "too much recursion" in case object is
      // a proxy whose defineProperty handler might unwittingly trigger this
      // getter again.
      delete object[name];
      const value = lambda.apply(object);
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

// module.js
import ModuleWrapper from "./ModuleWrapper";
// any imports would need to be wrapped as well

function MyModule() {
 // ... all of the work of the module
}

export default ModuleWrapper({}, "MyModule", MyModule);

// parent.js
import wrappedModule from "./module";

function Foo() {
  wrappedModule.MyModule.bar() // first use
}
```

However, this solution doesn't cover deferring the loading of submodules of a lazy graph, and would
not acheive the characteristics we are looking for.

#### Why `import defer *` gives a different namespace object from `import *`?

If a deferred module throws while being evaluated, `ns.foo` will throw the evaluation error:

```js
// module-that-throws1
export let a = 1;
throw new Error("oops");
```
```js
// main1.js
import defer * as ns1 from 'module-that-throws';
try { ns1.a } catch (e) { console.log('caught', e) } // logs "oops"
```

Module namespace objects of modules that are _already evaluated_ and threw during evaluation do now re-throw an error on
property access:
```js
// module-that-throws2
import * as ns2 from 'module-that-throws';
globalThis.ns2 = ns2;
export let a = 1;
throw new Error("oops");
```
```js
// main2.js
import("module-that-throws").finally(() => {
  try { ns2.a } catch (e) { console.log('caught', e) } // Doesn't throw
});
```

This is not a problem today, because having access to the namespace object of a module that threw during evaluation is incredibly rare. However, it would be incredibly more common with `import defer` declarations. To guarantee that the behavior of `main1.js` is not affected by module previously loaded (and to avoid race conditions), `ns2.foo` must throw even if `module-that-throws` is already evaluated, and thus it cannot be the same namespace object as `import *`.

Another approach we considered (and discarded) was to always suppress evaluation errors on namespace property access, so that in the ecample above `ns1.a` would be guaranteed to _never_ throw and thus not be affected by unrelated modules that might have already triggered evaluation of `module-that-throws`.

#### Why not re-use import attributes (`import * as ns from "mod" with { defer: true }`)?

There are two reaosns why we chose to use an "import modifier" rather than an attribute:
1. Import attributes affect what a module _is_, but cannot change basic semantics of how ECMAScript modules behave: they are similar to adding query parameters to the imported URL, except that attributes are handled by the running environment rather than by the server. For example, `with { type: "json" }` behaves as if the imported module was a JavaScript file wrapped in ``export default JSON.parse(` ... the file contents ... `);``. `import defer` changes how namespace objects behave (by making them side-effectul, while before this proposal property access on namespace objects couldn't trigger any side effect): it cannot be expressed as a wrapped/modified "classic" ECMAScript module.
2. Together with the [source phase imports proposal](https://github.com/tc39/proposal-source-phase-imports), we are exposing multiple "phases" of module loading. The phases we've identified are: resolving a module given a specifier, fetching the module (these two both happens in hosts and not in ECMA-262), attaching modules to their execution and resolution context, linking modules together, and finally executing them. We are using `import` modifiers to represent modules processed up to one of those phases, without going all the way to finishing execution. These modifiers give more guarantees than import attributes: while `import "x" with { attr1: "val" }` and `import "x" with { attr2: "val2" }` might be two completely different modules, `import source s from "x"`, `import defer * as ns from "x"`, and `import "x"` all are guaranteed to load the same module, and that module will be executed at most once regardless of which "phase" it gets temporarely paused at (and then continued from).
