---
title: Scalability-is-not-performance
published: 2025-10-16
updated: 2026-10-18
figure: example-job-rates
aliases:
---
%%%?
Scalability is sometimes confused with performance, but it’s not at all the same thing.

Let’s take a look!
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
\def\Concurrency{\term{Concurrency}{Pink}}
\def\HttpNotModified{\textbf{304 Not Modified}}
\def\HttpBadGateway{\textbf{502 Bad Gateway}}
\def\Box{\term{Box}{PineGreen}}
\def\Boxes{\term{Boxes}{PineGreen}}
\def\Productivity{\term{Productivity}{Apricot}}
\def\avg{\texttt{avg}}
\def\Value{\term{Value}{RoyalBlue}}
$$
Scalable systems should efficiently maintain consistent performance at different levels of demand. *Consistency* and *efficiency* are more important than raw speed. 

Before delving deeper, let’s take a closer look at how performance is defined in distributed systems.
# What’s performance?
Distributed systems are typically modelled using $\Jobs$ and $\Boxes$, though different authors use somewhat different terms for these.

- A $\Job$ is a computing task that needs to be completed.
- A $\Box$ is the thing processing jobs, like a container, VM, or a guy in a room.
- Our $\Demand$ is the number of $\Jobs$ we receive per $\Time$.

In such systems, performance typically refers to one of two metrics:

- $\Latency$ — the amount of time it takes the system to complete a $\Job$.
- $\Throughput$ — the number of $\Jobs$ the system completes per $\Time$.

The two metrics are related in Little’s law, found in ^[!doi:10.1287/opre.9.3.383 | http://fisherp.scripts.mit.edu/wordpress/wp-content/uploads/2015/11/ContentServer.pdf]:
$$\avg(\Throughput) \approx \frac{\avg(\Concurrency)}{\avg(\Latency)}$$
Where $\Concurrency$ is the number of jobs in the system. 

From this, we can see that reducing system $\Latency$ tends to increase system $\Throughput$. We often do that by optimizing the code actually being executed or reducing $\Latency$ between components.

But there is a limit to how much $\Throughput$ we can get this way. To freely control our $\Throughput$, we need to increase $\Concurrency$ while keeping $\Latency$ fixed.

We can do that by adding more resource $\Capacity$ to the system, so it can execute more jobs in parallel. That means adding more servers, more CPUs, more memory, and so on.

Of course, resources aren’t free; they $\Cost$ money. Any $\Capacity$ not going towards $\Throughput$ still costs the same, so we want to keep unneeded $\Capacity$ low.

In addition, more $\Boxes$ means more infrastructure to route between them, so increasing $\Capacity$ also raises $\Cost$ in other ways.
## So we don’t care about latency?
We certainly do.

From the perspective of a distributed system, $\Latency$ is less a performance metric and more a *Quality of Service (QoS)* metric. If the $\Latency$ is too high, our system isn’t just performing badly — it’s actually broken.

When we trade $\Latency$ for $\Throughput$, we do so *consistently*. On the other hand, when $\Demand$ far exceeds $\Throughput$, $\Latency$ can rise uncontrollably.
# Scalability
One formal definition for scalability is found in {!doi:10.1109/71.862209|https://www.sce.carleton.ca/faculty/woodside/pubs/scal00.pdf}. Here, we have a $\Value$ function which represents the system’s QoS. It’s often defined to decrease as $\Latency$ increases.

For a given level of $\Demand$, the system’s productivity is defined as:
$$
\footnotesize
\Productivity(\Demand)=\frac{\Throughput(\Demand)\times \Value(\Demand)}{\Cost(\Demand)}
$$
The system is scalable if $\Productivity$ stays around the same at different levels of $\Demand$. In other words, $\Throughput$ needs to rise semi-linearly with $\Cost$ and $\Latency$ should not increase too much.
## A note about elasticity
Astute readers might consider this definition to refer to elasticity, but that concerns the system’s response to $\Demand$ that changes over $\Time$.

When talking about scalability, we’re not considering dynamic responses to changes in $\Demand$, only $\Productivity$ at different levels of it.
## Regions of scalability
In practice, no system is infinitely scalable. Instead, systems are designed to handle $\Demand$ in a specific region. If the system is outside that region, things will start failing.

This will first be felt as an increase in $\Latency$ as queues fill up or $\Boxes$ take more jobs than they can efficiently process. After that, load failures will start happening. Things such as:

- Machines run out of memory
- Queues grow beyond operational constraints
- Network requests time out
## Granularity
Let’s say our $\Demand$ fluctuates between $0$ and $3000$ per $\Time$. Each $\Box$ has the $\Capacity$ to process $1000$ jobs per $\Time$.

At a $\Demand$ of $1001$, we might have two $\Boxes$ instead. But that configuration doubles our $\Cost$ while only increasing $\Throughput$ by $0.1\%$. 

What if, instead of getting another $\Box$, we queue $\Jobs$ instead?

This will raise $\Latency$ and the queue will slowly keep growing. When it’s big enough, we can finally introduce another $\Box$ to handle the added $\Jobs$. 

But then, the new $\Box$ will eat through the queue rapidly, and we’ll need to get rid of it again. 

A better solution might be to reduce our granularity by getting smaller $\Boxes$ that cost less, or even several different kinds of $\Boxes$ to more accurately match $\Capacity$ to $\Demand$.
# Failing at cost efficiency
Some systems can have high $\Capacity$ without scaling at all. While they can provide a lot of $\Throughput$, they fail at doing so *efficiently*. 

Let me present one such system:

```canva key=silly-big-server ;; size=208x208 ;; alt=A single rectangle saying 'HPE Scale-Up 3200'
https://www.canva.com/design/DAHYY0hbWOk/co6gpuuOZAylV9cdRNflUw/view
```

A single HPE Compute Scale-Up 3200, a thing that takes *vertical scaling* to its full conclusion. Although not technically a supercomputer, we can still equip it with some pretty impressive specs.

- **32× 60-core Intel Xeon Platinum 8490H CPUs** — totaling **1920 cores**, each clocking at up to **3.5GHz**.
- **32× 8× 2× 128GB DDR5 memory** for a sweet **64 TiB**, or **2 TiB per CPU**.
- **192× 15.36 TB PCIe 5.0 NVMe SSDs** in **RAID 0** configuration, giving us **3 PB of raw storage** at **~2 TiB/s**.

You can’t normally have 32 CPUs in the same memory space, except that the Scale-Up 3200 is a *glued NUMA* machine, an unholy union of eight server chassis bound with solid copper and a dash of mad science.

Now, consider running all your code directly on the OS. No virtualization, no containers, nothing. Just 64 terabytes of memory, four thousand hardware threads, and an I/O bandwidth that rivals RAM.

This one machine provides enough $\Capacity$ to rival a major distributed system. After all, it could instead be hosting 1000 VM instances doing exactly the same work.

On the other hand, it also costs millions to purchase and hundreds of thousands a year to maintain.

While it can deliver high $\Throughput$ when $\Demand$ is high, we’re paying the same $\Cost$ even when $\Demand$ is low. Scaling up isn’t an issue, but we can’t scale down at all.

Operating a really big $\Box$ is only viable when you can guarantee high levels of $\Demand$.
# Conclusion
Performance is important, but it’s just one metric that distributed systems need to optimize for. Scalable systems need to be efficient at different levels of $\Demand$, something that’s considerably harder.