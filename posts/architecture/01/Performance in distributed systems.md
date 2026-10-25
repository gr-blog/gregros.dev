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
%%%?
In distributed systems, we sometimes need to reduce performance to get better performance.

What?
%%
The line 
In software systems, scale costs $\Latency$. Latency is a

# Load balancing
When you have a bunch of replicas, you need something on the network to actually balance the traffic between them. That thing costs latency. It’s a serial cost 

When a system scales, 


In distributed systems, performance refers to two very specific metrics: 
Distributed systems are typically modelled using $\Jobs$ and $\Boxes$, though different authors use somewhat different terms for these.

- A $\Job$ is a computing task that needs to be completed by some workload.
- A $\Box$ is the thing processing jobs, like a container, VM, or a guy in a room.

In such systems, performance typically refers to one of two metrics:

- $\Latency$ — the amount of time it takes a $\Box$ to complete a single $\Job$.
- $\Throughput$ — the number of $\Jobs$ all the $\Boxes$ can complete over time.

While reducing $\Latency$ increases $\Throughput$, doing so is very hard. It primarily happens at the level of the workload itself and there is a limit to how low it can go.
 
Instead, we increase $\Throughput$ by raising $\Capacity$, the number of $\Jobs$ our system can process at a time. This often comes *at the expense* of $\Latency$.

From the perspective of a distributed system, $\Latency$ is less a performance metric and more a *Quality of Service (QoS)* metric. If the $\Latency$ is too high, our system isn’t just performing badly — it’s actually broken.

When we trade $\Latency$ for $\Throughput$, we do so *consistently*. On the other hand, when $\Demand$ exceeds $\Throughput$, $\Latency$ rises uncontrollably.
# Scales of latency
It’s best to view latency in terms of scale:

- CPU cycles
- Microseconds
- Milliseconds
- Seconds
- Minutes
- Hours
- Days

When you have second latency in series of microsecond latency, microsecond latency can often be ignored.




ranging from a single CPU cycle to around a hundred, to
## Nanosecond / CPU cycle
- Simple instructions like integer additions 

## Cache read
An L1 cache read takes also takes a few nanoseconds, though the 

1. CPU cycles typically take around a nanosecond.
2. 

# Everything costs latency
In distributed systems, every component and piece of infrastructure has a latency cost. That goes for every load balancer, queue, service, gateway, VPN, and so on.

The lowest $\Latency$ is found inside a single machine, where basic operations cost nanoseconds. Outside the machine, nothing is that cheap. 

Everything has a microsecond cost at the very least, which increases to milliseconds for long roundtrips and even seconds once queues become involved.

# All systems are real-time
All systems are soft real-time systems to some extent — every single $\Job$ has a best

When adding a component to a system
# Latency costs a lot
All jobs have a best-by date, some time after which they are worth less and less to complete, until they become totally irrelevant.

This is true for requests, 
All systems are, to some extent, soft real-time systems. 

All $\Jobs$ have a *latency budget*



