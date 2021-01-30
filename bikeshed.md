# The Bikeshed

This file exists to explore different options for the API, naming, etc.

The language in general here needs some bikesheddding, as "Lazy" in the context of JS is often associated with "Lazy Loading", but this is not what is happening here. For now, lacking the right word, "lazy-eval" will sometimes be used.


## The API

The api can take a couple of forms, here are some suggestions:

### Using import attributes

Example using a hypothetical import attribute named "lazyInit" (subject to discussion, some alternatives below):

```js
import {x} from "y" with { lazyInit: true }
import defaultName from "y" with { lazyInit: true }
import * as ns from "y" with { lazyInit: true }
```

Benefits:

* easy to read, clear intent
* clearly shows in the file that imports a lazy module, that there is lazy work being done
* per-use basis -- different files can import this as either lazy-eval or eager-eval

Drawbacks:

* requires the import attributes proposal
* per-use basis -- if you import a file lazily, you may want to import it lazily everywhere.

Alternative names:

* `... with { preferLazy: true}`
* `... with { lazyHint: true}`

### Using a custom Keyword

Example using the keyword "lazy" as in `lazy import` (subject to discussion):

```js
lazy import {x} from "y";
lazy import defaultName from "y";
lazy import * as ns from "y";
```

Benefits:

* easy to read, clear intent
* clearly shows in the file that imports a lazy module, that there is lazy work being done
* per-use basis -- different files can import this as either lazy-eval or eager-eval
* doesn't require any support proposals

Drawbacks:

* per-use basis -- if you import a file lazily, you may want to import it lazily everywhere.

### Using a directive

Example using the string "lazy-eval" (subject to discussion):

```js
"use lazy-eval";
import foo from "./bar";
//etc
```

Benefits:

* easy to read, clear intent
* per-module basis -- always lazy!
* doesn't require any support proposals

Drawbacks:

* per-module basis, you may want to be certain that the file is loaded
* requires the use of a directive :(
* hides laziness from importing files. It isn't clear that the execution order will be changed from
    the importer.

### Assert that modules are pure instead

Rather than specifying that something can be lazily initialized, we can instead assert that a module is pure:

```js
import {x} from "y" assert { pure: true }
```

This would mean that you wouldn't need to explicitly make something lazy, it would be treated as lazifiable by the engine instead. This solves the side effect issue, but it makes other aspects more difficult. For example, can you only export function or can you export constants? What kind of constants? How do we detect clever ways to break this etc. 

It also makes a distinction between what can and cannot be lazy. Throught the lazy getter, any module can be lazy. In this case, only a subset off modules can be lazy

### Address this via prefetched dynamic import

Raised in issue [#5](https://github.com/tc39/proposal-defer-import-eval/issues/5) it is possible to instead have a `prefetch` on dynamic import, and build this feature out of that.


```js
// showing a "simplified" version.
function lazyAMethod(...args) {
   const { aMethod } = import("./my-module", { with { prefetch: true }}); 
   return aMethod(...args);
}

Object.defineProperty(globalThis, "someConst", {
  get: function() {
    delete globalThis.someConst;
    const myModule = import("./my-module", { with { prefetch: true }}); 
    Object.defineProperty(globalThis, "someConst", {
      myModule.someConst,
      writable: true,
      configurable: true,
      enumerable: true,
    });
    return myModule.someConst;
  },
  configurable: true,
  enumerable: true,
});


function rarelyUsedA() {
  // ...
  lazyAMethod();
}

function alsoRarelyUsedA() {
  // ...
  use(someConst);
}
```

However the polyfill is quite heavy for this. Instead the ideal would be something that clearly communicates both cases.
