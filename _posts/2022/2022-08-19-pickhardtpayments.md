---
layout: post
title: Pickhardt payments in the Lightning Network
subtitle: A review of Pickhardt-Richter paper
tags: [bitcoin]
---

Payments in the Lightning Network (LN) are performed along paths (ie. a sequence of channels) 
from the sender to the recipient. 
In this process one must take into account two aspects:
1. the sender must pay a fee to every node who must forward the payment on his behalf;
2. a node owner can forward a payment through a certain channel if and only if the
outbound liquidity of that channel is larger or equal than the amount he is asked to forward.
Notice that the total amount in every channel is known by all nodes, but how much of these
funds are allocated inbound or outbound is kept private by owners of the channel.

The traditional LN implementations
are designed so that the payment fees are minimized.
They do not take into account the second aspect and as consequence several 
payment attempts fail with this approach
because of insuficient liquidity at some channel through the choosen path.

## Multi-part payments

Each payment can be routed through a combination of different paths, this is 
called a *Multi-Part Payment* (MPP).
In order to find an optimal MPP, ie. with the least fees, one could model the LN as a directed graph 
where
each channel connecting nodes $$i$$ and $$j$$ is represented by two directed arcs $$(i,j)$$
and $$(j,i)$$.
Each arc $$(i,j)$$ would have an associated *capacity* $$X_{(i,j)}$$ which encodes the outbound 
liquidity and a *cost* function $$c_{(i,j)}$$ which represent the
fees required for forwarding payments.
If $$x$$ satoshis where to be sent through $$(i,j)$$, one must satisfy the constraints
$$x \le X_{(i,j)}$$ and pay a fee of $$c_{(i,j)}(x)$$.
With this model a MPP can be computed using a *minimum cost flow* (MCF) algorithm to optimize the fees.
Yet, likewise the single path payments, the value $$X_{(i,j)}$$ is unknown.
For this reason the MCF networks approach with arc costs associated to fees
gives an incomplete description of the routing problem in MPP, just like the single path payment.

## Uncertainty network

Rene Pickhardt and Stefan Richter [1] have proposed an alternative algorithm for payments, by
introducing the following novel idea:
the LN can be modelled as a directed graph where the arcs represent one way channels and are
characterized by a random variable
that represents the partial knowledge of the channels outbound liquidity and hence their
ability to forward a certain amount during a payment. 

Formally, let $$G(V,E)$$ be the directed graph that represents the LN,
where $$V$$ is the set of all nodes and $$E$$ is the set of all arcs (notice that for every channel
in LN we have two associated arcs going in opposite directions).
We have a capacity function $$u\colon E \to N$$, that encondes the total channel capacity
and let $$f\colon E \to N$$ be a *flow* function that encodes the forwarding request for
every arc in order to place a MPP. Clearly, one must have $$ 0 \le f(e) \le u(e) $$ for every 
$$e \in E$$.
The outbound liquidity $$ X_e $$ of every $$e\in E $$ is unknown, hence we treat it as a 
random variable $$ \hat X_e $$.
The probability of success of a forwarding request of $$ x $$ sats over the arc $$ e $$
will be then $$ P(\hat X_e \ge x) $$.
The probability of success of the MPP represented by $$ f $$ is then
the multiplication of the probability of success for every arc

$$ 
    P(f) = \prod_{e\in E} P(\hat X_e \ge f(e))
$$

The search of $$f$$ that maximizes $$P(f)$$ is equivalent to the minimization 
of $$ -\log P(f) $$, which turns the problem into a search of $$f$$
which minimizes a separable cost function

$$ 
    C(f) = -\log P(f) = \sum_{e\in E} c_e(f(e))  
$$

where $$c_e(f(e)) = -\log P( \hat X_e \ge f(e) )$$.

The tuple $$ ( G(V,E), u, c ) $$ is then called the *Uncertainty Network*.

Pickhardt-Richter suggest that once a MPP is constructed and onions are sent,
the LN will respond either with the success of the payment or the failure of some onions.
With the knowledge of the failed forwarding attemps we can update our knowledge 
of the network, that translates in an upate of the cost function,
and we can try again to construct one more MPP for the remaining amount of money.
The process continues until the entire payment has been sent.


## The probability distribution for the channels

$$ P(\hat X_e \ge f(e)) $$, ie. the probability that the outbound
funds in $$ e $$ are greater or equal to $$ f(e) $$, is naturally constructed from 
an uniform probability distribution of the outboud liquidity.
That is, for an arc $$ e $$ for which we know that the outbound liquidity is
bounded by $$ a_e \le \hat X_e \le b_e$$
the probability of having exactly $$ X $$ outbound satoshis is

$$
P(\hat X_e = X) = \begin{cases} \frac{1}{b_e-a_e+1} & a_e \le X \le b_e \\
                       0 & \text{otherwise}
         \end{cases}
$$

This is because $$ \hat X_e $$ can be any value from $$ a_e $$ to $$ b_e $$, hence the $$ b_e-a_e+1 $$ in the 
denominator.
Then that arc $$e$$ is able to forward a payment of $$ f(e) $$ satoshis if and only if
$$ \hat X_e \ge f(e) $$, that corresponds to all states of $$ \hat X_e $$ from $$ f(e) $$ until $$ b_e $$.
Therefore the probability of success of that forwarding request is

$$
P(\hat X_e \ge f(e)) = \begin{cases} 
		 \frac{ b_e-f(e)+1}{b_e-a_e+1} & a_e \le f(e) \le b_e \\
		 1 & f(e) < a \\
                 0 & \text{otherwise}
         \end{cases}
$$

Initially $$ a_e = 0  $$ and $$ b_e = u(e) $$ but 
in the case that a forwarding request $$f(e) $$ fails along the arc $$e$$ we update
our knowledge of the random value of the outbound liquidity $$ X $$, because now 
we know that it is less than or equal to $$ f(e) $$, hence a new value for 
$$b_e$$ is set to $$ f(e)-1 $$ and $$a_e$$ is kept the same.
Again a uniform probability distribution is the best knowledge we have.
On the other case, when the forwarding of $$ f(e) $$ succeeds 
we can deduce that the outbound liquidity was actually greater than or equal to $$ f(e) $$
and we update $$ a_e = f(e) $$ and keep $$ b_e $$ the same.

## Computing an optimal MPP

It is not difficult to show that the cost function $$ c_e(x) = - \log P( \hat X_e \ge x) $$
is always a convex function and the problem of computing an MPP represented by 
$$ f $$ that corresponds to the optimal value of the cost $$ C(f) $$,
ie. with the maximum probability of success, 
reduces to a minimum cost flow problem
for which efficient polynomial algorithms are known, see [2].

So far this problem optimizes the probability of success of
a MPP, but it doesn't tackle the first aspect discussed in the introduction about the fees of the
payment. However Pickhardt-Richter idea can be extended with the construction of a more complicated
cost function $$ c_e $$, as long as it is convex, that combines the probability of success of
forwarding and the fee that one must pay over that request.
For example if there were no base fees at all one could define 

$$
    c_e(x) = -\log P(\hat X_e \ge x) + \mu \cdot x \cdot r_e
$$

where $$ r_e $$ is the fee rate on that channel and $$ \mu $$ is any positive constant whose value
can be tuned to give more or less importance to the success probability over the payment fees.

## Conclusions

With Pickhardt-Richter we might be seeing pretty soon Lightning libraries implementing the LN 
as an Uncertainty Network. With this information-based model, payments requests will be not just
optimal in the sense of fees but also more reliable.

## References

[1] Rene Pickhardt, Stefan Richter. *Optimally Reliable & Cheap Payment Flows on the Lightning Network* https://arxiv.org/abs/2107.05322

[2] R.K. Ahuja, T.L. Magnanti, and J.B. Orlin. *Network Flows: Theory, Algorithms, and
Applications*. Prentice Hall, 1993.
