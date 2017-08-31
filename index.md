---
title: Introduction
---

# Introduction
<img style="display:inline; float: right; height: 256px; width: 256px;" src="https://raw.githubusercontent.com/bobtail-dev/bobtail-dev.github.io/master/mascot/logo.png">
Bobtail is a simple and lightweight Javascript reactive programming framework, combined with declarative DOM 
construction. There's no magic and no new templating language; it's all Javascript.

Unlike many reactive programming frameworks, rather than starting with event streams, the core abstraction of Bobtail 
is the observable cell, which other cells can subscribe to through using the `bind` function. 

```
const x = rx.cell(3);
const y = rx.cell(4);
const hyp = rx.bind(() => (x.get()**2 + y.get()**2)**0.5);  
console.info(hyp.get())  # 5
x.set(5);
y.set(12);
console.info(hyp.get())  # 13
```

This is combined with a small templating DSL to allow you to quickly and easily stitch together user interfaces which
react to changes in their dependencies.

```
const $a = tags.input.number({value: 3, min: 1, style: {display: 'inline-block'}});
const $b = tags.input.number({value: 4, min: 1, style: {display: 'inline-block'}});
const a = bind(() => $a.rx('val').get());
const b = bind(() => $b.rx('val').get());
const a2 = bind(() => a.get());
const b2 = bind(() => b.get());
const hyp2 = bind(() => a2.get() + b2.get());
const hyp = bind(() => hyp2.get() ** 0.5);

$('#container').append(tags.div([
    tags.div(["a: ", $a, " b: ", $b]),
    tags.div([
        "√(",
        tags.span({id: 'a2'}, bind(() => a2.get())),
        " + ",
        tags.span({id: 'b2'}, bind(() => b2.get())),
        " = √",
        tags.span({id: 'hyp2'}, bind(() => hyp2.get())),
        " = ",
        tags.span({id: 'hyp'}, bind(() => hyp.get())),    
    ])
]));
```

In this example, the contents of `#a2`, `#b2`, `#hyp2`, and `#hyp` will only update if their respective cell updates.

In addition, support for observable Arrays, Maps, and Sets is provided out of the box, as well as a number of utility
functions giving finer grained control of dependency subscription and updating.

## Installation
To install bobtail, simply run `npm i --save bobtail`! 
Or, if you only want the reactive primitives without the `rxt` templating functions, `npm i --save bobtail-rx`.

## License
Bobtail is released under the [MIT License](https://opensource.org/licenses/MIT). Note that some of the packages in our [ecosystem](ecosystem.md) have been
released under the [Apache 2.0 license](http://www.apache.org/licenses/LICENSE-2.0).

## Learn more
You can find our [API Documentation here](api.md) and [tutorial](tutorial.md)!

# Project History

Bobtail was originally developed by [Yang Zhang](yang.github.io) under the name [Reactive Coffee](yang.github.io/reactive-coffee). 
While working with Yang at [Infer](www.infer.com), Richard Mehlinger gradually took over the role of primary 
maintainer. The framework was originally intended primarily for use with CoffeeScript, to take advantage of its terser
syntax for the templating DSL and bind specifications.

We rebranded away from Reactive Coffee for two reasons. First, the framework has nothing at all to do with 
[React.js](https://facebook.github.io/react/). Second, it's not limited to CoffeeScript, and with the 
release of ES2015, which adopted many of CoffeeScript's best features, it made sense to refocus the framework 
toward vanilla JavaScript. The rebranding became official with the 2.0.0 release, which saw the codebase 
converted to JavaScript and the core reactive primitives split into a separate repository.

So why Bobtail? Easy: Richard is a cat person. He saw this 
[video about Kurilian bobtails](https://www.animalplanet.com/tv-shows/cats-101/videos/kurilian-bobtail) and was 
overwhelmed by cuteness.

Bobtail was developed under the auspices of [Infer, Inc.](www.infer.com) Continued development on Bobtail is with the
support of [Dropbox](https://blogs.dropbox.com/dropbox/).

# Contributors

*   [Travis Athougies](https://github.com/tathougies)
*   Cassie Doll
*   [Elik Eizenberg](https://github.com/eizenberg)
*   [Lorefnon](https://github.com/lorefnon)
*   [m1sta](https://github.com/m1sta)
*   [Mateus Maso](https://github.com/mateusmaso)
*   [Richard Mehlinger](https://github.com/rmehlinger)
*   [David Moshal](https://github.com/dmoshal)
*   [Chris Poirier](https://github.com/cpoirier)
*   [Dean Radcliffe](https://github.com/chicagogrooves)
*   [Chung Wu](https://github.com/chungwu)
*   [Artem Yavorsky](https://github.com/aqson)
*   [Yang Zhang](https://github.com/yang)

Last, but certainly not least, is the amazing Adèle Boulie, who designed our new mascot, Charlie the Bobtail.

----
Bobtail Ⓒ 2017 bobtail authors.
Charlie the Bobtail Ⓒ Adèle Boulie 2017. Used by permission.