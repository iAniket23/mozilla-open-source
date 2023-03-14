# Bug 1820882
Convert toolkit/crashreporter to ES modules
[Bug 1820882](https://bugzilla.mozilla.org/show_bug.cgi?id=1820882)

# Description
ESmify the toolkit/crashreporter modules.  
Mentor - standard8.   
[Patch D172545](https://phabricator.services.mozilla.com/D172545)

# What are JSMs and ESMs?
JSM is a Mozilla-specific module system used in the Firefox codebase.  It’s a plain JavaScript file with `.jsm` file extension, and `EXPORTED_SYMBOLS` global variable, that defines the list of exported symbols (global variables) that can be imported from other JS files, by `ChromeUtils.import` API.
JSM is a per-process singleton, shared among all consumers in the process.
ESM is an acronym of the standard ECMAScript Module.  It has dedicated syntax for importing/exporting symbols. Usually, `.mjs` file extension is used.
System ESM replaces the Mozilla-specific JSM module system based on the standard ECMAScript module system. It uses `.sys.mjs` file extension. System ESM is also a per-process singleton.
