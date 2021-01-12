# Lazy Module Initialization

## Status

Champion(s): *Yulia Startsev and others*

Author(s): *Yulia Startsev and others*

Stage: 0

## Motivation

JS applications can get pretty large. It gets to the point that loading them incurs a significant performance cost, and usually, this happens later in an application's life span -- often requiring invasive changes to make it more performant.

Lazy loading is a common solution to this problem and modules present a natural boundary for loading,
as they also encapsulate meaningful information about a program. However, the current tools we have
for this are somewhat cumbersome, and reduce the ergonomics and readibility of code. The best tool
right now is `import()` but requires that all code relying on a lazily loaded module becomes async.

As a result, several solutions to hide the async nature of lazy loading have been introduced, both
server-side and client-side. This proposal seeks to explore the problem of loading large applications through this perspective.

## Background

Lazy loading is a common solution for startup performance. However this is a detail rather
meaningful information for programmers, and appears in its most problematic form on projects with
some complexity seeking additional performance. We can use the following example:

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

### Where dynamic import falls short

The pure js solution to this problem, as it is now, would be to use dynamic import. However, due to
the nature of dynamic import, we end up with large scale refactoring that is often not meaningful
for programmers in order to get the desired performance characteristics. For example, we end up with
the following:

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

This is an issue for a couple of reasons:

1) introduces a large refactoring task, including knockon effects in potentially distant code such
as `eventuallyCalled` in this example.
2) introduces information about async work, when the only goal here is to *hint* that something can
be deferred, not a semantic change to the code.

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

or using a custom keyword, in this case using `lazyImport` as a stand in:

```js
lazyImport {x} from "y";
lazyImport defaultName from "y";
lazyImport * as ns from "y";
```

## Semantics

Modules can be simplified to have three points at which laziness can be implemented:

1) Load
1) Parse

    1) if an import statement is found, go to step 1. Repeat for all imports.

1) Evaluate

In existing polyfills for this behavior, some implement laziness at point 1. This would mean that we do not fully build the module graph at load time. Implementing laziness at step 1 would potentially invalidate Run to completion semantics and require pausing in sync code. The desire for a "hint" for engines might not be enough of a reason to relinquish this invariant on it's own (please see the alternatives section for more info).

One problematic point to implement laziness is at point 2, due to point 2.i, which builds the rest of the module graph. Without parsing, we will be unable to have a complete module graph. We want the module graph to be present and complete, so that other work can also be
done.

Implementing lazy modules at point 3, the point of evaluation would relinquish some significant
performance advantages. At the very least implementations would be able to have a light-weight parse step for stage 2, and loading would be required.

Deferring the evaluation of a lazy module will be semantically different from what "regular" modules
currently do, which will evaluate the top level of the module eagerly.

The proposal in it's simplest form will pause at point 3, and for the present moment this is the
approach taken. Please see [Alternatives](#alternatives) for more ideas.

## Existing workarounds

### Client-side

Most modern frontend frameworks have a lazy loading solution built-in based off dynamic import. One characteristic is that
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
those not working in frameworks.

#### Lazy load it, but hide the async-ness

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
with is ["suspense" and "lazy" components](https://reactjs.org/docs/concurrent-mode-suspense.html).
Simplified example [here](https://gist.github.com/sebmarkbage/2c7acb6210266045050632ea611aebee).

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
  const aMethod =lazyAMethod();
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
    const temp = ChromeUtils.import(resource);
    return temp[symbol || name];
  });
};
```

This is only possible due to an exposed function, `ChromeUtils.import` which allows us to
synchronously load the file.

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
[alternatives](#alternatives) or, we need to somehow disallow this.

There are two possibilities for how to handle this case:
1) Throw an error if we come across an async module during the parse step
2) roll back the "lazy parse", and treat it as a normal module import -- possibly with a warning

### Impact on web related code

The solution described here assumes that the largest cost paid at start up is in building the
Abstract Syntax Tree and in evaluation, but this is only true if loading the file is fast. For
websites, this is not necessarily true - network speeds can significantly slow the performance. It
has also resulted in a delayed adoption of ESMs for performance sensitive code.

This said, the solution may lie in the same space. The batch preloading presentation by Dan Ehrenberg in the
[November 2020 meeting](https://github.com/tc39/notes/blob/master/meetings/2020-11/nov-19.md#batch-preloading-and-javascript) pointed a direction for JS bundles, which would offer a delivery mechanism. This would mean that the modules may be available on disk for quick access. This would mean that the same technique used for server side applications would also work for web applications.

This interplay is also clear in the ecosystem, where [webpack
codespliting](https://webpack.js.org/guides/code-splitting/) parses files to find imports, to then
have them split out modules in the bundle.

In the case that batch preloading outline above works, no extra handling will be necessary. In the
interim, bundlers would be able to use the syntax outlined here to do similar optimizations.

## Alternatives

TODO: Fill this out more.

### Co-routines

The issue highlighted above regarding top level await, and regarding async fetch requests, could be
mitigated by the introduction of co-routines. This would remove the invariant of run to completion.
There is some need for this potentially, as illustrated by the React usecase.

### Delegating work to blocking workers

This is loosely based off an idea of "transactional JS". This would delegate work to the Host, who
would run it in the context of a worker and block further execution until the result is ready.
Objects which require manipulation would need to be isolated (read-only to the worker),
and the modification could only be applied when the worker returns the result. This approach would
not invalidate Run to Completion, as the guarentee provided by Run to Completion is that the data
will not change in memory while the sync function runs unless it is changed by that function itself.

This leaves the issue of blocking the main thread.

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

