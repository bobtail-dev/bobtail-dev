# Wishlist

There are a number of areas for improvement in Bobtail, including but not
limited to the following:

## `rx` improvements

### standardized utility functions for asynchronous and streaming subscribtions

Asynchronous programming in JavaScript is the norm. All but the simplest applications require asynchronous 
communication with backend services of one sort or another. There are some 
[draft ideas](https://github.com/bobtail-dev/bobtail-util/blob/master/src/main.coffee) for an implementation of 
an `AjaxDepCell` type (as well as a partially-reloadable bind, allowing one to refresh a part of a query), but 
nothing set in stone. Settling on a clean, simple API for working with Promises is probably our highest priority.

And while we are not going to try to reimplement RxJS or Bacon.js, an API for subscribing to event streams
seems highly desirable.

### transaction squashing

Currently, `rx.transaction` only delays events until the transaction
finishes. It still fires every event that was registered in the
transaction. It would instead be preferable if we could compress
multiple invocations of the same event. Consider:

```
x = rx.cell(0);
rx.transaction(() => {
    x.set(1);
    x.set(2);
    x.set(3);
});
```

At the end of the transaction, `x.onChange` will fire three times, with arguments
`[0, 1]`, `[1, 2]`, and `[2, 3]`. For many if not most applications, we'd prefer for
these changes to be compressed into a single change event, with argument `[0, 3]`.

The basic implementation would be fairly straightforward: Events would be rewritten
to be called with variadic arguments. Each event would know how to com

While it is easy to envision how this could be accomplished for `ObsCell`s, there 
are wrinkles with the other data types. In particular, `ObsMap`'s three events
are not amenable to squashing. 

In addition, we need to ensure that, once we've squashed the events, that we fire
them in topographical order, so that we do not end up refreshing cells more than 
once. The algorithm we're currently using to propagate refreshes calculates a
topologically sorted DAG from a single source node, but a transaction inherently
propagates events from multiple source nodes, and the topo sort algorithm we are
currently using will not work in this case.

Finally, there is the question of whether we should change the default `transaction`
behavior, or introduce another function, such as `digest`, that will squash events. 

### `ObsMap` events
Unlike the other data types, `ObsMap`'s currently fire up to three events on a 
mutation: `onAdd`, `onRemove`, and `onChange`. This seemed like a good idea at 
the time. However, it's inconsistent with the other data structures, makes working
with `ObsMap`s unnecessarily complicated, and is standing in the way of implementing
transaction squashing (above). 

### `ObsArray`

#### `map` and`indexed`

Currently, the `ObsArray.map` callback signature only takes a single argument,
whereas `Array.map`'s callback takes three: `currentValue`, `index`, and `array`.
To get a version with an indexed map function, it's necessary to construct an
`IndexedArray` with the `.indexed` method. This has led to convolution and code
duplication inside pur codebase, and is counterintuitive. Instead, the basic
`ObsArray.map` method should behave the way `IndexedArray.map` does today.

#### more efficient refreshes

Currently, `ObsArray.map` is the only `ObsArray` method optimized to
minimally refresh. `reduce`, `filter`, et. al. all reference `.all`,
and thus any change to the parent array will force a full recalculation,
going over the entire array. It would be nice if we could instead perform
partial refreshes, for instance by having `.every` check that only
newly added elements satisfy the condition, rather than re-checking
the entire array.

## `rxt` improvements

### implicit binds

One of the main sources of noise in the templating DSL is the need to liberally scatter `bind` calls throughout. One bit
of syntactic sugar that could be used to reduce this noise is to allow subscriptions via function. That is, if a
function is supplied as a child or an attribute, wrap that function in a bind, like so:

```
let x = rx.cell(5);
let cls = rx.cell('red');
let $div1 = tags.div({class: () => cls.get()}, () => x.get());
```

This would be equivalent to:
```
let $div1 = tags.div({class: rx.bind(() => cls.get())}, rx.bind(() => x.get()));
```

... just with a somewhat terser syntax.

There is a slight wrinkle to this, which is that the event fields already expect a function. In theory, one can have
the event handler bound to observables. There is, after all, no reason why the value of an `ObsCell` cannot be a 
function. In practice this is rarely useful--any switching you want to do can be handled
inside the handler, so we could special case those fields to ignore the bind-wrapping.

### spreadable child arguments

Currently tag functions can take at most two arguments. The first argument
is either an attributes object, a single child element, or an array, either
primitive or reactive, of child elements. If the first argument is an 
attributes object, then the second is used to hold the child/children.

This was fine in CoffeeScript, with its terser syntax. However in Javascript,
this leads to a proliferation of parentheses and brackets, like so:

```
tags.ul({style: {listStyle: 'list-unstyled'}}, [
    tags.li('hello'),
    tags.li('world')
]);
```

If however we children could be specified with spread or variadic arguments,
then we could remove the square brackets, and write something like:

```
tags.ul({style: {listStyle: 'list-unstyled'}},
    tags.li('hello'),
    tags.li('world')
);
```

### auto-flatten tag contents

Currently, tag contents cannot be a nested array or nested reactive object. Nor, if the contents are a primitive array,
can any of its elements be an `ObsCell`. You can call `rx.flatten` to get around this, however it would be nice if 
instead we auto-flattened tag contents before proceeding.

### self-referential and mutual dependencies

Consider the case where we want to have the class of an input
dependent on its value.

Or, consider a case where I ran into recently, where I wanted to
enforce that the sum of two numeric inputs could not be greater
than 7.

There is no easy way to do this without resorting to imperative 
programming.

One partial solution would be to allow users to update element
attributes with a reactive bind after they've been created:

```
let $in1 = tags.input.number({min: 1});
let $in2 = tags.input.number({min: 1});
let max1 = bind(() => 7 - $in1.rx('val').get()));
let max2 = bind(() => 7 - $in2.rx('val').get()));
$in1.attr('max', bind(() => max1.get())
$in2.attr('max', bind(() => max2.get())
```

This is imperative in style, however. A declarative approach would be greatly preferable.

### individually bindable style attributes

Currently, it is possible to create a tag where we bind the entire `style` attribute to a cell, but not any of its sub-
attributes. That is: 
```
let pixels = rx.cell(50);
// this is OK
tags.div({style: bind(() => {height: pixels.get()})});
// this is not OK 
tags.div({style: {height: bind(() => pixels.get()}});
```

It would be nice if we could bind individual style attributes like that, and then if any of them were to update, to
only change that style attribute on the tag, rather than calling `$.css`.

### document SVG tags

Currently, Bobtail ships with functions for creating SVG tags in `rxt.svg_tags`, that are analagous to creating the
regular tags in `rxt.tags`. Unfortunately, it's not currently documented, and there's no example code that I know of.
It'd be good to bring this up to full support.

### better `rx('checked')` support for radio buttons

As mentioned, radio button change events only fire when a button is selected, not when it's deselected. Having an easy
way to bind to the currently selected radio button in a group seems like a useful and valuable proposition.

### better support for `select` elements.

There are a number of wrinkles with `select` elements. For one thing, when children are added or removed, the value of
the `select` can change without an event firing. A built-in `select` tag function that handles cases like this would be
very useful.

### transitions
Ability to apply animations and effects to things like entrances, exits, and
reorderings.  See the 
[d3 transitions API](https://github.com/mbostock/d3/wiki/Transitions) for 
possible inspiration.

### reduce jQuery dependency

jQuery is a large library, but we only use it for a few things. It would greatly reduce bobtail's minified size
if we could make the jQuery dependency optional (one would of course then lose the jQuery extension function), or
swappable with a lighter weight framework such as Zepto.

### refresh HTML elements on `requestAnimationFrame`

Currently, if a cell that a Bobtail template depends on changes, we will immediately attempt to recompute and redraw
the relevant DOM subtrees. This, however, is inefficient. The average screen only refreshes 60 times per second, and 
indeed we could find ourselves attempting to redraw multiple times between screen refreshes. Instead of immediately 
refreshing the DOM, we could instead mark the sections that need to be refreshed, and wait redraw those when the screen
next rerenders, using the 
[`window.requestAnimationFrame` API](https://developer.mozilla.org/en-US/docs/Web/API/window/requestAnimationFrame).