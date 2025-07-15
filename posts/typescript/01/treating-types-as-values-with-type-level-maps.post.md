---
title: Treating types as values with type-level maps
published: "2025-07-13"
updated: "2025-07-13"
---
%%%?
With a shift of perspective, TypeScript’s complex types become type-level code where types are values. 

Type-level code can even include data structures. Let’s take a look at one of them – the type-level map!
%%
TypeScript is famous for its complex type system.

In this article, I’d like to introduce a different approach for thinking about these complex types: as another kind of code that happens to execute during compilation.

This approach contrasts **runtime code** with **type-level code**, while looking at the similarities between them.

- Runtime code *does the thing*.
- Type-level code *makes it easier to do the thing.*
- The values in runtime code are *strings, numbers, and objects*.
- Meanwhile, in type-level code, the values are **types themselves.**

In type-level code, we work with types using operators, call type-level functions (which are just generic types), and so on. We do this in the same way runtime code works with numbers and strings. 

Uniquely, TypeScript’s type-level code has a wide assortment of **data structures** it can use. Since in type-level code all values are types, these data structures are also types. But they still support all the operations a data structure would.

None of them was designed to be a type-level data structure. But that’s what they’ve become.

Let’s take a look at one of them – the type-level map. 
# What is a type-level map?
A type-level map is just an object type. We typically define it using an interface:

```ts
interface TypeLevelMap {
    Key_1: MyType;
    Key_2: MyOtherType;
    Key_3: true;
    Key_4: object;
}
```

While the interface typically represents the shape of a runtime object:

```js
{
	Key_1: {name: "MyType"},
	Key_2: {name: "MyOtherType},
	Key_3: true,
	Key_4: new Date()
}
```

When we’re looking at it as a type-level map, it becomes an *immutable dictionary*. In this dictionary, *types are values* and *the keys are strings*. The strange naming convention is meant to make it stand out from other object types.

We might visualize it like this:
$$
\begin{align*}
\mathtt{Key\_1}&\Rightarrow\mathtt{MyType} \\
\mathtt{Key\_2}&\Rightarrow\mathtt{MyOtherType} \\
\mathtt{Key\_3}&\Rightarrow\mathtt{true} \\
\mathtt{Key\_4}&\Rightarrow\mathtt{object}
\end{align*}
$$
Whatever you call them, these structures form a critical part of modern TypeScript APIs. 
# Working with type-level maps
A shift to “type-level maps” means finding operations that mimic how we might work with maps in runtime code. That means:

- Listing keys
- Looking up values by key
- Merging two maps
- Transforming a map

If this was math, we’d call this an *isomorphism* between dictionaries and object types. Which means we can regard one as the other.
## Listing keys
We can list the keys in a type-level map using the `keyof` operator:

```ts
type Map = { Key_1: 42; Key_2: 123 }

type Keys = keyof Map // "Key_1" | "Key_2"
```

We can compare this to using `Object.keys` on a JavaScript object:

```ts
const map = { key1: 42, key2: 123 }

const result = Object.keys(map) // ["key1", "key2"]
```
## Lookups
We look up values by key using TypeScript’s lookup types:

```ts
type Map = { Key: 42 }

type Result = Map["Key"] // 42
```

That example shows what I like to call a *static lookup*. We can also do a *generic lookup* based on a type parameter, like this:

```ts
type MapValueOfKey<Key extends keyof Map> = Map[Key]
```

We can compare this to the `[]` lookup in JavaScript:

```ts
const map = { key: 42 }

const result = map["key"] // 42
```
## Merging
We can merge two type-level maps using the `&` operator:

```ts
type Map1 = { A: 1 }
type Map2 = { B: 2 }

type Result = Map1 & Map2 // {A: 1; B: 2}
```

Meanwhile, we can merge JS objects using the `...` operator:

```ts
const map1 = { a: 1 }
const map2 = { b: 2 }

const result = {
    ...map1,
    ...map2
} // {a: 1, b: 2}
```

## Transforming
Since type-level maps are immutable, we can’t change them like we would a JavaScript dictionary. But we can still transform one map into another map.

We do this using a mapped type:

```ts
type Map = { Key_1: 42; Key_2: 123 }

type Result = {
    [Key in keyof Map]: `${Map[Key]}`;
} // {Key_1: "42"; Key_2: "123"}
```

The closest JavaScript equivalent here is the `mapValues` function from `lodash`:

```ts
import { mapValues } from "lodash"
const map = { key1: 42, key2: 123 }

const result = mapValues(map, x => `${x}`) // {key1: "42", key2: "123"}
```
# A somewhat simple use-case
Imagine we’re building a browser automation platform. This platform tells the browser what to do using `Command` objects.
## Command objects
Every command object has three things:

- A string **name** which is its unique identifier.
- A set of named arguments bundled into a single **input object**.
- A **return type**, which is always wrapped in a Promise.
### Running a command
We run a command using the syntax `Browser.call(name, args)`.
### Possible commands
Let’s take a look at three possible commands:

1. **`Click`** Emulates a mouse click at position $(x, y)$.
2. **`Goto`** Navigates to a webpage at the address `url`. Returns the new URL after the page has loaded.
3. **`GetLocation`** Gets the current webpage address as a string.
## Implementation
There are a few different ways to *implement* this API at the type level.
### Using overloads
One way is to define an overload for every command, like this:

```ts
declare class Browser {
    call(name: "Click", args: { x: number, y: number }): Promise<void>
    call(name: "Goto", args: { url: string }): Promise<string>
    call(name: "GetLocation", args: {}): Promise<string>
}
```

This has quite a few problems, though:

- We keep repeating ourselves.
- We can’t reference the input or return types of a command.
- Extending the list of commands is error-prone.

Notice how these are the same design issues that we might find in runtime code.
### Using type-level maps
This approach works like this:

1. Define one type-level map that acts as a source of truth, describing all the commands.
2. Then create one generic method that behaves just like the overloads we saw earlier.

The type-level map uses command names as it keys. Each one of its values is another type-level map that has the structure:

```ts
type CommandType = {
    Args: object
    Returns: unknown
}
```

Here’s the entire map:

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

Now we can define a `call` function with a computed signature that accesses the type-level map:

```ts
declare class Browser {
    call<Name extends keyof CommandsMap>( // Key list
        name: Name,
        args: CommandsMap[Name]["Args"] // A lookup
    ): Promise<
        CommandsMap[Name]["Returns"] // Another lookup
    >
}

const browser = new Browser()

browser.call("Click", {
	x: 100,
	y: 500
})
```
# Conclusion
Type-level code is another way of looking at type declarations. This approach lets us explain complexity using the same principles and tools we’ve learned to deal with runtime code.

In this article, we’ve taken a look at a type-level map. We saw how it corresponds to a runtime dictionary, and how using one can solve fundamental design issues at the type level. 

I hope you’ll join me in exploring these concepts in the future!

