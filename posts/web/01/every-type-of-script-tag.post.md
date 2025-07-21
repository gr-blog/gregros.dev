---
title: "How many types of script tags are there?"
description: ""
published: "2025-01-01"
updated: "2025-07-21"
---
%%%?
There used to be just one type of script tag, but after decades of web standards development, we have tons of them.

Let’s take a look at every single one!
%%

I like to divide script tags based on two primary classifications.

1. The **type axis**, which describes **how a script tag executes.**
2. The **source axis**, which says **where it gets its content from**.

We can picture this using a simple tabular diagram:

```canva size=445x305 ;; key=script-tag-types ;; alt=Script tag types diagram
https://www.canva.com/design/DAGby5He9jU/1PyY3PGdsH74HG0GVjaimA/view?utm_content=DAGby5He9jU&utm_campaign=designshare&utm_medium=link2&utm_source=uniquelinks&utlId=h43df53d6b0
```

# Type axis
There are three possible options here, and they’re all determined by the script’s `type` attribute.

- **Non-executable** script tags.
- **Classic** script tags
- **Module** script tags

Let’s take a closer look at each one.
## Non-executable
You’ll get this kind of script tag if your `type` is present but **doesn’t** have one of the recognized values for executable scripts:

- A JavaScript MIME type – `"text/javascript"`
- Module – `"module"`
- No value or the empty string `""`

These script tags won’t execute and won’t fetch any resources. Instead, they’re used to embed data into the page through the body of the tag, usually in the form of JSON.

It’s best practice to use the `type` attribute to show what type of data it has.

Some common values for the attribute include:

- `"application/json"`
- `"application/yaml"`
- `"application/xml"`

In principle, you could use any HTML tag for this purpose, but script tags have a few advantages over other types of tags.

- They're invisible.
- They can appear in the `<head>` part of the page.
- They let you avoid escaping most special characters used in HTML, like `&` and `<`.

Here's an example of a non-executable script tag:

```html
<!-- a non-executable script tag -->
<script type="application/json">
  {
    "analytics": true,
    "options": {
      "theme": "no",
      "antigravity": "ǝnɹʇ",
      "lasers": "pew pew",
      "foo": "bar",
      "canBeHacked": false
    }
  }
</script>
```

## Module
You’ll get this kind of script tag if you have the `type` attribute set to `"module"`.

These script tags are entrypoints to the ES module system. They always fetch and execute asynchronously.

Here are a few examples of ES module script tags of different kinds:

```html
<!-- An inline module script tag -->
<script type="module">
  import { myFunction } from './module.js'
  myFunction()
</script>

<!-- an external module script tag -->
<script
    type="module"
    src="https://example.com/module.js"
></script>

<!-- a data module script tag -->
<script
    type="module"
    src="data:text/javascript,import './module.js'"
>
</script>
```

### The new normal for the web
Nowadays, pretty much everyone uses module script tags.

While porting old code that uses classic script tags can be a thorny proposition, you should absolutely use these if you’re building something new.

One drawback they do have is **compatibility.** Major browsers have supported them since around 2017, but some users continue to use obsolete browsers that don’t support them.

This includes the infamous IE11, but also some built-in mobile browsers.

These browsers account for up to 5% of active users, but this percentage can be much lower or much higher, depending on the specific demographics your product is targeting.

If you’re, say, writing a tech blog, it’s not something you have to worry about. But a government service, a pension fund, or a bank might have stricter requirements.
## Classic
You’ll get this kind of script tag if the `type` attribute:

- Doesn’t exist
- Has a value of `text/javascript` or another valid JavaScript MIME type.
- Has no value, or the value `""`.

Classic script tags have been around since JavaScript became a thing, and they make up the majority of script tags found on websites today.

In spite of their age – or perhaps because of it – they're actually more complicated than the newer module type script tags.

When the browser encounters a classic script tag as part of parsing a webpage, immediately fetches and executes its content synchronously. This blocks the rest of the page from loading.

This means that some of the page might not have loaded yet when the script executes.

For instance, if you place the script before the `<body>` tag, you’ll find `document.body` to be `null`. If you place it before a `<div>`, that `<div>` won’t exist yet.

Some elements, such as images and fonts, can load asynchronously. This means the script could execute while the geometry of the page hasn’t settled yet, changing the results of functions such as `getClientRect`.

Any top-level declarations made here become page-wide globals, accessible from any other script tag.

That's a particularly nasty and error-prone feature, and you’ll often see script tags use scoping constructs like self-executing functions to control it.

Here's an example of a script tag using this technique:

```html
<!-- a classic inline script tag
    using the 'self-executing function'
    technique
-->
<script>
  (function() {
    const myDiv = document.createElement('div')
    myDiv.textContent = 'Hello, world!'
    document.body.appendChild(myDiv)
  })()
</script>
```

Here are examples of different kinds of classic script tags:

```html
<!-- external classic script tag -->
<script
    src="https://example.com/script.js"
></script>

<!-- inline classic script tag -->
<script>
  alert(1)
</script>

<!-- data classic script tag -->
<script
    src="data:text/javascript,alert(1)"
></script>

<!-- with type attribute -->
<script
    src="https://example.com/script.js"
    type="text/javascript"
></script>

<!-- another one -->
<script
    src="https://example.com/script.js"
    type=""
></script>
```

### Client-side infrastructure
The ES module system has replaced classic script tags for frontend development, but that doesn’t mean classic script tags are now obsolete.

Instead, they’ve simply transformed into a specialized tool for low-level applications. While in the last section I phrased it as a drawback, the ability to choose exactly when your code executes is actually very powerful.

For example, classic script tags that appear before the `<body>` tag execute before any visible component of the page has loaded. That guarantees the user hasn’t had the chance to interact with anything yet.

This technique is crucial for many pieces of client-side infrastructure, ranging from analytics packages to security systems, which must come online early to avoid missing security threats or events.

That doesn’t mean they get a free pass to block the page for as long as they like, though.

Rather, this power comes with the responsibility of ensuring as little disruption to the page as possible. If a security system blocks for too long, hurts user experience, and causes clients to lose KPIs – they'll simply switch to something else.
# Source axis
The source axis determines where a script’s JavaScript content comes from. This axis has four possibilities:

- **Inline scripts**, which get their JavaScript contents from the element’s body.
- **External scripts**, which reference a script file by address.
- **Data scripts**, which use a URI with the `data:` pseudo-protocol.
- **Blob scripts**, which use a URI with the `blob:` pseudo-protocol.

The `src` attribute of a script tag determines which group it belongs to.
## Inline scripts
These kinds of scripts don’t have an `src` attribute and embed JavaScript content in the body of the tag.

The browser has special rules for parsing the bodies of script tags. These rules let you avoid escaping special characters like `&` or `<`. But it’s not like the HTML parser tries to parse JavaScript either.

Instead, it simply looks for the string `</script>` and closes the tag as soon as it finds it. It doesn’t matter if it appears in the middle of JavaScript code, as part of a string, or in a JavaScript comment.

So, for example, the following content causes the script to break:

```html
<script type="module">
  const script2 = "<script>alert(1)</script>"
// parsing fails here ↑
</script>
```

Here are examples of valid inline script tags:

```html
<!-- classic inline script tag -->
<script>
	alert(1)
</script>

<!-- module inline script tag -->
<script type="module">
	import { myFunction } from './module.js'
</script>
```

## External script
These scripts have an `src` attribute that points to an HTTP/S URL.

When compared to inline scripts, external scripts have some advantages that make them the most common type of script tag in use today.

1. They let you avoid sending the same bit of JavaScript with every request, leveraging the browser’s caching mechanism and reducing total bandwidth.
2. They have an address that appears in the stack trace, making them far easier to debug.
3. They mean you can use different hosting strategies for different parts of your site, optimizing delivery and potentially reducing costs.
4. They allow for better code organization.

The main disadvantage they have against inline scripts is the extra indirection, which increases latency, at least on the first page load. Whether they're more or less secure than inline scripts is a thorny question that’s hard to answer.

Here are some examples of external script tags:

```html
<!-- classic external script -->
<script
    src="https://example.com/script.js"
></script>

<!-- module external script -->
<script
    type="module"
    src="https://example.com/module.js"
></script>
```

## Data scripts
Data scripts have an `src` attribute that points to a URI that uses the `data:` pseudo-protocol.

As a pseudo-protocol, `data:` doesn’t actually point to the location of a resource. Instead, this pseudo-protocol lets you embed content verbatim into the URI itself, and have the browser load that content as though it came from the network.

`data:` URIs are very handy for many purposes, and you’ll occasionally see scripts loaded this way. We should compare then with inline scripts, rather than external scripts, since they embed JavaScript content instead of referencing another resource.

Data scripts use more consistent escaping rules than inline tags, and can use lots of encodings and character sets, which are often specified as part of the data URI. One of the most popular options is base64, which avoids the need to escape anything.

At the same time, data scripts have the massive disadvantage of usually being illegible. This makes them one of the vectors of choice when attackers inject malicious scripts.

Here are some examples of data script tags in action:

```html
<!-- classic data script tag -->
<script
    src="data:text/javascript,alert(1)"
></script>

<!-- classic base64 encoded data script tag -->
<script
    src="data:text/javascript;base64,YWxlcnQoMSk="
></script>

<!-- classic base64+utf8 encoded data script tag -->
<script
    src="data:text/javascript;charset=UTF-8;base64,YWxlcnQoMSk="
></script>

<!-- module data script tag -->
<script
    type="module"
    src="data:text/javascript,import './module.js'"
></script>
```

## Blob scripts
You can only create Blob scripts from JavaScript. They’re pretty weird.

In JavaScript, a Blob is kind of like a Stream. It represents a bunch of data, without specifying its source or shape.

You can get Blobs as the result of `fetch` requests or from files uploaded by the user. You can also create them using the `Blob` constructor, like this:

```ts
var blob = new Blob(
    ['alert(1)'], 
    { 
        type: "text/javascript" 
    }
)
```

You can then use `URL.createObjectURL` to get a URI with the `blob:` pseudo-protocol. Again, being a pseudo-protocol, it doesn’t point to a location of a resource. In this case, the URI points to the dynamically allocated blob.

This URI looks something like this:

```js
`blob:http://localhost:1234/60e5ba14-5bd0-4333-bb19-7782bf17cf4a`
```

There are solid reasons to use these, though I have to admit they’re pretty weird.

For one, we can only construct them from JavaScript. This means attackers can't insert them using certain kinds of XSS attacks, making them somewhat more secure.

But they have some security risks too. They're sometimes used by attackers to obfuscate malicious scripts, since they’re harder to trace.

Here's some code that creates a script tag using this kind of URI:

```js
// Construct the blob
var blob = new Blob(
    ['alert(1)'], 
    { 
        type: "text/javascript" 
    }
)
// Generate a blob URI for it
var uri = URL.createObjectURL(blob)

// Create a new script tag
var script = document.createElement('script')

// Set its src to the URI
script.src = uri

// Insert it into the page
document.body.appendChild(script)
```

The resulting script tag looks like this:

```html
<script
    src="blob:http://localhost:1234/60e5ba14-5bd0-4333-bb19-7782bf17cf4a"
></script>
```

# Conclusion
Decades of development have given us a huge range of script tags of different types. There's a lot more I didn’t have the time to cover, of course, and I hope you’ll join me on future deep dives into the subject.
