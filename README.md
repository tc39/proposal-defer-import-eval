# Lazy Module Initialization

## Status

Champion(s): *Yulia Startsev and others*

Author(s): *Yulia Startsev and others*

Stage: 0

## Motivation

JS applications can get pretty large. It gets to the point that loading them incurs a significant performance cost, and usually, this happens later in an application's life span -- often requiring invasive changes to make it more performant.

Introducing "laziness" – deferring non-essential work until later – is a common solution to this problem and modules present a natural boundary for loading,
as they also encapsulate meaningful information about a program. However, the current tools we have
for this are somewhat cumbersome, and reduce the ergonomics and readibility of code. The best tool
right now is `import()` but it forces all code relying on a lazily loaded module to become async.

This proposal seeks to explore the problem of loading large applications through this perspective.

## Background

Prior to the standardization of ES Modules, RequireJS was used by many developers as a module
system. There was one important difference between requireJS and what we have today, that is --
requireJS loads modules on demand, whereas ES Modules load them eagerly.

The reasoning for the semantics of ES Modules is well founded. However, in some cases this
difference in semantics has made it difficult for projects to adopt ES Modules in place of Require.
Namely, the performance cost of eagerly loading, parsing and evaluating modules is too great.

Lazification is a common solution for startup performance. At present, the only solution here
is dynamic loading, which carries with it an additional cost of async-ifying everything.
We can use the following example:

```js
import aMethod from "a.js";

function rarelyUsedA() {
  // ...
  aMethod();
}

function alsoRarelyUsedA() {
  // ...
  aMethod();
}

// ...

function eventuallyCalled() {
  rarelyUsedA();
}
```

And consider a naive implementation with dynamic import:

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

The goal of the programmer is likely _not_ to make everything lazy, but instead to allow work to be
deferred in return for performance.

This problem space has partially been solved by code splitting, a technique employed by bundlers and built on top
of dynamic import. The two techniques are complimentary. Specifically, Dynamic import coupled with code splitting can reduce startup I/O cost whereas lazy import can reduce initialization cost.

## Description

Lazy import can defer the evaluation, and a large part of the parsing, of a module for the purposes
of ergonomic touch ups to performance.

## Proposed API

The api can take a couple of forms, here are some suggestions:

Using import attributes:

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

## Semantics

Lazy import would defer some of the work done eagerly by modules, and only do that work when the module's exported
object or it's properties are accessed.

So, the question is, at which point do we start deferring work? Modules can be simplified to have three points at which laziness can be implemented:

1) Before Load (i.e. do not fetch the target module)
1) Before Parse (i.e. fetch but do not parse the target module)

    1) Note: after Parse, if an import statement is found, go to beginning for that resource. Repeat for all imports.

1) Before Evaluate (i.e. parse the module graph - the target and its children - but do not evaluate any of it)

In existing polyfills for this behavior, some implement laziness at point 1. This would mean that we do not fully build the module graph at load time. Implementing laziness at point 1 would invalidate run-to-completion semantics and require pausing in sync code.

One problematic point to implement laziness is at point 2, due to point 2.i, which builds the rest of the module graph. Without parsing, we will be unable to have a complete module graph. We want the module graph to be present and complete, so that other work can also be
done.

Implementing lazy modules at point 3, before evaluation would relinquish some significant
performance advantages. At the very least implementations would be able to have a light-weight parse step for stage 2, and loading would be required.

Deferring the evaluation of a lazy module will be semantically different from what "regular" modules
currently do, which will evaluate the top level of the module eagerly.

The proposal in it's simplest form will pause at point 3, and for the present moment this is the
approach taken. However, we can't be certain that we will get desirable performance characteristics
from this alone, or that it will be as useful for client side applications as it might be for
server-side applications (more on that in the next section). Some [Alternative](./alternatives.md) explorations have been proposed.

## Code Splitting: a complimentary tool.

At present, the tool available to developers is dynamic import, often used as part of a technique
called code splitting. This technique works really well in a number
of cases. However, in some cases, it forces code to be async when it may not need to be. For
example, below any function which relies on `aMethod` which is imported lazily becomes async.

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

To use Dynamic import in the case where lazy import would be a more appropriate solution, we come
to the following issues:

1) It introduces a large refactoring task, including knockon effects in potentially distant code such
as `eventuallyCalled` in this example.
2) It introduces information to the programmer that async work is being done, when the only goal here is to *hint* that modules can
be deferred, not change the meaning of the code to be asynchronous rather than synchronous.

To elaborate a bit more on point 2, if we consider server side applications, they often are not interested in making a fetch
request, they are accessing local resources. In the Firefox codebase, [we have dedicated utilities to load files synchronously](https://searchfox.org/mozilla-central/rev/07342ce09126c513540c1c343476e026cfa907bf/js/xpconnect/loader/mozJSComponentLoader.cpp#1184-1298). Attempts to use dynamic import on the other hand, [requires us to manually turn the
event loop (see lines 249-251)](https://phabricator.services.mozilla.com/D18858), something developers most certainly do not have access to.


## Existing workarounds

### Client-side

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

As mentioned above, there may be cases in which dynamic import is used, where dynamic initialization
might be more appropriate.

#### Lazy-load it, but hide the async-ness

This strategy involves a hook that the developer can add that is called by the framework. The
framework has an update loop that is async, but the developer's code for the most part still
"appears" sync. This effectively allows the developer to write their code as though it was sync,
with the framework handling the rest.

- [angular](https://angular.io/guide/lazy-loading-ngmodules)
- [vue](https://vueschool.io/articles/vuejs-tutorials/lazy-loading-and-code-splitting-in-vue-js/)
- [ember](https://medium.com/zonky-developers/lazy-loading-modules-in-emberjs-e4f880b15aa0)
- [react has a hook "use effect" -- but this is primarily for data](https://www.robinwieruch.de/react-hooks-fetch-data)

#### Sync-Async

This strategy is closer in many ways to what server-side applications are able to do. This is a form
of a "co-routine". Effectively, the framework uses a loop with a try/catch, which continuously
throws until the promise resolves. The only framework currently exploring this technique is React,
with its ["suspense" and "lazy" components](https://reactjs.org/docs/concurrent-mode-suspense.html).
Simplified example [here](https://gist.github.com/sebmarkbage/2c7acb6210266045050632ea611aebee).

This work also points to a potential lack of dynamic import for all cases on the front end. This
requires more investigation, but it looks like lazy import and dynamic import are complimentary on
the frontend as well as serverside.

## Server-side and JS Applications

For JavaScript that runs locally and outside of a browser context, the story is a bit different.
There has been a great deal of inertia in moving away from requireJS due to the performance impact.
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

In the case of Firefox DevTools, we have several implementations of the following:

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

Or something like this.

### Impact on web related code

The solution described here assumes that the largest cost paid at start up is in building the
Abstract Syntax Tree and in evaluation, but this is only true if loading the file is fast. For
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

TODO.
