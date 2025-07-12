---
title: Scalabilty is not performance
published: 2025-07-08
updated: 2025-07-08
---
Scalability is sometimes confused with performance, but it’s not at all the same thing.

In this article, we’ll explore the differences between them using a simple model!

---
$$
%! style=hidden
\newcommand{\term}[2]{\textcolor{#2}{\mathbf{\textrm{#1}}}}
\def\Job{\term{Job}{yellow}}
\def\Jobs{\term{Jobs}{yellow}}
\def\System{\term{System}{ProcessBlue}}
\def\Output{\term{Output}{Cyan}}
\def\Throughput{\term{Throughput}{green}}
\def\Latency{\term{Latency}{orange}}
\def\Cost{\term{Cost}{red}}
\def\dol{\term{\$}{white}}
\def\Efficiency{\term{Efficiency}{Apricot}}
\def\JobRate{\term{JobRate}{RedViolet}}
\def\Time{\term{Time}{Olive}}
\def\Utilization{\term{Utilization}{RoyalBlue}}
\def\Capacity{\term{Capacity}{Pink}}
\def\HttpNotModified{\textbf{304 Not Modified}}
\def\HttpBadGateway{\textbf{502 Bad Gateway}}
\def\Box{\term{Box}{PineGreen}}
\def\Boxes{\term{Boxes}{PineGreen}}
$$
We’ll start by answering a more fundamental question. It will allow us to determine the scope of our model.
# What is performance?
When people talk about the *performance* of a distributed system, they’re usually considering one of two metrics:

- $\Latency$, which is the time it takes the system to process a single job (on average).
- $\Throughput$, which is the number of jobs the system can process in a given period.

Low $\Latency$ automatically means having higher $\Throughput$. But in spite of that, software architecture typically focuses on increasing $\Throughput$ through other means.

This is because reducing $\Latency$ is something that can only be done on a per-component basis, it’s subject to severe diminishing returns, and there are inherent limits to how low you can go.

It’s much easier to increase $\Throughput$, even if it comes *at the expense* of $\Latency$. So that’s what our model is going to focus on.
# Boxes and jobs
This model involves just two entities — $\Boxes$ and $\Jobs$.

A $\Job$ stands for pretty much anything a server could be doing — like answering an HTTP request, processing a credit card transaction, or running a database query.

$\Jobs$ are processed by $\Boxes$. A $\Box$ could stand in for a VM, container, or maybe even a process inside a single machine.

The rules for how $\Boxes$ process jobs are very simple:

- Every $\Job$ takes a fixed amount of $\Time$ to complete. That is, $\Latency$ is fixed.
- Every $\Box$ is identical to any other. They all process $\Jobs$ at the same rate.
- Each $\Box$ can only process one job at a time.

None of these things is true of real-world jobs, but these simplifications are what makes the model easy to reason about.

And here it is so far, in all its glory:

```canva key=one-box-model ;; size=500x190 ;; alt=A single Box processing one job at a time
https://www.canva.com/design/DAGlFywvS4Y/G4OZ4DQv84f-a-Sp6hI8Xg/view?utm_content=DAGlFywvS4Y&utm_campaign=designshare&utm_medium=link2&utm_source=uniquelinks&utlId=h46b5637248
```

With a single $\Box$, our performance metrics are directly connected:

$$
%! style=big

\Throughput=\frac{1}{\Latency}
$$

But it’s hardly a distributed system if there’s only one $\Box$! Which is why we have lots of them:

```canva key=many-boxes ;; size=500x330 ;; alt=Many boxes processing jobs
https://www.canva.com/design/DAGlGFv94Po/kXO_HjoZH3TUX4r1GNmzJg/view?utm_content=DAGlGFv94Po&utm_campaign=designshare&utm_medium=link2&utm_source=uniquelinks&utlId=h478ac77bb9
```

We’ll call the number of these boxes $\Capacity$. In that case, the expression becomes:

$$
%! style=big

\Throughput=\Capacity\times\frac{1}{\Latency}

$$
While $\Latency$ is fixed, $\Capacity$ isn’t. We can change it however we want. To increase $\Throughput$, we just get more $\Boxes$!

Sadly, these $\Boxes$ aren’t free. They actually $\Cost$ money.

While in the real world, different boxes have different price tags, in our little model we just have one. And it comes at a fixed $\Cost$ per unit of $\Time$.
$$
%! style=big

\Cost \propto \Capacity\times\Time

$$
Since $\Throughput$ is proportional to $\Capacity$ times a constant factor, we can actually simplify the relation to just:
$$
%! style=big
\Cost \;\propto\; \Throughput
$$
# The JobRate
So we have this system and jobs are coming in. But *how fast are they going?*

Having a $\Throughput$ of, say, $100$ means we have the $\Capacity$ to handle $100$ jobs per second (or something). But are we getting $100$ jobs per second?

Maybe we’re getting $10$ or $10,000$. This is the $\JobRate$, and whatever it is, our system must be prepared to handle it.

The $\JobRate$ is not constant — it’s actually a function of time, or $\JobRate(\Time)$, and in almost all cases we can neither predict nor control it. It could be a flat line or it could fluctuate like crazy.

Here are some examples of how it might look like:

```canva key=example-job-rates ;; size=700x450 ;; alt=Shows four job curves with different peaks and valleys
https://www.canva.com/design/DAGlG86qqak/lKUMM69vLfs-Aqqc3pvnzQ/view?utm_content=DAGlG86qqak&utm_campaign=designshare&utm_medium=link2&utm_source=uniquelinks&utlId=he1cd796a6d
```

## Missed jobs
If $\Throughput\;<\;\JobRate$, there are jobs that aren’t being handled.

This usually means the jobs end up being queued somewhere (possibly multiple *somewhere*s). For example:

1. Your HTTP server.
2. The user’s web browser.
3. A message broker like Kafka.

As jobs get queued, one of the following happens:

- They get less and less relevant. Maybe they’re stock orders that become less valuable the longer we wait to fulfill them.

- Their worth is constant until the deadline, after which it drops to $0$, like HTTP transactions that time out.

- So many jobs get queued that they cause the messaging infrastructure to crash, leading to total system failure.

The specifics don’t actually matter for the purposes of this article — in almost all cases, not handling jobs is a Very Bad Thing$\texttrademark$ and somewhere we really, really don’t wanna be.

We’ll just consider it a fail state of the model.
# Utilization
But what about having more $\Throughput$ than we need? That is, a situation where:
$$
%! style=big

\Throughput\;>\;\JobRate{}

$$
That sounds nice, but since $\Cost$ is proportional to $\Throughput$, it basically means we’re paying for infrastructure that's going to waste.

We can get a figure for $\Utilization$ — how much of our system is actually being used to process these jobs, versus standing idle and racking up money.

It’s simply:

$$
%! style=big

\Utilization=\frac{\JobRate}{\Throughput}

$$

- $\Utilization<1$ is the scenario we’re just described. We have *more than enough* $\Capacity$ to handle jobs. We’re still paying for that extra $\Capacity$ though.

- $\Utilization=1$ means we’re right on the money — every cent we’re pumping into our system is translated into jobs that get processed. It’s the ideal scenario. It’s also dangerously close to failure.

- $\Utilization>1$ just means we’re not handling jobs, which is a Very Bad Thing$\texttrademark$ and something we’re not considering right now.

The theoretical maximum here is $1$, but having lower $\Utilization$ is almost always better than missing jobs, so you usually want to have some $\Capacity$ left over. That translates to having a $\Utilization$ that’s a little less than $1$, like $0.8$.

The exact amount of headroom you want is a hard question to answer and really depends on what you’re actually doing.

Which is why we’ll answer a different question instead. How would you want $\Utilization$ to *change over time* as the $\JobRate$ changes?

**The answer is that you don’t!**

If you can keep $\Utilization$ at $0.5$ whether the $\JobRate$ is $10$ or $10,000$, you’re playing it safe — half of your infrastructure is idling — but you’re still winning!

On the other hand, failure looks like $\Utilization$ going down into $0.01$ or above $1$. Both of those should be avoided.

Of course, if $\JobRate$ keeps changing, the only way to keep $\Utilization$ the same is changing your $\Throughput$ to match. $\Latency$ is fixed, so we have to change $\Capacity$ instead.

That’s *scaling*. Being able to do that is called *scalability*.
# The point of it all
The goal of scaling your system is to meet your target $\Utilization$. That doesn’t just mean getting more $\Throughput$ — it also means getting less of it as needed.

People tend to focus on the *getting more* part. I think that’s because it’s more glamorous, in the same way people tend to talk about climbing up mountains, even though climbing back down is often harder.

In any case, building a system like that is much harder than just getting high $\Throughput$. A real-world system is a heterogenous mass of different components that have very different scaling characteristics.

In some cases, it may not be obvious how some of these components can be scaled at all.

What’s more, as a system scales, its structure changes, and it becomes vulnerable to different sorts of issues. Problems that occur at one level of scale may not occur in another, making such issues hard to debug.

This kind of stuff happens whether you’re scaling up or down.
## The cloud
In fact, scalability is so difficult that few companies attempt to achieve it by themselves.

Instead, they pay huge cloud providers to solve some parts of these problems for them, even if it means also paying a pretty steep markup on actual processing power — and $\Throughput$.

The best example of all of this is the **serverless platform**.

One paper^[https://arxiv.org/pdf/1812.03651] found that, when compared to a virtual machine, serverless platforms cost $100$ times more for the same $\Throughput$, with $20$ times the $\Latency$.

That would be lunacy from a *performance* standpoint, but it makes perfect sense when considering *scalability* — your $\JobRate$ might vary by a factor of $100,000$ over a single day.

Compared to that, even a factor of $100$ might not be so scary, especially if it lets you avoid paying for so many engineers.
# Conclusion
In short, *scalability* is being able to change your system’s throughput based on demand.

Sometimes, people talk about *improving scalability* when they actually just mean making stuff run faster.

That’s important too, but serverless platforms (and the cloud computing model in general) are proof that you can have scalability without high performance, and that people will happily pay lots of money to have it.

I hope you’ll join me for future articles, in which we’ll use slightly more complicated models to look at more advanced qualities of distributed systems!
