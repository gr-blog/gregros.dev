---
title: Fixing type inference in generic rest parameters
published: 2025-07-11
updated: 2025-07-11
---
It turns out that combining generics with rest arguments is kind of tricky in TypeScript. Let’s see where the normal approach fails and how we can use tuples to fix it!

---
Let’s say we have a type called `List<T>` that looks like this:

```ts
interface List<T> {
    at(n: number): T;
}
```

We want to define a constructor function `List` that creates an instance of `List<T>` from any number of elements.

Since the code `[x, y, z]` always compiles, no matter what `x`, `y`, and `z` happen to be, so should the code `List(x, y, z)`.

This seems like a classic use case for a rest argument. Something like this:

```ts
declare function List<T>(...args: T[]): List<T>
```

But there is a problem here. While this approach does work if we just use values like `1` and `2`:

```ts
const list: List<number> = List(1, 2)
```

It [fails](https://www.typescriptlang.org/play/?#code/AQSwdgLgpgTgZgQwMZWAGRAZwgHgCoB8wA3gFDDAIQAUYAXMGAK4C2ARrAJQN4DcpAX1KkAJlCQAbBDFRwmYJBBAB7MOiy5C1AHS7pAc0w8A2gF1u67PgKkkq7MAkbgAXks0AjABpgAIgAWUBISysAA7sowEiK+nLxAA) if we add in something else, like a string:

```ts
const list = List(1, "hello world")
// ⛔ Argument of type `string` is not assignable to parameter of type `number`
```

We get a different version of this problem if we instead pass unrelated object types:

```ts
List({ a: 1 }, { b: 2 })
```

This does compile. But where we might expect to get a neat disjunction type, the compiler widens it into something [pretty weird](https://www.typescriptlang.org/play/?#code/AQSwdgLgpgTgZgQwMZWAGRAZwgHgCoB8wA3gFDDAIQAUYAXMGAK4C2ARrAJQN4DcpAX1KkAJlCQAbBDFRwmYJBBAB7MOiy5C1AHS7pAc0w8A2gF1u67PgKkkq7MAkbgAXks1ilBgEZgAgDQkwGwMAEx+nLxAA):

```ts
List<{ a: 1; b?: undefined } | { b: 2; a?: undefined }>
```

Why is this happening, and is there a way to fix it?
# When type inference fails
Because we’re not specifying the type parameter `T`, we’re asking TypeScript to figure out what it should be on its own.

In other words, we’re asking the compiler to *infer* it – use the types it does know about to figure out what it should be.

Type inference is a convenience feature, so it can get pretty murky. TypeScript can basically choose whatever method it wants.

Some of these methods always succeed – like always inferring `unknown` – but others can lead to compilation errors. That’s what happened here.

In fact, I’m not exactly sure how inference works in this case. Here’s what I’ve learned:

- It widens types, but only up to a point.
- Literal types, objects, and primitive types are all treated differently.
- The order in which the parameters appear can matter.
- But in other cases, TypeScript ignores it.

Here are some [tests](https://www.typescriptlang.org/play/?importHelpers=true&experimentalDecorators=true#code/CYUwxgNghgTiAEkoGdnwDIEtkBcA8AKgHzwDeAUPPFDgBQB2AXPPQK4C2ARiDAJTMEA3OQC+5UEjjwAZq3pgcmAPb0M2fMVoA6HbADmyAQG0AuvzW5CRYeTArcLDtxjwAvPACM1NGy49b9jjwAPpe7l4ongH0DsEATG7wCZFx0Q5QGYlGHgA0SXkARAAWIBAQSgUmaUFyANb0SgDuqu4ADN7wdQ3N5Fi4tKF58bzwAPSjnvAAPkm96gO5ZCIj45OAMuRz-YPwBQUrE14bfXTbu0Nx++vkm3SkUMweInmknMxxy2MTpDpaYscLhQKeS6TXovButDOIUWw0+kyAA) – if you manage to piece it together, I’d love to hear about it in the [Discord](https://discord.gg/ePjFUSRfPh)!

I understand the overall motivation, though – TypeScript is trying to avoid inferring a `T` that’s “too broad”. It’s a general solution that probably works for most APIs. That means it fails for some, and our array-like List constructor is just one example.
# Where tuples come in
Since defining an array always works, TypeScript uses different logic for inferring its type – it just takes the disjunction of all the element types. Since we want to reproduce this logic, we’d like TypeScript to do the same with `List`.

```ts
const example = [ 1, "2", true ] satisfies (1 | "2" | true)[]
```

We can make use of this logic ourselves by rephrasing the signature using a generic tuple argument, which we’ll use to annotate the rest parameter:

```ts
declare function List<Ts extends readonly any[]>(...items: Ts): List<Ts[number]>
```

This isn’t a completely different signature – we’re just being a bit more specific about how we want inference to happen.

Defined like this, the following code compiles just fine, [giving us](https://www.typescriptlang.org/play/?#code/AQSwdgLgpgTgZgQwMZWAGRAZwgHgCoB8wA3gFDDAIQAUYAXMGAK4C2ARrAJQN4DcpAX1IATKEgA2CGKjhMwSCCAD2YdFlx5MwKAA9oYYVukJhK8QE9KYcwG0AugWoA6FyGgtMPTNzXZ8mG2Z2WAd+JBVsYHF1YABeXxpiSgYARmABABoSYDYGACZ0zjCIiCj1AviMbGoUrIAiAAsocXElYAB3JRhxYTqspIRUwuAgA) a clean disjunction of value and object types:

```ts
List(1, "hello world", { a: 1 }) satisfies List<number | string | { a: number }>
```

# Conclusion
Using generics with rest parameters can sometimes cause type inference to fail. In this post, we’ve looked at a way of overcoming this problem using tuples – leading to more flexible variadic signatures.

You should use this approach whenever you want to allow rest arguments to accept any combination of types, including some that seem incompatible with each other.
