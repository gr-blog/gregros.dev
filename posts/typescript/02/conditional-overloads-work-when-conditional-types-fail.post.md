---
title: Conditional overloads that work when conditional types fail
published: 2025-07-08
updated: 2025-07-08
description: ...
---
How can we define methods that have different signatures depending on a type parameter? In this article, we’ll take a look at conditional types – but end up using a more subtle pattern using overload resolution.

Let’s take a look!

---

Let’s say we have a `List<T>`, and we want a `map` method that maps each element of the list to something else. We can define it like this:

```ts
declare class List<T> {
    map<S>(iteratee: (x: T) => S): List<S>
}
```

But what if we had a `List<Promise<T>>`? A user who wants to use our `map` method asynchronously would need to write something like this:

```ts
list.map(async p => {
    const v = await p
    return projection(v)
})
```

That’s pretty awkward! What if we instead allow the user to project the *awaited value* of each Promise, rather than the Promise itself?

At runtime, we can check every element to see if it’s a Promise, and if it is, project it using `then`. Simple enough.

But what about at compile-time? We would need a method that has a different signature depending on what `T` happens to be! 
# Trying to use conditional types
Naturally, the first thing we’d reach for is conditional types. The idea is to phrase both the argument and the return type based on what `T` actually is:

```ts
declare class List<T> {
    map<R>(
        iteratee: (e: T extends Promise<infer S> ? S : T) => R
    ): List<T extends Promise<any> ? Promise<R> : R>
}
// You can also use Awaited<T> to the same effect
```

But something interesting happens when `T` is actually generic at the point of call, such as in the following code:

```ts
function example<T>() {
    return new List<T>()
	    .map(x => x) // no-op
	    .map(x => x) // no-op
}
```

Intuitively, we would expect the return type of `example<T>` to be `List<T>`, since we basically just added a few no-ops.

Unfortunately, TypeScript disagrees. Instead, its return type turns out to be:

```ts
List<(T extends Promise<any> ? Promise<T extends Promise<infer S> ? S : T> : T extends Promise<infer S> ? S : T) extends Promise<...> ? Promise<...> : (T extends Promise<any> ? Promise<T extends Promise<infer S> ? S : T> : T extends Promise<infer S> ? S : T) extends Promise<...> ? S : T extends Promise<any> ? Promise<T extends Promise<infer S> ? S : T> : T extends Promise<infer S> ? S : T>
```
Even though it *looks* like some of these conditionals should simplify, TypeScript doesn’t even try. Instead, it lets them build on themselves until we get this thing.

TypeScript _will_ resolve all these conditionals if we call the method as `example<number>`. But if we keep the code generic, we’re going to end up with these unreadable types just lying around all over the place.

The only way to deal with these things is to use lots of type assertions and `any`, which probably won’t pass code review. 

Is there a better approach? Turns out, yes! But it involves a clever trick that combines three TypeScript features in a way you might not expect...
# The conditional overload
The three features are:

- The `this` annotation.
- Overload resolution
- Type parameter inference.

Let’s start by taking a look at each feature separately. We’ll start with the `this` annotation.
## The this annotation
In JavaScript, the `this` argument is what you call the function *on*. For example, the code `thing.doStuff()` calls `doStuff` with `thing` as the `this` argument.

TypeScript takes this idea but goes a bit further – functions can declare a *`this` annotation* as part of their argument list, like this:

```ts
function example(this: { a: 1 }) {
    return this.a
}
```

TypeScript treats that `this` parameter just like any other argument. It only gets checked when you actually call the function.

So the following code, for example, will fail to compile because the `this` argument is missing:

```ts
example()
```

We can only call it by passing `this` explicitly:

```ts
example.call({ a: 1 })
```

It’s a pretty obscure feature – but we can use it to get some interesting results. 
## Methods with a this annotation
Strangely, a `this` annotation on a method works the same as on a function – it doesn’t take into account where the method is actually defined.

This means we can define a method on a class that can’t be called normally:

```ts
declare class Example {
    doStuff(this: { a: 1 }): void
}

let a = new Example()

a.doStuff() // ❌ `this` argument not assignable to {a: 1}
```

But by extending the class, we can make the method callable:

```ts
declare class Example2 extends Example {
    a: 1
}

let b = new Example2()

b.doStuff() // ✅ works
```

I like to call these *conditional methods*, and they’re kind of cool – but they’re also something we need to be careful about. These methods aren’t **callable** but they’re still **visible**. That leads to a lot of confusion when users try to call them.

Luckily, the next stage of the pattern solves this problem!
## Conditional overloads
The trick is using *conditional methods* as overloads, while ensuring that some version of the method is callable no matter what `this` happens to be. 

Here is an example:

```ts
declare class Example {
    doThing(this: { a: 1 }): { a: 1 }
    doThing(): {}
}
```

TypeScript will pick the first overload that matches – in other words, order matters here. We need to put the specialized overloads first and the relaxed overloads last.

Now we’re finally close to a solution, but we’re not quite there yet. Let’s take a look at how conditional overloads interact with type parameters!
## Rephrasing type parameters
For this part, let’s look at the signature of a `push` method. We’ll declare it in two different ways:

```ts
declare class List<T> {
    pushNormal(item: T): void
    
    pushWeird<S>(this: List<S>, item: S): void
}
```

Although these signatures seem different, they actually end up behaving in exactly the same way – thanks to TypeScript’s type parameter inference.

Type parameter inference works at the point of call, using the types of the value parameters to figure out what the type parameters should be.

Since the `this` argument is treated as just another argument by TypeScript, it’s also part of this inference process. It goes like this:

- The value of `this` has the type `List<T>`
- The `this` annotation on the method is `List<S>`
- The compiler infers $S \equiv T$. 
- The inferred signature becomes `pushWeird(this: List<T>, item: T)`.

It’s not very useful when used this way, of course. But using this pattern, we can also rephrase what `S` refers to. Take a look:

```ts
declare class List<T> {
    map<_T, S>(
        this: List<Promise<_T>>,
        iteratee: (x: _T) => S
    ): List<Promise<S>>
}
```

Based on what we know about the `this` argument:

- `this` must be assignable to `List<Promise<_T>>` for some `_T`.
- Inference will figure out what `_T` should be.

In other words, this approach lets us “look inside” `T`, shaping the signature of `map` depending on what it happens to be.

Notice how the declared type parameter `T` is not used in the signature. Worse, using it will actually break type checking! That’s why I recommend shadowing the original `T` when doing this:

```ts
declare class List<T> {
    map<T, S>(
        this: List<Promise<T>>,
        iteratee: (x: T) => S
    ): List<S>
}
```

To top it off, we just add a second overload to cover the case when `T` is not actually a promise:

```ts
declare class List<T> {
    map<T, S>(
    this: List<Promise<T>>,
    iteratee: (e: T) => S
  ): List<Promise<S>>
  
    map<S>(iteratee: (e: T) => S): List<S>
}
```
# Testing it out
With this version, our code works as expected. It works when we know `T` is a promise:

```ts
new List<number>()
    .map(async x => x)
    .map(x => x + 1) satisfies List<Promise<number>>
```

While retaining the return type in case of a no-op:

```ts
function example<T>() {
    return new List<T>().map(x => x).map(x => x) satisfies List<T>
}
```
# Conclusion
Conditional types can be a bit fragile, especially when generic code is involved. 

In this post, we looked at a way of having conditional logic without conditional types, using TypeScript’s overloads feature together with `this` annotations.

I hope you find it useful!