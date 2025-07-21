---
title: Scalability is not performance
published: "2025-07-16"
updated: "2025-07-20"
figure: example-job-rates
---
%%%?
Scalability is sometimes confused with performance, but it’s not at all the same thing.

In this post, we’ll take a closer look at what both of those things mean, with the help of a simple model!
%%
$$
%! style=hidden
\def\term#1#2{\textcolor{#2}{\mathbf{\textrm{#1}}}}
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
Scalability is sometimes confused with performance, but it’s not at all the same thing.

To show that, we’re going to build a simple model. But before we start, we need to answer a more basic question. It’ll allow us to decide the scope of our model.
# What is performance?
When people talk about the *performance* of a distributed system, they’re usually considering one of two metrics:

- $\Latency$, which is the time it takes the system to process a single job (on average).
- $\Throughput$, which is the number of jobs the system can process in a given period.

Lower $\Latency$ automatically raises $\Throughput$. But in spite of that, distributed systems generally ignore it.

That's because reducing $\Latency$ is very hard. There's no surefire way of doing it and it suffers from severe diminishing returns.

It’s much easier to increase $\Throughput$, even if it comes *at the expense* of $\Latency$. That’s how modern distributed systems work, and what we’re going to focus on.
# Boxes and jobs
This model involves just two entities – $\Boxes$ and $\Jobs$.

A $\Job$ stands for anything a server could be doing – like answering an HTTP request, processing a credit card transaction, or running a database query.

$\Jobs$ get handled by $\Boxes$. A $\Box$ could stand in for a VM, container, or maybe even a process inside a single machine.

The rules for how $\Boxes$ handle jobs are very simple:

- Every $\Job$ takes a fixed amount of $\Time$ to complete. This is the $\Latency$.
- Every $\Box$ handles $\Jobs$ at the same rate.
- Every $\Box$ can only process one job at a time.

None of these things is true of real-world jobs, but these simplifications are what makes the model easy to reason about.

## A single box
Here is the bare-bones version:

```canva key=one-box-model ;; size=500x190 ;; alt=A single Box processing one job at a time
https://www.canva.com/design/DAGlFywvS4Y/G4OZ4DQv84f-a-Sp6hI8Xg/view?utm_content=DAGlFywvS4Y&utm_campaign=designshare&utm_medium=link2&utm_source=uniquelinks&utlId=h46b5637248
```

With a single $\Box$, our performance metrics are directly connected:

$$
%! style=big

\Throughput=\frac{1}{\Latency}
$$
## Lots of boxes
But it’s hardly a distributed system if there’s only one $\Box$. Which is why we have lots of them:

```canva key=many-boxes ;; size=500x330 ;; alt=Many boxes processing jobs
https://www.canva.com/design/DAGlGFv94Po/kXO_HjoZH3TUX4r1GNmzJg/view?utm_content=DAGlGFv94Po&utm_campaign=designshare&utm_medium=link2&utm_source=uniquelinks&utlId=h478ac77bb9
```

Where do these $\Boxes$ come from, though? The rules are really simple:

- We can create $\Boxes$ instantly.
- When a $\Box$ appears, it immediately starts accepting jobs.
- We can also remove $\Boxes$.
- But if a $\Box$ is processing a $\Job$, we first wait for that $\Job$ to complete.

We’ll call the number of these boxes $\Capacity$. In that case, our expression becomes:

$$
%! style=big

\Throughput=\Capacity\times\frac{1}{\Latency}

$$
While $\Latency$ is constant, $\Capacity$ isn’t. We can change it in any way we want. To increase $\Throughput$, we just get more $\Boxes$!
## No such thing as a free lunch
Sadly, these $\Boxes$ aren’t free. They actually $\Cost$ money.

While in the real world, different boxes have different price tags, in our model they have a fixed $\Cost$ per unit of $\Time$. We'll pick a cost of $1$ for simplicity, giving us:
$$
%! style=big

\Cost = \Capacity\times\Time

$$
Combining with the last equation, we find that $\Throughput$ and $\Cost$ are directly correlated:
$$
%! style=big
\Cost = \left(\Throughput\times \Latency\right)\times\Time
$$
This tells us that if we want to handle more jobs, we need to spend more money.
# The JobRate
So we have this system and jobs are coming in. But *how fast are they going?*

Having a $\Throughput$ of, say, $100$ means we have the $\Capacity$ to handle $100$ jobs per second. But are we getting $100$ jobs per second?

Maybe we’re getting $10$ or $10,000$. That's the $\JobRate$, and whatever it is, our system needs to handle it.

The $\JobRate$ isn't constant – it’s actually a function of time, or $\JobRate(\Time)$, and in almost all cases we can neither predict nor control it. It could be a flat line or it could fluctuate like crazy.

Here are some potential $\JobRate$ curves:

```canva key=example-job-rates ;; size=700x450 ;; alt=Shows four job curves with different peaks and valleys
https://www.canva.com/design/DAGlG86qqak/lKUMM69vLfs-Aqqc3pvnzQ/view?utm_content=DAGlG86qqak&utm_campaign=designshare&utm_medium=link2&utm_source=uniquelinks&utlId=he1cd796a6d
```

## Missed jobs
If $\Throughput<\JobRate$, we're not handling some of the jobs.

In the real world, that's inevitable. But in our model, any $\Job$ we're not handling has nowhere to go.

It becomes a *missed job*, which is a Very Bad Thing$\texttrademark$ in any system. We'll just say it's a *fail state* of the model.
# Utilization
But what about having more $\Throughput$ than we need? That is, a situation where:
$$
%! style=big

\Throughput > \JobRate{}

$$
That sounds nice, but since $\Cost$ is proportional to $\Throughput$, we’re paying for hardware that's going to waste.

We can get a figure for $\Utilization$ – how much of our system is actually used to process these jobs, versus standing idle and racking up money.

It’s simply:

$$
%! style=big

\Utilization=\frac{\JobRate}{\Throughput}

$$

- $\Utilization<1$ is the situation we’ve just described. We have *more than enough* $\Capacity$ to handle jobs. We’re still paying for that extra $\Capacity$, though.

- $\Utilization=1$ means we’re right on the money. We're translating every cent we’re pumping into our system into processing jobs.

- $\Utilization>1$ just means we’re not handling jobs, which is the Very Bad Thing$\texttrademark$.

In our model, we just want to have a $\Utilization$ of $1$. But in the real world, we really don’t want to miss any jobs, so it’s better to have some $\Capacity$ left over.

That translates to having a $\Utilization$ that’s a bit less than $1$, like $0.8$.

The exact amount of headroom we want is outside the scope of our model, and really depends on what we’re actually doing.

Which is why we’ll answer a different question instead. How would we want $\Utilization$ to *change over time* as the $\JobRate$ changes?
## We don’t want it to change
If we can keep $\Utilization$ at $0.5$ whether the $\JobRate$ is $10$ or $10,000$, we're playing it safe – half of our infrastructure is idling – but we're still winning!

On the other hand, failure looks like $\Utilization$ going down into $0.01$ or above $1$. Both are situations we want to avoid.

Of course, if $\JobRate$ keeps changing, the only way to keep $\Utilization$ the same is changing our $\Throughput$ to match. We're fixing $\Latency$ in place, so we have to change $\Capacity$ instead.

In other words, we need the ability to create and remove $\Boxes$ based on the amount of work we need to do.

That’s what it means to *scale*. Being able to do that is called *scalability*.
# Why systems scale
The goal of scaling our system is to meet our target $\Utilization$. That doesn’t just mean getting more $\Throughput$ – it also means getting less of it as needed.

A real-world system is a mess of different components that behave and scale very differently. Building a system that can grow and shrink on demand is both harder and more important than raw performance.
## Buying scalability
In fact, scalability is so difficult that few companies try to achieve it by themselves.

Instead, they pay huge cloud providers to solve some parts of these problems for them, even if they have to pay a pretty steep markup on hardware – and $\Throughput$.

The **serverless platform** is the best example of this.

One paper^[https://arxiv.org/pdf/1812.03651] found that, when compared to a virtual machine, serverless platforms cost $100$ times more for the same $\Throughput$.

That would be lunacy from a *performance* standpoint, but it makes perfect sense when considering *scalability* – our $\JobRate$ might vary by a factor of $100,000$ over a single day.

Compared to that, even a factor of $100$ might not be so scary, especially if it lets us reduce costs in other ways.
# Conclusion
*Scalability* is being able to change our system’s throughput based on demand, while *performance* typically refers to that throughput.

Sometimes, people talk about *improving scalability* when they just mean *making stuff run faster*.

That’s important too, but serverless platforms are proof that many people care more about scalability than performance.

Stay tuned for future posts, where we’ll dive into more advanced models of distributed systems.
