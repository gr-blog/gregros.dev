
This is a dual article to [[NO NO NO NO NO|Scalability is not performance]]. That article was aimed towards developers or people who aren’t entirely familiar with distributed system terminology. 

To those who are familiar, my definition may or may not seem strange, depending on the literature and sources you’ve read.

You may, for example, believe that my definition is actually *elasticity* and scalability means something else. 
# Able to handle growing demand
> The ability to handle a growing amount of work.

This definition does not reference cost-effectiveness or addition of resources, so by it, a system consisting of a single giant machine is scalable. It can handle a vast 

# Doing so by adding resources
> The ability to handle a growing amount of work by adding resources.

In this case, a single giant machine is not scalable, but 

I’ve generally heard three definitions for scalability. 

# Addition of resources
This definition goes like this:

> A system is scalable 


# Dealing with changing demand
$\Demand$ is never constant and rarely predictable. In fact, it’s a function of $\Time$, and can be plotted as a curve. Here are some potential examples:

```canva key=example-job-rates ;; size=700x450 ;; alt=Shows four job curves with different peaks and valleys
https://www.canva.com/design/DAGlG86qqak/lKUMM69vLfs-Aqqc3pvnzQ/view
```

A good distributed system seeks to match its $\Capacity$ to every level of $\Demand$.  

## Scaling delays
There is always a notable delay between rising $\Demand$ and a matching increase in $\Capacity$. 

This delay, even if small, has a big impact on how systems scale. It comes from two sources:

- **Monitoring delay:** A system can only scale on metrics that were computed in the past. This means it can only see what the $\Demand$ used to be, rather than what it currently is.
- **Startup delay:** Systems scale by adding resources. These resources take time to come online. 

We try to deal with these problems in various ways, such as using queues, keeping a fixed $\Capacity$ overhead, and trying to interpolate the $\Demand$ curve.

# Buying scalability
The ability to deal with rapidly changing demand is really expensive in both $\Cost$ and $\Latency$.

A cloud platform has $\Boxes$ that appear or disappear at the press of a button or based on an automatic policy. But behind all of its abstractions, 

Cloud scalability is possible because cloud providers take up the purchase and operating costs of these machines, using software that provisions them based on our demand.

That software uses virtualization, which has invariable overhead. Virtual machines are also subject to performance fluctuations, bleedover from other workloads on the same physical hardware.

In spite of that, VMs are significantly more expensive than leasing physical servers (in terms of resources per USD). We’re still happy to pay that cost for the ability to rapidly scale our system.

Serverless platforms take things a step further, solving many of the problems discussed in the previous sections, scaling invisibly behind the scenes.

The ability to scale is so important that most software companies pay a lot of money to get it, in spite of all the overhead it introduces.