# Tutorial

<img src="https://raw.githubusercontent.com/bobtail-dev/bobtail-dev.github.io/master/mascot/chasing.png" class="header-image"/>

## Getting Started

Start by cloning the [Bobtail Project Seed](https://github.com/bobtail-dev/bobtail-project-seed):

```git clone git@github.com:bobtail-dev/bobtail-project-seed.git```

Then, install dependencies and start the server:
```
npm i
npm run start
```

Now navigate to [localhost:5000](localhost:5000). You should see something like this:

<img src="https://raw.githubusercontent.com/bobtail-dev/bobtail-dev.github.io/master/images/sample.png">

This seed project comes preconfigured with Browserify and live reload. Any changes you make to 
`src/main.js` should cause your browser to reload with your latest version.

All right, let's dive in!

## Cells

Bobtail's core building block is an _observable cell_, the `ObsCell`.

```
let x = rx.cell();
```

A cell is just a container for a value (initialized to `undefined` above, but you could also have passed in an initial 
value). You can `get` and `set` this value:

```
x = rx.cell(0)
x.set(3)
x.get() # 3
```

Cells can hold anything you like, including arrays, objects, functions, and even other cells!

## Events

The special thing about observables is that you can _subscribe_ to events on them, where events are fired when the value 
of the cell changes in some way. For simple cells like the above, there's just a single `onSet` event type. 

```
let subscriptionId = x.onSet.sub(([oldVal, newVal]) => console.log(`x was set from ${oldVal} to ${newVal}`));
```

While you can directly subscribe to events, you are typically better off using the `rx.autoSub` utility function, which 
will automatically unsubscribe the event listener as part of [garbage collection](advanced.md):

```
let subscriptionId = rx.autoSub(x.onSet, ([oldVal, newVal]) => console.log(`x was set from ${oldVal} to ${newVal}`));
```

The listener is just a simple callback. All event types take callbacks of a single argument—the type of that argument 
is event-specific, and in the case of `onSet` it's a pair of `[old value, new value]`. The `sub` method and `autoSub` 
utility method return a unique identifier for this subscription, which can later be used to manually unsubscribe it:

```
x.onSet.unsub(subscriptionId)
```
Once subscribed, you can start reacting to its events. For instance:
```
let firstName = rx.cell('John');
// This ensures .name will always reflect the firstName

rx.autoSub(firstName.onSet, ([oldVal, newVal]) => $('.name').text(newVal));
firstName.set('Jane');
```

Events immediately invoke a newly added listener to bring them up to speed on the current value, but you can suppress 
this first invocation with the simple rx.skipFirst utility function:

```
rx.autoSub(x.onSet, rx.skipFirst(([oldVal, newVal]) => ...));
```

## Dependent Cells

The above is a simple way of observing and responding to events on cells, but it's a bit verbose. In most UIs, you 
really just want to be able to declaratively express your views in terms of a model (data binding), rather than 
explicitly manage listeners.

To extend the above example, let's say you now had a displayed name that depended on two cells (comprising your 
"model"). You could just create explicit subscriptions and listeners:

```
const firstName = rx.cell('John');
const lastName = rx.cell('Smith');

const updateName = () => $('#name').text(`${firstName.get()} ${lastName.get()}`);
rx.autoSub(firstName.onSet, updateName);
rx.autoSub(lastName.onSet, updateName);

firstName.set('Jane');
lastName.set('Doe');
```

This, however, remains frustratingly imperative. Again, we want to avoid listener wrangling as much as possible, 
because that way lies spaghetti-code hell. The problem with this approach is that it's impossible to reason about
what the contents of `#name` are at any given point. Any number of other listeners, from Bobtail or not, might be 
mutating `#name`.

Instead, Bobtail relies on the `bind` function to compose cells, which lets you simply write an expression or function 
in terms of the cells it depends on.

```
const fullName = rx.bind(() => `${firstName.get()} ${lastName.get()}`);
```

Now, `fullName`'s value is always bound to `firstName` and `lastName`. Key here is that no explicit subscription 
management is necessary. This scales well to more complex dependencies, and is more readable and declarative:

```
const displayName = rx.bind(() => {
  if (showRealName.get()) {
    return `Full name: ${fullName.get()}`;
  } else {
    return `Fake name: ${fakeName.get()}`;
  }
});
```

## Subscription, Side Effects, Mutation, and Infinite Loops

This subscription is not magic. Under the hood, the function passed to `bind` is called in a context with a recorder 
object, which subscribes dependencies introduced by calls to `.get` on `ObsCell`s referenced within that function. 
Calling `foo.get()` within a `bind` context introduces a dependency on `foo`'s `onSet` event in the `DepCell` object 
returned by the `bind` call. Any time `foo.onSet` is called--i.e., whenever `foo`'s value changes, that function will be 
re-evaluated, and dependencies will be re-recorded from scratch.

If you want to peek at a cell's value without subscribing a dependency, you can call the `raw` method instead:

```
let fullName = rx.bind(() => {
  console.info(`current alias: fakeName.raw()`);
  return `${firstName.get()} ${lastName.get()}`;
});
```

Or, you can use `rx.snap(f)`. This will call `f`, while preventing the recorder from subscribing any dependencies.
`this.raw())` is simply shorthand for `rx.snap(() => this.get())`.

This will log the value of `fakeName` whenever `fullName` is re-evaluated, without however subscribing `fullName` to
`fakeName`. Had we instead called `fakeName.get()`, whenever `fakeName` changed, `fullName` would be re-evaluated, 
despite the fact that `fakeName` has no bearing on `fullName`'s computed value.

This leads us to a very important point: reading a cell's value in Bobtail is **not** side effect free. Recording the
dependency is inherently a side effect. In particular, this means it is possible to trigger an infinite loop by calling
`get`. The most common cause of an infinite loop is when we mutate a cell to which we have already subscribed from 
within the same bind context where we subscribed to it. For instance:

```
let count = rx.cell(0);
let fullName = rx.bind(() => {
  count.set(count.get() + 1);
  return `${firstName.get()} ${lastName.get()}`;
});
```

This seemingly innocuous bit of code will trigger an infinite loop. When we call `count.get()` inside of the `bind`, we
subscribe a dependency on `count`, meaning `fullName` will refresh whenever `count` changes. We then increment `count`.
This will force `fullName` to refresh, subscribing a dependency on `count`, which it then increments, forcing `fullName`
to refresh... 

Calling `count.raw()` instead of `count.get()` will prevent the infinite loop. However, in general it is safer to simply
never call mutator functions from within `bind` contexts.

To help guard against this, Bobtail will emit a warning if you call a mutator such as `set` from within a `bind` 
context.

## Transactions

Sometimes you want to make a more complex update on some cells, while preserving certain semantics/invariants over them. 
For instance, consider two cells representing the balances in two different bank accounts. When transferring funds from 
one to the other, there will transiently be a moment when the total funds across both will be either greater or less 
than what it really should be:

```
const a = rx.cell(5);
const b = rx.cell(3);
const sum = rx.bind(() => a.get() + b.get());
// sum should always be 8, but when transferring 1 from a to b...
a.set(a.get() - 1);
// sum is now 7!
b.set(b.get() + 1);
// sum back to 8
```

To prevent these "glitches" from being visible, we can bundle up the operations into a transaction. Throughout the 
duration of this transaction, events will not be propagated. For instance:

```
rx.transaction(() => {
  a.set(a.get() - 1);
  // sum will not yet have received any events
  return b.set(b.get() + 1);
});
// now sum will have received notifications, but it will see that the `a`
// and `b` still sum to 8.
```

## Promises and Asynchronicity

Synchronous JavaScript does not get you very far nowadays. Everyone wants data and resources loaded asynchronously,
for the obvious reason that blocking your render loop while you wait for a 50k request to finish is _bad_. Bobtail
supports asynchronous requests with the `promiseBind` helper method. `promiseBind` takes three arguments, `init`, 
`promiseFn`, and, optionally, `catchFn`, and returns a `DepCell`. 

`init` is the initial value of the `DepCell`, while `promiseFn` is a 0-ary function that returns a `Promise`. `catchFn` is a 
0- or 1-ary function. If the Promise is rejected, `catchFn` is called with the `reason` why it the `Promise`
was rejected; the cell is updated with its return value. The default `catchFn` function simply sets the cell's
value to `null`.  

Here's an example using jQuery's [`$.get`](https://api.jquery.com/get/):

```
let catType = rx.cell('bobcats');
let catName = rx.cell('charlie');
let appearance = rx.promiseBind(
  {}, 
  () => $.get('/' + catType.get() + '/' + catName.get() + '/appearance').then(({size, color}) => {size, color}),
  (reason) => {error: reason}
);
```

### Handling Errors

What happens if the URL we are trying to get is invalid? Well, there are two options. The first is
that we can catch the 404 error returned by the server, using `catchFn`. The problem with this, of course, is that
we have to wait for the server to return. If, however, we know ahead of time that there's not enough information for the 
request, we can resolve or reject the Promise ourselves, depending on whether we think this is a problem:

```
let appearance = rx.promiseBind(
  {},
  () => {
    if(catType.get() && catName.get()) {
      return $.get(`/${catType.get()}/#{catName.get()}/appearance`);
    }
    return Promise.resolve({color: '', size: ''});
  } 
  (reason) => {error: reason}  // save this for unexpected errors.
);
```

### Post-processing

Once the Promise has resolved, you are likely going to want to do some sort of post-processing on the result.
Perhaps you need to stitch it with other data, or format it for display. You can postprocess rejections
similarly

If, however, you want that data to be formatted _reactively_, you should create another cell and bind it to
the processed value of your `promiseBind`:

```
let cat = rx.bind(() => ({
  breed: catType.get(),
  name: catName.get(),
  color: appearance.get().color,
  size: appearance.get().size
}))
```

## Arrays, Maps, and Sets

Cells are the most general type of observable container. However, there are also observable containers with special 
support for JavaScript's Array, Map, and Set objects. These objects are represented by the `ObsArray`, `ObsSet`, and
`ObsMap` classes, each of which comes with `Src` and `Dep` versions analagous to `SrcCell` and `DepCell`.

This specialized support is for more efficient and fine-grained 
event types reflecting changes to particular sub-parts rather than an all-encompassing `onSet` event whenever any part 
of the array or map changes.

For instance, arrays commonly have elements inserted into or removed from them, in which case we'd like to avoid 
re-rendering entire dependent sections of the DOM or otherwise needing to figure out what parts have changed (the 
relationship between these reactive data structures and DOM UI rendering will be explained in later sections).

### Arrays

Arrays support an onChange event. It fires with a triple `[index, removed, added]`, where `index` is the offset into the 
array where the change is happening, `removed` is the sub-array of elements removed, and `added` is the sub-array of 
elements inserted. Example:

```
const xs = rx.array([1,2,3]);
xs.onChange.sub(function([index, removed, added]) {
  return console.log(`replaced ${removed.length} element(s) at offset ${index} with ${added.length} new element(s)`);
});
xs.push(4);
// replaced 0 element(s) at offset 3 with 1 new element(s)
```

From a declarative point of view, you can define a _dependent array_ as a mapped transformation of a _source array_:

```
const ys = xs.map(x => 2 * x);
// ys.all() is now [2, 4, 6, 8]
xs.push(5);
// ys.all() is now [2, 4, 6, 8, 10]
```

All `ObsArray`s come with methods analogous to the functional programming methods JavaScript provides on 
[Arrays](https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Global_Objects/Array):
`map`, `filter`, `reduce`, `some` and so forth. In addition, `SrcArray`s come with analagous versions of all of the 
mutator functions defined on the base JavaScript `Array` object, including `push`, as shown above.

You can bind to individual indices of an `ObsArray` using the `at` method, and its length using the `length` method:

```
let queue = rx.array([]);
let head = bind(() => queue.at(0));
let tail = bind(() => queue.at(queue.length() - 1);
```

### Maps

Maps were introduced in ES2015, and are distinct from Objects in that they are explicitly intended to be mutable stores
of key-value pairs. Maps offer a number of utility methods along these lines, without risk of interference from e.g.
prototype chains and the like. Furthermore, unlike with Objects, a Map's keys can be _anything_.

```
const months = rx.map();
months.put('Mar', 3);
months.put('Aug', 8);
months.put('Oct', 10);
months.get('Mar'); // 3
months.remove('Oct'); // 10
months.all(); // {"Mar": 3, "Aug": 8}
```

With maps, there are three different event types:

```
const months = rx.map({"Mar": 3, "Aug": 8});
months.onAdd.sub(function(...args) {
  const [key, val] = Array.from(args[0]);
  return console.log(`set ${key} to ${val}`);
});
// set Mar to 3
// set Aug to 8
months.onChange.sub(function(...args) {
  const [key, oldVal, newVal] = Array.from(args[0]);
  return console.log(`updated ${key} (which had mapped to ${oldVal}) to ${newVal}`);
});
months.onRemove.sub(function(...args) {
  const [key, val] = Array.from(args[0]);
  return console.log(`removed ${key} which had mapped to ${val}`);
});

months.set('May', 5);
// set May to 5
months.set('Mar', 'three');
// updated Mar (which had mapped to 3) to three
months.remove('Oct');
// removed Aug which had mapped to 8
```

The key to maps is that you can bind to changes to just certain keys. For instance, if you had:

```
const mar = rx.bind(() => months.get('Mar'));
const dec = rx.bind(() => months.get('Dec'));
```

Then `mar` will be re-evaluated only when the "Mar" key is changed/removed, and `dec` will only when the "Dec" key is 
inserted. One can also check if a key is present in the Map using the `has` method. This will only refresh if the key
is added or removed, not if its value changes:

```
const hasMar = rx.bind(() => months.has('Mar'));
```

### Sets

Introduced along with Maps in ES2015, 
[Sets](https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Global_Objects/Set) are a kind of cross 
between a Map and an Array. JavaScript Sets are ordered, but each element in a set can appear only once, and it takes 
O(1) lookup time to check if an element is in a Set.

`ObsSets` come with only a single event, `OnChange`. As with `ObsMap`s, one can check to see if an element is present in
the `ObsSet` using the `has` method. There is no `get` method.

### Casting
To cast from one reactive type to another, every `Obs` object comes with `toCell`, `toArray`, `toMap`, and `toSet` 
methods, which return a `Dep` object of the corresponding type bound to this observable's value. The exception is
when trying to cast an observable to itself, in which case, for legacy reasons, the method simply returns the original
observable. If you want a `Dep` version of an observable (useful for exposing read-only interfaces), you can use the
`readOnly` method.

Meanwhile, `rx.cell.from`, `rx.array.from`, `rx.map.from`, and `rx.set.from` allow you to construct `Dep` observables 
from both regular and reactive objects.
```
let arr1 = rx.array.from([42]);  // DepArray [42]
let arr2 = rx.array.from(arr1);  // DepArray [42]
let arr3 = rx.array.from(bind(() => [42]));  // DepArray [42]
```

Finally, the `rxt.cast` method allows you to freely cast any primitive or observable to an observable:

```
let map1 = rx.cast({a: 1, b: 2}, 'map');
```

### `rx.flatten`

The `rx.flatten` utility method is an extremely useful tool for working with nested data structures. It recursively 
flattens both primitive and reactive data structures, removing `null`/`undefined` values, and returns a `DepArray`. 
For example:

```
xs = rx.array(['b','c']);
ys = rx.array(['E','F']);
i = rx.cell('i');
let zset = rx.set(['F', 'L', [], 'U', 'E', [new Set(['FLUE!'])]]);
let flattened = rx.flatten([
  rx.cell([
    'A',
    xs.map(x => x.toUpperCase()),
    'D'
  ]),
  ys.map(y => y),
  ['G','H'],
  bind(() => i.get().toUpperCase()),
  zset.all()
]);
console.info(flattened.raw());
// ['A','B','C','D','E','F','G','H','I','F','L','U','E','FLUE!']
```

This is obviously an extreme case, however it's not uncommon to need to deal with, for instance, an `ObsArray` of 
`ObsArray`s. Currently, `ObsMaps` are not supported by `flatten`, because it's not clear what they should flatten
_to_.

## `ObsBase` common methods

`ObsCell`, `ObsArray`, `ObsSet`, and `ObsMap` all extend the `ObsBase` class. You can too, if you want to implement your
own custom reactive objects. However, you should never directly instantiate an `ObsBase`. In a more object oriented 
framework, it would be an abstract class. `ObsBase` comes with a number of useful methods, the most notable of which are
`.all`, `.raw`, `.readonly`, and `.to[Cell/Array/Set/Map]`. 
* `all` returns the current value of the observable, subscribing
to any events it might emit. This is useful, but note the inherent inefficiency:
* `raw` returns the current value of the observable _without_ subscribing.
* `readonly` is an "abstract" method, in that every subclass of `ObsBase` implements it. It returns a `Dep` version of
the current object. Thus, calling `readonly` on a `SrcArray` would return a `DepArray`. Calling it on a `DepCell` would
return another `DepCell`.
* `.to[Cell/Array/Set/Map]` casts the current object to a different type of observable. For instance:
```
let foo = rx.cell([1,2,3]);
let bar = foo.toArray();
foo.set([1,2,3,4]);
console.info(bar.raw()); // [1,2,3,4]
```

If you attempt to cast an object to its own type, the method will simply return the source object itself.

## Static Templates

Now for a brief jump to something entirely different from reactive programming...

One of the most unique and powerful aspects of JavaScript is the expressivity of the DOM and CSS together. The browser's
model has become so powerful that even desktop software is now being written as HTML/CSS/JavaScript applications, using
frameworks such as Electron.

Many frameworks have historically supported interacting with the DOM through pseudo-HTML templating
languages. This goes back to the 90s, with ASP.Net and PHP, and has been handed down through Rails
to Angular, Ember, and others. React, on the other hand, uses JavaScript to handle its templating,
with the concept of a component being key. Like React, we express our templates using JavaScript.

```
let tags = rx.rxt.tags;
let $sidebar = tags.div({class: 'sidebar'}, [
  tags.h2('Send a message'),
  tags.form({action: '/msg', method: 'POST'}, [
    tags.input({type: 'text', name: 'comment', placeholder: 'Your message'}),
    tags.select({name: 'recipient'}, [
      tags.option({value: '0'}, 'John'),
      tags.option({value: '1'}, 'Jane')
    ]),
    tags.button({class: 'submit-btn'}, 'Send')
  ])
]);
```

That translates to the following HTML:

```
<div class="sidebar">
  <h2>Send a message</h2>
  <form action='/msg' method='POST'>
    <input type='text' name='comment' placeholder='Your message'/>
    <select name='recipient'>
      <option value='0'>John</option>
      <option value='1'>Jane</option>
    </select>
    <button class='submit-btn'>Send</button>
  </form>
</div>
```

Because this is just JavaScript, templates can be easily broken apart and composed using functions,
loops, and conditionals.

```
// Loops:

tags.ul({class: 'friends'}, names.map(name => tags.li({class: 'friend'}, name)));

// Conditionals

tags.div({class: 'profile'}, [
  tags.img({src: `${user.picUrl}`}),
  signedIn ? tags.button(`Add ${user.name} as a friend.`) : tags.p(`Sign up to connect ${user.name}!`)
]);
```

Even recursive templates--always problematic for HTMLish templating langauges--become easy:
```
function Tree(tree) {
  return tags.ul(tree.map(subtree => tags.li(
    if(Array.isArray(subtree)){
      return functionRecList(subtree);
    }
    else {
	  return subtree;
	}  
  )));
}
```

Bobtail's tag functions have a flexible argument structure: `tag(attrs, children)`. 
Both the `attrs` and `children` arguments are optional; one can create an empty div with `R.div()`.
Basically, if the first argument is a non-Array Object, then it's used as the tag's attrs.
Otherwise, it's interpreted as the tag's child or children.

Tags are really just functions that return DOM elements (wrapped in jQuery objects), so you are
free to attach behaviors to them:

```
const $button = tags.button({class: 'submit-btn'}, 'Click Me!');
$button.click(function() { return $(this).text('I been clicked.'); });
```

### JSX Support

Recently, with the rise in popularity of React, JSX become a popular method for writing frontend templates. Rather than
extending HTML, JSX opts to extend JavaScript itself to provide templating functionality. Like HTML-ish templating
languages, JSX also requires a separate compilation step, in which the tag syntax in .jsx files are
[converted into function calls](https://reactjs.org/docs/jsx-in-depth.html)--by default, React's `createElement`
function.

As of version 2.3.0, Bobtail includes a `createElement` function in its `rxt` namespace, which behaves superficially
similarly to React's. Lowercase tags are instantiated using Bobtail's `createTag` function. If a tag is uppercase,
it will be instantiated using a function or class in the current scope. If it's a class, it needs to have a `render`
method defined, as in React.

The following example uses JSX with a function, a class, and basic tags:

```
function Counter(opts={}) {
    let count = rx.cell(opts.count || 0);
    return (
        <div>
            {count}
            <button click={() => count.set(count.raw() + 1)}>+</button>
            <button click={() => count.set(count.raw() - 1)}>-</button>
        </div>
    );
}

class ShapedContainer {
    constructor(opts={}, contents=[]) {
        this.color = rx.cell(opts.color || 'red');
        this.shape = rx.cell(opts.shape || 'square');
    }
    render() {
        return <div class={bind(() => this.color.get() + " " + this.shape.get()}>{contents}</div>
    }
}

function GreenCircleCounter() {
    return (
        <ShapedContainer color="green" shape="circle">
            <Counter />
        </ShapedContainer>
    );
}
```

To use JSX with react, you need to use a transpiler, like Babel, and set the pragma field to the `rxt.createElement`
function--however you've imported it. The following is a sample .babelrc, assuming you import Bobtail into the `rx`
namespace:

```
{
  "presets": ["es2015"],
  "plugins": [
    ["transform-react-jsx", {
      "pragma": "rx.rxt.createElement"
    }]
  ], ...
}
```

## Reactive Templates

<img src="https://raw.githubusercontent.com/bobtail-dev/bobtail-dev.github.io/master/mascot/sneak.png" class="body-image"/>

Of course, if we were only writing static templates, we would be better off using vanilla HTML. 

Now, you _could_ just write explicit imperative code to transform the DOM in a way that
consistently reflects the bindings you're interested in. For instance:

```
$('body').append(tags.input({
  class: 'name passive',
  type: 'text',
  placeholder: 'Name',
  value: ''
}));
displayName.onSet.sub(function(...args) {
  const [oldVal, newVal] = Array.from(args[0]);
  return $('.name').val(newVal);
});
isActive.onSet.sub(function(...args) {
  const [oldVal, newVal] = Array.from(args[0]);
  if (newVal) {
    return $('.name').removeClass('passive').addClass('active');
  } else {
    return $('.name').removeClass('active').addClass('passive');
  }
});
```

However, more complex transformations can become more involved.

```
names = rx.array(['1','2','3','4','5'])
$('body').append($nameList = tags.div {class: 'name-list'})
spans = names.map (name) -> tags.span name
spans.onChange.sub ([index, added, removed]) ->
  # Homework: fill in logic here for efficiently inserting/removing DOM
  # nodes!
```

This is inherently clumsy. You shouldn't have to repeatedly code this logic any time you want to make bindings, and 
using jQuery selectors to mutate the DOM breaks componentization; that is, it's easy to end up modifying something other
than what you intended to. It would be both clearer and safer to specify the template deceptively:

```
$('body').append(
  tags.input({
    class: rx.bind(() => `name ${isActive.get() ? 'active' : 'passive'}`),
    type: 'text',
    placeholder: 'Name',
    value: rx.bind(() => displayName.get())
  })
);

$('body').append(
  tags.div({class: 'name-list'}, names.map(name => span(name)))
);
```

And this is where Bobtail shines. The primitives on their own are clever, but not hugely useful.
The templating language on its own is a toy. Together, however, they offer an incredibly expressive
approach to building user interfaces.

Bobtail lets you declare what the UI should _always look like at any given time_. It thus frees you
from the responsibility of maintaining and transitioning state. Meanwhile, the `promiseBind`
function allows us to write UIs without worrying about the synchronicity or asynchronicity of our
data model.

Here's another quick example, this one of a todo list. 

```
const tasks = rx.cell(['Get milk', 'Take out trash', 'Clean up room']);
$('body').append(
  tags.ul({class: 'tasks'}, rx.bind(() =>
    tasks.get().map(task => tags.li({class: 'task'}, `User: ${task}`))
  ))
);
```

Notice how here we are using a raw array in a cell. This is fine to start with, but is inefficient when dealing with 
large amounts of data. Any time anything in the array changes, the entire `<ul>` will need to be re-rendered. We can
instead map over a `SrcArray`. When the `SrcArray` changes, only elements that have been inserted, deleted or modified
are re-rendered:

```
const tasks = rx.array(['Get milk', 'Take out trash', 'Clean up room']);
$('body').append(
  tags.ul({class: 'tasks'}, bind(() =>
    tasks.map(task => tags.li({class: 'task'}, `User: ${task}`))
  ))
);
```

Tag attributes can also bind to cells:

```
input {value: bind -> displayName.get()}
```

Contents can be a cell that returns an array (of strings or elements):

```
tags.span({}, rx.bind(() => [displayName.get()]));
tags.div({}, rx.bind(() => names.all()));

// shorthand:
tags.span(rx.bind(() => displayName.get()));
tags.div(rx.bind(() => names.all()));
```

or an observable array:

```
let names = rx.array(['1','2','3','4','5']);

tags.div({}, names);
// results in: <div>01234</div>

// simplified:
tags.div(names);

tags.div({}, names.map(name => tags.span({}, [name])));
// results in: <div><span>0</span><span>1</span>...</div>

// simplified:
tags.div(names.map(name => tags.span(name))
);
```

### Nested `bind`s

One of the chief advantages of Bobtail is that you have very fine-grained control over the re-rendering process. 
Dependencies in nested `bind`/`map` calls are insulated from outer calls, so if something changes in an inner bind,
we only re-render what's necessary. For instance, say we had a model like the following:

```
class User {
  constructor(id, name) {
    this.id = rx.cell(id);
    this.name = rx.cell(name);
  }
}

class App {
  constructor(users) {
    this.users = rx.array(users);
  }
}

const app = new App([
  new User('John', 0),
  new User('Jane', 1)
]);
```

If we wanted to just re-render individual elements, we could do that with:

```
tags.select(app.users.map(user => 
  tags.option({value: rx.bind(() => `${user.id.get()}`)}, bind(() => `${user.name.get()}`))
));
```

By using `map`, we ensure that if we add or remove users, we only change rerender the added or removed elements. 
Meanwhile, by binding the `<option>`s' value and contents, we ensure that we only change the contents or attributes
of the `<option>` if its `name` or `id` have changed, respectively, and that these changes will not force the entire
`<select>` to re-render.

On the other hand, if we wanted to re-render the entire section whenever anything changed, we could do so with:

```
tags.select(rx.bind(() => app.users.all().map(user => option({value: `${user.id.get()}`}, `${user.name.get()}`)))
);
```

Remember that if you only care about a cell's value at the current moment, you can use `snap` or `raw` to prevent it
from being subscribed as a dependency:

```
let x = rx.cell('hello');
let y = rx.cell(0);
div({class: 'editor'}, bind(() => [
  input({type: 'text', value: snap(() => x.get())})
  input({type: 'number', value: snap(() => y.raw())})
]));
```

Here, no dependencies will be subscribed; the call to `bind` is in fact superfluous. Without the `snap`, anytime `x` 
changes, the entire `<div>` would be re-rendered, but with a `bind`, the text box's contents would be re-rendered. 
Similarly, calling `y.raw()` prevents dependencies from subscribing; it is equivalent to `rx.snap(() => y.get())`.

### auto-flattening

Sometimes you will want to append an observable array after some element. Or you may want to make one element among a 
set conditionally rendered. Previously, you would have needed to use `rx.flatten` to achieve this:

```
tags.div(rx.flatten([
  tags.h1('Header'),
  headerItems.map(item => render(item)),
  rx.bind(function() { if (showSeparator.get()) { return separator; } }),
  tags.h1('Footer'),
  footerItems.map(item => render(item))
]));
```

Since version 2.0.0, however, we automatically flatten tag contents, so explicitly calling `rx.flatten` is unnecessary.
You can just pass in a primitive Array, and it will work fine.

```
tags.div([
  tags.h1('Header'),
  headerItems.map(item => render(item)),
  rx.bind(function() { if (showSeparator.get()) { return separator; } }),
  tags.h1('Footer'),
  footerItems.map(item => render(item))
]);
```

### More on Arrays and Reactive Templates

Arrays also support an `indexed` method which returns a copy of the array, but which actually tracks the indices of its 
elements, such that when you map the array, the mapper function is given the index (as a reactive cell) in addition to 
the current element. This way you can react to shifts in the position of each element.

For instance, if you had a list where you wanted the rows to be styled based on whether they're prime:

```
const xs = rx.array(['foo', 'bar', 'baz', 'zzyzx', 'quux']);
tags.ul(xs.indexed().map((x,i) => 
  tags.li({class: bind(function() { if (isPrime(i.get())) { return 'prime'; })}, x)
));
// shifts the entire array, but still only prime rows have class "prime"
xs.removeAt(0);
```

This is not built into `map` since it would make maps more expensive, and in general this is a less-frequently used 
feature.

### Inputs

We can treat `<input>`, `<select>`, and `<textarea>` DOM elements' attributes as cells, thus binding to them as well.

Consider a type-ahead search component, where the completion list only displays items that prefix-match the given query. 
Furthermore, we want the completion list to only show up if it's enabled by the checkbox and when we actually have 
focused on the input text field:

We are using three different types of reactive element attributes. `rx(property)` returns the value of the given 
property (one of `focused`, `checked`, or `val`) as a `SrcCell`. This is a form of simple two-way data-binding: mutating 
the returned cell will cause the linked attribute to update in the DOM, and changing that attribute on the DOM will 
cause the cell to update.

## Special Attributes

Since often times you may be working with deeply nested templates structures where it's clumsy to tack on 
behaviors afterward, you can for convenience supply a function in an attribute named `init`, which is immediately 
invoked with the current element bound to `this`:

```
tags.table({}, properties.map(function(prop) {
  return tags.tr([
    tags.td(prop.name),
    tags.td([
      tags.input({
        type: 'text',
        value: prop.value,
        placeholder: 'Enter property value',
        init() { return this.blur(() => setProperty(prop, this.val())); }
      })
    ])
  ]);
}));
```

`init` is an example of a special attribute. These are just attributes that are intercepted and handled differently 
from others, which are simply passed on as regular HTML attributes on the DOM element. Besides init, built-in special 
attributes include:

* `style`, which in addition to regular style strings accepts jQuery-style mappings like `{fontSize: 10}`, 
so you can write:

```
tags.div({
  style: rx.bind(() => ({
    width: score.get(),
    color: team.get() === 'a' ? 'red' : 'blue',
    display: gameMode.get() !== 'running' ? 'none' : undefined
  }))
});
```

* `class`, which in addition to regular class strings accepts an array of class names (filtering out `null` and 
`undefined`), e.g.:

```
tags.button({
  class: rx.bind(() => [
    'btn',
    formComplete.get() ? 'btn-primary' : undefined
  ])
});
```

You can also access this transform directly via `rxt.smushClasses`.

* and all jQuery events: `click`, `focus`, `mouseleave`, `keyup`, `load`, etc. So we could have bound the blur event in 
the earlier example directly (note it also takes a jQuery object as this):

```
tags.input({
  type: 'text',
  value: prop.value,
  placeholder: 'Enter property value',
  blur() {
    $row.css('opacity', .5);
    return setProperty(prop, this.val());
  }
});
```

## Next Steps

Congratulations! You now have a working knowledge of Bobtail. Interested in learning more? Check
out the [advanced topics](http://bobtailjs.io/advanced.html) and our
[nascent ecosystem](http://bobtailjs.io/ecosystem.html). Want to contribute? Take a look at our
[development guidelines](http://bobtailjs.io/development.html)!