---
layout: main
permalink: /index.html
title: browser-modules
---

<script type="text/javascript" src="browser-modules.js"></script>
<script type="text/javascript" src="install.js"></script>

The _browser-modules_ library is a synchronous, in-browser implementation of [CommonJs](http://wiki.commonjs.org/wiki/Modules/1.1). If you've landed here by chance, then it's likely your needs might be better served by [browserify](http://browserify.org/).


## Installing

_browser-modules_ provides a `realm` object, having `define()` and `require()` methods that you can use to define and require things with, for example:

``` javascript
realm.define('stuff', {});
var stuff = realm.require('stuff');
```

Typically however, and to allow _browser-modules_ to be CommonJs compliant, you will install the `realm` first:

``` javascript
realm.install();
define('stuff', {});
var stuff = require('stuff');
```

**Warning:** Having a global `define()` method may be enough to trigger AMD definitions in some buggy UMD modules &mdash; they should really check for `define.AMD`, but not all of them do.

## Defining Modules

In addition to being able to define and require _stuff_, you can also define and require JavaScript modules, for example:

``` javascript
define('acme/module1', function(require, exports, module) {
  exports.stuff = 'more stuff';
});
```

While _browser-module's_ `define()` method looks a little like the [CommonJs Wrapper](http://requirejs.org/docs/api.html#cjsmodule) described in the RequireJS documentation, it's synchronous, so that everything that will be required must be pre-loaded, and pre-defined.

One module can require another module internally, for example:

``` javascript
define('acme/module2', function(require, exports, module) {
  var acmeModule1 = require('acme/module1');
  exports.stuff = acmeModule1.stuff + ', etc';
});
```

and this can also be written using a relative _require-path_ since, logically, both modules reside within 'acme', for example:

``` javascript
define('acme/module2', function(require, exports, module) {
  var acmeModule1 = require('./module');
  exports.stuff = acmeModule1.stuff + ', etc';
});
```

## Node.js Modules

_browser-modules_ supports Node.js style modules too, so we could instead define our two modules like this:

``` javascript
define('acme/module1', function(require, exports, module) {
  module.exports = 'more stuff';
});

define('acme/module2', function(require, exports, module) {
  var moreStuff = require('./module1');
  module.exports = moreStuff + ', etc';
});
```

## Module Transpilation

The _browser-modules_ library is designed to be paired with an _out-of-process_ (e.g.  _file-watcher_ or _web-server_) module transpiler. So that, a CommonJs module that looks like this:

``` javascript
var acmeModule1 = require('./module');
exports.stuff = acmeModule1.stuff + ', etc';
```

can trivially be transpiled into this:

``` javascript
define('acme/module2', function(require, exports, module) {
var acmeModule1 = require('./module');
exports.stuff = acmeModule1.stuff + ', etc';
});
```

where the line numbers in the transpiled code are only one out, compared with the original code. Specifically, we created _browser-modules_ for [BladeRunnerJS](https://github.com/BladeRunnerJS/brjs), but it can equally well be used in other systems too.


## Circular Dependency Detection

The _browser-modules_ library provides a circular dependency feedback mechanism that would not be possible with a 100% CommonJs compliant implementation. Consequently, while this mechanism does not affect Node.js modules, it may prevent what would otherwise be valid CommonJs modules from loading, instead leading to circular dependency errors being thrown.

Specifically, _browser-modules_ will throw a circular dependency error if a CommonJs module is loaded that forms part of a circle, if that module has not exported any values before it requires the next module within the circle.

For example, given the following two modules:

``` javascript
define('module1', function(require, exports, module) {
  exports.stuff = 'stuff';
  require('module2');
});

define('module2', function(require, exports, module) {
  require('module1');
  exports.stuff = 'stuff';
});
```

then in a fully CommonJs compliant system these modules could be loaded in any order, but with _browser-modules_ it will only work provided 'module1' is required first. If 'module2' is required first then the following error will be generated:

```
Circular dependency detected: module2 => module1 -> module2
```

Although relaxing CommonJs compliance may seem strange, we believe there are good grounds for doing it. The rationale is as follows:

  1. We prefer using Node.js style modules to CommonJs modules, and for Node.js modules this change is wholly positive, since it makes it debugging circular dependency issues trivial.
  2. It's trivial to 'fix' a CommonJs module so that it works again when an error is encountered.
  3. It complements [BladeRunnerJS](https://github.com/BladeRunnerJS/brjs), which exports libraries as a _bag-of-classes_ rather than as a single encapsulated module, so that circular dependencies can be happened upon in third-party libraries.


## Sub-Realms

To aid testing, _browser-modules_ supports hierarchical realms, providing comparable functionality to what [mockery](https://github.com/mfncooper/mockery) provides to Node.js developers. This allows developers to install a sub-realm for the duration of a test, where arbitrary definitions within the realm can be replaced for the duration of that test, leaving the original realm unchanged and unaffected.

Specifically, sub-realms must be used exactly as follows:

  1. Install the sub-realm.
  2. Override any definitions within the realm.
  3. Start requiring things.
  4. Un-install the sub-realm.

Consider the following test for example:

``` javascript
describe('module2', function() {
  var subrealm;

  beforeEach(function() {
    subrealm = realm.subrealm();
    subrealm.install();
  });

  afterEach(function() {
    subrealm.uninstall();
  });

  it('appends ", etc" to the value in module1', function() {
    define('acme/module1', function(require, exports, module) {
      module.exports = 'less stuff';
    });

    expect(require('acme/module2')).toBe('less stuff, etc');
  });
});
```

If any of the steps are done out of order, then things will start to go wrong.

For example, if you start requiring modules before you've finished overriding all the definitions, then the transitive dependencies of the modules you've required may well cause you to end up using the original definitions of modules you planned to override.


### Augmenting Definitions WIthin Sub-Realms

If you need to augment a module within a sub-realm, rather than completely replace it, then you need some way to get a reference the original definition in the sub-realm, even though you've overridden it. This can be done using the `subrealm.recast()` method, as follows:

``` javascript
define('acme/module1', function(require, exports, module) {
  var moreStuff = subrealm.recast('acme/module1');
  module.exports = moreStuff + '?';
});
```
