---
title: Advanced Concepts
---

## Side Effects and Resource Cleanup

Sometimes you will have side effects associated with certain `bind`s or components, such as establishing a connection 
or setting an interval timer:

```
blinkingBox = opts => {
  let active = rx.cell(false);
  setInterval(() => active.set not active.get(), opts.ms ? 500);
  return tags.div({class: bind(() => if active.get() then 'red-box' else 'black-box')});
}

$('body').append(
  div({class: 'container'}, boxes.map(() => blinkingBox()))
)
```
In these cases, you'll want to do proper resource management/cleanup whenever the `bind` context is removed or 
refreshed—in this case, stopping the interval timer. For these situations, you can use `rx.onDispose(callback)`:

```
blinkingBox = opts => {
  let active = rx.cell(false);
  setInterval(() => active.set not active.get(), opts.ms ? 500);
  rx.onDispose -> clearInterval(interval)
  return tags.div({class: bind(() => if active.get() then 'red-box' else 'black-box')});
```
## Garbage Collection

Dependencies hold references to their dependents in order to know who to propagate updates to. We do currently 
unsubscribe a dependent observable from its dependencies whenever that dependent is refreshed, but if the dependent 
observable itself creates nested binds, those nested binds must also be disconnected, and so on, recursively.

For instance, say we have:

```
let showUnread = rx.cell(true);
let unread = rx.cell(0);
let $unreadDiv = div({class: 'scoreboard'}, bind(() =>
  if(showUnread.get()) 
	return tags.div({class: 'unread'}, bind(()=> `${unread.get()} message(s)`));
  else return '(nothing shown)';
));
```

Initially, `unread` has a subscriber, as expected: the inner `bind`. When we update `showUnread`, the outer `bind` 
will re-evaluate and create a new inner `bind` (for the nested `div`), completely forgetting about the old one. 
Without the nested bind cleanup, `unread` now has _two_ subscribers, old and new inner `bind`s. And since the old 
inner `bind` is still referenced from `unread`'s list of subscribers, it can't be garbage-collected. Ergo, memory leak.

What instead happens: `bind` tracks any nested `bind`s. On re-evaluation, these inner `bind`s are _disconnected_, 
unsubscribing themselves from all their dependencies. This disconnection also occurs recursively down to _their_ nested 
`bind`s, and so on.

One question is what happens at the top level or when you want to break out and do your own thing. We don't have the 
luxury of weak refs in JS, and as a result it's less forgiving if you do "silly" things like create binds that go 
nowhere and that you don't want to keep.

(But even with weak references, one can't tell if you added a bind only to subscribe a console.log caller to its 
changes, save to localStorage, or some other side effect that you _do_ want to keep.)

For instance, you don't want to write the above as the following:

```
let showing = false;
$('.toggle-button').click(() => {
  if(showing) $('.scoreboard').html(div {class: 'unread'}, bind -> "#{unread.get()} message(s)")
  else $('.scoreboard').text('(nothing shown)');
  showing = not showing
});
```
The key for these situations is that `bind`s should go all the way to the top, or else you'll want to manually 
disconnect your (top-level) binds using their `disconnect` method:

```
let showing = false;
let topBind = null;
$('.toggle-button').click(function() {
  if (showing) {
    $('.scoreboard').html(
      div({class: 'unread'}, (topBind = bind(() => `${unread.get()} message(s)`)))
    );
  } else {
    $('.scoreboard').text('(nothing shown)');
    topBind.disconnect();
  }
  return showing = !showing;
});
```

## Mutations Inside Binds

Say you had some dependent cell:

```let plusOne = bind(()=> src.get() + 1);```

If you try something like the following, i.e. explicitly wire up a dependent cell to modify a source cell:

```
bind(() => {
  let x = plusOne.get();
  dst.set(val);
});
src.set(2)
```

you'll get a warning that you're trying to perform a cell-mutating operation within the subgraph of a dependent cell.


This is discouraged because a well-structured program should exhibit a clean DAG of dependencies. When you do 
something like this, you're potentially creating cycles, which can lead to infinite loops. It also muddies what 
`dst` _should_ reflect, when you have multiple bind locations calling `dst.set(...)`—it's often clearer to express 
the value of `dst` as some expression of its dependencies.

Usually you'll instead want to have `dst` be a dependent cell of `plusOne` rather than be its own `SrcCell`:

```
let dst = bind(() => plusOne.get());
```
However, in case you need an escape hatch in a bind (har har), you can always set up a manual listener:

```
plusOne = bind(() => rx.autoSub(src.onSet, rx.skipFirst(([old, val]) => val + 1));
```

We need `skipFirst` since by default the listener is immediately invoked with the current value of `src`.

Note that `dst` will not be considered a dependency of `src` or `plusOne`. This connection is unmanaged (or rather, managed by you).

So if this is happening within some bind context, you must also unsubscribe on context disposal. This is handled by 
`rx.autoSub`, which will automatically unsubscribe the listener on disposal.

Lastly, there may come a point where you still need to trigger a mutation from within a `bind` context, and you 
are _certain_ that doing so is safe. In that case, you can use `rx.hideMutationWarnings` to suppress the warning
message, like so:

```
let a = rx.cell(5);
let loading = rx.cell(false);
let b = bind(() => rx.hideMutationWarnings(() => {
  try {
    loading.set(true);
    return a.get() ** 2;
  }
  finally {
    loading.set(false);
  });
})
```

## Asynchronous Binds

So far we've seen _synchronous_ binds. However, let's say you want to have introduce a delay between when a bind's dependencies are updated and when that bind is evaluated. This can be useful, for instance, if you have an expensive bind over some dependencies, where the dependencies can change in bursts of high frequency, and you don't want the app to block on re-evaluating the bind every time the dependencies change. Similarly, you may want a UI component to update in response to the user entering text into a text box, but you don't want a jarring user experience and want to instead only update the UI once the user stops typing for some time.

There is in fact a `lagBind`:

```
let y = rx.lagBind(500, 0, () => expensiveFunction(x.get()));
```
Here, `y` isn't immediately evaluated every time `x` changes; rather, `y` (which is initialized to 0) waits for
`x` to settle down for at least half a second before evaluating the expensive function of `x`.

There's also a similar function, `postLagBind`, contributed by [@eizenberg](https://github.com/eizenberg), which instead 
introduces a lag but _after_ the function evaluation/re-capture (which is immediate upon any changes to dependencies). 
This allows you to specify a dynamic delay as part of the result, where the delay duration comes from observables. 
In this example, the tootip appears instantly, but disappears only after a delay:

```
const showTooltip = rx.postLagBind(false, function() {
  if (focused.get() && showTooltips.get()) { return {val: true, ms: 0};
  } else { return {val: false, ms: 300}; }
});
```
More generally, you can implement your own asynchronous binds. `bind`, `lagBind`, and `postLagBind` are all expressed 
in terms of `asyncBind`. You supply a function that runs the asynchronous lifecycle. At a single point in the lifecycle,
you can call `this.record(f)`, passing it a function which will record and subscribe to dependencies. This can be 
called only once; otherwise things like disposal of prior subscriptions gets very complex and multiple in-flight 
lifecycles gets more complex (if you need multiple stages of bindings separated by asynchronous gaps, you can use a 
chain of multiple asynchronous binds). Upon completion, `this.done(x)` must be called with the result.

Here's an example that immediately evaluates/records its dependencies and then feeds that to an AJAX call; upon completion, the result is passed into `@done`.

```
y = rx.asyncBind(0, () => {
  let input = this.record(() => x.get());
  ajaxCall({data: input, done: output => @done(output)});
})
```

Users interested in `asyncBind` are also encouraged to take a look at the (very short) 
[implementations](https://github.com/bobtail-dev/bobtail-rx/blob/master/src/main.js) of `bind`, `lagBind`, and 
`postLagBind`.


## Object-Oriented Programming: A Pattern for Public and Private Cells

Often, when using an object-oriented approach, one wants to expose values to the user without permitting them to change 
those values directly. When using reactive, we would also want the user to be able to subscribe listeners to these 
private cells. In addition, we still need to be able to set the value of the cell internally. To achieve this, we can 
make use of `rx.bind`, which returns a `DepCell` object, or the `readonly` method discussed above. These objects do 
not have a set method, and thus cannot be directly mutated by the user:

```
class Incrementer {
  constructor() {
    const _value = rx.cell(0); // private field which can be freely changed within this constructor's scope.
    this.value = bind(() => _value.get()); // public field; its value mirrors that of _value, but cannot be directly changed by the user.
    this.increment = () => _value.set(value.get() + 1);
  }
}

const inc = new Incrementer();
inc.value.onSet.sub(function(...args) { const [oldVal, newVal] = Array.from(args[0]); return console.log(newVal); });
inc.increment(); // prints 1
inc.value.set(42); // error, not allowed.
```

The usefulness of this pattern is not limited to object oriented programming. We can use this any time we wish to 
return a read-only cell:

```
const autoIncrementedCell = function(start) { 
  const value = rx.cell(start);
  setInterval(value.set(value.get() + 1), 1000);
  return value.readonly();
};

const c = autoIncrementedCell(0);
c.onSet.sub(function(...args) { const [oldVal, newVal] = Array.from(args[0]); return console.log(newVal); });
```