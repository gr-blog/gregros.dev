---
title: Treating types as values with type-level maps
published: "2025-07-14"
updated: "2025-07-16"
---
%%%?
With a shift in perspective, we can view complex type declarations as another kind of code. Code where types are values.

There’s no better example of this than the type-level map. Let’s take a closer look!
%%
TypeScript is famous for its complex types and what they allow framework authors to achieve.

In this post, I’d like to introduce a different approach for thinking about these complex types: as **type-level code**, code that happens to execute during compilation.

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
    Key_1: MyType;
    Key_2: MyOtherType;
    Key_3: true;
    Key_4: object;
}
```

While interfaces usually represents the shape of a runtime object, in our case one like this:

```js
const a = {
    Key_1: { name: "MyType" },
    Key_2: { name: "MyOtherType" },
    Key_3: true,
    Key_4: new Date()
}
```

When we’re looking at it as a type-level map, it becomes an *immutable dictionary*. In this dictionary, *types are values* and *the keys are strings*.

We show this shift of perspective by changing naming conventions. Where normal objects have keys in `camelCase`, here we might use `Pascal_Snake_Case` .

We might visualize it like this:
$$
\begin{align*}
\mathtt{Key\_1}&\Rightarrow\mathtt{MyType} \\
\mathtt{Key\_2}&\Rightarrow\mathtt{MyOtherType} \\
\mathtt{Key\_3}&\Rightarrow\mathtt{true} \\
\mathtt{Key\_4}&\Rightarrow\mathtt{object}
\end{align*}
$$
Although they aren’t typically named as such, these structures form a critical part of modern TypeScript APIs.
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
type Map = { Key_1: 42; Key_2: 123 }

type Keys = keyof Map // "Key_1" | "Key_2"
```

The union type we get is really a **type-level set**. But that’s outside the scope of this post.

Instead, let’s compare it to listing keys on a JavaScript object:

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

We can compare this to the same lookup in JavaScript:

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
# Use-case: handling events
Type-level maps have a wide variety of use-cases. One of the most common ones is **handling events**. In fact, you might have encountered this one yourself! 

In this example, we have a `Button` object that has a bunch of events, like `click`, `mount`, and `hover`. Every event comes with a distinct information object:

- `click` says which button was clicked, either `"left"` or `"right"`.
- `hover` gives the $(x,y)$ of the pointer.
- `mount` doesn’t have any special information.

As is customary, events are managed using three main methods:

- `on` defines a handler for an event.
- `off` removes a handler.
- `emit` emits an event together with an information object.

We could define all three methods with no type information. This works, but we’re not type checking anything, introducing the possibility of sneaky bugs:

```ts
export type Handler = (name: string, info: object) => void

declare class Button {
  emit(name: string, info: object);
  on(name: string, handler: Handler)
  off(handler: Handler)
}
let button = new Button()
button.emit("clikc", {
	button: 1
})
```

Ideally, we’d like TypeScript to check event names and info objects, as well as handler signatures. 

One way to achieve that is to hand-code an overload signature for each method and every type of event, like this:

```ts

type ClickEventInfo = { button: "left" | "right" }

type HoverEventInfo = { x: number; y: number }

type Handler<Name, Info> = (name: Name, info: Info) => void

declare class Button {
	// click events:
	emit(name: "click", info: ClickEventInfo)
	on(name: "click", handler: Handler<"click", ClickEventInfo>)
	off(handler: Handler<"click", ClickEventInfo>)

	// hover events:
	emit(name: "hover", info: HoverEventInfo)
	// ...
}
```

This solution shows that many of the problems we encounter in runtime code are also present in type-level code. In this case, **we’re not DRY** – we keep repeating ourselves.

That makes expanding the `Button` with additional events time consuming and error-prone, and it also means we can’t easily extend the event infrastructure to cover other types of elements.

We can compare this to copy-pasting the runtime code for registering an event handler:

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

In runtime code, the solution is pretty obvious — *just use a Map*.

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

It turns out the same logic applies to type-level code. We just need to create a **type-level map**. This type-level map will match every event name to its information object:

```ts
export interface ButtonEventsMap {
	click: ClickEventInfo
	hover: HoverEventInfo
	mount: {}
}
// Get the map's keys and store them:
type ButtonEventNames = keyof ButtonEventsMap

// A function that returns the info object type
// for each event:
type HandlerForEvent<
	Name extends string
> = ButtonEventsMap[Name]

// Combining the two:
declare class Button {
	on<Name extends ButtonEventNames>(
		name: Name,
		handler: HandlerForEvent<Name>
	)
	
}
```

Now, in order to build the signatures of the `on`, `emit`, and `off` methods, we use the operators we talked about earlier. 

Let’s take a look at the `on` method first:
```ts
declare class Button {
  on<Name extends keyof ButtonEvents>(
    name: Name,
    handler: Handler<Events[Name]>
  ): void;
}
```

Even better, we can extract these methods into their own interface, with a type parameter standing in for the event map. This encapsulates the type-level event logic and lets us reuse it with ease.

```ts
interface SupportsEvent<Events extends Record<string, object>> {
  on<Name extends keyof Events>(
    name: Name,
    handler: Handler<Events[Name]>
  ): void;
  off<Name extends keyof Events>(
    name: Name,
    handler: Handler<Events[Name]>
  ): void;
  emit<Name extends keyof Events>(
    name: Name,
    handler: Handler<Events[Name]>
  ): void;
}
```




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

Notice how these are the same design issues that we might find in runtime code. They also have similar solutions.
### Using type-level maps
This approach works like this:

1. Define one type-level map that acts as a source of truth, describing all the commands.
2. Then create one generic method that behaves just like the overloads we saw earlier.

The type-level map uses command names as it keys. Each one of its values is another type-level map that matches the type-level interface:

```ts
type CommandType = {
    Args: Record<string, unknown>
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
 
In this post, we’ve taken a look at type-level maps. We saw how it corresponds to a runtime dictionary, and how using one can solve design issues at the type level.

I hope you’ll join me in exploring these concepts in the future!
