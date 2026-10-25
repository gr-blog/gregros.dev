



We might say that our scaling solution has high *granularity*. It becomes inefficient at specific levels of $\Demand$.

Using smaller $\Boxes$ allows us to avoid this problem. An even better solution involves having $\Boxes$ of different sizes. Both of these introduce additional complexity to the scaling solution and come with problems of their own.
- only scale based on metrics that were computed in the past

The first problem is our inability to see the future. In fact, a scaling solution can’t even see the present. We can only scale based on metrics, and those must have been computed in the past.

The second problem is the startup time. A $\Box$ doesn’t appear instantaneously and just start accepting jobs. The startup time varies between $\Boxes$, but ideally it should be as low as possible.

Overcoming scaling delay typically requires trying to predict changes in $\Demand$ while maintaining some headroom in the form of idle $\Capacity$.
such as machines running out of memory, queues becoming overfull, network requests timing out, and so on.




Scalability is being able to match $\Capacity$ to this changing $\Demand$ to achieve high $\Throughput$, while maintaining consistent $\Latency$.


# Scaling solutions
As mentioned earlier, we typically scale by changing $\Capacity$, the number of $\Jobs$ our $\Boxes$ process at the same time.

We can increase $\Capacity$ by raising the number of $\Boxes$ or using bigger ones, which can handle more $\Jobs$. We can drop $\Capacity$ by doing the opposite.

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

## Granularity

## Eviction and cordoning
