---
title: What is an iframe? An overview
series: web
description: ""
published: 2025-01-01
updated: 2025-01-11
---

Iframes are omnipresent and extremely useful, yet they're also complicated and confusing.

Let’s take a broad look at how they work.

---

# But what is an iframe?
An iframe is a box with another webpage in it — a webpage within a webpage.

Some iframes are visible and obvious, used for social media components, payment portals, and embedded videos.

Others are completely invisible, created by third-party scripts for a variety of purposes.

Iframes are incredibly complicated, but for all that complexity, they appear as just another HTML tag. Here is an example of one:

```html
<iframe id="gregros" src="https://gregros.dev/iframe"></iframe>
```

Iframe tags are always empty and don’t reflect their contents. Like an `img` tag, their content comes from *elsewhere*. That elsewhere happens to be another webpage.

The webpage inside the iframe functions just like a normal webpage. It has its own CSS rules, JavaScript, and DOM. It can make its own requests from its own origin and use all standard web APIs.

JavaScript inside the iframe runs on the same thread as the main page, but in a separate copy of the web environment.

Some iframes are isolated from the pages they’re embedded in, while others aren’t. If an iframe isn’t isolated from its parent, the webpage inside it can run JavaScript that [[iframes-and-when-javascript-worlds-collide.post|directly interacts with the parent page]].

Meanwhile, the parent page can control the kinds of things code in the iframe can do. For example, it can prevent the iframe from using the microphone and limit its scripts to a certain domain.

# What do they look like?
Like most elements iframes appear as rectangular areas with stuff inside them. Unlike other elements, though, that stuff can’t affect their size, which is entirely determined by the styling rules applied by the parent page.

Iframes don’t overflow like normal elements. They’re more like a small browser window stuck inside a webpage.

The webpage can try to fill its boundaries, but it can’t make the window larger or smaller, and it can’t draw outside of it either.

Webpages inside iframes are also unaffected by most CSS rules in the parent page, just like a webpage isn’t directly affected by the user’s browser theme.

All of these qualities tend to make iframes stand out from the page they’re embedded in, which is often the point. Other times a lot of work has to go into making them blend in with everything.

# What’s inside them?
Each iframe has a complete DOM, which starts with having the base HTML structure of:

```html
<!DOCTYPE html>
<html>
    <head></head>
    <body></body>
</html>
```

Some iframes don’t contain anything else, except maybe a `<script>` tag or two. These iframes are frequently invisible, constructed by the main page, and used for their JavaScript environment.

Other iframes, in particular all visible ones, contain something that looks more like a normal webpage. They have scripts tags, stylesheets, and normal elements like `<div>` and `<input>`.

Some iframes also contain other iframes.

# Are they secure?
Iframes are generally regarded as secure and are a core component of the web as we know it. Payment portals, for instance, wouldn’t be possible without their involvement.

They don't by themselves solve all client-side security issues, but they're an important component of anything that tries to do so.

Iframe security is based on the browser’s same-origin policy, which means that resources from different websites shouldn’t be able to interact with each other.

This means that an iframe from another website — such as a payment provider — can’t be accessed from outside, has its own cookies and local storage, and runs isolated JavaScript. This security feature can also be enabled for iframes from the same site.

These security features are sometimes used by attackers to run malicious code out of sight of the main page.

# Anything to look out for?
Iframes aren’t a simple technology, and using them has both performance and development costs.

The performance cost comes from having the browser create this entirely new webpage inside of an existing webpage, increasing the page’s overall memory footprint.

Plus, since it runs on the same thread, anything it does can stall the main page, depriving it of resources.

There are also specific kinds of bugs that can only happen if you’re trying to run code in both an iframe and the main page. Doing so is quite tricky, and the browser actually lets you screw up your web environment in terrible ways.

# Use-cases
Let's take a broad look at the different ways iframes are used.

## Embeds
An embed is a component from one site that’s integrated into another site. An embed can be any kind of component, whether interactive or not, including:

- A video player
- A payment portal
- A social media feed
- An advertisement

While some embeds can be done without using iframes, the benefits to the embed provider are usually so great that they'll insist on using them, whether they’re needed or not.

This is why the Facebook Like button is an iframe. It doesn’t *have* to be one, but it gives enormous advantages to Facebook if it is.

While it’s tempting to try to categorize embeds into separate groups like *ads*, *content embeds*, *payment portals*, and so on, very few embeds actually do only a single thing.

There are clear-cut cases, but the same embed can also serve radically different purposes depending on who’s looking at it.

Users might see it as A, while the host puts it on their site for B, but the provider instead uses it for X, Y, Z.

Embeds are a really complicated topic, and I don’t feel I can give it justice here, so let’s leave it at that.

## Integration

A lot of webpages modify their global environment in ways that can be incompatible with third-party scripts.

One straight-forward way to overcome this issue is to avoid running code in the main page as much as possible. Instead, a third-party script can create a dedicated, invisible iframe, and run the main body of its code there, only executing as much code outside the iframe as absolutely necessary.

The idea here isn’t to use the security features of the iframe. In fact, this kind of solution relies on some degree on the ability to run code across its boundaries, which would be impossible if the security features were in play.

Instead, it relies on the fact the iframe has its own JavaScript environment that’s independent from its parent, and so doesn’t have any of those changes.

As I mentioned earlier, working with iframes, especially trying to run code across them, isn’t easy. However, other solutions tend to make the entire software development lifecycle more expensive.

There are alternative solutions to iframes that do something similar, such as web workers, but whether they’re applicable depends on what the script is trying to do and what APIs it needs access to.

## Security zones

While the security features of iframes normally only work between different sites, it’s possible to enable them for the same site too.

This can be used to isolate different areas of the same webpage from each other. Each area can have its own codebase and security standards, and some areas can be used to run less-trusted code in a kind of sandbox, out of which said code shouldn’t be able to escape from.

Another variant on the security zone architecture is used by some web development platforms, where a single webpage is going to contain content from both the platform as a whole, as well as content from specific web application. That can include custom JavaScript.

Such platforms frequently make use of iframes to separate the two, so that a user’s web application can’t be used as an attack vector for the platform as a whole.

Other times, iframes are used for browser-based development platforms in order to execute potentially untrusted code in a closed-off environment.

# Conclusion

In this article, I’ve tried to give a general, top-down view of the benefits of iframes, how they work, and why you might want to use one.

However, iframes aren’t like a function or event a development technique — they’re a major piece of client-side architecture, one that can be as complicated as the entire rest of the webpage.

I hope that you’ll join me on future deep dives into this topic, as I feel it’s the only way to do it justice.
