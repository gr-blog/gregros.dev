---
title: Debug webpages with code using the inspector's internal API
published: 2025-07-08
updated: 2025-07-08
description: ...
---
Using the right approach, we can get past the Chrome inspector’s UI and invoke its internal API directly.

This lets us debug live webpages using JavaScript instead of buttons!

---
# How it works
The Chrome inspector is actually just a webpage hosted locally inside the browser. If we could interact with its code using a console, we could access the API behind its interface.

The inspector’s own console can’t do that, since it’s just a fancy textbox that’s part of the webpage. It can run commands on the *end-page* it’s inspecting, but that’s about it.

That means all we need to do is *inspect* the inspector using another instance of itself!

I call this technique *meta-inspection*, and here is how it works:
```canva key=meta-inspection ;; size=570x370 ;; alt=Illustrates the end-page, an inspector, and the meta-inspector
https://www.canva.com/design/DAGb7Bi_Ezk/vgeH-J7-CSHwTsrEz28HpQ/view?utm_content=DAGb7Bi_Ezk&utm_campaign=designshare&utm_medium=link2&utm_source=uniquelinks&utlId=hb2b009fec8
```

- First, press `Ctrl + Shift + I` on the page you want to *meta-inspect*.
- Detach the inspector into its own window.
- Press `Ctrl + Shift + I` again on the inspector window, opening up a second inspector. We’ll call it the *meta-inspector!*
- Now you can reattach the first inspector, but don’t close it.

Now, just like with any other webpage, we can use the *meta-inspector* to do all kinds of things:

- Examine the inspector’s UI
- Put breakpoints in its code
- And if we get references to the right objects, invoke its internal API from the console!

That internal API contains all the information about the *end-page* — the webpage we actually want to debug — and it’s all in the form of juicy JavaScript objects.
## But why?
**The power. Why else?**

This technique lets you automate stuff that you could only do using the UI before. That opens up a world of possibilities:

- You can automate breakpoints
- Perform complex searches on live network data
- And do lots of other stuff!

Now, as an internal API, a lot of the code I’m going to show you might break in the future.

But the point of this article is the technique itself, not the specific code I’m going to use. The code is just an example of what’s possible!

I’ll show you exactly how I figured it out, so you can do the same if it breaks. 

That said, I want to keep the examples working, so if they break do [send me a line](mailto:gregros@gregros.dev) and I’ll fix them!
# Inspector architecture
The inspector itself is written in TypeScript (surprise!), but you’ll only see the compiled JavaScript.

It might also be bundled and minified to some extent, which makes it a little hard to work with. 

You can view the source directly at the [project repo](https://github.com/ChromeDevTools/devtools-frontend/tree/main), though, and I’m going to link to it quite frequently in this post.
## Figuring stuff out
The code doesn’t have much documentation. It’s well-organized, but if you want to figure out how it works, your best bet is to start with the UI.

The inspector UI is divided into individual Panels, such as the [NetworkPanel](https://github.com/ChromeDevTools/devtools-frontend/blob/main/front_end/panels/network/NetworkPanel.ts), the [ConsolePanel](https://github.com/ChromeDevTools/devtools-frontend/blob/main/front_end/panels/console/ConsolePanel.ts), and so forth. Most display elements that can be dragged, toggled, or resized are Panels of some sort.

Panels can contain other panels, as well as other components called Views.

With that in mind, let’s say you want to figure out how to do X using JavaScript. Your workflow is going to look something like this:

1. Go to the source code of the Panel or View that’s related to X
3. Figure out the API it uses to get data
4. Follow that API to the correct lower-level component

So when I was trying to figure out how to get the network data, I first went to the `NetworkPanel` and quickly saw it uses the [NetworkLog](https://github.com/ChromeDevTools/devtools-frontend/blob/main/front_end/models/logs/NetworkLog.ts) to get most of its information. 

Whenever I got lost, I just went to the specific UI related to the thing I wanted. 

For example, when I had trouble figuring out where to get the request payload, I went to the [RequestPayloadView](https://github.com/ChromeDevTools/devtools-frontend/blob/ed19c1e8985293025be2e812b86ea7619185fcfd/front_end/panels/network/RequestPayloadView.ts#L119).
## Quick tip
You might notice that the *meta-inspector* doesn’t refresh, even if you refresh the end-page being inspected or navigate it somewhere else.

On one hand, it’s quite convenient, since any functions and variables you defined previously will still be available.

But it also means the *meta-inspector*’s memory is going to fill up with data from lots of different pages, which can lead to unbounded memory usage. 

You should be careful to refresh it every so often with a quick `Ctrl + R`.
## Importing stuff
The inspector uses ES modules, and we’ll need to dynamically import them in the *meta-inspector*’s console if we want to use its code. 

While the modules themselves don’t change often, their import paths can change a lot, depending on the Chrome version and how it was built. 

For example, the `logs.js` module might be imported using one of the following paths:
  
```
./models/logs/logs.js
./devtools-frontend/front_end/models/logs/logs.js
```

Luckily, there is a pretty stable API that lets us import modules using the same path. Here is how it works:

```js
var Logs = await runtime.loadLegacyModule(
    "models/logs/logs.js"
)
```

Internally, it just does a dynamic import from one of the inspector’s script files. Nothing fancy. But convenient!
## Object architecture
The inspector API largely consists of singleton classes. These classes mostly follow the same structure, which makes them easy to work with. 

For example, we can access the [NetworkLog](https://github.com/ChromeDevTools/devtools-frontend/blob/main/front_end/models/logs/NetworkLog.ts) instance from the `Logs` module we imported earlier using:

```js
var Logs = await runtime.loadLegacyModule(
    "models/logs/logs.js"
)
var NetworkLog = Logs.NetworkLog.NetworkLog.instance()
```

`Logs` is a module with an export `NetworkLog` that also happens to be a module, finally exporting the `NetworkLog` class.

Here is a quick breakdown of the whole thing:
```canva size=525x340 ;; key=network-log-pattern ;; alt=Breaking down the path
https://www.canva.com/design/DAGb1HApUCg/RoKVEOP8aBwFgYfxUbn7tQ/view?utm_content=DAGb1HApUCg&utm_campaign=designshare&utm_medium=link2&utm_source=uniquelinks&utlId=ha093f46606
```
# Processing network data
Let’s take a look at the first use case of this technique — filtering and processing network data using JavaScript.

We do this using the `NetworkLog` I showed in the last section — it’s actually the source of all the data in the Network panel. To access its data, we just need to call:

```js
NetworkLog.requests()
```

Which returns an array of [NetworkRequest](https://github.com/ChromeDevTools/devtools-frontend/blob/main/front_end/core/sdk/NetworkRequest.ts) objects. These are mutable objects that get updated in real-time as network data arrives. 

These `NetworkRequest` objects can also represent non-HTTP requests, blocked requests, and things that aren’t really requests at all, like resolved data URIs. 

It can be helpful to exclude these using the methods [`isHttpFamily`](https://github.com/ChromeDevTools/devtools-frontend/blob/ed19c1e8985293025be2e812b86ea7619185fcfd/front_end/core/sdk/NetworkRequest.ts#L1435) and [`wasBlocked`](https://github.com/ChromeDevTools/devtools-frontend/blob/ed19c1e8985293025be2e812b86ea7619185fcfd/front_end/core/sdk/NetworkRequest.ts#L725) 

```js
NetworkLog.requests().filter(x => 
    x.isHttpFamily() && 
    !x.wasBlocked()
)
```

Let’s take a look at some code examples!
## Traffic volume by host
If you’re looking at a complicated web application with lots of dependencies, each making tons of different requests — you might want to know where most of the traffic is coming from.

Using the *meta-inspector*, you can figure it out using JavaScript. We just group the log entries by [`domain`](https://github.com/ChromeDevTools/devtools-frontend/blob/4590d3a54ca7023ca9f61f0dc46f2d821401c118/front_end/core/sdk/NetworkRequest.ts#L911) and sum by [`resourceSize`](https://github.com/ChromeDevTools/devtools-frontend/blob/ed19c1e8985293025be2e812b86ea7619185fcfd/front_end/core/sdk/NetworkRequest.ts#L649).

```js
var trafficByHost = Object.create(null)
for (var rq of NetworkLog.requests()) {
    let currentTraffic = trafficByHost[rq.domain ?? ""] ?? 0
    currentTraffic += rq.resourceSize ?? 0
    trafficByHost[rq.domain] = currentTraffic
}
trafficByHost
```
## Security reports
TLS 1.2 is widely considered to be obsolete, but it’s still being used on the web in some cases. 

We can use the `NetworkLog` to summarize the security protocols used by each request and find any that use TLS 1.2.

We do this using the [`securityDetails()`](https://github.com/ChromeDevTools/devtools-frontend/blob/ed19c1e8985293025be2e812b86ea7619185fcfd/front_end/core/sdk/NetworkRequest.ts#L649) method, which returns a raw  [`SecurityDetails`](https://chromedevtools.github.io/devtools-protocol/tot/Network/#type-SecurityDetails) object.

This object gives lots of info about the security protocols, ciphers, key exchanges, and certificates used by each request.

Here is the code:

```js
var reqList = NetworkLog.requests()
    .map(rq => {
        return { // simplify objects:
            url: rq.url(),
            security: rq.securityDetails()?.protocol ?? ""
        }
    })

// Grouping requests by security protocol:
Object.groupBy(reqList, x =>
    x.security
)
```

I looked around, and I quickly found some webpages using TLS 1.2 using this technique:
```canva key=internal-api-tls-1.2 ;; size=500x190 ;; alt=Examples of requests using TLS 1.2
https://www.canva.com/design/DAGb7Dkn3BU/8HS2t1_K3sM0K3lk2vcxMg/view?utm_content=DAGb7Dkn3BU&utm_campaign=designshare&utm_medium=link2&utm_source=uniquelinks&utlId=hb839f9a980
```
## Searching in request bodies
While we can search for stuff in response bodies using the inspector UI, that doesn’t work for searching inside *request* bodies. But using this technique, we can do it with code!

Getting the request payload is actually an async operation, so our code is going to be a bit more complicated than the previous examples.

We’ll use the [`requestFormData`](https://github.com/ChromeDevTools/devtools-frontend/blob/4590d3a54ca7023ca9f61f0dc46f2d821401c118/front_end/core/sdk/NetworkRequest.ts#L988) method to retrieve the body of a request. It doesn’t just work for form-encoded data. It just returns it as a string.

```js
var searchPromises = NetworkLog.requests()
    .filter(x => x.requestMethod == "POST")
    .map(async x => {
        let payload = await x.requestFormData()
        // skip no payload:
        if (!payload) { 
            return false
        }
        // search in the contents:
        if (!payload.includes("zionSp")) {
            return false
        }
        // if pass, return simplified object:
        return {
            url: x.url(),
            body: payload
        }
    }
    )

// await all and filter out false values:
    (await Promise.all(searchPromises)).filter(x => x)
```
## Other stuff
There is a lot more you can do with this network data. For example:

- Search for specific headers or header combinations.
- Analyze network timing across multiple requests.
- Search for all requests with a specific script in the initiator chain.
- Search for strings in responses from specific hosts

# Conclusion
Accessing the inspector’s internal API is a little complicated, but it’s a powerful debugging technique. If used correctly, it can literally save you hours of work.

In this post, I’ve mainly tackled how it can be used for processing network data. In the future, I’ll tackle advanced DOM searches, automating breakpoints, and more!

[I’d love to know](mailto:gregros@gregros.dev) what you think I should tackle next.

Good luck, and have fun!