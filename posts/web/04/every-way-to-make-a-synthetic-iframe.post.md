---
title: "Every way to make a synthetic iframe: an overview"
description: ""
published: 2025-01-01
updated: 2025-01-11
---
A synthetic iframe is one that doesn’t have web address. It's filled by its parent page instead.

Let’s take a look at every single way you can create one.

---
I define an iframe as *synthetic* if:

1. It has an `src` attribute.
2. That points to a web address.

We can visualize it using a decision tree:

```canva size=540x420 ;; key=synthetic-iframe-decision-tree ;; alt=Synthetic iframe decision tree
https://www.canva.com/design/DAGby_r5FO0/_WUVxhryforZ-iF7yI8ppA/view?utm_content=DAGby_r5FO0&utm_campaign=designshare&utm_medium=link2&utm_source=uniquelinks&utlId=h64c24c39ab
```

Synthetic iframes can have an `src` attribute, but one that contains a non-web address. They might also have no attributes at all.

Synthetic iframes are a bit strange, but they’re also simpler than non-synthetic ones. That’s because there are way fewer factors that can change their behavior.

You can also figure them out just by looking at the tag itself.

Meanwhile, non-synthetic iframes always make an HTTP/S request, and that request can have headers that change how the page functions. That includes the Content Security Policy (CSP), but also the Permissions Policy and other stuff too.

When a page builds a synthetic iframe, it also determines everything that's in it. That's why they can only provide security in one direction – the page is protected from the iframe, but not the other way around.

Let’s take a look at every method for constructing a synthetic iframe and compare their advantages.
# srcdoc attribute
A straight-forward method that probably resolves 90% of use-cases.

We can use the `srcdoc` attribute to set the contents of a synthetic iframe verbatim. We can do that as part of the iframe's HTML or using JavaScript.

Note that we can set the attribute before we attach the frame to the page.

Here, we're setting it via HTML:

```html
<!-- srcdoc attribute via HTML -->
<iframe id="myFrame" srcdoc="<h1>hello world</h1>"></iframe>
```

And here we set it via JavaScript:

```js
// Create the iframe 
var iframe = document.createElement("iframe")

// Set the srcdoc attribute
iframe.srcdoc = "<h1>hello world</h1>"

// Attach it to the page
document.body.appendChild(iframe)
```

A srcdoc iframe doesn’t have an origin. That means its origin defaults to the page that created it, which in turn means it’s not isolated from that page by default.

- Content specified verbatim
- Can be inserted using JavaScript or be part of the HTML page itself
- Loads synchronously
- Not isolated from the parent page.

# Empty – no src or srcdoc
An iframe with neither an `src` nor `srcdoc` attributes starts out empty. Its content is manually constructed using DOM operations and JavaScript, via the iframe element’s `contentWindow` property.

The iframe generates its `contentWindow` property when it's attached to the page. Before this happens, the property is empty.

You can use pretty much any method you want, but if you use DOM objects, make sure to use the iframe’s `createElement` function. (-- ref iframes)

```js
// Create the iframe
var iframe = document.createElement("iframe")

// Attach it to the page
document.body.appendChild(iframe)

// Now we can access its document
var doc = iframe.contentWindow.document

// Create an h1 element
// Remember to use the iframe’s createElement function
var iframeH1 = doc.createElement("h1")

// Set its text content
iframeH1.textContent = "hello world"

// Append it to the body
doc.body.appendChild(iframeH1)
```

- Constructed using JavaScript.
- Everything DOM update is applies synchronously.
- Not isolated from the parent page.

# Data URI
Another kind of iframe uses the `src` attribute, but with a data URI in it. That's very similar to the `srcdoc` attribute, in that it allows us to specify the iframe’s contents verbatim in the attribute.

But using a data URI is more flexible, since it allows different encodings. We can just use text:

```html
<iframe src="data:text/html,<h1>hello world</h1>"></iframe>
```

But we can also use base64:

```html
<iframe src="data:text/html;base64,PGgxPmhlbGxvIHdvcmxkPC9oMT4="></iframe>
```

An iframe with a data URI has an “opaque” origin, which means it’s isolated from everything. It doesn’t have any persistent storage and can’t access the parent page.

This restriction also goes the other way – the parent page can’t access it either – but this doesn’t protect the iframe since the parent page created it in the first place.

We can achieve something similar using the `sandbox` attribute, but the parent can remove that attribute at any time, whereas this method provides more thorough protection.

- Inserted using JavaScript or part of the page.
- We can specify content verbatim using any encoding we choose.
- Securely isolated from the parent page.
- Loads after an asynchronous delay.

# Blob URI
This kind of iframe can only be created using JavaScript. It uses a Blob URI, a really weird mechanism for generating a URI pointing to a dynamically allocated binary object.

To create one, we need to create a Blob object with the contents we want the iframe to have:

```js
var blob = new Blob(["<h1>hello world</h1>"], { type: "text/html" })
```

Then we need to create a URI for it:

```js
var uri = URL.createObjectURL(blob)
```

And finally, we can set the iframe’s `src` attribute to that URL:

```js
var iframe = document.createElement("iframe")
iframe.src = uri
document.body.appendChild(iframe)
```

The result looks something like this:

```html
<iframe src="blob:http://localhost/703363e4-f8a8-401f-90d7-b258264b4d60"></iframe>
```

Unlike data URIs, iframes constructed like this actually do inherit the parent’s origin.

- Constructed using JavaScript
- Content specified verbatim as part of the blob
- Loads after an asynchronous delay
- Not isolated from the parent page

# About URI
Using an `src` attribute with the `about:` pseudo-protocol doesn’t actually construct an iframe. It sets its address without specifying what’s in it. Kind of like giving a nickname to a pet.

You can combine this with the [[#srcdoc attribute]] to set the frame’s location to something meaningful. Can be helpful when debugging.
# Conclusion
Synthetic iframes are simpler than non-synthetic ones, but there are lots of different ways to create them, each with its own sets of benefits and drawbacks.

I’m pretty sure I covered literally every single method in this article. Please [contact me](mailto:gregros@gregros.dev) if I missed anything!
