---
title: Every language is fast
description: ""
published: "2026-10-10"
updated: "2026-10-10"
---
%%%?  
Rust is obviously fast. C’s performance is second-to-none. Assembly is probably even faster. But I’ve also heard Node.js described as fast too, while others say none of this even matters because I/O.

Can all these claims be true?
%%  
I’ve heard every single mainstream language described as fast over the years. 

Rust is obviously fast. C’s performance is second-to-none. Assembly is probably even faster. But I’ve heard Node.js described as fast too, while others say none of this even matters.

**All of those claims are true,** for a given value of _fast_. 

It depends on the kind of workload we’re looking at, whether we’re comparing typical or aggressively optimized code, whether we’re looking at *latency* or *throughput*, and even language conventions.

Let’s go through a bunch of totally different examples to examine what *fast* really means.
# Latency, throughput, and scale
Suppose we compare the cost of calling a function in Python:

```python
def add(a, b): return a + b

add(1, 2)
```

Against roughly the same thing in C:

```c
// Assume this is not inlined
int add(int a, int b)  
{
	return a + b;
}

int main(void)
{
    return add(1, 2);
}
```

The code looks similar, but the workload that results from executing each is totally different.

Let’s consider the C code first, because the machine code that ends up executing is practically trivial. You could hand-compile it with ease. In fact, let me do so now:

```x86
main:
    mov edi, 1
    mov esi, 2
    call add
    ret

add:
    lea eax, [rdi + rsi]
    ret
```

Python has more to do. It sets up a Python function frame and binds `a` and `b` to Python integer objects. The interpreter loads those objects, performs the addition, and returns the result.

That involves way more instructions than the tiny example we showed above. 

I ran a small benchmark, executing both functions in a loop, while making sure the function call remains intact.

The C function took around **3 nanoseconds**, while CPython 3.12 managed it in around **50 nanoseconds**. That’s around 17 times slower!
## Zooming out
But what if that code was processing an HTTP request?

```http
GET /add?a=1&b=2 HTTP/1.1
Host: example.com
```

Now, `add` is just a step in something much larger. The request has to reach the server, engage IP, TCP, and HTTP stacks.

Network latency alone can take over a hundred milliseconds, depending on distance and the path packets take. Routing and processing within the machine can take a hundred microseconds. 

Then suppose that our endpoint is also reading a file or something. Now we have to wait on storage too. That might take another 5ms.

When you consider that, the latency improvements from using C versus Python seem not to matter at all.
## Zooming sideways
But wait.

Our `add` function is actually sandwiched by the standard library’s HTTP stack and the framework’s routing logic.

Both of these are written in Python. Processing a single request might involve calling a dozen functions, building dictionaries, and parsing strings — all before serializing the response.

Those costs add up.

While request *latency* is still dominated by I/O, the overall CPU time tells us how much processing power our box needs to keep serving requests, concurrently, at a high throughput.

Suppose we measure a particular Python server and find that it uses an extra 10 microseconds of CPU time per request compared with a similar C implementation. 

At 100k requests per second, those 10 microseconds add up to one CPU-second of extra work every second. In other words, an entire core. Those cost money!

These higher CPU requirements also mean we’ll start to run into [[scalability-is-not-performance.post|scaling issues]] earlier. In settings where CPU time is expensive and you can’t just scale out, like IoT, they are prohibitive. 

Doesn’t matter how much I/O you have to do.
# The (performance) cost of abstractions
Rust and Go are both compiled languages known for their high performance. However, Rust has an edge: its abstractions have lower performance costs.

Let’s look at one example: how each language handles abstracting over data types.

Both languages have a notion of `struct`, a data type with fields and methods. 

Both languages provide a mechanism for abstracting over structs. In Go, it’s called an interface, while in Rust it’s called a trait.
## Go
A Go interface allows us to abstract over different structs that define members with the same names. It looks like this:
```go
type Drawable interface {
    Draw()
}

func render(value Drawable) {
    value.Draw()
}

```

Calling a method on an interface forces the call to go through dynamic dispatch. The address of the corresponding function needs to be looked up in a table and the call can’t be inlined.
## Rust
Meanwhile, in Rust, traits are primarily used in generic code:

```rust
trait Drawable {
    fn draw(&self);
}

fn render<T: Drawable>(value: &T) {
    value.draw();
}
```

This works quite differently. Instead of emitting a single function that uses dynamic dispatch, the Rust compiler creates a specialized `render` function for every struct type `T`.

This means the call to `draw` can be dispatched statically and can even be inlined. The code that’s emitted depends on `T` itself, but each version is inherently more efficient than Go’s.
## What this means
Most abstractions, unless very carefully designed, have runtime performance costs. Abstraction typically involves some level of indirection, whether type checking, a lookup table, or something else.

Rust’s flagship abstractions are carefully designed to avoid this. It’s one reason Rust typically performs better than Go.
# What about C?
C is very fast. In some ways, it’s the fastest programming language. But what’s the secret behind its legendary performance?

There’s just nothing to slow it down.
## It doesn’t have nice things
Earlier, we looked at how Rust and Go handle polymorphism. C takes a third approach — not having any.

C’s abstractions are extremely basic and map directly to machine code. While Rust prides itself on its low-cost abstractions, C’s abstractions cost even less. They barely exist.
## It doesn’t check your work
A lot of languages, including Rust, need to make runtime checks to make sure the emitted machine code doesn’t try to read or write uninitialized memory. 

This prevents dangerous security vulnerabilities and makes tracking and identifying bugs much easier, but it does add a bit of overhead to every program.

In C, this kind of thing is covered by *undefined behavior*, which is another way of saying the language just doesn’t care.

Just another feature C doesn’t have.
## Compilers are easier to write
There is another cost to abstractions, besides runtime performance — the difficulty of compiling them to machine code. High-level languages are hard to compile.

C’s abstractions, what little of them exist, were designed around machine code in the first place. This makes bad C compilers a relatively easy thing to write. This was, in fact, one of C’s design goals.

And yet modern C compilers for mainstream platforms have seen decades of work. A lot of that work has gone into one thing — runtime performance.
# Hurry up and wait
Now let’s look at something totally different: the way in which conventions affect the performance of a typical bit of code.

In Node.js, the default version of an expensive operation — such as reading a file — is always async. Meaning, it uses the runtime’s event loop to wait on I/O without blocking.

While a sync version often exists, using it in production code is typically just regarded as a mistake.

This is because JavaScript is single-threaded. One blocking I/O call stalls the whole program and prevents it from responding to requests.

(Incidentally, while JavaScript developers are quite used to having only one thread, in any other language it would be a little insane.)

Meanwhile, similar functions in C#, such as `File.ReadAllText`, are blocking. They capture the entire thread to do this one specific task. C# does have `File.ReadAllTextAsync`, but that’s not the one developers tend to reach for.

Now, C# does have more than one thread, so you might think that’s okay.

But it’s not. Although threads are available, they are still viewed as a valuable resource. ASP.NET only maintains so many, and at some point [it will run out](https://learn.microsoft.com/en-us/dotnet/core/diagnostics/debug-threadpool-starvation). Once this happens, future requests will start to queue up.

Seeing blocking I/O in C# is [quite common](https://learn.microsoft.com/en-gb/aspnet/core/fundamentals/best-practices?view=aspnetcore-10.0#avoid-blocking-calls), even though everyone agrees it shouldn’t be. Python suffers from similar issues for much the same reasons. 

I think this is why I’ve heard Node.js described as fast. Its threading model and API conventions force developers to avoid blocking code unless absolutely necessary. This lead to endpoints that can feel more responsive.

While using async I/O in C# is not very hard, there can be quite a big difference — in both performance and effort — between the absolute peak of what’s possible and what you can typically expect.
# Is C faster than assembly?
It’s well known that optimizing C compilers can emit assembly code that’s way faster than what a human can write.

Except not really. 

If an expert really tries, using platform-specific instructions carefully arranged to saturate the specific pipeline they are targeting, and using sly workload-specific trivia that a compiler can’t infer, they can do better.

How much better? According to one [FFMpeg](https://ffmpeg.org/pipermail/ffmpeg-devel/2024-May/326959.html) benchmark, almost a factor of 2!

That’s not the only library with hand-written implementations, either. The same goes for `dav1d`, `x264`, and `OpenSSL`. 

But in practice, it’s just not the kind of code a human is typically going to write. It takes a very long time, and all these devilish optimizations can easily lead to equally demonic and inexplicable bugs.

Nor are you likely to see similar performance gains. 

A more typical example of hand-written assembly will probably perform worse than a compiled C counterpart, and for most code, the immense effort to do better is just not worth the bother.

I still find it a little thrilling that the art is far from dead.
# Conclusion
Languages like C and Rust beat almost everything at the nanosecond time scale, but that becomes less important when code is I/O bound anyway.

In that environment, perhaps Node’s strong support for async primitives provides an advantage.

Yet high-performance languages can still come out on top when looking at the maximum possible throughput a server can sustain.

Then there is assembly. It can potentially beat everything, but taking advantage of that potential is very hard.

So… which language is the fastest?