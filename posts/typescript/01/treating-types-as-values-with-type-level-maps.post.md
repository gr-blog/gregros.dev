---
title: Treating types as values with type-level maps
published: 2025-07-14
updated: 2025-07-14
---
TypeScript is very good at talking about object structure. With a shift of perspective, we can use this ability to talk about dictionaries where types are values. 

Let’s see how that works!

---
Here is what a *type-level map* looks like:

```ts
interface TypeLevelMap {
    Key1: MyType;
    Key2: MyOtherType;
    Key3: true;
    Key4: object;
}
```

From one point of view, it’s just a funny-looking object. From another, it’s a dictionary that maps strings to types:
$$
\begin{align*}
\mathtt{Key1}&\Rightarrow\mathtt{MyType} \\
\mathtt{Key2}&\Rightarrow\mathtt{MyOtherType} \\
\mathtt{Key3}&\Rightarrow\mathtt{true} \\
\mathtt{Key4}&\Rightarrow\mathtt{object}
\end{align*}
$$
When viewed like this, the types are actually *values*, like `42` or `"hello world"`. 
# Working with type-level maps
TypeScript has some great tools for working with type-level maps – which makes sense, since this pattern is built right into the language.

Since a type-level map is like a dictionary with types as values, we’ll compare each tool to JavaScript code that does the same thing to a dictionary object.
## List keys
Here’s how we can get an array of all the keys in an object in runtime code:

```ts
const map = { key_1: 42, key_2: 123 };
const result = Object.keys(map); // ["key_1", "key_2"]
```

The equivalent here is the `keyof` type operator:

```ts
type Map = { Key_1: 42; Key_2: 123 };
type Keys = keyof Map; // "Key_1" | "Key_2"
```
## Look up values
Looking up a property on an object can look like this:

```ts
const map = { key: 42 };
const result = map["key"]; // 42
```

Meanwhile, here is the type-level version:

```ts
type Map = { Key: 42 };
type Result = Map["Key"]; // 42
```

This example shows what I like to call a *static lookup*. We can also do a *generic lookup* based on a type parameter, like this:

```ts
type MapValueOfKey<Key extends keyof Map> = Map[Key];
```
## Merging
We can merge JS objects using the `...` spread operator:

```ts
const map1 = { a: 1 };
const map2 = { b: 2 };
const result = {
    ...map1,
    ...map2
}; // {a: 1, b: 2}
```

Meanwhile, in the land of types, we can merge two type-level maps using the `&` operator:

```ts
type Map1 = { A: 1 };
type Map2 = { B: 2 };
type Result = Map1 & Map2; // {A: 1; B: 2}
```
## Mapping
The mapped type lets us project every “type value” using an expression:

```ts
type Map = { Key_1: 42; Key_2: 123 };
type Result = {
    [Key in keyof Map]: `${Map[Key]}`;
}; // {Key_1: "42"; Key_2: "123"}
```

It doesn’t have a built-in JavaScript equivalent, but we can compare it to `mapValues` from lodash:

```ts
import { mapValues } from "lodash";
const map = { key_1: 42, key_2: 123 };
const result = mapValues(map, x => `${x}`); // {key_1: "42", key_2: "123"}
```
# Use-cases
Type-level maps are a cornerstone of modern TypeScript APIs. Most of the TypeScript packages you’re familiar with use them in one way or another.

Let’s take a look at two simple use-cases.
## Automatic overloads
Imagine we’re building a browser automation platform. Automation happens through command objects. 

Every command object has three things:

- A string **name** which is its unique identifier.
- A set of named arguments bundled into a single **input object**.
- A **return type**, which is always wrapped in a Promise.

We invoke a command using the syntax `Browser.call(name, args)`.

Let’s take a look at three possible commands:

1. **Click:** Emulates a mouse click at position $(x, y)$.
2. **Goto:** Navigates to a webpage at the address `url`. Returns the new URL after the page has loaded.
3. **GetLocation:** Gets the current webpage address as a string.

How do we represent this API using TypeScript?
### Using overloads
One way is to define an overload for every command, like this:

```ts
declare class Browser {
    call(name: "Click", args: { x: number, y: number }): Promise<void>;
    call(name: "Goto", args: { url: string }): Promise<string>;
    call(name: "GetLocation", args: {}): Promise<string>;
}
```

This has quite a few problems, though:

- We keep repeating ourselves.
- We can’t reference the input or return types of a command.
- Extending the list of commands is error-prone.

Let’s take a look at a better way.
### Using type-level maps
This method works like this:

1. Define one type-level map that acts as a source of truth, describing all the commands.
2. Then create one generic method that behaves just like the overloads we saw earlier.

The type-level map will be keyed using the command name. Its value will *also* be a type-level map that always has the structure:

```ts
type CommandType = {
    Args: object
    Returns: unknown
};
```

It looks like this:

```ts
export interface CommandsMap {
    Click: {
        Args: {
            x: number
            y: number
        }
        Returns: void
    }
    Goto: {
        Args: {
            url: string
        }
        Returns: string
    }
    GetLocation: {
        Args: {}
        Returns: string
    }
}
```

Since we’re using `CommandsMap` as a type-level map, it’s not really the type of any runtime object. But it does let us phrase the `call` method like this:

```ts
declare class Browser {
    call<Name extends keyof CommandsMap>(
        name: Name,
        args: CommandsMap[Name]["Args"]
    ): Promise<
        CommandsMap[Name]["Returns"]
    >;
}
```

Note how we combine a *generic lookup* into the `CommandsMap` with a *static lookup* into the command structure itself. 
## Simplifying types
Let’s say we’re building a library for querying the DOM, kind of like JQuery.

This library lets us search the DOM, yielding either individual elements or collections. In both cases, we want to keep track of the possible element types a given object can represent.

So, for example, a query like `$("div")` will only return `div` elements, and we want to make that part of its return type.

We clearly need a generic type to represent this, but there are several ways to define it. Let’s start by looking at an example *without* type-level maps.
### Using HTMLElement
One way is to use the `HTMLElement` interface, which all HTML element types extend:
```ts
interface Tag<TElement extends HTMLElement> {}
```

This has a few issues, though.

For one, `HTMLElement` is a pretty big object type. This generic signature means every instantiation must be compared to that object structurally, which will make type checking a lot slower.

Besides that, the canonical name of every element type has this `HTML*Element` structure. Naming types like this is good out of context, but in this case, it’s going to make the name of our `Tag` type very long.

```ts
type Example = Tag<
  HTMLDivElement | HTMLButtonElement | HTMLCanvasElement | HTMLSomeOtherElement
>;
```

This isn’t just cosmetic – it will truncate compilation errors and make parsing out the names of these types basically impossible.

On top of that, the compilation messages produced won’t actually be that meaningful. TypeScript is a structural language, but [[but-what-is-a-dom-node.post|DOM nodes are not structural]]. Comparing DOM nodes structurally is pointless.
### Using tag names with a type-level map
Instead of using explicit element types, we can reference elements by their tag names. 

We’ll need a type-level map to do this, but luckily one already exists. Namely, the `HTMLElementTagNameMap` which is built into the DOM type declarations. Here’s a snippet:

```ts
interface HTMLElementTagNameMap {
    a: HTMLAnchorElement;
    abbr: HTMLElement;
    address: HTMLElement;
    area: HTMLAreaElement;
    article: HTMLElement;
    aside: HTMLElement;
    audio: HTMLAudioElement;
    b: HTMLElement;
    // ...
}
```

Using this type-level map, we can define our wrapper like this:

```ts
export type TagNames = keyof HTMLElementTagNameMap;
export interface TagWrapper<Tag extends TagNames> {}
```

And here’s what using it looks like:

```ts
type Div = TagWrapper<"div">;
type Canvas = TagWrapper<"canvas">;
type Either = TagWrapper<"div" | "canvas">;
```

This solution is even better in terms of API design! We’re letting users reference each tag using its canonical HTML name, rather than the constructor name. 

We can even use the entire `TagNames` type to define a wrapper for _any_ element, but without considering element structure at all.

```ts
type Any = TagWrapper<TagNames>;
```
# Conclusion
Type-level maps are a powerful TypeScript pattern that’s a staple of advanced type definitions.

In this article, I tried to explain it by comparing it to regular object operations that we already know. 

I hope you liked it!