## Simplifying types
Let’s say we’re building a library for querying the DOM, kind of like JQuery.

This library lets us search the DOM, yielding either individual elements or collections. In both cases, we want to keep track of the possible element types a given object can represent.

So, for example, a query like `$("div")` only returns `div` elements, and we want to make that part of its return type.

We need a generic type to represent this, but there are a few ways to define it. Let’s start by looking at an example *without* type-level maps.
### Using HTMLElement
One way is to use the `HTMLElement` interface, which all HTML element types extend:

```ts
interface Tag<TElement extends HTMLElement> {}
```

This has a few issues, though.

For one, `HTMLElement` is a pretty big object type. This generic signature means that TypeScript must compare it to every instantiation, making type checking a lot slower.

Besides that, the canonical name of every element type has this `HTML*Element` structure. Naming types like is generally a good thing, but in this case, it’s going to make the name of our `Tag` type quite long.

```ts
type Example = Tag<
  HTMLDivElement | HTMLButtonElement | HTMLCanvasElement | HTMLSomeOtherElement
>
```

This isn’t just cosmetic – it'll truncate compilation errors and make parsing out the names of these types impossible.

On top of that, the compilation messages produced won’t be that meaningful. TypeScript is a structural language, but [[but-what-is-a-dom-node.post|DOM nodes aren't structural]]. Comparing DOM nodes structurally is pointless.
### Using tag names with a type-level map
Instead of using explicit element types, we can reference elements by their tag names.

We’ll need a type-level map to do this, but luckily one already exists. Namely, the `HTMLElementTagNameMap` built into the DOM type declarations. Here’s a snippet:

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
export type TagNames = keyof HTMLElementTagNameMap
export interface TagWrapper<Tag extends TagNames> {}
```

Here’s what using it looks like:

```ts
type Div = TagWrapper<"div">
type Canvas = TagWrapper<"canvas">
type Either = TagWrapper<"div" | "canvas">
```

This solution is even better for API design! We’re letting users reference each tag using its canonical HTML name, rather than the constructor name.

We can even use the entire `TagNames` type to define a wrapper for *any* element, but without considering element structure at all.

```ts
type Any = TagWrapper<TagNames>
```

# Conclusion
Type-level maps are a powerful TypeScript pattern that’s a staple of advanced type definitions.

In this post, I tried to explain it by comparing it to regular object operations that we already know.

I hope you liked it!

Type-level maps are a cornerstone of modern TypeScript APIs. Most of the TypeScript packages you’re familiar with use them in one way or another.

Some of the use-cases include:

- Maps of $\mathtt{EventName\Rightarrow EventArgs}$
- Maps of $\mathtt{TagName\Rightarrow ElementType}$
- Maps of $\mathtt{LogLevel \Rightarrow LogLevelName}$

Let’s take a look at a similar but somewhat exotic use-case.
## Automatic overloads

# An example use-case
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

This signature combines listing the map’s keys and performing some lookups.