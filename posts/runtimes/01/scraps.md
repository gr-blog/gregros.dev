# Actually solving performance problems
The aggressive optimizations JavaScript engines use make our code run faster, but they also make pinning down performance issues really hard.

You shouldn’t view a bit of JavaScript code as being fast or slow. Instead, you need to look at the performance of the entire application. Avoid benchmarking, since it typically gives misleading results.

Similarly, profiling doesn’t always tell you where the problem is. A function can be slow because its source code is doing a lot of work, but it can also be slow because its runtime representation (which you can’t easily see) is inefficient. 

Both might be true as well. If a function does a lot of computation, having an inefficient representation can make it a hundred times slower than it could be.

In that case, the issue lies elsewhere, in how the function is being called or how its callees behave. You can only find this out using engine-specific debugging tools, hidden commands, and deoptimization logs.

It might not be possible to modify the call sites to keep the function monormormphic, since they can be 
## Practical solutions
If a function does a lot of computation, it’s a good idea to create a hidden implementation function and a public wrapper that just normalizes the inputs.

Since JavaScript engines optimize functions separately, the wrapper can be megamorphic, but if it doesn’t do any work that doesn’t really matter. 

For example, let’s say you want to define your own sort function for some reason. You’d like it to accept any `Iterable<T>`, but doing so would make it megamorphic. 

You can instead split it into two, like this:

(BAD EXAMPLE)

```js
function _sort(arr) {
	// your favorite sorting algorithm
}

export function sort(iterable) {
	const arr = Array.from(iterable)
	_sort(arr)
	return arr
}
```


For example, let’s say your organization has a package hierarchy, with a lot of packages depending on `@org/utils`. You find out that some perofrmance
A utility function defined by your organization might be called from a dozen different packages and each package does can do it differently. Are you going



In practice, it may not be possible to fix deoptimization problems because of their pervasiveness. 
When profiling, the fact a specific function is slow doesn’t mean the problem actually lies in that function. The function might be deoptimized because of how it’s being called somewhere else.


When you profile something, you won’t typically get information about the optimization state of a function, and 

## Functions are mutable
From the above process, it’s easy to see that the implementation of a JavaScript function **is inherently mutable,** even though the source code never changes.



Every time the function is invoked or invokes something else, its internal implementation can change, sometimes unpredictably.


Their state changes depending on how they are invoked and how they behave, something that can change unpredictably as the code runs.

This means that a function doesn’t have stable performance characteristics and nor does JavaScript code. Its performance can change based on non-local factors.


## A specific example

Let’s take a look at the following function:

```ts
function magnitudeSquared(point) {
    return point.x * point.x + point.y * point.y
}
```

After enough calls, V8 might have learned that:

* `point` always has the same shape, `{x, y}`.
* `point` always has the prototype `Object.prototype`.
* `point`’s invisible runtime representation is always the same (that includes offsets, whether it’s in fast mode, and so on).
* `x` and `y` are always numbers.

It uses this info generate the following specialized C code:

```c
typedef struct {
    double x;
    double y;
} Point;

double magnitudeSquared(Point point) {
    return point.x * point.x + point.y * point.y;
}

// This is just an illustration.
// V8 generates assembly with a lot of guard-rails and other stuff.
```

But then!

Maybe someone passes in `{x, y, label: "my_point"}`. Even if `label` is unused by the function, the runtime representation is different. **Deoptimized!**

Someone could pass `{x: 1n, y: 10n}`, using bigints instead of regular numbers. **Deoptimized!**

Someone makes the terrible blunder of passing in `{y, x}` instead of `{x, y}`. According to JavaScript, those are the same thing. But according to V8, they aren’t. **Deoptimized!**

Use `new Point(x, y)` instead of `{x, y}`. That’s a totally different prototype chain. **Deoptimized** again!

This one bit of code can go through a dozen different transformations depending on seemingly unrelated things, and each version will perform differently.
# How big is the difference?

If you’re looking for a single number, a consistent factor that measures the difference between optimized code and not, you’re going to be disappointed.

V8’s own measurements give us a [good example](https://v8.dev/blog/maglev). The pure-JavaScript JetStream benchmark showed optimizations improving performance by **a factor of 4.3**.

Meanwhile, the Speedometer benchmark measures a web app’s responsiveness. It uses real-world code and a set of popular frameworks and libraries. There, the **factor was only around 1.5.**

But we can see the pattern: pure JavaScript running traditional benchmarks benefits heavily from these optimizations. Messy, real world code benefits a lot less.

# What’s being benchmarked here?

I’ve also seen benchmarks testing the performance of `JSON` against analogs in other languages, such as `System.Text.Json` in C#.

In JavaScript, computationally expensive parts of the standard library are provided by the runtime and implemented in native code.

That includes:

* Everything in `Math`
* `JSON.parse` and `stringify`
* Regular expressions

Benchmarks these things is a valid and informative comparison. It’s just not representative of JavaScript code as a whole.
# Conclusion
JavaScript benchmarks don’t even measure the performance of a specific bit of code, because that bit of code can be compiled in lots of different ways. Each version comes with its own performance characteristics.
