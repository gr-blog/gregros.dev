---
title: Benchmarking JavaScript is hard
description: ""
published: 2026-09-02
updated: 2026-09-02
---
%%%?
Benchmarks try to measure the performance of a bit of code in isolation. Using them in JavaScript can have some pretty misleading results.

To see why, let’s take a closer look at how JavaScript engines work.
%%
Benchmarks try to measure the performance of a bit of code by isolating it from everything else.

This works in most languages. But in JavaScript, isolating code can drastically change its performance in ways that are very hard to predict.

To see why, let’s take a look at how JavaScript engines work.
# How JavaScript is executed
To execute JavaScript, the engine first compiles the source code into its own bytecode. This mapping is typically one-to-one, so the engine never needs to interpret the source code directly.
## Figuring things out
As this bytecode executes, the runtime monitors the shapes or types of a function’s arguments, the return values of the functions it calls, and its variables. 

The *shape* of a value (sometimes called *hidden class*) is the runtime representation of a heap-allocated object. 

That includes its structure, like a TypeScript object type, but also includes other things that are runtime-specific, such as:

- Whether a number is a float or int
- Property order
- The prototype chain

Let’s take a look at an example.

```js
function example(input) {
	let a = func(input.x)
	return a
}
```

In this function, the runtime will record:

- The shape of`input`, which could be `{x: int}`.
- The shape returned by `func`, which might be `int`.
## Making assumptions
If these things stay the same for long enough, the runtime will assume them and generate a specialized implementation of the function in machine code with those assumptions baked in.

A function in this state — where all values always have the same type and shape — is called *monomorphic*.

We can illustrate this machine code version using simple C code, though the real implementation is a lot more complicated. 

```c
typedef struct {
    int x;
} Input;

int example(Input *input) 
{
    int a = func(input->x);
    return a;
}
```

Here is a pretty illustration:
```canva size=700x450 ;; key=assumptions-and-js-compilation ;; alt=A diagram showing assumptions leading to compilation
https://www.canva.com/design/DAHXiLuNspw/RLJ3WJdtM1x9GRqdyFs8xw/view
```

The assumptions made by the runtime can be broken at any time. Some code might invoke `example`  differently or `func` might return something else in the future.

If this happens, the function is *deoptimized*. The specialized machine code is scrapped and the runtime returns to executing it as bytecode. Here are some example invocations of that would cause this to happen:

```js
// A different shape, even if it's compatible:
example({x: 1, y: 2})

// Using a float instead of an integer:
example({x: 0.1})

// Using an object with a different prototype chain:
class Thing { constructor(x) {this.x = x}}
example(new Thing(5))
```

Note that having shared structure is not enough. The shape has to match exactly for the optimized version to work.

After being deoptimized, the runtime will typically try to optimize the function again. It returns to keeping track of the function’s behavior and how it’s invoked.

If the only difference is that `func` sometimes returns floating point values, the runtime might use similar machine code, just using a `double` instead of `int` for the return type of `func`.

On the other hand, if the shape of `input` changes, the function will be recompiled as *polymorphic*. This kind of function uses a lookup table to support different object shapes.

Each time the runtime notices a different shape, the compiled version will be modified to support it.
## Giving up
Compiling a function costs CPU time, and every time the function is recompiled, it becomes less specialized and so performs worse, while its code and surrounding data (like lookup tables) become larger.

Eventually, the runtime will decide the cost outweighs the benefit and stops trying. From that point, the function will just execute as bytecode.

This state is called *megamorphic.* Once a function reaches it, there is no going back. It will never be optimized again.
# Implementations are mutable
The process we described in the last section shows that **the runtime implementation of a JavaScript function is mutable.**

Although it always does the same thing, its performance characteristics change wildly depending on its hidden internal state — information the runtime has recorded about the code’s behavior.

This internal state can change when the function is called and based on the behavior of the functions it calls. These are non-local events. 

Here is an illustration of a few stages. In practice, functions might be compiled more or less often than this. Some things can cause the runtime to give up immediately too.

```canva size=700x450 ;; key=javascript-function-states ;; alt=A flow chart showing changes to a JS function's state
https://www.canva.com/design/DAHXieSTKGg/6-hfd9QU6GBY0mqmZGoHTw/view
```

If we benchmarked the function after each change in its implementation, we would get very different results.

In the same way, isolating the function would make some of the events fail to happen. The function might end up being monomorphic on a test bench, even though it ends up being megamorphic in our codebase.

In other words, a function’s performance is tied to the code that surrounds it.

This is a big factor of why JavaScript appears fast in benchmarks but performs a lot worse than other languages in practice.
## Built-ins are built different
A built-in function doesn’t always act the same way as ones you define in your own code.

- It could be written in C++ and manipulate JavaScript objects through their C++ interface.
- It could have a compiled representation that works with lots of different input types and shapes.
- It might be replaced with inline machine code.
- All of the above could be true and apply in different situations.

Built-ins would be inherently megamorphic otherwise. For example, functions defined on `Object.prototype` receive totally different inputs through their `this` parameter, but that’s just how they are supposed to work.

A lot of work has also been done on functions like `Function.prototype.call` and `bind` for similar reasons.
## How deep does the shape go?
Unlike TypeScript object types, the runtime shape of an object isn’t recursive.

In other words, an argument value of `{x: {a: 1}}` will just have the shape:

```ts
{x: Shape}
```

While an argument value of `{x: {a: 1}, y: {b: 2}}` will have the shape:

```ts
{x: Shape, y: Shape}
```

This is because `Shape` is represented as runtime pointer, which has the same structure no matter what it’s pointing to. Polymorphism only happens if code inside the function tries to access `x` and finds a different shape inside.


# Conclusion
JavaScript engines use lots of aggressive performance optimizations that make our code run faster, but they make measuring that performance kind of tricky.

They mean that you can’t really view a piece of JavaScript code as being fast or slow. Instead, you need to look at the entire codebase that interacts with it.

It’s not all guessword, though. There are ways to write high performance JavaScript code. We’ll get into those methods in another article.

