# Deferring Module Evaluation

previously known as "Lazy Module Initialization"

## Status

Champion(s): *Yulia Startsev and others*

Author(s): *Yulia Startsev and others*

Stage: 1

[Stage 1 Slides](https://github.com/tc39/proposal-defer-import-eval/issues/5#issuecomment-770208479)

## Motivation

JS applications can get pretty large. It gets to the point that loading them incurs a significant performance cost, and usually, this happens later in an application's life span -- often requiring invasive changes to make it more performant.

Introducing "laziness" – deferring non-essential work until later – is a common solution to this problem and modules present a natural boundary for loading,
as they also encapsulate meaningful information about a program. The best tool
right now is `import()` but it forces all code relying on a lazily loaded module to become async,
without necessarily reflecting the intention of the programmer.

In some cases, a programmer way want to sacrifice some performance to guarentee that the modules
they are working with are sync. This is where deferring module evaluation comes in. Since a module
can be deferred, some optimizations such as a preparse might be applied to get further startup
speedups.

## Background

Prior to the standardization of ES Modules, CommonJS, and later RequireJS was used by many developers as a module
system. There was one important difference between CommonJS/RequireJS and what we have today, that is --
the former load modules on demand, whereas the ES Modules system loads them eagerly. This change was
an important one to make modules possible on the web.

In some cases this difference in semantics has made it difficult for projects to adopt ES Modules in place of CommonJS.
Namely, the performance cost of eagerly loading, parsing and evaluating modules is too great. On the
client side, this was largely addressed by the introduction of Dynamic Import used as part of code
splitting. However, it also introduces a few issues. The main one is that it forces a codebase to be
async. This has two implications: 1) there is a high refactoring cost, and 2) there may be races.

This is not to say that dynamic import should be changed. It is, on it's own, a very useful tool and already contributing significantly to production codebases. The two techniques -- code splitting and deferring evaluation, are complimentary. Specifically, Dynamic import coupled with code splitting can reduce startup I/O cost whereas lazy import can reduce initialization cost. Different applications may favor one or the other.

## Description

Deferring the evaluation of a module may be desirable when the intention is not to run a module upon
loading. This can help startup performance, and may be especially useful when the goal is not to
load something from the network but to use local resources.

## Proposed API

The api can take a couple of forms, here are some suggestions:

Using import attributes (using `lazyInit` here to illustrate, but it could also be something like `deferEval` or similar:

```js
import {x} from "y" with { lazyInit: true }
import defaultName from "y" with { lazyInit: true }
import * as ns from "y" with { lazyInit: true }
```

or using a custom keyword, in this case using `lazy import` as a stand in:

```js
lazy import {x} from "y";
lazy import defaultName from "y";
lazy import * as ns from "y";
```

There are other proposals, but the discussion of the naming may derail the discussion of the
semantics, which should be agreed on first and would likely guide the design of the API.
A full writeup of these options can be found in the [Bikeshed](./bikeshed.md).

## Semantics

So, the question is, at which point do we start deferring work? Modules can be simplified to have three points at which work can be deferred. Let's take a look at each one.

1) Before Load (i.e. do not fetch the target module)
1) Before Parse (i.e. fetch but do not parse the target module)

    1) Note: after Parse, if an import statement is found, go to beginning for that resource. Repeat for all imports.

1) Before Evaluate (i.e. parse the module graph - the target and its children - but do not evaluate any of it)

Starting with **1) Before Load**: At present, we already have a technique for deferring at load, dynamic import. Dynamic
import must be asynchronous, otherwise we would break run-to-completion semantics. Dynamic import is
already a powerful tool in use on many applications.

If we consider **2) Before Parse**, we see that it is ill-advised to laziness is at this point, due to the note at 2.i. Without parsing, we will be unable to have a complete module graph. We want the module graph to be present and complete, so that other work can also be
done. So, **we do not want to defer parsing**. However, many engines may implement an optimization
to only partially parse the module, and find early errors and imports.

If we consider **3) Before Evaluate**, we would relinquish some significant performance advantages.
However, if those are important, a developer would be able to fall back on dynamic import, and trade
in sync code for more performance. Secondly, we would change the evaluation order. Presently, we
have an invariant that all child modules of a parent module will complete evaluation before their
parent module completes evaluation. What lazy-eval would introduce is a new concept. We would have a
subgraph that does not evaluate with the parent, but instead waits untill it is called.

### Impact on execution order

|                                | Static Import        | Dynamic Import               | "Lazy" Import  |
|--------------------------------|----------------------|------------------------------|----------------|
| Top level exectution of module | Load, Parse, Execute |                              |  Load, Parse   |
| First use                      |                      | Load, Parse, Execute         |  Execute       |

The proposal in it's simplest form will pause at point 3, and for the present moment this is the
approach taken. However, we can't be certain that we will get desirable performance characteristics
from this alone, or that it will be as useful for client side applications as it might be for
server-side applications (more on that in the next section).

### Rough sketch

If we split out the components of Module loading and initialization, we could roughly sketch out the
intended semantics:

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

## Code Splitting: a complimentary tool.

At present, the tool available to developers is dynamic import, often used as part of a technique
called code splitting. This technique works really well in a number
of cases.

Most modern frontend frameworks have a solution built-in based off dynamic import. One characteristic is that
they all mask the async nature of the code for the benefit of the programmer. This example is from vue:

```js
// file1.js
export default {
  components: {
    lazyComponent: () => import('Component.vue')
  }
}

// file2.js, lazy component is loaded based on a given trigger like scrolling or clicking a button.
<lazy-component v-if="false" />
```

For frontend developers working with a framework, the solutions provided by the framework are often
quite good. However a language feature providing this benefit would likely be welcome especially by
those not working in frameworks. This works well with code splitting -- where bundlers such as
webpack parse files for `import(..)` and create additional files that can be loaded async.

This strategy involves a hook that the developer can add that is called by the framework. The
framework has an update loop that is async, but the developer's code for the most part still
"appears" sync. This effectively allows the developer to write their code as though it was sync,
with the framework handling the rest.

- [angular](https://angular.io/guide/lazy-loading-ngmodules)
- [vue](https://vueschool.io/articles/vuejs-tutorials/lazy-loading-and-code-splitting-in-vue-js/)
- [ember](https://medium.com/zonky-developers/lazy-loading-modules-in-emberjs-e4f880b15aa0)
- [react has a hook "use effect" -- but this is primarily for data](https://www.robinwieruch.de/react-hooks-fetch-data)


This is a great solution in a lot of cases. Sometimes though, it introduces more problems than solutions.
It forces code to be async which could cause hard to find bugs. It may also be that deferring
loading is not very important. For example, below any function which relies on
`aMethod` which is imported lazily becomes async.

```js
async function lazyAMethod(...args) {
   const aMethod = await import("./a.js");
   return aMethod(...args);
}

async function rarelyUsedA() {
  // ...
  const aMethod = await lazyAMethod();
}

async function alsoRarelyUsedA() {
  // ...
  const aMethod = await lazyAMethod();
}

// ...

async function eventuallyCalled() {
  await rarelyUsedA();
}
```

In the case that the work that needs to be done is truely async (that is, we need to wait on I/O),
then this solution and cost makes sense. However, if we are more interested in splitting the cost of
initialization, it makes less sense.

To use Dynamic import in the case where lazy-eval import would be a more appropriate solution, we come
to the following issues:

1) It introduces a large refactoring task, including knockon effects in potentially distant code such
as `eventuallyCalled` in this example.
2) It introduces information to the programmer that async work is being done, when the only goal here is to *hint* that modules can
be deferred, not change the meaning of the code to be asynchronous rather than synchronous.
3) It introduces sometimes hard to debug race conditions.

To elaborate a bit more on point 2, if we consider server side applications, they often are not interested in making a fetch
request, they are accessing local resources. In the Firefox codebase, [we have dedicated utilities to load files synchronously](https://searchfox.org/mozilla-central/rev/07342ce09126c513540c1c343476e026cfa907bf/js/xpconnect/loader/mozJSComponentLoader.cpp#1184-1298). Attempts to use dynamic import on the other hand, [requires us to manually turn the
event loop (see lines 249-251)](https://phabricator.services.mozilla.com/D18858), something developers most certainly do not have access to.


## Existing workarounds

### Client-side

There is some evidence on the front end, that the async nature of dynamic import can be problematic,
and that a sync solution is necessary. The best known example comes from react.

Effectively, the framework uses a loop with a try/catch, which continuously
throws until the promise resolves. The only framework currently exploring this technique is React,
with its ["suspense" and "lazy" components](https://reactjs.org/docs/concurrent-mode-suspense.html).
Simplified example [here](https://gist.github.com/sebmarkbage/2c7acb6210266045050632ea611aebee).

This is resource intensive. It may also result in some unexpected behavior. As described in the
documentation for suspense, this has its best application in cases where you really want things to be sync,
but there is a performance issue. A situation like this may very well benefit from a different technique than dynamic import.
For example, a server can eagerly push resources via HTTP/2 to the client, and they can be loaded via lazy-eval import rather
than via dynamic import, as the cost for the load would be lower and the trade off may be worth it.

### JS Applications

For JavaScript that runs locally and outside of a browser context, the story is a bit different. In
the case of server side applications, having a slow start may not have a significant impact. On the
other hand, for tools on the CLI, or startup sensitive applits, it is a rather significant issue.

As a result, there has been some inertia in moving away from requireJS due to the performance impact.
Loading sync with require is by default, so the following is possible:

```js
function lazyAMethod(...args) {
   const aMethod = require("./a.js");
   return aMethod(...args);
}

function rarelyUsedA() {
  // ...
  const aMethod = lazyAMethod();
}

function alsoRarelyUsedA() {
  // ...
  const aMethod = lazyAMethod();
}

// ...

function eventuallyCalled() {
  rarelyUsedA();
}
```

In the case of Firefox DevTools, we have several variations of the following:

```js
function defineLazyGetter(object, name, lambda) {
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
}
exports.defineLazyModuleGetter = function(object, name, resource, symbol) {
  defineLazyGetter(object, name, function() {
    const temp = ChromeUtils.import(resource); // this calls the module component loader
    return temp[symbol || name];
  });
};
```

This is only possible due to an exposed function, `ChromeUtils.import` which allows us to
synchronously load the file.

Speaking to the Firefox case specifically, it was reported on DevTools that our most significant cost was
the full parsing and evaluation of the modules. At present, our custom solution excludes the ability
to use certain tooling, such as typescript, to validate our code and also restricts the
optimizations that we can perform. In an ideal world, we would transition all of our frontend code
to ES modules, rather than implementing a custom solution.

## Known issues

### Top Level Await

Top Level Await introdues a wrench into the works. If we delay evaluation until a module is used,
then we may have a module with top level await evaluated in the context of sync code, for example:

```js
// module.js
// ...
const data = await fetch("./data");

export { data };

// main.js
import { data } from "./module.js" with { lazyInit: true };

function run() {
  doSomeProcessing(data);
}
```

In order for this to work, we would need to pause in the middle of `run`, a sync function. However,
this is not allowed by the run to completion invariant. We would either need to break this (see
[alternatives](./alternatives.md) or, we need to somehow disallow this.

There are two possibilities for how to handle this case:
1) Throw an error if we come across an async module during the parse step
2) roll back the "lazy parse", and treat it as a normal module import -- possibly with a warning

There are also two implications here. The first is, whatever we choose, must be true for any
dependencies of a lazy module as well. So in the traversal of a "lazy" graph, we will need to throw
if any dependency is async. We may also need to include an additional piece of syntax, something to
the effect of:

```js
import { item } from "./file" assert { sync: true } with { lazyInit: true }
```

Another solution here, would be to eagerly evaluate any async modules, while leaving the rest of the
lazy graph, lazy. The lazy graph will in any case have some already-evaluated nodes, as other parts
off the module graph may share a resource with it.

### Impact on web related code

The solution described here assumes that the largest cost paid at start up is in building the
AST and in evaluation, but this is only true if loading the file is fast. For
websites, this is not necessarily true - network speeds can significantly slow the performance. It
has also resulted in a delayed adoption of ES Modules for performance-sensitive code.

This said, the solution may lie in the same space. The batch preloading presentation by Dan Ehrenberg in the
[November 2020 meeting](https://github.com/tc39/notes/blob/master/meetings/2020-11/nov-19.md#batch-preloading-and-javascript) pointed a direction for JS bundles, which would offer a delivery mechanism. This would mean that the modules may be available on disk for quick access. This would mean that the same technique used for server-side applications would also work for web applications.  Alternatively, HTTP2 servers could push resources eagerly to a client.

## Other Languages

The standard libraries of these programming languages includes related functionality:
- Ruby's `autoload`, in contrast with `require` which works in the same way as JS `import`
- Clojure `import`
- Most LISP environments

Our approach is pretty similar to the Emacs Lisp approach, and it's clear from a manual analysis of billions of Stack Overflow posts that this is the most straightforward to ordinary developers.

## Implementations

### Native implementations

None so far.

## Q&A

Q: what can we do in current JS to approximate this behavior?

A: The closest we can get is the following:

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
