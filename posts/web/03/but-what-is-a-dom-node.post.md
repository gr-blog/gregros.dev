---
title: What is a DOM node? A peek under the hood
description: ""
published: "2025-01-01"
updated: "2025-07-15"
---
%%%?
What makes an object a DOM node? Is it the prototype or something else?

The answer turns out to be surprisingly complicated!
%%

To investigate what the browser sees as a DOM node, we'll find a function that accepts one, pass it different inputs, and see what it does.

The classic example is `appendChild`. This method accepts a DOM node and inserts it as the child of another node. If you pass the method just a regular old object, it errors instead.

Here is some code to illustrate this:

```js
// Create an element
var div = document.createElement("div")

// Insert it into the page
document.body.appendChild(div)
// Works!

// Let's try to insert a regular object instead
document.body.appendChild({})
// Uncaught TypeError: Failed to execute 'appendChild' on 'Node': 
//     parameter 1 is not of type 'Node'.
```

# Mad web science
Now let’s perform a series of bizarre experiments that subvert this code in strange and unusual ways, in the name of mad web science!
## Messing up a DOM node
In this variation, we create the element as normal, but we then mess it up by removing its prototype and deleting all its keys.

This should result in an object that’s functionally indistinguishable from `{}`, something that should be totally non-functional.

Here is the code:

```js
// Create an element
var div = document.createElement("div")

// Remove its prototype
Object.setPrototypeOf(div, null)

// Delete all of its keys
for (const key of Reflect.ownKeys(div)) {
    delete div[key]
}

// Insert it into the page
document.body.appendChild(div)
```

## Trying to fake one
Now, here is the second variation:

```js
// Create an object with the HTMLDivElement prototype
var div = Object.create(HTMLDivElement.prototype)

// Insert it into the page
document.body.appendChild(div)
```

In this variation, we use the `Object.create` function to make a new JavaScript object with the `HTMLDivElement` prototype.

It’s the opposite of what we did in the last variation. We’re making something that looks like a functional JavaScript object, but we’re not using the correct API to do so.

## The question
So… which variation actually works?

- Does the first one work, in spite of the object being totally empty?
- Does the second one work, in spite of how we created it?
- Do neither of them work, because an object needs to have both the correct prototype and be created in the right way for it to count?

Feel free to try to run the code in your browser console and check for yourself!
## The answer
It turns out that the first object – the empty one – **is** recognized as a DOM node, but the second one isn’t. That means the browser doesn’t use an object’s prototype to recognize DOM nodes at all. It’s doing something else.

That’s not to say getting rid of the prototype doesn’t do anything. You can no longer call instance methods, for example, since they're defined on the prototype and that prototype is missing.

But no matter how you screw up a DOM node, if you get a reference to one of those methods, you can still call it and it'll work just fine. Here is an example:

```js
// Create a div elemenmt
var div = document.createElement("div")

// Unset its prototype
Object.setPrototypeOf(div, null)

// Insert it into the DOM
document.body.appendChild(div)

// Get the `setAttribute` function
const { setAttribute } = HTMLElement.prototype

// Invoke it using `call`:
setAttribute.call(div, "id", "this-actually-works")
```

Weird, right? Don’t worry, it's going to make more sense once we zoom out a bit.
# Beyond JavaScript
But by *a bit*, I actually mean a lot. Because to truly understand this weirdness, we have to leave the realm of JavaScript altogether and take a look at browser architecture instead.

Browsers are complicated things with many separate systems that interact in lots of different ways. In particular, they all include two critical yet separate components:

- The JavaScript engine, which executes JavaScript.
- The rendering engine, which renders the HTML document.

In the Chrome browser, these are called V8 and Blink, respectively. These two separate systems are connected by the JavaScript Web API. This takes the form of a thin layer of bindings embedded in V8 that translate JavaScript function calls to native method calls on Blink objects.

These bindings do very little. Once we call a DOM operation, JavaScript is mostly out of the picture and everything resolves in native code.

```canva size=380x320 ;; key=chromium-architecture ;; alt=Browser architecture diagram
https://www.canva.com/design/DAGby_a3rDM/N8CbBV9kIAUZeGTcAXWw4A/view?utm_content=DAGby_a3rDM&utm_campaign=designshare&utm_medium=link2&utm_source=uniquelinks&utlId=hbc0c0bb1df
```

The rendering engine doesn't follow the rules of JavaScript and generally *tries* not to know what JavaScript even is. It does know what a DOM Node is though. In fact, **one of the rendering engine’s primary jobs is to allocate and manage DOM nodes.**

These DOM nodes don’t have anything to do with prototype chains or JavaScript. They're native C++ objects called `Node` that are passed by reference. They literally implement methods called `appendChild` and `insertBefore`.

The V8-Blink bindings form the link between the two. There, each JavaScript DOM node is mapped to a `Node` object, and this mapping just works by reference.

When we call `appendChild`, each JavaScript DOM node gets resolved to its native counterpart, and then Blink executes the operation. This means, in turn, that **JavaScript DOM nodes are just handles to Blink DOM nodes.**

That's why removing the prototype of a DOM node didn’t break it – *it was never functional to begin with*. The only thing that matters is the mapping. Blink created it as soon as we called `createElement`.  The JavaScript properties of the object were always irrelevant.

Next, Blink allocates DOM nodes, it’s impossible to create a DOM node within JavaScript – which is what we tried to do using `Object.create`. That’s kind of like trying to use a random number as the handle to a file.

The OS knows what files we opened, since it’s responsible for opening them; we’re not fooling anyone.
# Conclusion
JavaScript objects aren't actually DOM nodes at all. DOM nodes are native objects managed by the rendering engine, and JavaScript objects are just handles to those objects, kind of like pointers.

The browser gives out these *handles* and puts them on a list. To check if an object is a handle to a DOM node, all it needs to do is check if it’s on that list. The state of the object, like its prototype, is irrelevant.

And that’s it.
