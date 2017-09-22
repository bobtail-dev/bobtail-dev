# 2.2.0

* Fixed bug with casting to `DepMaps`
* Fixed bug where one couldn't create an `ObsMap` from a `Set`
* `promiseBind` updated to use Promise API (`then`) rather than `done`; now officially
documented and supported
* `ObsBase` objects now come with `toCell`, `toArray`, `toMap`, and `toSet` methods, 
which are equivalent to `rx.cast(this, <type>)` and `<type>.from(this)`
* Upgrade to Jasmine 2.8.0
* Babel now transpiles from ES2017 to ES5

## 2.2.1
* Moved `eslint` and `uglify-es` out of `dependencies` (how did they get there?) and into `devDependencies`.

## 2.2.2
* Upgraded to `bobtail-rx@2.2.1`, which includes following changes:
** added support for resolving `promiseBind`s if the `Promise` rejects.
** added missing `keys` method to `ObsSet`.
** `flatten` now supports `Function`s. Any function `f` found by `flatten` will be `bind`ed to a cell, which will itself
then be flattened. This is to support desired semantics of the `rxt` templating DSL.