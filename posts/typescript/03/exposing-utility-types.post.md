---
title: Exposing utility types
description: ""
published: "2026-05-04"
updated: "2026-05-04"
---
%%%?
What do you do when you have a bunch of utility types that are closely related to a class or interface? Should you expose them or not?
%%

Packages can expose multiple functions and classes. Some of these can have their own miniature world of utility types.

Should you expose those types or not? If you do expose them, what’s the best way to do so? Let’s find out!

---
Let’s say we have a class called [Seq](https://github.com/gregros/doddle?tab=readme-ov-file#seq), a wrapper for Iteraables tht provides a set of powerful and convenient operators.

This wrapper has a bunch of types associated with it, like `Iteratee` which is a projection function type. This type is used in the `map` method, as follows:

```ts
const mapped = seq([1, 2, 3]).map(x => x + 1)
//                                ^Iteratee<number, number>
```

First, let’s think about what happens if you expose the type versus if you don’t.
# Not exposing it
If you don’t expose it, it will be harder or impossible to build utility functions and types around your library, or use them in variable declarations.

For example, users might want to define the projection separately from the call to `map`, in its own variable. They might also want to define shorthand types, such a version of `Iteratee` that just works on numbers:

```ts
type NumberIteratee = Iteratee<number, number>
```

Besides, if there is a function that uses this type, users won’t be able to re-export it, something that comes up every so often.
# Type-only namespaces
Namespaces used to be very popular and unavoidable, but nowadays they’re kind of discouraged. Nevertheless, they’re the only construct (other than a module) that lets you declare and organize types.

Users generally don’t expect to see or dig through namespaces, so you should avoid using them for critical things — making them a decent choice for exporting utility types.

You _still_ have to export them, typically from the main export path, but since you’re just exposing one export that contains a dozen types or more, it’s still worth it.

The nesting also helps users navigate a potentially confusing landscape of complicated types.

Some more tips about using namespaces like this:

- Divide your utility types into categories.
- Use clear naming conventions.
- Declare them _inside_ namespaces and use them yourself.
- Export all of these namespaces from your `index.ts`.

Here is an example of how we might expose an `Iteratee` type through a namespace:

```ts
// You should obviously pick a better name
export namespace MyExtraTypes {
  export type Iteratee<In, Out> = (item: In, index: number) => Out;
}
```

Like I said, you should define these types inside namespaces. However, you don’t absolutely have to. There _is_ a little-known feature that lets you use a namespace to export types from other modules.

It’s the weird `export import` statement!

```ts
import * as SeqTypes from "./seq.js";
export namespace MyOtherTypes {
  export import Iteratee = SeqTypes.Iteratee;
}
```

Don’t use it.

It’s an exotic and obsolete construct from ancient times. It tends to confuse IDEs and is rarely supported by newer export-related features. It can also break refactoring.

We can go one step further, though. Let’s take a look at TypeScript’s version of associated types.
# Associated types
The name is taken from a Swift language feature, and appeared in an early [TypeScript proposal](https://github.com/microsoft/TypeScript/issues/17588) that didn’t end up happening. Nevertheless, we can emulate it using namespaces pretty well.

You can use this pattern when you have a bunch of utility types that are closely connected to a specific class or interface. In our example, `Iteratee` is closely connected to `Seq`, so we can use it.

Here’s how this pattern works:

- Define a namespace with the same name as the class.
- That namespace should be in the same file. Otherwise users will get conflicts.
- The namespace must only contain types.

Here’s how it looks like:

```ts
export class Seq<T> {
  // ...
}
export namespace Seq {
  export type Iteratee<In, Out> = (item: In, index: number) => Out;
}
```

This is a special case of declaration merging. The compiler will pretend there’s just one thing called `Seq` and it contains both the contents of the class and the namespace. 

Declaration merging can modify runtime values. For example, if the namespace contained a function, it would be added as a member of the `Seq` constructor.

However, in this case, the namespace doesn’t contain any functions. It only has types. This means that the class won’t be modified and IntelliSense will only ever show the contents of one of them — either the class or the namespace, never both.

```ts
// Reference in a type position resolves the class type:
declare let s: Seq<number>;

// Reference in runtime code resolve the class constructor:
const b = Seq.apply;

// References to a namespace when `Seq.` is in a type position:
declare let s: Seq.Iteratee<number, string>;
```
