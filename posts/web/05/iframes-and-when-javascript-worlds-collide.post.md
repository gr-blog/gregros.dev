---
title: iframes and when JavaScript worlds collide
description: ""
published: "2025-01-11"
updated: "2025-07-15"
---
%%%?
Every iframe gets a complete copy of the JS web environment, down to every prototype chain.

Let’s see what horrifying things can happen when we interact with it!
%%
# What gets copied exactly?
When the [[but-what-is-an-iframe.post|iframe]] attaches to its parent page, it gets a fresh copy of the JS web environment. This copy doesn’t include any modifications made by the parent page.

That includes:

- The global `window` object
- The `HTMLElement` constructor
- The `Array` constructor
- Every function
- Every prototype chain

## Can’t it just share?
Not really. Pages change their environments all the time, and these modifications can be incompatible with each other.

For example, I prefer to set up my JS environment like this:

```js
// reduce memory footprint:
delete window.Object
// improved developer experience:
JSON.parse = eval
// so I don't miss anything:
console.error = alert
// for debugging:
console.log = document.write
```

Keeping the environments separate is the only way to make sure the pages stay consistent.
## Isn't it expensive?
Yup!

It’s one of the reasons using iframes is generally a bad idea.

That said, there are some things that only iframes can do, so they'll probably never go away.
# Making an iframe
Before can interact with an iframe, we need to create one and attach it to the page.

While we can use a separate webpage, it’s faster and easier to create a [[every-way-to-make-a-synthetic-iframe.post|synthetic iframe]] using JavaScript and populate it using the `srcdoc` property.

We’ll encapsulate all that in a function, and also have that function insert the iframe into the page, since otherwise things won’t work properly.

```js
function makeIframe(contents) {
    var iframe = document.createElement("iframe")
    iframe.srcdoc = contents
    document.body.appendChild(iframe)
    return iframe
}
```

# Accessing the JavaScript environment
It’s pretty easy to access an iframe’s JS environment, as long as it’s not isolated by security features.

We can do that using the iframe’s `contentWindow` property, which exposes *iframe*’s global `window` object. Let’s use it to run a few quick checks:

```js
// create the iframe with no contents:
var iframe = makeIframe("")

// get its window:
var i_win = iframe.contentWindow

// and run some checks:
console.assert(
    // It's a window
    i_win === i_win.globalThis
)
console.assert(
    // but not *our* window
    i_win !== window
) 
console.assert(
    // It has a different Array
    i_win.Array !== Array
)
console.assert(
    // And a different `setTimeout` function
    i_win.setTimeout !== setTimeout
)
```

Great! Now let’s mess around with everything and see what happens.

We’ll start by creating an array using the iframe’s `Array` constructor.

```js
var i_array = new i_win.Array([1, 2])
console.assert(
    i_array[0] === 1
)
console.assert(
    i_array.length === 2
)
```

The result seems to work like a normal array, but don’t be fooled. Any check involving the array’s prototype reveals the alien array’s true nature:

```js
console.assert(
    !(i_array instanceof Array)
)
console.assert(
    !(i_array instanceof Object)
)
```

Weird, right?

The worst part is that these objects are really hard to tell apart from normal ones when debugging. Leaving a bunch of them lying around is sure to cause all sorts of horrible bugs.

But let’s ignore that for now and focus on messing around some more!

For example, what about defining a function inside the iframe, and calling it from outside the iframe? Will it use the caller’s environment or do something else?

Let’s find out!
# Functions from other worlds
We’ll run the experiment in two different ways and see if the results line up.

- We’ll create an iframe that just has a script tag with a function.
- We’ll insert another function into the iframe from the outside.

Both functions will return an array literal, and we’ll check to see which function returned which version of `Array`!

```js
var iframe = makeIframe(`
<script>
    function getArray1() {
        return [1, 2, 3]
    }
</script>
`)
var i_win = iframe.contentWindow

i_win.getArray2 = function() {
    return [1, 2, 3]
}

var x_array1 = i_win.getArray1()
var x_array2 = i_win.getArray2()

console.log(
    "version 1:",
    array1 instanceof Array
)
console.log(
    "version 2:",
    array2 instanceof Array
)
```

What do you think?

- Maybe both will return `Array` because they inherit the caller’s environment.
- Maybe both will return `i_win.Array` because they inherit the environment they’re bound to.
- Or they could return different Arrays for some reason!

You can just run the code in your console to find out! (don’t forget to define `makeIframe` from earlier).
## The result
It turns out that:

- `getArray1` returns `i_win.Array`.
- `getArray2` returns `Array`.

That seems confusing, until you consider the critical difference between the two functions: where we wrote their code.

It turns out that when a script loaded by a webpage, it’s **permanently bound** to that webpage’s environment. Any functions defined by that script are part of it, and so use the same environment.

When we defined `getArray1`, we created a new script inside the iframe, but `getArray2` was actually created in the parent page. The fact we assigned it to the iframe afterwards didn’t change its origin.

This makes sense, but it also means that far from worrying *just* about alien objects, we should be more concerned about alien functions!

If we put a function defined in one environment into another, it'll keep producing alien objects, and it might break if we pass it any parameters of our own.

Scary!
# Tags from other worlds!
The function example is pretty damn weird, but it’s just scratching the surface when it comes to weird iframe behavior.

An iframe has its own copy of the DOM prototype chains, and every element within it is an instance of one of those prototypes. We can create these alien elements using the iframe’s `createElement` function.

But what happens if we insert one of them into the DOM of the parent page?

```js
var iframe = makeIframe("")
var i_win = iframe.contentWindow

var i_doc = i_win.document
var i_div = i_doc.createElement("div")

i_div.id = "find-me"
document.body.appendChild(i_div)
```

This one is a bit tricky! Here are some possibilities:

- It might throw an exception because doing this makes no sense.
- Maybe it won’t do anything.
- Possibly, it'll switch out the element’s prototype before inserting it.
- It could clone the element, attach the correct prototype, and then insert the copy.

## What actually happens
What ends up happening, though, is that Chrome inserts the element as-is.

We get an alien element in the DOM, and it’s just sitting there. We can even look it up!

```js
var i_div_after = document.querySelector("#find-me")

console.assert(
    i_div === i_div_after
)
console.assert(
    !(i_div instanceof Element)
)
console.assert(
    i_div instanceof i_win.Element
)
```

This really took me for a spin, because it *feels* like something the browser shouldn’t allow.

I’d expect the element to be broken or non-functional, the document to be in an invalid state, or… something like that.

But no, the element is totally fine. We could check the document’s `innerHTML`, and find that everything is normal. Invoke `setAttribute` and see its attributes change.

We could even insert child nodes into the element’s subtree, with the correct prototype this time.

```js
i_div.setAttribute("data-blah", "xyz")
i_div.appendChild(
    document.createElement("div")
)

console.log(i_div.outerHTML)
```

It would all work just fine. If we didn’t know any better, we wouldn’t even know anything is wrong.

So what’s going on?
## It’s not allowed to care
The W3C specification, which describes everything about the DOM as we know it, very rarely uses the term *prototype*, but it does define the *DOM Interfaces*.

JavaScript represents them as the constructors we all know and love – `Node`, `Element`, `HTMLElement`, and so forth.

But as we learned back in my [[but-what-is-a-dom-node.post|post about DOM nodes]], DOM nodes and JavaScript objects aren’t the same thing. The rendering engine manages DOM nodes, and it follows a different set of rules.

Specifically, the W3C’s set of rules. And according to the W3C, there's just one set of *DOM interfaces* – no copies.

That's why you should be able to create a DOM node in one *browsing context* and stick it in another *browsing context*. At least, as long as none of them are isolated by security features.

In fact, copying prototype chains is actually something web browsers do *by convention*, not according to any sort of spec.

When it comes down to DOM operations, they have to give way to what the spec says and pretend they didn’t do it. Which just results in yet more weirdness!
# Conclusion
When two JavaScript environments interact, the result can get pretty weird and confusing.

Browsers happily let you pollute your JS environment and even the DOM itself with alien objects that aren’t part of any prototype chain. And you won’t find out until everything breaks a few weeks later.

More than the performance impact, the horrifying bugs that result from working with iframes are probably the best reason to stay away from them.
