---
title: Treating types as values with type-level maps
published: 2025-07-14
updated: 2025-07-16
figure: type-level-map
---
%%%?
With a shift in perspective, we can view complex type declarations as another kind of code. Code where types are values.

There’s no better example of this than the type-level map. Let’s take a closer look!
%%
TypeScript is famous for its complex types and what they allow framework authors to achieve.

I'd like to introduce a different way to think about complex types: as **type-level code**, code that happens to execute during compilation.

We'll contrast it with **runtime code** – the normal code that gets stuff done.

Runtime code has strings, numbers, and objects as values; meanwhile, in type-level code, the **values are types themselves.**

In type-level code, we:

- Apply type-level operators.
- Call type-level functions (these are simply generic types).
- Define type-level interfaces, which we enforce using generic constraints.
- Make use of type-level data structures.

In this post, I’d like to focus on **type-level data structures**, and specifically **type-level maps**. They’re a great example of what type-level code really means.
# What is a type-level map?
A type-level map is just an object type. We typically define it using an interface:

```ts
interface TypeLevelMap {
    key1: MyType;
    key2: MyOtherType;
    key3: true;
    key4: object;
}
```

While interfaces usually represents the shape of a runtime object, in our case one like this:

```js
const a = {
    key1: { name: "MyType" },
    key2: { name: "MyOtherType" },
    key3: true,
    key4: new Date()
}
```

When we’re looking at it as a type-level map, it becomes an *immutable dictionary*. In this dictionary, *types are values* and *the keys are strings*.

Here’s how we can visualize it:

```canva size=580x330 ;; key=type-level-map ;; alt=A diagram showing strings mapped to types
https://www.canva.com/design/DAGtzwsCdfc/CUBX4Z0ZkUGew5-exmTVUA/view
```

That’s a nice diagram, but how do we use them?
# Working with type-level maps
To justify the shift to “type-level maps”, we need to find operations on object types that mimic how we might work with maps in runtime code. That means:

- Listing keys
- Looking up values by key
- Merging two maps
- Transforming a map

If we were doing math, we’d call this an **isomorphism** between dictionaries and object types. That means we can regard one as the other.
## Listing keys
We can list the keys in a type-level map using the `keyof` operator:

```ts
type Map = { key1: 42; key2: 123 }

type Keys = keyof Map // "key1" | "key2"
```

The union type we get is really a **type-level set**. But that’s outside the scope of this post.

Instead, lets compare it to listing keys on a JavaScript object:

```ts
const map = { key1: 42, key2: 123 }

const result = Object.keys(map) // ["key1", "key2"]
```

## Lookups
We look up values by key using TypeScript’s lookup types:

```ts
type Map = { key: 42 }

type Result = Map["key"] // 42
```

That example shows what I like to call a *static lookup*. We can also do a *generic lookup* based on a type parameter, like this:

```ts
type MapValueOfKey<Key extends keyof Map> = Map[Key]
```

We can compare this to the same lookup in JavaScript:

```ts
const map = { key: 42 }

const result = map["key"] // 42
```

TypeScript has an extra feature, though – you can look up lots of values at once!
### Multi-value lookups
TypeScript lets us look up more than one value by passing a bunch of keys:

```ts
type Map = { a: 1; b: 2; c: 3 }

type AB = Map["a" | "b"] // 1 | 2
```

### Listing values
Taken to its logical conclusion, if we pass all the keys, we’ll get all the values:

```ts
type Map = { a: 1; b: 2; c: 3 }

type Vals = Map[keyof Map] // 1 | 2 | 3
```

## Transforming
Since type-level maps are immutable, we can’t change them like we would a JavaScript dictionary. But we can still transform one map into another map.

We do this using a mapped type:

```ts
type Map = { key1: 42; key2: 123 }

type Result = {
    [Key in keyof Map]: `${Map[Key]}`;
} // {key1: "42"; key2: "123"}
```

The closest JavaScript equivalent is the `mapValues` function from `lodash`:

```ts
import { mapValues } from "lodash"
const map = { key1: 42, key2: 123 }

const result = mapValues(map, x => `${x}`) // {key1: "42", key2: "123"}
```

Next, we’ll take a look at a use case.
# Use case: handling events
Type-level maps have lots of use cases. One of the most common ones is **handling events**. In fact, you might’ve encountered it yourself.

Let’s say we have a `Button` object that has a bunch of events. Every event comes with its own information object:

- `click` says which button the user clicked, either `"left"` or `"right"`.
- `hover` gives the $(x,y)$ of the pointer.
- `mount` doesn’t have any special information.

As is customary, we manage events using three main methods:

- `on` defines a handler for an event.
- `off` removes a handler.
- `emit` emits an event together with an information object.

There are a few ways we can implement this pattern at the type level.
## Approach A: loosely typed
We could define all three methods with no type information. Like this:

```ts
export type Handler = (info: object) => void

declare class Button {
    emit(name: string, info: object): void
    on(name: string, handler: Handler): void
    off(handler: Handler): void
}
let button = new Button()
button.emit("clikc", {
    button: 1
})
```

This works, but we’re not type checking anything, introducing the possibility of sneaky bugs. Ideally, we’d like TypeScript to check event names and info objects, as well as handler signatures.

Let’s look at a second approach.
## Approach B: hand-coding everything
One way to achieve that is to hand-code an overload signature for each method and every type of event, like this:

```ts
type ClickEventInfo = { button: "left" | "right" }

type HoverEventInfo = { x: number; y: number }

type Handler<Info> = (info: Info) => void

declare class Button {
    // click events:
    emit(name: "click", info: ClickEventInfo): void
    on(name: "click", handler: Handler<ClickEventInfo>): void
    off(handler: Handler<ClickEventInfo>): void

    // hover events:
    emit(name: "hover", info: HoverEventInfo): void
    // ...
}
```

This solution shows that many of the problems we meet in runtime code are also present in type-level code. In this case, **we’re not DRY** – we keep repeating ourselves.

That makes expanding the `Button` with more events time consuming and error-prone, and it also means we can’t easily extend the event infrastructure to cover other types of elements.

We can compare this approach to copy-pasting the runtime code for registering an event handler:

```js
class Button {
    _clickHandlers = []
    _hoverHandlers = []
    _mountHandlers = []
    on(name, handler) {
        if (name === "click") {
            this._clickHandlers.push(handler)
        }
        if (name === "hover") {
            this._hoverHandlers.push(handler)
        }
        if (name === "mount") {
            this._hoverHandlers.push(handler)
        }
    }
}
```

In runtime code, the solution is pretty obvious – *just use a Map*.

```js
class Button {
    _handlers = new Map()
    on(name, handler) {
        let existing = this._handlers.get(name)
        if (!existing) {
            existing = new Set()
            this._handlers.set(name, existing)
        }
        existing.add(handler)
    }
}
```

We can do the same thing with our type-level code!
## Approach C: using a type-level map
We start by defining a type-level map that maps every event name to its information object:

```ts

export interface ButtonEventMap {
    click: ClickEventInfo
    hover: HoverEventInfo
    mount: {}
}
```

This type-level map also keeps track of all the valid event names.

We can then use local utility types, which are like variables in type-level code, to store the results of operations on the map:

```ts
// Get the map's keys:
type ButtonEventNames = keyof ButtonEventMap

// Transform each key to create a map of handlers:
type ButtonEventHandlerMap = {
    [Name in ButtonEventNames]: Handler<
        ButtonEventMap[Name]
    >
}
```

Finally, we can declare the `Button` class itself, using our utility types together with the map operations we talked about earlier:

```ts
declare class Button {
    // A generic lookup to get a handler:
    on<Name extends ButtonEventNames>(
        name: Name,
        handler: ButtonEventHandlerMap[Name]
    ): void

    // A generic lookup to get the info object:
    emit<Name extends ButtonEventNames>(
        name: Name,
        info: ButtonEventMap[Name]
    ): void

    // Allow only valid handlers by listing values:
    off(handler: ButtonEventHandlerMap[ButtonEventNames]): void
}
```

Looks interesting, but let’s make sure it actually works!
### Testing it
First, we create a button:

```ts
const button = new Button()
```

We’ll define some handlers to see it compiles:

```ts
button.on("click", obj => {
    console.log(`User clicked the ${obj.button} button!`)
})

button.on("hover", obj => {
    console.log(`Mouse pointer is in (${obj.x}, ${obj.y})!`)
})
```

But the whole point was to catch typos and stuff like that. Let’s check that works too:

```ts
// @ts-expect-error clikc is not a valid event
button.on("clikc", obj => {})

// @ts-expect-error Missing property 'y'
button.emit("hover", {
    x: 1
})

```

You can try it out yourself in this [playground link](https://www.typescriptlang.org/play/?importHelpers=true&experimentalDecorators=true&ssl=32&ssc=28&pln=31&pc=1#code/C4TwDgpgBAwgNgSwMYGsCiA3CA7YBJbAMwHsoBeKAbygCMBXYYY7ALigCI4JDh2oAfDgCcEAcwAWvKAF8AULNCQoACWJYhmHPiKkK1AB5tsdALY0IQgNxQQR0+aEz5i6MoCG2ACZchAHgIkAHzkUAAUCDpsAcQAlOTBGMQInvIQ+mDEQsBQEcAWhG5I0ABCDEzYmrgAsm5gVLJQjVBIiKhs8MjoWLjRDU3iahZsquqV2iR9jSbEdLhslHJysgD0y1AA4hDZwOLQJrUA5ADOUCgQIEcANM3M6tukcMQA7hZIbkcQCuAlZcxjAHJuEwQE4UM4gYiEKClRh-brAGpgeSrKAAFSEHiOJCEJigEEK4lO5ygTGaQnxeSgbig+zqkKg4g83gsRxYXyUMPKY3cXh8iJClEmUAA2oDgTlsNDfhV4WKQQBdYZMny+TlwrSI0VAiDywKyJaeCAtNzk5pwd4nNWSwVNKAogCCUFEOAsyCgj2IKDodVJzuy1MZvKGQuYvjleP0eS8lulAO1R0CoWw2rYcuugeZQjYVu5yosmrl8pibESyXktodTpdIiQ7uInu9JNIfpJuwlJCgxBoACsjcA2baICYEMAw9qI1HPDHYTKtHKE0mU1A0+3iNnY-CC9qiyWkikhQ64I8np3sHAQFAMG5EJ4GXmhCcaBfEEdgBFRJfr3QQQOmpDCKEGY+OuM65kGQiajmsrxjul57vq8hIMwr60NKITYBAJ5WqEMTyPQM4AHTMKE7AtJ07DXF23bxPUtpIdgRzEFwBGPKIoQAAYAKofI4ZGoBAt47NAAAklBUQR+HlNIqEzgAhOxuHSLhsiScwRHYCRAzqBRnY9jRNpNPRjHMaxHFVDMHxQBkuQWDkJwRGEonifo0jXE5PYESASnyYpykogAAsARwALRpJASDAKFQhCJkZoICgtYICc2DEP6n43ni8IqdK6kkWRCU6VR+lKciayBSFYV9lFMWOFUSVHO+VkxZAWQXgcIAHNlhFDiOmmDEIOkGY0hhQAAjPquFAA)!
# Conclusion
Type-level code is another way of looking at type declarations. This approach lets us explain complexity using the same principles and tools we’ve learned to deal with runtime code.

In this post, we’ve taken a look at type-level maps. We saw how it corresponds to a runtime dictionary, and how using one can solve design issues at the type level.

I hope you’ll join me in exploring these concepts in the future!
