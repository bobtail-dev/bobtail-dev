# API Documentation

<img src="https://raw.githubusercontent.com/bobtail-dev/bobtail-dev.github.io/master/mascot/listening.png" class="header-image"/> 

# `rx` Namespace

This contains the core reactive programming primitives. These are the core data structures:

* `ObsBase`: base class for all other observable objects
*   `ObsCell`: observable cell base class
    *   `SrcCell`: a source cell, one that can be directly mutated
    *   `DepCell`: a dependent cell, one whose value is some function of another observable cell
*   `ObsArray`: observable array base class
    *   `SrcArray`
    *   `DepArray`
*   `ObsSet`: observable Set base class
    *   `SrcSet`
    *   `DepSet`
*   `ObsMap`: observable Map base class
    *   `SrcMap`
    *   `DepMap`
*   `Ev`: an event node; serves as a pub-sub node for events of a certain type

## Constructor aliases

*   `cell(value)`: return a `SrcCell` initialized to the given value (optional; defaults to `undefined`)
*  `cell.from(value)`:  casts `value` to a `DepCell`, or if `value` is already an `ObsCell`, return `value`.  
*   `array(value[, diff])`: return a `SrcArray` initialized to the given array (optional; defaults to `[]`) and the 
given `diff` function for computing/propagating the minimal set of change events (defaults to `rx.basicDiff()`).
*  `array.from(value)`: casts `value` to a `DepArray`, or if `value` is already an `ObsArray`, return `value`.  
*   `set(value)`: constructs a `SrcSet` initialized to the given value. If value is not already a Set, will attempt
to cast it to one.
*   `set.from(value)`: casts `value` to a `DepSet`, or if `value` is already an `ObsSet`, return `value`.  
*   `map(value)`: constructs a `SrcMap` initialized to the given value. If value is not already a Map, will attempt
to cast it to one. In particular, to preserve backwards compatibility, `value` here can be a regular `Object`.
*   `map.from(value)`: casts `value` to a `DepMap`, or if `value` is already an `ObsMap`, return `value`.  
*   `cellToArray(cell[, diff])`: given an `ObsCell` of an array, return a `DepArray` that issues the minimal set of 
change events by diffing the contents of the array to determine what has changed. Similar in mechanism to 
`SrcArray.update`. `diff` is an optional diff function (see `SrcArray.update` documentation) and defaults to 
`rx.basicDiff()`.
*   `cellToSet(cell)`: given an `ObsCell` of a primitive JS object, return a `DepSet` that, in response to future new 
values of the object, issues the minimal set of events by diffing the contents of the object to determine what has 
changed.
*   `cellToMap(cell)`: given an `ObsCell` of a primitive JS object, return a `DepMap` that, in response to future new 
values of the object, issues the minimal set of events by diffing the contents of the object to determine what has 
changed.
*   `cast(...)`: two forms:
    *   `rx.cast(opts, types)`: convenience utility for use in preparing arguments for a component. `opts` is the 
    arguments object mapping, and `types` is an object mapping fields of `opts` to either `cell` or `array`. `cast` 
    returns a copy of `opts` but with the fields casted based on the specified desired `types` (yielding observable 
    cells and arrays).
    *   `rx.cast(data, type)`: lift the given datum to the given type; acts as identity function if data is already of
    the proper type.

## Subscription
*   `bind(fn)`: given a 0-ary function, return a `DepCell` whose value is bound to the result of evaluating that 
function. The function is immediately evaluated once, and each time it's evaluated, for any accesses of observables, 
the `DepCell` subscribes to the corresponding events, which may subsequently trigger future re-evaluations.
* `promiseBind(init, fn, [catchFn])`: given a 0-ary function `fn` which returns a `Promise`, returns a `DepCell` whose 
value is initialized to `init`, execute `fn` and record dependencies, and update the value of the cell when the 
`Promise` resolves. If the Promise fails, use a 0- or 1-ary `catchFn` to update the value of the cell based on the
rejection reason. The default `catchFn` is `() => null`--that is, if a `catchFn` is not specified and the Promise
rejects, the cell's value will be reset to `null`.
*   `snap(fn)`: evaluate the given 0-ary function while insulating it from the enclosing bind. This will be evaluated 
_just once_ and prevents subscriptions to any observables contained therein.
*   `transaction(f)`: run the given function in a _transaction_, during which updates to observables will not emit 
change events to their subscribers. Only once the transaction ends will all events be fired (at which time the state 
of the source cells will no longer be in flux, and thus a consistent view of the universe can always be maintained).
*   `asyncBind(init, fn)`: create an asynchronous bind. The given `fn` is expected to call `this.record(f)` up to at 
most one time at some point during its potentially asynchronous duration before calling `this.done(result)`. The call 
to `this.record(f)` evaluates `f` while recording the implicit subscriptions triggered by accesses to observables. 
(You can call `done` from within `record` but the result won't be set on this cell until after `record` finishes.) The 
cell's initial value is `init`. Note that the use of `this` means that `fn` _cannot_ be an arrow function.
*   `lagBind(lag, init, fn)`: same as `bind` but waits a 500ms delay after a dependency changes (or after 
initialization) before the `DepCell` refreshes. For the first 500ms, the cell's value is `init`; you can set this to 
e.g. `snap(fn)` if you want to immediately populate it with the result of evaluating your given `fn`.
*   `postLagBind(init, fn)`: immediately evaluates `fn`, which is expected to return a `{val, ms}`, where `ms` 
indicates how long to wait before publishing `val` as the new value of this `DepCell`. Until the first published value,
the cell is initialized to `init`.
*   `onDispose(callback)`: add a (0-ary) cleanup listener, which gets fired whenever the current (enclosing) bind is 
removed or refreshed. This is useful for proper resource disposal, such as closing a connection or removing an interval
timer.
*   `skipFirst(f)`: wrap a 1-ary function such that the first invocation is simply dropped. This is useful for 
subscribing an event listener but suppressing the initial invocation at subscription time.
*   `autoSub(ev, listener)`: Subscribes a listener to an event but such that if this is called from within a `bind`
context, the listener will be properly cleaned up (unsubscribed) on context disposal. Returns the subscription ID.
*   `subOnce(ev, listener)`: Subscribes a listener to an event, suppresses the initial invocation of the event
at subscription. The listener is unsubscribed after the first time it is called.
*   `hideMutationWarnings(f)`: Run the given function, but suppress console logging of mutation warnings and firing of 
the `onMutationWarnings` event. This is useful when you need to mutate a cell when already within a bind context, but 
you *know* that doing so is safe.

## `Ev`

*   `sub(listener)`: subscribes a listener function for events from this event node and returns a unique ID for this 
subscription, which can be used as the handle by which to unsubscribe later. The listener takes a single argument, 
whose type depends on the event, and is called every time an event is fired on this event node. NB: you are almost
certainly better off using the `rx.autoSub` utility method, which ensures that subscriptions are garbage collected.
*   `pub(data)`: publish an event, described by the data in `data`. `data` is what will be passed to all the subscribed 
listeners.
*   `unsub(subscriptionId)`: detaches a listener from this event.
*   `scoped(listener, contextFn)`: subscribes to the listener, runs `contextFn()`, and unsubs the listener. Returns the
result of `contextFn()`.
*   `observable`: the observable object to which the `Ev` is attached.

## `ObsBase`
`ObsCell`, `ObsArray`, `ObsSet`, and `ObsMap` all extend the `ObsBase` class, which provides a few common methods:
* `flatten()`: returns `rx.flatten(this)` .
* `raw()`: returns the current value of the reactive object without subscribing. Short for `rx.snap(() => this.all())`.
* `all()`: returns current value of the reactive object, and subscribes to any changes to it. Abstract method 
implemented by all subclasses.
* `readonly()`: returns a `Dep` version of the current object. An `ObsCell` returns a `DepCell`, an 
`ObsMap` returns a `DepMap`, and so forth. Abstract method implemented by all subclasses. Subtly different from calling
e.g. `new SrcCell(42).toCell()`, since `readonly` always returns a `Dep` version of the current object, while casting
an object to its own type via `cast`, `.from`, or `to<type>` will simply return the pre-existing object.
* `.toCell()`: returns a `DepCell` whose value is bound to that of this object. 
unless the object is an `ObsCell`, in which case it returns the object itself. Equivalent to `rx.cell.from(this)`.
* `.toArray()`: returns a `DepArray` whose value is bound to that of this object. 
unless the object is an `ObsArray`, in which case it returns the object itself. Equivalent to `rx.array.from(this)`.
* `.toSet()`: returns a `DepSet` whose value is bound to that of this object.
unless the object is an `ObsSet`, in which case it returns the object itself. Equivalent to `rx.set.from(this)`.
* `.toMap()`: returns a `DepMap` whose value is bound to that of this object. 
unless the object is an `ObsMap`, in which case it returns the object itself. Equivalent to `rx.map.from(this)`.

### `Obs` objects

In general, you should not instantiate `Obs` objects. They cannot be subscribed to other objects, nor do they come
with mutator functions. As a result, they are really only useful as base classes.

### `Src` objects
While `Src` objects do not share a common mixin, they do all provide an `update` method, in addition to the built in
`ObsBase` methods and those particular to each implementation. 
`update(x)`: updates the object's value, emits change events, and returns its old value.

### `Dep` objects
With the exception of `DepCell` (see below), `Dep` objects inherit their `Obs` parents, and provide no new methods.
Their constructors all follow the same pattern:

`constructor(f)`: Constructs an observable whose value is bound to `f`. Runs `f`. Use `this.record` to subscribe to 
dependencies, and `this.done` to publish a new value. If any of recorded dependencies change, `f` will be re-run, its
dependencies cleared and re-transcribed, and the value of `this` observable will be updated.

Note that `DepArrays` optionally take a second argument, `diff`, to their constructor. This argument is used to compute
the diffs  

## `ObsCell`

*   `get()`: returns current value of the cell
*   `onSet`: the event that is fired after the value of the cell is changed. The event data is an array of two elements,
`[oldVal, newVal]`.

### `SrcCell`

*   `set(x)`: alias for `update`, included to maintain backwards compatibility and consistency.

### `DepCell`

Note: these methods are for advanced use only, and are key to the internal workings of Bobtail.
*   `disconnect`: unsubscribes this cell from its dependencies and recursively disconnects all nested binds; useful for
manual disposal of `bind`s
*   `refresh`: this method recalculates the value of a cell, as well as its dependencies, and then publishes its
`onSet` event.

## `ObsArray`

`ObsArray` implements most of the non-mutating methods of the native JS `Array` class, in reactive form. 

*   `onChange`: the event that is fired after any part of the array is changed. The event data is an array of three
elements: `[index, added, removed]`, where `index` is the index where the change occurred, `added` is the sub-array of
elements that were added, and `removed` is the sub-array of elements that were removed.

*   `get(i)`: returns element at `i` 
*   `at(i)`: alias for `get`. Included for backwards compatibility.
*   `all()`: returns array copy of all elements
*   `raw()`: returns raw array of all elements; this is unsafe since mutations will violate the encapsulated invariants,
but serves as a performance escape hatch around `all()`
*   `length()`: returns size of the array
*   `transform(f)`: returns a `DepArray` with value bound to `f(this.all())`.
*   `indexed()`: returns a `DepArray` that mirrors this array, but whose `map` method passes in the index as well. 
The mapper function takes `(x,i)` where `x` is the current element and `i` is a cell whose value always reflects the 
index of the current element.

*   `map(fn)`: returns `DepArray` of given function mapped over this array. For legacy reasons, not quite analagous to 
`Array.map`, in that fn is only a 1-ary function, as opposed to the 3-ary `(currentValue, index, array)` signature
of `Array.prototype.map`.
*   `filter(f)`: returns a `DepArray` with value bound `this.all().filter(f)`. analagous to `Array.prototype.filter`.
*   `slice(x, y)`: returns a `DepArray` with value bound `this.all().slice(x, y)`. analagous to `Array.prototype.slice`.
*   `reduce(f)`: returns a `DepArray` with value bound `this.all().reduce(f)`. analagous to `Array.prototype.reduce`.
*   `reduceRight(f)`: returns a `DepArray` with value bound `this.all().reduceRight(f)`. Analagous to 
`Array.prototype.reduceRight`.
*   `every(f)`: returns `this.all().every(f)`.
*   `some(f)`: returns `this.all().some(f)`.
*   `indexOf(val, from)`: returns `this.all().indexOf(val, from)`.
*   `lastIndexOf(val, from)`: returns `this.all().lastIndexOf(val, from)`.
*   `join(separator)`: returns `this.all().join(separator)`.
*   `first()`: returns the first element in the `ObsArray`.
*   `last()`: returns the last element in the `ObsArray`.

### `SrcArray`
`ObsArray` implements most of the mutator methods of the native JS `Array` class, in reactive form. None of the mutator
functions specified here should cause events to subscribe.

*   `SrcArray([xs[, diff]])`: constructor that initializes the content array to `xs` (note: keeps a reference and does 
not create a copy) and the diff function for `update` to `diff` (defaults to `rx.basicDiff()`).
*   `splice(index, count, additions...)`: replace `count` elements starting at `index` with `additions`
*   `insert(x, i)`: insert value `x` at index `i`
*   `remove(x)`: find and remove first occurrence of `x`
*   `removeAt(i)`: remove element at index `i`
*   `push(x)`: append `x` to the end of the `ObsArray`
*   `pop()`: removes and returns the last element of the `ObsArray`
*   `unshift(x)`: prepend `x` to the front of the `ObsArray`
*   `shift()`: removes and returns the first element of the `ObsArray`
*   `reverse()`: reverses the order of elements of the `ObsArray` and returns its new value, *without* subscribing.
*   `put(i, x)`: replace element `i` with value `x`
*   `replace(xs)`: replace entire array with raw array `xs`
*   `update(xs[, diff])`: replace entire array with raw array `xs`, but apply the `diff` algorithm to determine the 
minimal set of changes for which to emit events. This enables flexible updating of the array, representing arbtirary 
transformations, while at the same time minimizing the downstream recomputation necessary (particularly DOM 
manipulations).
*   `move(src, dest)`: Moves the element at `src` to before `dest` as a single mutation.
*   `swap(i1, i2)`: Swaps the elements at i1 and i2 as a single mutation.
*   inherits `ObsArray`

## `ObsSet`
A reactive version of the `Set` object introduced by ES2015.
* `has(key)`: returns if `key` is in the set.
* `keys()`: returns a copy of the underlying `Set`. alias for `.all`, to match the ES2015 `Set` API.
* `values()`: returns a copy of the underlying `Set`. alias for `.all`, to match the ES2015 `Set` API.
* `entries()`: returns a copy of the underlying `Set`. alias for `.all`, to match the ES2015 `Set` API.
* `size()`: returns the number of elements in the `ObsSet`.
* `union(other)`: returns a new `DepSet` whose value is bound to the union of `this` and `other`, which can
be either a primitive or `ObsBase`, and which will be cast to a Set.
* `intersection(other)`: returns a new `DepSet` whose value is bound to the intersection of `this` and `other`, which can
be either a primitive or `ObsBase`, and which will be cast to a Set.
* `difference(other)`: returns a new `DepSet` whose value is bound to the difference of `this` and `other`, which can
be either a primitive or `ObsBase`, and which will be cast to a Set.
* `symmetricDifference(other)`: returns a new `DepSet` whose value is bound to the 
[symmetric difference](https://en.wikipedia.org/wiki/Symmetric_difference) of `this` DepSet and `other`, which can
be either a primitive or `ObsBase`, and which will be cast to a Set.

### `SrcSet`
* `put(item)`: Puts `item` to the Set.
* `add(item)`: Alias for `put`.
* `delete(item)`: Deletes `item` from the Set.
* `remove(item)`: alias for `delete`.
* `clear()`: removes all entries from the Set.

## `ObsMap`
A reactive version of the `Map` object introduced by ES2015.

*   `onAdd`: the event that is fired after an element is added. The event data is a map of added keys to their 
corresponding values.
*   `onRemove`: the event that is fired after an element is removed. The event data is a map of removed keys to their 
original values.
*   `onChange`: the event that is fired after a key's value is changed. The event data is a map of changed keys to a
two-element list of `[oldValue, newValue]`.
*   `get(k)`: returns value associated with key `k`
*   `has(k)`: returns whether key `k` is in the Map
*   `size()`: returns the number of entries in the Map.

### `SrcMap`

*   `put(k, v)`: associates value `v` with key `k` and returns any prior value associated with `k`
*   `set(k, v)`: alias for `put`.
*   `delete(k)`: removes the entry associated at key `k`
*   `remove(k)`: alias for delete.
*   `clear()`: removes all entries from the set.
*   inherits `ObsMap`


## flatten
* `rx.flatten(xs)`: given an array of either `Array`s, `ObsArray`s, `ObsCell`s, `ObsSet`s, primitives, or objects, 
flatten them into one dependent array, which will be able to react to changes in any of the observables. It 
furthermore strips out `undefined`/`null` elements (for convenient conditionals in the array). This operation is a 
"deep flatten"---it will recursively flatten nested arrays. Furthermore, reactive objects which themselves return 
reactive objects are also recursively flattened: `bind(() => [bind(() => [42]).toArray())` will resolve to `42`.
Any `Function`s found in the recursive descent are first bound, and the resulting cell then flattened.

This function exists primarily to support the templating DSL. All non-primitive `contents` passed to an `rxt.tags`
function are run through `rx.flatten` to convert it into something we can use to construct HTML tags. The reasoning 
behind this is that, when writing templates, nested structures can easily develop. We would prefer not to have to wrap
these in semantically meaningless `div` or `span` tags (especially problematic when, say, working with tables). This 
means that, unlike in older versions of Bobtail and Reactive Coffee, you do not have to explicitly call `flatten` to
simplify complex templating code; we do it for you automatically. 

## Arrays
*   `concat(arrays...)`: returns a dependent array that is the result of concatenating multiple `ObsArray`s 
(efficiently maintained).
*   `basicDiff([key])`: returns a diff function that takes two arrays and uses the given key function, which defaults 
to `rx.smartUidify`, to efficiently hash-compare elements between the two arrays. Used by `rx.cellToArray`.
*   `smartUidify(x)`: returns a unique identifier for the given object. If the item is a scalar, then return its JSON 
string. If the item is an object/array/etc., create a unique ID for it, insert it (this is a standard "hashing hack" 
for JS) as a non-enumerable property `__rxUid`, and return it (or just return any already-existing `__rxUid`). I.e., 
this implements reference equality. Used by `rx.basicDiff`.

## Implicitly reactive objects
*   `lift(x[, spec])`: convert an object `x` containing regular non-observable fields to one with observable fields 
instead. `spec` is a mapping from field name to either `'cell'`, `'array'`, or `'map'`, based on what kind of 
observable data structure we want. By default this is supplied by `rx.liftSpec`.
*   `liftSpec(x)`: given an object `x`, return a spec for `rx.lift` where any field that is an array is to be stored in 
an `rx.array`, and otherwise in an `rx.cell`.
*   `reactify(obj, spec)`: mutates an object to replace the specified fields with ES5 getters/setters powered by an observable cell/array, and returns the object. `spec` is a mapping from field name to `{type, val}`, where `type` is `'array'` or `'cell'` and `val` is an optional initial value.
*   `autoReactify(obj)`: mutates an object to `reactify` all fields (specifying `array` for arrays and `cell` for everything else), and returns the object

# `rx.rxt` Namespace

<img src="https://raw.githubusercontent.com/bobtail-dev/bobtail-dev.github.io/master/mascot/stalking.png" class="header-image"/>

This contains the template DSL constructs. The main thing here is the _tag function_, which is what constructs a DOM element.

*   `mktag(tag)`: returns a tag function of the given tag name. The various tags like `div` and `h2` are simply aliases;
e.g., `div = mktag('div')`. 
Tag functions themselves take `(attrs, contents)`, both optional, where `attrs` is a JavaScript object of HTML 
attributes and/or _special attributes_  
and `contents` is an array (`ObsArray` or regular `Array`) of child nodes (either raw elements, `RawHtml`, or jQuery 
objects) and/or strings (for 
text nodes). You can also pass in a singular such node without the array. The function returns an instance of the 
specified element wrapped in a jQuery object. See `specialAttrs` below for more on special attributes. 
* `rxt.tags.<tagName>(opts, children)`: We generate a function using `mktag` for each of the standard HTML tags. See the 
"Supported Tags" section below for the full list of tag functions.
*   `rawHtml(html)`: returns a `RawHtml` wrapper for strings that tags won't escape when rendering; example: 
`div {}, [rawHtml('<span>hello</span>')]`
* `specialChar(code, tag)`: Creates a `tag` wrapping the single special character `&<code>;`
* `unicodeChar(code, tag)`: Creates a `tag` wrapping the single unicode character `\u<code>;`
* `rxt.smushClasses(classes)`: convenience utility for programmatically constructing strings suitable for use as a 
`class` attribute. Takes an array of strings or `undefined`s (useful for conditionally including a class), and filters 
out the `null`/`undefined`.

## jQuery plug-in

*   `rx(property)`: lazily create (or take the cached instance) and return a `SrcCell` whose value is maintained to 
reflect the desired property on the element. If the cell's value is changed, the property on the element will update, 
and vice versa. `property` can be `'checked'`, `'focused'`, or `'val'`. `checked` and `val` update on the `onchanged` 
event, while `focus` (naturally) updates on the `onfocus` and `onblur` events. This means that, by default, calling 
`$elem.val(newValue)` will *not* cause the cell to update its value, because doing so does not fire the `onchanged`
event. You must do so manually if you want the cell to update. Note also that radio buttons do not fire an `onchanged`
event when they are deselected. But see below.

## Input tags
In addition to the normal `tags.input` function, we take advantage of the fact that JavaScript
functions are also objects to add shorthand methods for each input type:

* `rxt.tags.input.<type>(attrs, children)`: Shorthand for `tags.input(_.extend(attrs, {type}), children)`. In addition,
for checkboxes and radio buttons, we patch the jquery `prop` function so that using it to programmatically set checked
state updates the `rx('checked')` cell. For all other inputs, we similarly patch the jquery `val` function to update the 
`rx('val')` cell.

## Special attributes

_Special attributes_ are handled differently from normal. The built-in special attributes are primarily events like 
`click`:

```
button {click: -> console.log('hello')}, 'Hello'
```

Special attributes are really just a convenient short-hand for running the handler functions after constructing object. 
The above example is equivalent to:

```
$button = button 'Hello'
$button.click -> console.log('hello')
```

The built-in special attributes available include:

* `init`: which is a 0-ary function that is run immediately upon instantiation of the element.
* `style`: if the value is an 
* `class`: automatically transforms compacts, flattens, and joins the given value if it's an array, then strips any 
extra whitespace 
* events like `click`, `focus`, `mouseleave`, `keyup`, `load`, etc.,: these are available for all jQuery events. 
The handlers are similar to those you would pass to a jQuery event method?they take a jQuery event and return false to 
stop propagation?but the context is the jQuery-wrapped element instead of the raw element. The full list:

    *   `blur`
    *   `change`
    *   `click`
    *   `dblclick`
    *   `error`
    *   `focus`
    *   `focusin`
    *   `focusout`
    *   `hover`
    *   `keydown`
    *   `keypress`
    *   `keyup`
    *   `load`
    *   `mousedown`
    *   `mouseenter`
    *   `mouseleave`
    *   `mousemove`
    *   `mouseout`
    *   `mouseover`
    *   `mouseup`
    *   `ready`
    *   `resize`
    *   `scroll`
    *   `select`
    *   `submit`
    *   `toggle`
    *   `unload`

`specialAttrs`: an object mapping special attribute names to the functions that handle them. When an element specifies 
a special attribute, this handler function is invoked. The handler function takes `(element, value, attrs, contents)`, 
where:

*   `element` is the element being operated on
*   `value` is the value of this special attribute for `element`
*   `attrs` is the full map of attributes for `element`
*   `contents` is the content/children for `element` (which can be a string, `RawHtml`, array, `ObsCell`, or `ObsArray`)

So for instance, we can create a special attribute `drag` that just forwards to the jQuery `[jdragdrop]` plug-in:

```rxt.specialAttrs.drag = (elt, fn, attrs, contents) -> elt.drag(fn)```

and then use it like so in your templates:

```div {class: 'block', drag: -> $(this).css('top', e.offsetY)}```

### Supported input types
* `color`
* `checkbox`
* `date`
* `datetime`
* `datetimeLocal`
* `email`
* `file`
* `hidden`
* `image`
* `month`
* `number`
* `password`
* `radio`
* `range`
* `reset`
* `search`
* `submit`
* `tel`
* `text`
* `time`
* `url`
* `week`

### Supported tags
```
a, abbr, address, area, article, aside, audio, b, base, bdi, bdo, blockquote, body, br, button, canvas, caption, 
cite, code, col, colgroup, data, datalist, dd, del, details, dfn, div, dl, dt, em, embed, fieldset, figcaption, figure, 
footer, form, h1, h2, h3, h4, h5, h6, head, header, hr, html, i, iframe, img, input, ins, kbd, keygen, label, legend, 
li, link, main, map, mark, math, menu, menuitem, meta, meter, nav, noscript, object, ol, optgroup, option, output, p, 
param, pre, progress, q, rp, rt, ruby, s, samp, script, section, select, slot, small, source, span, strong, style, sub, 
summary, sup, svg, table, tbody, td, template, textarea, tfoot, th, thead, time, title, tr, track, u, ul, var, video, 
wbr
```