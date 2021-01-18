# The Bikeshed

This file exists to explore different options for the API, naming, etc.

The language in general here needs some bikesheddding, as "Lazy" in the context of JS is often associated with "Lazy Loading", but this is not what is happening here. For now, lacking the right word, "lazy-eval" will sometimes be used.


## The API

The api can take a couple of forms, here are some suggestions:

### Using import attributes

Example using a hypothetical import attribute named "lazyInit" (subject to discussion):

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

