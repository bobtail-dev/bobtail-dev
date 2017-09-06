# Ecosystem

## Related Packages

In an effort to begin building out useful functionality without
bloating the two core projects, we've been working hard on building
out additional extensions and integrations.

### [`bobtail-json-cell`](github.com/bobtail-dev/bobtail-json-cell)

The `bobtail-json-cell` takes an alternative approach to declaring
reactive objects. Rather than using special classes like `SrcMap`
and `DepArray`, the `Obs`, `Dep` and `SrcJsonCell`s provided by this package
use ES2015's 
[Proxy feature](https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Global_Objects/Proxy)
to fire change events and subscribe dependencies. This proxy is applied
recursively to all levels of the provided object, allowing it to be 
referenced like a regular variable as well as subscription to deeply 
nested sub-fields, without subscribing to changes on the entire object.
Furthermore, it uses [Benjamín Eidelman's](https://github.com/benjamine) 
[`jsondiffpatch` package](https://github.com/benjamine/jsondiffpatch)
package for its change event notation to fire only minimal refreshes.
ß∂
### [`bobtail-form`](github.com/bobtail-dev/bobtail-form)

The `bobtail-form` package combines `bobtail-json-cell` with 
[Mario Izquierdo's](https://github.com/marioizquierdo) 
[`serialize-json`](https://github.com/marioizquierdo/jquery.serializeJSON) package
and [Rafael Weinstein's](https://github.com/rafaelw) 
[`mutation-summary`](https://github.com/rafaelw/mutation-summary) package
(built on the `MutationObserver` API) to give Bobtail two way data binding.
We do this by creating an `ObsJsonCell` whose value
is essentially bound to the form's serialized value. Whenever the form's serialization
changes, either by value or other state changes on its inputs, the serialization cell's value
updates. The form itself is specified with a function that takes the serialization cell
as an input, thus allowing the form's HTML representation to depend upon its currently
serialized value.

### [`firetail`](github.com/bobtail-dev/firetail)

Firetail is an extension of `bobtail-json-cell` that integrates with Google's 
[Firebase](https://firebase.google.com/) database layer. It supports declaring 
reactive JSON objects by Firebase database reference, and includes
support for cells (which fully refresh on the `value` event), lists 
(which partially refresh on the `child_added`, `child_changed`, and 
`child_removed` events), and 
[synchronized arrays](https://firebase.googleblog.com/2014/05/handling-synchronized-arrays-with-real.html).

In addition, the list and cell objects come in both read-only and 
writable versions (the synchronized array currently only supports
read-only). Both versions update locally whenever the reference change, 
while the writable version also pushes any local changes back to the
Firebase database.

## Projects using Bobtail

### [`soykaf`](github.com/rmehlinger/soykaf)

[Soykaf](https://soykaf.rmehlinger.com) is a character building application 
for _Shadowrun 5e_ that leverages `bobtail-json-cell`, `bobtail-form`, and
Firetail to allow users to build characters and track their progression.

## Ideas for further work

### RxJS integration

While Bobtail focuses on cell-based reactive programming, [RxJS](https://github.com/ReactiveX/rxjs) is focused
on working with streams. An integration would allow Bobtail cells and templates to react to RxJS `Observables` 
representing data streams.

### d3 integration
d3's chaining syntax does not gel well with Bobtail's declarative style templating. Some kind of integration
library between d3 and Bobtail seems like it could be helpful.

### server side rendering 

Developers are increasingly using server side rendering to speed page loads and improve performance. Having examples,
documentation, and possibly a helper package for using Bobtail with server side rendering would be very convenient.