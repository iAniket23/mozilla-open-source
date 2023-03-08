# bug 1820882
Convert toolkit/crashreporter to ES modules

# description
ESmify the toolkit/crashreporter modules


# ESMification Docs
Public

ESMification: In-tree Migration Phase 1

The first phase of ESMification focuses on converting the existing JSM to ESM with minimal changes, keeping the behavior the same as much as possible.

The purpose of this document is to help the first phase of the JSM to ESM migration done by each team.

TL;DR: JSM-to-ESM conversion Walkthrough 
Index
This is a big document, so to help navigate it, here are some handy links:

Background and Frequently Asked Questions
What are JSMs and ESMs?
What’s new in system ESM?
What doesn’t change with system ESM?

New APIs and and compatibility layer
New APIs
Restrictions in system ESM
Compatibility Layer

Are you doing a conversion and just want to jump into it? Here is the quick start guide.
JSM-to-ESM conversion Walkthrough 
Terminology
What are JSMs and ESMs?
JSM is a Mozilla-specific module system used in the Firefox codebase.  It’s a plain JavaScript file with `.jsm` file extension, and `EXPORTED_SYMBOLS` global variable, that defines the list of exported symbols (global variables) that can be imported from other JS files, by `ChromeUtils.import` API and friends.
JSM is a per-process singleton, shared among all consumers in the process.

// JSM file : foo.jsm
const EXPORTED_SYMBOLS = [“X”];
const X = 10;


// Consumer JS file
const { X } = ChromeUtils.import(
  "resource://gre/modules/foo.jsm"
);
console.log(X); // 10


// Other consumer JS file that uses “lazy getter”
const lazy = {};
XPCOMUtils.defineLazyModuleGetter(lazy, {
  X: "resource://gre/modules/foo.jsm"
});
console.log(lazy.X); // 10

ESM is an acronym of the standard ECMAScript Module.  It has dedicated syntax for importing/exporting symbols. Usually, `.mjs` file extension is used.

System ESM replaces the Mozilla-specific JSM module system based on the standard ECMAScript module system. It uses `.sys.mjs` file extension. System ESM is also a per-process singleton.

// System ESM file : foo.sys.mjs
export const X = 10;


// Consumer system ESM file
import { X } from "resource://gre/modules/foo.sys.mjs";
console.log(X); // 10


// Other consumer JS file
const { X } = ChromeUtils.importESModule(
  "resource://gre/modules/foo.sys.mjs"
);
console.log(X); // 10


// Other consumer JS file that uses “lazy getter”
const lazy = {};
ChromeUtils.defineESModuleGetter(lazy, {
  X: "resource://gre/modules/foo.sys.mjs"
});
console.log(lazy.X); // 10
What’s new in system ESM?
Static import
Inside system ESM, other system ESMs can be imported with static `import` declaration.

import { AppConstants } from "resource://gre/modules/AppConstants.sys.mjs";

The static import is handled as part of the parsing of the module file, as specified in ECMAScript. Once all dependencies are compiled, the module script is executed.

Static import supports relative URI, that’s resolved based on the module’s URI.

import { AppConstants } from "./AppConstants.sys.mjs";

The primary difference between web based loading of modules and loading of a system ESM is that system ESMs are loaded synchronously. 

See ES module in chrome-priv xhtml  for how `ChromeUtils` API and static `import` works inside and outside the system module.
Always strict mode
ESM is always strict mode and `”use strict”;` is no longer necessary.

Example notable differences are the following:
no `with` statement
no legacy octal (e.g. `0377`)
the following are disallowed in variable names:
`let`, `static`, `implements`, `interface`, `package`, `private`, `protected`, `public`, `yield`, `await`
no duplicate parameter name (e.g. `function (a, a) {}`)

The automated script mentioned below automatically removes `”use strict”;` once bug 1792398 patch is landed.
No per-JSM global `this` object

The JSM environment has some incompatibility with the ES module, and that resulted in some new code conventions explained below.

JSM has the per-JSM global `this` object that holds the global variables, but the ES module doesn’t have the global `this` object.

Below, you will find different cases for using the per-JSM global `this`. Most of these have been rewritten already, however you may find some cases in-tree. Here is what they do, which may help you rewrite them:
Defining global variables
You will sometimes see  `this.foo = ...`. This form has been used to define global variables in JSM. ESMs do not have a `this`, and needs to be done by regular `var`, `let`, `const`, `function`, or `class` declarations (see bug 1610653). This has an additional use, explained in “Exporting Lexical Variable to `Cu.import` for compatibility”. 

Exporting lexical variable to `Cu.import` for compatibility

This is a *legacy* API that relied on the `this` value, as described above in “Defining global variables”.  

In the past, `Cu.import` returned the per-JSM global object with only non-lexical variables exposed. As a result, to expose fields to the importer, the importee required the `this.Foo = Foo;` expression to export lexical variables. In order to make it possible to automatically convert these files, we had to remove this pattern.

Bug 1768060 changed the behavior of `Cu.import`, and now lexical variables are also exposed, and `this.Foo = Foo;` hack is no longer necessary. However, in the future, these should be rewritten as exports `export var Foo = …`.

Note: In the ESM case, this causes slight de-optimization on the global non-lexical variable (see the comment for the CompileOptions field, for the details), and this will be removed once all `Cu.import` consumers are rewritten with the new API.
Defining lazy getters

The per-JSM global `this` has been used to define lazy getters, e.g. with `XPCOMUtils.defineLazyModuleGetters`, in order to achieve lazy getters that look like global variables. 

Again, given ESM does not have the global `this`, this is not possible within ESM, and the lazy getter needs to be defined on a plain object (see bug 1608279). Therefore, we have introduced a convention to define these properties on a `lazy` object.

Re-exporting a lazy getter (no longer possible)

In some cases, code will define a lazy getter on the global `this` object, and then re-export that value from itself by putting the name in `EXPORTED_SYMBOL`.  This is not possible within ESM. `EXPORTED_SYMBOL` only has access to names on the global object and no other object. So, this needs to be done differently. (see bug 1771463, bug 1771464, bug 1771476, and bug 1771478).

What doesn’t change with system ESM?
Per-process singleton

System ESM is also a per-process singleton and loaded into the shared global. The shared global is shared among all JSMs and ESMs in the process, and loading the same module multiple times doesn’t result in creating multiple instances of the module file.

NOTE: Any change to the shared global from JSM will be visible from ESM, and vice versa.

JSM and ESM files can be mixed

A single module import graph can contain both JSM and ESM.

Interface to the consumer

The interface exposed to the consumer of the system module doesn’t change, as long as the module exports the same symbols.

Lazy getters

System ESM can be used with lazy getters.

One restriction around lazy getter is that, ESM doesn’t have global `this`, and it needs to be defined on a plain object `lazy`.

Outside of system ESM (e.g. script loaded into chrome window), lazy getters for system ESM can be defined on global `this` and can be used in the same way as lazy getters for JSM.

URI except for the filename extension

System ESM can be loaded both from `chrome://` URI and `resource://` URI.
Definition in `moz.build` file
System ESM files referred to with`resource://` URI can be defined with `EXTRA_JS_MODULES` and `TESTING_JS_MODULES` in `moz.build` files.
Definition in `jar.mn` file

There’s no change in how the system module is referred to by `jar.mn` file, except for the filename extension.
New APIs

To work with system ESM, the following new APIs are added:
ChromeUtils.importESModule
`ChromeUtils.importESModule` synchronously loads the specified system ESM file, and returns the module’s namespace object.

const { AppConstants } = ChromeUtils.importESModule(
  "resource://gre/modules/AppConstants.sys.mjs"
);
// Use AppConstants

This corresponds to `ChromeUtils.import` for JSM.

ChromeUtils.defineESModuleGetters
`ChromeUtils.defineESModuleGetters` defines lazy getters on the given object, that synchronously loads the specified system ESM and replaces the getter with a data property with the exported symbol value.

const lazy = {};
ChromeUtils.defineESModuleGetters(lazy, {
  AppConstants: "resource://gre/modules/AppConstants.sys.mjs",
});
// Use lazy.AppConstants

This corresponds to `XPCOMUtils.defineLazyModuleGetters` for JSM.
Cu.isESModuleLoaded
`Cu.isESModuleLoaded` returns whether the given ESM is already loaded to the shared global.

if (Cu.isESModuleLoaded("resource://gre/modules/AppConstants.sys.mjs")) {
  ...
}

This corresponds to `Cu.isModuleLoaded` for JSM.
Cu.loadedESModules
`Cu.loadedESModules` returns an array of loaded ESM’s URIs. This API is discouraged. 

for (const uri of Cu.loadedESModules) {
  ...
}

This corresponds to `Cu.loadedModules` for JSM.
“esModuleURI” key for ChromeUtils.registerWindowActor
`ChromeUtils.registerWindowActor` is an existing API that registers a window actor.
JSM is currently registered with `moduleURI` parameter.
ESM can be registered with the `esModuleURI` parameter.

ChromeUtils.registerWindowActor("Screenshot", {
  parent: {
    esModuleURI: "resource:///modules/HeadlessShell.sys.mjs",
  },
  child: {
    esModuleURI: "resource:///modules/ScreenshotChild.sys.mjs",
  },
});

“esModuleURI” key for ChromeUtils.registerProcessActor
`ChromeUtils.registerProcessActor` is an existing API that registers a process actor.

ChromeUtils.registerProcessActor("AboutCertViewer", {
  parent: {
    esModuleURI: "resource://gre/modules/AboutCertViewerParent.sys.mjs",
  },
  child: {
    esModuleURI: "resource://gre/modules/AboutCertViewerChild.sys.mjs",
  },
});
“esModule” key for components.conf
`components.conf` is a pre-existing file that defines static components, possibly with JS-implementation.
JSM is used with the `jsm` property.
ESM can be used with the `esModule` property.

Classes = [
  {
    'cid': '{7913837c-9623-11ea-bb37-0242ac130002}',
    'contract_ids': ['@mozilla.org/embedcomp/prompt-collection;1'],
    'esModule': 'resource:///modules/PromptCollection.sys.mjs',
    'constructor': 'PromptCollection',
  },
]

Change in existing APIs
Cu.getModuleImportStack

`Cu.getModuleImportStack` keeps working for system ESMs, but has different behavior for static `import` declarations, because static `import` is handled at compile time, and it does not have runtime JS stack.

Until bug 1792404 gets fixed, the import stack for a module which is imported by static `import` is not collected.  Once the bug is fixed, the import stack persists with 2 parts:
static import (`* import [“URL”]`)
runtime JS stack (`FRAME_INDEX FUNCTION_NAME [“URL”:LINE:COLUMN]`)
Restrictions in system ESM

System ESM has the following restrictions, compared to non-system, standard ESM.
dynamic `import()` is not available
top-level `await` is not available

This restriction comes from the module loader being synchronous, which performs the load without using the event loop.

While dynamic `import()` isn’t available, the same thing can be done by calling `ChromeUtils.importESModule`, in the same way as `ChromeUtils.import` is currently used.

During Phase 1 migration, our goal is to migrate existing JSMs. Existing JSMs do not have access to top level await or dynamic import, as they don’t have a module system. For now, this behavior will be excluded from system ES Modules, even though they are available on Web based modules. Eventually, we may add this behavior. 
Compatibility Layer
During migration, we will have a period of incompatibility both for the in-tree code base, and out-of-tree codebases. To address this and simplify migration, the following compatibility layer is added:
Redirect ChromeUtils.import(“foo.jsm”) to ESM

While converting a single JSM, it can be overwhelming to convert all of its consumers. A good example of a difficult JSM to convert is `resource://gre/modules/AppConstants.jsm`, of which there are over 100 consumers. To make this easier to do in smaller chunks, we’ve made it possible to only convert a JSM to ESM, without converting its consumers. Concretely, you can convert a file from a JSM to an ESM, without touching any of its consumers. In this case, `ChromeUtils.import` for `*.jsm` or `*.js` or `*.jsm.js` URI is automatically redirected to `ChromeUtils.importESModule` for `*.sys.mjs`, and the conversion of the importers can happen later.

The algorithm is the following:

When `ChromeUtils.import` is called with `*.jsm` or `*.js` or `*.jsm.js` URI:
If the previous redirect is found in the cache, then:
return the `.sys.mjs` module in the cache
Else:
Try loading the file, as usual
If the load is successful, then:
Return the exports object, as usual
Else:
Try loading ES module, with the URI replacing `.jsm` or `*.js` or `*.jsm.js` suffix to `.sys.mjs` suffix
If the load is successful, then:
Put the module in the cache for redirect (used by step 1)
Return the module namespace object
Else:
Throw error that says no module is found, as usual

A System ESM loaded with `*.jsm` or `*.js` or `*.jsm.js` URI is the same instance as the one loaded with `*.sys.mjs` URI.  There are no duplicate instances.

Migration of a module can be done independently of its consumers as long as the following conditions are satisfied:
The set of exported symbols is same
The behavior of the exported symbols is same
The URI is same, except for the extension
The old extension is `jsm` or `*.js` or `*.jsm.js`
The new extension is `sys.mjs`

In case any of the above isn’t satisfied, check the “If the redirect doesn’t work” section below.

The redirect works for most of the legacy APIs:
`Cu.import`
`ChromeUtils.import`
`XPCOMUtils.defineLazyModuleGetter`
`XPCOMUtils.defineLazyModuleGetters`
`ChromeUtils.defineModuleGetter`
`ChromeUtils.defineLazyGetter`
`Cu.isModuleLoaded`
`Cu.loadedModules`
NOTE: Resource URI is visible always with `*.jsm` extension, even if it’s loaded with `*.js` extension
and anything that internally calls `mozJSModuleLoader::Import`

This allows migrating a single system module file, leaving the consumer of the module in other files as is. Here, the “consumer” includes out-of-tree files, such as privileged extensions and non-sandboxed AutoConfig files.

This compatibility layer will be kept until all in-tree and out-of-tree code is migrated to new APIs.

Cu.isJSModuleLoaded and Cu.loadedJSModules

Given `Cu.isModuleLoaded` and `Cu.loadedModules` returns the status of ESM as well, the following 2 new APIs are added to purely handle JSMs:

`Cu.isJSModuleLoaded`
Returns true if the JSM is loaded. This doesn’t check the corresponding ESM
`Cu.loadedJSModules`
Returns an array of loaded JSM’s URIs. The array doesn’t contain the URI for ESMs (with replacing the `.sys.mjs` with `.jsm`)

If the redirect doesn’t work

There can be some case that automatic redirect doesn’t work, for example with the following reason:
The old file extension doesn’t match any of `.jsm`, `.js`, or `.jsm.js`
The new file extension cannot be `.sys.mjs` due to harness incompatibility or something
The set of exported symbols or the behavior change
The filename or directory cannot be same

In such case, you can still keep the compatibility by the following:
Move the implementation to new ESM file (if tools etc doesn’t accept `.sys.mys` extension, it can use different extension than`.sys.mjs`, such as `.sys.mjs.js`)
Make the existing JSM a thin wrapper for the new ESM file
If necessary, add a thin wrapper function (see the example) in JSM file that keeps the existing behavior

Simple case
// old.jsm
const EXPORTED_SYMBOLS = [“SomeFunction”];
const { SomeFunction } = ChromeUtils.importESModule(“new.sys.mjs”);


// new.sys.mjs
export function SomeFunction() {
  ...
}
Complex case
// old.jsm
const EXPORTED_SYMBOLS = [“FunctionWithOldName”];
const { FunctionWithNewName } = ChromeUtils.importESModule(“new.sys.mjs”);
/* A thin wrapper to absorb the difference */
function FunctionWithOldName(a, b, c) {
  return FunctionWithNewName({a, b, c});
}


// new.sys.mjs
export function FunctionWithNewName({a, b, c}) {
  ...
}

JSM-to-ESM conversion Walkthrough for Phase 1
Most of the steps for the phase 1 migration are automated with `./mach esmify` command.

$ ./mach configure  # In case this is not yet performed
$ ./mach esmify [options] PATH_TO_JSM_OR_DIRECTORY

The command automatically converts specified JSMs into ESMs, and rewrites consumers in the specified file or inside the specified directory and subdirectories to use new APIs as much as possible.

# Example 1: When JSMs are used across the entire tree.
#
# This works for all cases, but takes some time (5~10 min)


#   * Convert all JSMs inside browser/components/pagedata into ESMs
$ ./mach esmify --convert browser/components/pagedata

#   * Rewrite consumers inside the entire tree that imports
#     the already-ESMified files in browser/components/pagedata with new API
#
#     NOTE: `--prefix` is optional. It’s the prefix of `.sys.mjs` file,
#           to specify the target of import, in case there are other unrelated
#           JSM imports for already-ESMified files and you don’t want to
#           rewrite them.
$ ./mach esmify --imports . --prefix=browser/components/pagedata

# Example 2: When all JSMs and consumers are inside a specific directory.
#
# This works when the JSM is local to the directory. This is faster than
# the example 1.


#   * Convert all JSMs inside browser/components/pagedata into ESMs
#   * Rewrite consumers inside browser/components/pagedata that import
#     already-ESM-ified JSMs with new API
$ ./mach esmify browser/components/pagedata

The scripts will hit some edge cases, and that part needs manual rewriting. The rest of this document will be dedicated to a description of what each step does, and how it may also be applied manually (in case any problems arise). The two last steps (Steps 2-g, 2-h) are not automated and optional, but recommended.
Step 1. Convert JSM to ESM (`./mach esmify --convert …`)
Step 1-a. Automatically renames JSM with `*.jsm` or `*.js` or `*.jsm.js` to `*.sys.mjs`

The filename extension of the system ESM must be `.sys.mjs` as much as possible, for the following reasons:
Make it clearer it’s system ESM
There are some differences between system ESM and the non-system ESM (see Restrictions section above), and the developers need to be aware of it when reading/writing the code
To apply different ESLint rules for system ESM and non-system ESM
Make the compatibility layer works, and avoid unnecessary breakage for out-of-tree code
The filename except for the extension shouldn’t be renamed as a part of the phase 1 migration.

Example
AppConstants.jsm => AppConstants.sys.mjs

Step 1-b. Automatically replaces references to `*.jsm` in build file, .e.g. `moz.build`

There’s no change on how `moz.build` works, between JSM and ESM. Just replacing the filename extension works.
Example
JSM
EXTRA_JS_MODULES += [
  "AppConstants.jsm",
]

ESM
EXTRA_JS_MODULES += [
  "AppConstants.sys.mjs",
]
Step 1-c. Automatically replaces `EXPORTED_SYMBOLS` with `export` declaration

JSM uses the `EXPORTED_SYMBOLS` global variable to list all exported symbols in the file.
In ESM, each declaration can be declared as a `export` declaration.

In the phase 1 migration, each exported symbol must be converted to a named export, instead of a default export, so that the consumer can still access the symbol by `const { AppConstants } = ...`.

Example
JSM
var EXPORTED_SYMBOLS = [“Foo”, “Bar”];
function Foo() {}
var Bar = 10;

ESM
export function Foo() {}
export var Bar = 10;

Step 1-d. Automatically rewrites ChromeUtils.importESModule with static import

Added by bug 1780301.

This is a rewrite on the result of step 2-a.

If `ChromeUtils.import` in a JSM is rewritten to `ChromeUtils.importESModule` by step 2-a, and the JSM is converted to ESM after that, the `ChromeUtils.importESModule` at the top-level of the ESM must be converted to static import.
Example

JSM
const { AppConstants } = ChromeUtils.importESModule(
  "resource://gre/modules/AppConstants.sys.mjs"
);
// Use AppConstants

ESM
import { AppConstants } from "resource://gre/modules/AppConstants.sys.mjs";
// Use AppConstants
Step 2. Automatically rewrites consumers to use new APIs  (`./mach esmify --imports …`)

All these steps are optional, given the above compatibility layer.

It is possible to import modules in two ways, through `ChromeUtils.importESModule` or `import { ... } from “module.sys.mjs`(static import). However there are caveats to both approaches:
`ChromeUtils.importESModule` works anywhere
`import { ... } from “module.sys.mjs”;` cannot be used outside of the top-level of system ESMs
The redirect mentioned in the compatibility layer section doesn’t apply to both APIs (the redirect happens only for JSM-to-ESM direction), and the new APIs can be used only when the target module is ESMified

The script will automatically select the appropriate rewrite, however -- if you do it manually, you may want to treat it as two separate steps:
Step 2-a. Automatically replaces `ChromeUtils.import` with `ChromeUtils.importESModule`

System ESM can be imported using the new API `ChromeUtils.importESModule`.

`ChromeUtils.importESModule` works in ESM, JSM, and non-system-module scripts.

WARNING: This is for the consumer of the ESMified modules. `ChromeUtils.import` inside ESMified modules can be replaced with `ChromeUtils.importESModule` only when the module imported is also ESMified. The Importer may be a JSM or an ESM. 

Example

JSM
const { AppConstants } = ChromeUtils.import(
  "resource://gre/modules/AppConstants.jsm"
);
// Use AppConstants

ESM
const { AppConstants } = ChromeUtils.importESModule(
  "resource://gre/modules/AppConstants.sys.mjs"
);
// Use AppConstants

Step 2-b. Automatically replaces `ChromeUtils.import` with static import declaration

At the top-level script of the system ESM file, static `import` declaration can be used to import other system ESMs.

Static import declaration is recommended over `ChromeUtils.importESModule` whenever possible.

`./mach esmify` command always uses the absolute URI for static imports.

WARNING: This is for the consumer of the ESMified modules. `ChromeUtils.import` inside ESMified modules can be replaced with a static import only when the module imported there is also ESMified.

WARNING: Static import for `*.sys.mjs` file outside of the system ESM loads the ESM file in the non-shared global, e.g. window, and it’s not a per-process singleton. This is caught by an ESLint rule.
Example

JSM
const { AppConstants } = ChromeUtils.import(
  "resource://gre/modules/AppConstants.jsm"
);
// Use AppConstants

ESM
import { AppConstants } from "resource://gre/modules/AppConstants.sys.mjs";
// Use AppConstants

Step 2-c. Automatically rewrites lazy getter for the module

System ESM can be defined as a lazy module getter with `ChromeUtils.defineESModuleGetters`.  This works almost the same way as `XPCOMUtils.defineLazyModuleGetters`.

There’s no single-module variant like `ChromeUtils.defineModuleGetter` or `XPCOMUtils.defineLazyModuleGetter`.

Defining the lazy getter with alias (`XPCOMUtils.defineLazyModuleGetter`’s 4th parameter) is not supported, and any such call sites need a rewrite.  See bug 1772969 for more details.

Inside a system ESM, lazy getters must be defined on a plain object, given that ESM doesn’t have the global `this` object.  For consistency, the name of the object must be `lazy`. (the rule). If the getter needs to be defined on a different object, such as a nested object, if the number of the cases is small, please add `eslint-disable-next-line` with a comment why it needs to be named differently.  If there are many such cases, feel free to reach out.  We can look into modifying the rule.
Example
JSM
const lazy = {}
ChromeUtils.defineModuleGetter(
  lazy,
  ”AppConstants”,
  "resource://gre/modules/AppConstants.jsm"
);
// Use lazy.AppConstants


// or
XPCOMUtils.defineLazyModuleGetters(lazy, {
  ”AppConstants”: "resource://gre/modules/AppConstants.jsm"
});
// Use lazy.AppConstants

ESM
const lazy = {}
ChromeUtils.defineESModuleGetters(lazy, {
  ”AppConstants”: "resource://gre/modules/AppConstants.sys.mjs"
});
// Use lazy.AppConstants

Step 2-d. Automatically rewrites `components.conf` for the module

(This step is optional, given the above compatibility layer, but encouraged to do at the same time)

If the module is used as a static component, `components.conf` entry needs to be rewritten.
Replace `jsm` key with `esModule`
Replace `.jsm` extension with `.sys.mjs`
Example
JSM
Classes = [
  {
    'cid': '{7913837c-9623-11ea-bb37-0242ac130002}',
    'contract_ids': ['@mozilla.org/embedcomp/prompt-collection;1'],
    'jsm': 'resource:///modules/PromptCollection.jsm',
    'constructor': 'PromptCollection',
  },
]

ESM
Classes = [
  {
    'cid': '{7913837c-9623-11ea-bb37-0242ac130002}',
    'contract_ids': ['@mozilla.org/embedcomp/prompt-collection;1'],
    'esModule': 'resource:///modules/PromptCollection.sys.mjs',
    'constructor': 'PromptCollection',
  },
]

Step 2-e. Automatically rewrites `ChromeUtils.registerWindowActor` for the module

(This step is optional, given the above compatibility layer, but encouraged to do at the same time)

If the module is used as a window actor, the window actor option needs to be rewritten.
Replace `moduleURI` key with `esModuleURI`
Replace `.jsm` extension with `.sys.mjs`
Example
JSM
ChromeUtils.registerWindowActor("Screenshot", {
  parent: {
    moduleURI: "resource:///modules/HeadlessShell.jsm",
  },
  child: {
    moduleURI: "resource:///modules/ScreenshotChild.jsm",
  },
});

ESM
ChromeUtils.registerWindowActor("Screenshot", {
  parent: {
    esModuleURI: "resource:///modules/HeadlessShell.sys.mjs",
  },
  child: {
    esModuleURI: "resource:///modules/ScreenshotChild.sys.mjs",
  },
});

Step 2-f. Automatically rewrites `ChromeUtils.registerProcessActor` for the module

(This step is optional, given the above compatibility layer, but encouraged to do at the same time)

If the module is used as a process actor, the process actor option needs to be rewritten.
Replace `moduleURI` key with `esModuleURI`
Replace `.jsm` extension with `.sys.mjs`
Example
JSM
ChromeUtils.registerProcessActor("AboutCertViewer", {
  parent: {
    moduleURI: "resource://gre/modules/AboutCertViewerParent.jsm",
  },
  child: {
    moduleURI: "resource://gre/modules/AboutCertViewerChild.jsm",
  },
});

ESM
ChromeUtils.registerProcessActor("AboutCertViewer", {
  parent: {
    esModuleURI: "resource://gre/modules/AboutCertViewerParent.sys.mjs",
  },
  child: {
    esModuleURI: "resource://gre/modules/AboutCertViewerChild.sys.mjs",
  },
});
Step 2-g. (not automated) Rewrite `Cu.isModuleLoaded` for the module

(This step is optional, given the above compatibility layer, but encouraged to do at the same time)

Whether the system ESM is loaded or not can be queried with `Cu.isESModuleLoaded`.

It cannot be queried with `Cu.isModuleLoaded`.

NOTE: Given the above compatibility layer, ESMified modules are visible as *.jsm file in `Cu.isModuleLoaded`.

Example
JSM
if (Cu.isModuleLoaded("resource://gre/modules/AppConstants.jsm")) {
  ...
}

ESM
if (Cu.isESModuleLoaded("resource://gre/modules/AppConstants.sys.mjs")) {
  ...
}

Given there are not many consumers in tree, there’s no automated script for this
Step 2-h. (not automated) Rewrite `Cu.loadedModules` for the module

(This step is optional, given the above compatibility layer, but encouraged to do at the same time)

System ESM is not listed in `Cu.loadedModules`. It's listed in `Cu.loadedESModules`.

NOTE: Given the above compatibility layer, ESMified modules are visible as *.jsm files in `Cu.loadedModules`.
Example
JSM
for (const uri of Cu.loadedModules) {
  ...
}

ESM
for (const uri of Cu.loadedESModules) {
  ...
}

Given there are not many consumers in tree, and that this is also hard to automate, there’s no automated script for this

Edge cases that need manual migration or extra steps

There are multiple known edge cases where the automatic rewrite doesn’t work.
Empty lines are added or removed

There's a limitation in `./mach esmify` that it cannot preserve the empty lines properly, or inserts extra empty lines, in some cases.

A known case is when an object property spans multiple lines.  `./mach esmify` (`recast` module used there. see issue) inserts empty line before and after the property.

If this happens, please revert the addition or removal of empty lines manually.
JSM is loaded also as CommonJS

If a JSM is loaded with both `ChromeUtils.import` and CommonJS `require`, the file contains some tricks to make it compatible with both of them, and also converting the file to ESM will break the CommonJS consumers.

In such case, there are multiple options:
Duplicate the file into JSM and CommonJS, and only convert the JSM file
Rewrite the CommonJS consumers to use the standard ESM, and remove the CommonJS code
JSM is loaded from CommonJS in DevTools

Devtools loader supports loading system ESM with `.sys.mjs` extension (see bug 1778314), but the consumers are not rewritten by `./mach esmify` command.

`require(“.../*.jsm”)` keeps working with the compatibility layer, and also rewriting it to use the `.sys.mjs` extension makes it loads system ESM directly.
JSM is loaded also as worker script

If a JSM is loaded with both `ChromeUtils.import` and worker’s `importScripts`, the file contains some tricks to make it compatible with both of them. However, converting the file to ESM breaks the worker consumers, as `importScripts` doesn’t operate on modules. Module support in workers is currently in progress (bug 1247687), and these modules will need to wait until that is finished to be converted. Once this patch is landed, support code such as this example, can be removed.
Filename collision

If there are 2 JSMs in the same directory that have different file extensions (e.g. `.jsm` vs `.js`), applying the automatic rewrite results in filename collision because both get `.sys.mjs` extension.  In such case, revert the result of `./mach esmify` and rename one of the files to avoid the collision, and then re-apply `./mach esmify`.

A known case is `toolkit/mozapps/extensions/AddonManager.jsm` vs `toolkit/mozapps/extensions/addonManager.js`.
Code not compatible with strict mode

As explained in the “Always strict mode” section above, ESM is always in strict mode, and there are multiple restrictions.  If a JSM has code not compatible with strict mode, the `./mach esmify` command throws errors during applying eslint.  In such case, fix the code to make it compatible with strict mode, and re-apply `./mach eslint --fix`.

JSM was not covered by preparation

We’ve done some preparation to make almost all JSMs ready for the migration, there can be some JSMs slipped through the process.  For example, if the JSM wasn’t covered by ESLint, it will be overlooked.

In such case, preparation steps mentioned in the “No per-JSM global `this` object” section above need to be done at the same time as migration.

JSM file is auto-generated

If the JSM file is auto-generated from other files, e.g. vendored files, the migration needs to be done manually.

JSM file is referred from unsupported build files

If the JSM file is referred from unsupported files that’s used during running `./mach` commands, `./mach esmify` will hit error during running `./mach eslint --fix` internally.  In such case, please fix the reference in the file and apply `./mach eslint --fix` manually.

The load for the JSM file is monitored by performance-related test cases

If the JSM file is loaded early in the startup process, there are some test cases that monitor the load for JSM files.  In that case, URI in those tests needs to be updated manually.

Known tests are the following:
https://searchfox.org/mozilla-central/source/browser/base/content/test/performance/browser_startup.js
https://searchfox.org/mozilla-central/source/browser/base/content/test/performance/browser_startup_content.js
https://searchfox.org/mozilla-central/source/browser/base/content/test/performance/browser_startup_content_subframe.js
https://searchfox.org/mozilla-central/source/toolkit/components/backgroundtasks/tests/browser/browser_xpcom_graph_wait.js
https://searchfox.org/mozilla-central/source/toolkit/components/extensions/test/xpcshell/test_ext_startup_perf.js

The file is loaded with older binary

Some files are loaded by the Android hostutils which is based on the old tree.
In such case, there can be the following situation:
In the new code some JSMs are already ESMified
In the hostutils code, those JSMs are still JSMs

httpd.js is confirmed to be loaded by hostutils, and it needs to be compatible both with the latest and the old revisions.

New JSM files

`./mach esmify` uses `tools/esmify/map.json` file to map resource URI to the source file.
If the file is added after the last `map.json` update, the import for the file is not handled by `./mach esmify --imports`.
Monitoring exceptions thrown by APIs

The error thrown by `ChromeUtils.importESModule` and other new APIs can be different from `ChromeUtils.import` and other old APIs.

We’re looking into fixing the behavior (bug 1791292).

Known differences:
When imported file is not found:
`ChromeUtils.import` reports DOM exception with `ex.result == NS_ERROR_NOT_AVAILABLE` or `ex.result ==NS_ERROR_FILE_NOT_FOUND`
`ChromeUtils.importESModule` reports JS exception with `ex.message == “Failed to load <URL>”`

Timeline

There’s no deadline for the in-tree migration.  Each team can schedule the migration.

Once the in-tree migration finishes, out-of-tree code starts migrating to use ESMified modules with new API, and also to ESMify their code, and once all consumers for old API goes away, we can remove the old APIs.

We’ll publish another document for out-of-tree migration once it gets ready.
Terminology

There are a few new terms introduced to make the discussion of the new API easier. In addition, the definitions of older concepts here are made concrete, to ensure everyone has a clear understanding of what is being discussed.
 
System module (formerly “JSM”)
Code loaded by the mozJSModuleLoader (formerly mozJSComponentLoader). Until now known as “JSM”, this term is used from this point on to also refer to equivalent code written as an ES module.
ESM, or ES module
abbreviation of ECMAScript Module.
System ESM
A system module written as an ESM.
JSM
A system module written in regular JS “script” syntax, with a `.jsm` extension (or `.js` extension for exceptional case)
Non-system ESM
A standard ES module file that is not loaded as a system ESM (and is not restricted to the subset supported by system ESM loader).
Shared global
A global object shared by all system modules, introduced by bug 1186409, in order to reduce the memory consumption.  inside JSM and ESM, `globalThis` points the shared global
per-JSM global
an object that holds JSM’s global variables,  introduced by bug 1186409, in order to reduce the memory consumption.  This does not exist for system ESM.



