---
title: Scalability is not performance
published: 2025-10-16
updated: 2026-10-18
figure: example-job-rates
---
%%%?
Scalability is sometimes confused with performance, but it’s not at all the same thing.

So what is it?
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
\def\Demand{\term{Demand}{RedViolet}}
\def\Time{\term{Time}{Olive}}
\def\Utilization{\term{Utilization}{RoyalBlue}}
\def\Capacity{\term{Capacity}{Pink}}
\def\HttpNotModified{\textbf{304 Not Modified}}
\def\HttpBadGateway{\textbf{502 Bad Gateway}}
\def\Box{\term{Box}{PineGreen}}
\def\Boxes{\term{Boxes}{PineGreen}}
$$
Scalability in distributed systems is the ability to vary performance according to demand. 

Having high performance is nice, but it’s more important that this performance can change over time. In fact, we often willingly sacrifice performance in pursuit of scalability.
# The broad definition
This article is specifically about *scalability* as it’s defined in the study of distributed systems. If your knee-jerk reaction when reading this article was *that’s not scalability!* You’re probably thinking of one of these other definitions.

In the broad sense, scalability happens when we have $X$ and $Y$, and we can change $X$ such that $Y$ changes proportionally. In some contexts, we don’t really care about $X$ dropping, so we only look at what happens when it gets larger.

This definition is too vague to be useful. For the sake of clarity, I’d like to list some of these other meanings, so you’ll know what I’m *not* talking about.
## Making bigger
Some components, like databases, can’t always scale in the way that’s described here.

Scaling such components is typically a hands-on process that takes developer hours. It’s an effectively permanent change that increases the component’s $\Capacity$ while making it more expensive to operate.

This same definition also applies to physical hardware. We can scale a data center by getting more machines, but once again, the bigger data center becomes more expensive to own and operate.

The data center itself might be called scalable if it can accommodate a varying number of machines efficiently. 

Even though the number of machines changes, things like the network and cooling infrastructures might stay the same.
## Algorithmic complexity
Algorithms (or more generally workloads) are sometimes called scalable when they perform well for large inputs.

For example, the [bubble sort](https://en.wikipedia.org/wiki/Bubble_sort) algorithm doesn’t scale well, because it achieves very high $\Latency$ as the input gets large. Its time complexity is simply too high.
## Practical scalability
Other workloads aren’t scalable because they’re inherently inefficient, without regard to input sizes. Perhaps they consume too much memory or require too many cores, more than is reasonable for their specific task.

We can run such inefficient workloads when $\Demand$ is low and simply soak up the cost, since it can be small when compared to the rest or our system. But if $\Demand$ rises and we need more $\Capacity$, the cost becomes a problem.

Here, scaling the workload in a distributed system is still possible. It’s just not practical in the real world. We’re better off modifying the workload before trying to scale it.

This is somewhat of a slippery concept, though, since some workloads are just performance-intensive and no better solution exists.
## Other domains
The word *scalability* appears in other domains, including business, logistics, defense, and so on. Its meaning, while following the broad $X$ and $Y$ definition outlined earlier, is specific to those fields.
# Distributed systems
Let’s get back to distributed systems. Such systems are typically modelled with $\Jobs$ and $\Boxes$.

- A $\Job$ is a computing task that needs to be completed by some workload.
- A $\Box$ is the thing processing jobs, like a container, VM, or a guy in a room.

In such systems, performance typically refers to one of two metrics:

- $\Latency$ — the amount of time it takes a $\Box$ to complete a single $\Job$.
- $\Throughput$ — the number of $\Jobs$ all the $\Boxes$ can complete over time.

While reducing $\Latency$ increases $\Throughput$, doing so is very hard. It primarily happens at the level of the workload itself and there is a limit (sometimes a physical one) to how low it can go.
 
Instead, we increase $\Throughput$ by raising $\Capacity$, the number of $\Jobs$ are system can process at a time. This often comes *at the expense* of $\Latency$. 
## So we don’t care about latency?
We certainly do.

However, from the perspective of a distributed system, $\Latency$ is less a performance metric and more a *Quality of Service (QoS)* metric. If the $\Latency$ is too high, our system doesn’t have a performance problem — it’s just broken. 

Consider a distributed system that takes 24 hours to process a single $\Job$. In many situations, by the time that $\Job$ finishes, its results are worthless. 
# The biggest numbers
The lowest $\Latency$ and highest $\Throughput$ is found within the confines of a single machine, below the level of distributed systems.

Consider a single hardware node at a major cloud provider. An extreme example of such a node might involve:

- **8x 64-core Intel Xeon 6768P CPUs** — totaling **512 cores**, each clocking at a maximum of **3.9Ghz**.
- **4x 8x 128GB DDR5 8-channel** DIMMs, summing up to **8 TiB of RAM** clocking in at **3200Mhz**.
- NVIDIA ConnectX-6 NIC, providing a full duplex **100 GB/s**.

Now consider running a single Rust binary directly on such a node. No virtualization, no containers, nothing. Terabytes of memory space, a thousand hardware threads, and a latency lower than a virtualized service can dream of.

Such a binary would provide $\Throughput$ that rivals a distributed system. After all, this same node could instead host over 100 VM instances doing exactly the same work — and it doesn’t have to contend with virtualization overhead.

But do we really need all the power behind that specific workload at all times? Do we actually have 8 TiB worth of stuff to occupy that memory? If not… that hardware is crazy expensive both to purchase and operate.

Are we sure all this *performance* is what we’re looking for?
# Changing demand
So $\Throughput$ is about completing $\Jobs$. But these $\Jobs$ come from somewhere, and there is only so many of them.

How many? Well, it depends. What time is it?

Many real world systems don’t know how many $\Jobs$ they’ll need to handle ahead of time. A lot of them are driven by customer demand, which typically fluctuates across the span of hours, days, and months. 

This $\Demand$ is thus a function of $\Time$, and can be plotted as a demand curve. Here are some potential examples:

```canva key=example-job-rates ;; size=700x450 ;; alt=Shows four job curves with different peaks and valleys
https://www.canva.com/design/DAGlG86qqak/lKUMM69vLfs-Aqqc3pvnzQ/view?utm_content=DAGlG86qqak&utm_campaign=designshare&utm_medium=link2&utm_source=uniquelinks&utlId=he1cd796a6d
```
Scalability is not about having the most potential $\Throughput$ or highest $\Capacity$. If it were, we’d all be running Rust on massive hardware nodes. 

Rather, it’s about having *just the right amount*. Being able to fulfill the current $\Demand$ while wasting as little money as possible, whatever that demand might be.

A lot of times we focus on growing $\Demand$ and scaling $\Capacity$ upwards, because that’s where naive systems break down. But distributed systems have to scale down too, and doing so presents it own set of challenges that are nearly as hard to solve.
# How we scale
As mentioned earlier, we typically scale by changing $\Capacity$, the number of $\Jobs$ our $\Boxes$ process at the same time.

We can increase $\Capacity$ by raising the number of $\Boxes$ or using bigger ones, which are able to handle more jobs. We can drop $\Capacity$ by doing the opposite.

That might sound easy, but it’s actually quite hard. The mechanism responsible for this process is called the *scaling solution*. It consists of several things:

- **The metrics** — Measurements of current $\Demand$, $\Throughput$, $\Latency$, and other numbers.
- **The policy** — The function that maps a target $\Capacity$ to a specific $\Box$ configuration.
- **The autoscaler** — The thing that implements the policy and handles changes in the $\Box$ configuration.
- **The provisioner** — The thing that starts $\Boxes$ up and makes sure they’re running.
- **The deprovisioner** — The thing that kills $\Boxes$ while making sure the system keeps working properly.

Let’s take a look at some of the problems scaling tries to solve.
## Point of failure
Since scaling solutions are quite complicated, they add multiple points of failure to the system.

For example, sometimes, a $\Box$ will just refuse to start. It will go through the motions and simply get stuck. While this sometimes points to consistent bug in our infrastructure, other times *it just happens*.

A scaling solution needs to keep track of whether the $\Boxes$ it ordered actually started up or not. Since a typical $\Box$ is actually a stack, we sometimes need to check several times.

More generally, scaling solutions must be self-healing and fault-tolerant, capable of handling the myriad ways in which scaling can go wrong.
## Scaling delay
There is a significant delay between rising $\Demand$ and a matching increase in $\Capacity$. 

The first problem is our inability to see the future. In fact, a scaling solution can’t even see the present. We can only scale based on metrics, and those must have been computed in the past.

The second problem is the startup time. A $\Box$ doesn’t appear instantaneously and just start accepting jobs. The startup time varies between $\Boxes$, but ideally it should be as low as possible.

Overcoming scaling delay typically requires trying to predict changes in $\Demand$ while maintaining some headroom in the form of idle $\Capacity$.
## Granularity
Let’s say our demand fluctuates between $0$ and $3000$ and $\Box$ has a $\Capacity$ of $1000$. So we’re going to have between $0$ and $3$ boxes.

On one hand, this means that we don’t need to scale very often. This is good in some ways, because each scaling event brings with it the possibility of failure, and it reduces the impact of scaling delay.

But it means our scaling solution is quite inefficient for certain levels of $\Demand$.

For example, with a $\Demand$ of $1001$, we’ll have two $\Boxes$, but $49.9\%$ of our $\Capacity$ will remain unused. We’re essentially paying double for our infrastructure.

We might say that our scaling solution has high *granularity*. It becomes inefficient at specific levels of $\Demand$.

Using smaller $\Boxes$ allows us to avoid this problem. An even better solution involves having $\Boxes$ of different sizes. Both of these introduce additional complexity to the scaling solution and come with problems of their own.
## Eviction and cordoning
Let’s say our $\Demand$ has dropped and we want to reduce $\Capacity$. This means we have to get rid of some $\Boxes$.

But those $\Boxes$ are still live and accepting jobs. We can’t just randomly kill them. That would risk losing both jobs and the work put into completing them. 

We first need to *cordon* them, forcing them to stop taking new jobs. Then we need to *evict* any jobs they’re already executing, transferring them to other $\Boxes$ that aren’t marked for deletion.

Eviction isn’t always possible, and in some cases, we might just choose to *drain* the $\Boxes$ instead, cordoning them and waiting until they naturally complete the jobs they’re executing.
# Buying scalability
Scalability is like, really hard.

It’s so hard, very few companies try to achieve it themselves. Instead, they pay cloud providers a massive markup to solve it for them, and those solutions also come at the expense of $\Latency$ and $\Throughput$.

The **serverless platform** is the best example of this.

One paper^[https://arxiv.org/pdf/1812.03651] found that, when compared to a virtual machine, serverless platforms cost $100$ times more for the same $\Throughput$.

That would be lunacy from a *performance* standpoint, but it makes perfect sense when considering *scalability* – our $\Demand$ might vary by a factor of $100,000$ over a single day.

Compared to that, even a factor of $100$ might not be so scary, especially if it lets us reduce costs in other ways.
# Conclusion
*Scalability* is being able to dynamically change our system’s $\Capacity$ based on $\Demand$, while *performance* typically refers to the $\Throughput$ it sustains at a given time.

It’s important to understand the difference, because the two are often at odds. The highest performing systems are *not* scalable ones.

Stay tuned for future posts, where we’ll dive a little deeper into some of the things we’ve talked about in this article.