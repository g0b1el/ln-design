
# Payment Route Reservation

The idea behind Payment Route Reservation is to split the lightning network payment into two phases: reservation and commitment. During the
reservation phase, routing nodes will reserve a specified amount of satoshis without signing a new channel commitment state. Those reserved
funds may be used in the next(commitment) phase or not if the reservation is canceled. Reservations are automatically canceled if the
reservation is not committed within a specified timeout period(for instance, 1min). Reservation can be also canceled manually by any upstream node.
In the commitment phase, routing nodes will commit to a new channel state and revoke the previous commitment.

For example, `S` wants to send `amount` satoshis using `payment hash` to `R`:

```
          add_reservation(payment_hash, amount,...) ->
              ┌───┐         ┌───┐         ┌───┐
      ───────►│   ├────────►│   ├────────►│   ├────────►
    S         │ A │         │ B │         │ C │          R
      ◄───────┤   │◄────────┤   │◄────────┤   │◄────────
              └───┘         └───┘         └───┘
        <- reservation_success(fees_sum, cltv_delta_sum,...)
```


* `S` is a sender
* `R` is a receiver
* `A`, `B`, and `C` are routing nodes.

To initiate the payment process, `S` first generates a reservation onion for `A->B->C->R` route and forwards it to the first
node in the route, using the `add_reservation` call.
Each node on the route gets the same amount to reserve. So, there is no need to put the `amount` inside the onion, except for the receiver in some cases.
Note that onion now contains only next hop information. This fact will prove beneficial later for [Rendezvous Routing](todo).

Upon receiving the onion, `A` unwraps the onion and finds out the next hop `B`. If he has enough unreserved balances in his `A->B` channel, he will create
a new reservation for the requested amount and add it to the channel reservation set before forwarding the onion to B. Reservation in the reservation
set can be addressed by `payment hash`, while points can be used for PTLC. If `A` has an insufficient channel reservation balance, `A` will
return `reservation_failed` to the upstream node(sender `S`). Similarly, `B` and `C` will do the same. Assuming there are enough reservation
balances in all channels to reserve a payment, the reservation flow eventually reaches the final recipient, `R`. `R` unwraps the onion and checks
if he knows the preimage. If he knows, `R` returns `reservation_success` to upstream node `C`.

At this point, we have successfully reserved the payment route for `amount` satoshis from `S` to `R`. But our route nodes haven't included any fees
or cltv_delta. Each node in the route needs to account for not just his channel fees and cltv_delta, but the fees and cltv_delta of all downstream nodes.
To inform the neighbor downstream node of the minimum expected cltv delta for his funds in the channel, the upstream node will pass a `min_cltv_delta` as
part of the `add_reservation` call.
The `reservation_success` call returns a tuple `(fees_sum, upstream_node.min_cltv_delta + downstream.cltv_delta_sum)` to the upstream route node. Receiver `R` can
return tuple `(0, C.min_cltv_delta)` to `C` because he is the last node in the route. However, this will reveal to node `C` that `R` is the receiver.
To hide the fact that `R` is the receiver, `R` will return some non-zero value for a
fee(check [Amount Correlation Prevention](./Payment%20Correlation%20Attacks%20and%20Preventions.md#amount-correlation)) and semi-random cltv_delta_sum
value higher than `C.min_cltv_delta`.
`C` then will attempt to extend his reservation by fee and cltv sum from the `R`. If successful, `C` sends `reservation_success` to `B`, with a
tuple increased by his fees and his cltv delta. Fees can be ppm, base, or whatever fee function the node operator prefers. If `C` can't extend
the reservation, he sends to B `reservation_failure` and to `R` `cancel_reservation`. To ensure the node can always extend a reservation, we can
set aside part of the channel reserve balance just for this case (let's say 0.5 percent). `A` will do the same. Eventually, if everything is
in order with nodes `A` and `B`, `S` will receive a tuple (fees_sum, cltv_delta_sum). Now `S` knows precisely how much fees and cltv_delta route expect.
If the cost is acceptable, S can move on to the commitment phase. If not, S can cancel the current route and look for a less expensive alternative.

Let us examine the communication process between the hops in the current lightning network payment and the payment with route reservation in greater detail.

```
             Current LN payment                                         LN payment with route reservation

 +-------+                               +-------+             +-------+                               +-------+
 |       |--(1)---- update_add_htlc ---->|       |             |       |--(1)---- add_reservation ---->|       |
 |       |                               |       |             |       |<-(2)---- reservation_success--|       | reservation onion phase
 |       |                               |       |           ──|─────────────────────────────────────────────────
 |       |--(2)--- commitment_signed --->|       |             |       |--(3)--- commitment_signed --->|       |
 |   A   |<-(3)---- revoke_and_ack ------|   B   |    ==>      |   A   |<-(4)---- revoke_and_ack ------|   B   |
 |       |                               |       |             |       |                               |       | commitment phase
 |       |<-(4)--- commitment_signed ----|       |             |       |<-(5)--- commitment_signed ----|       |
 |       |--(5)---- revoke_and_ack ----->|       |             |       |--(6)---- revoke_and_ack ----->|       |
 |       |                               |       |             |       |                               |       |
 +-------+                               +-------+             +-------+                               +-------+
```

One `update_add_htlc` is now replaced with two calls - `add_reservation` and `reservation_success`. When node `A` receives
`reservation_success` with a tuple `(fees_sum, cltv_delta_sum)`, nodes `A` and `B` have all the information to commit a new
channel state. The new commitment transaction amount would be `amount + fees_sum`, and cltv delta would be `cltv_delta_sum`.

For now, thanks to the one additional call, we've just increased the latency. It should be noted that the reservation set is stored
exclusively in memory. No information needs to be stored in a database or replicated. If the node crashes with a live reservation, nothing bad can happen.
Thus, the increase in latency would be proportional to one additional call between hops.

But before we tackle latency, let's first address the reliability of payments.


## Payment Route Split Reservation

If the routing node is unable to create a new reservation due to insufficient channel reservation balance, the node may try to find some other route
to reserve the remaining amount for the next downstream node.

Let's consider a scenario where `A` wants to forward a reservation of 10 mBTC to `B`, but A->B channel has only 8 mBTC left of its channel
reservation capacity. The outbound liquidity can be greater than 10 mBTC, but the remaining reservation set has only 8 mBTC left.

           ┌───┐           ┌───┐
       10  │   │     8     │   │   10
      ────►│ A ├──────────►│ B ├───────►
           │   │           │   │
           └─┬─┘           └─▲─┘
             │               │
             │2    ┌───┐     │2
             │     │   │     │
             └────►│ C ├─────┘
                   │   │
                   └───┘


Rather than returning `reservation_failure` and losing the possibility to earn fees on the remaining 8 mBTC, `A` will try to find a route for
the remaining 2 mBTC. Suppose `A` picks A->C->B route. It is important to note that the new route should share the same payment hash as the
main route(->A->B->). For PTLC A->B and C->B payment hops need to commit to the same lock point. Now B will accept to forward the reservation and the payment of
10 mBTC further because he knows if he ever commits to a payment with `A` and `C`, and if the preimage is revealed, he will receive his 10 mBTC plus fees.

It is important to note that `A` now needs to take into account the fees and cltv_delta of the new route as well. A's fees for his upstream node
is now the sum of the fees in his A->B channel and the fees in the A->C->B route. Cumulative cltv_delta would be the maximum between his A->B channel
and A->C->B route.

There can be optimization in terms of fees. For instance, `A` can renounce his `A->C` channel fees. So the only additional fee the sender
needs to pay for the new route is 2 mBTC on C->B route. Assuming network convergences to roughly the same fees on all nodes, the cumulative
route split fees effect would be the same as if the whole amount was routed through A->B. Although this example may seem too perfect, it's quite
common on the plebnet part of the lightning network, where most nodes are created using Triangle Liquidity Swaps.

Note that `A` can try multiple reservation routes simultaneously, where each route can contain multiple nodes. Additionally, if `C` doesn't have
enough reserve balance in his C->B channel, `C` can split his route through some other nodes to `B`, etc.
To prevent infinite reservation recursion calls, in case of huge payment, the route split node will pass with `add_reservation` additional information
`max_fees` and `max_cltv_delta` that it will accept from the new route. Each node in the route will first deduct its fees and cltv_delta from those values.
If tuple `max_cltv_delta` is a huge number of blocks, and the routing node is not willing to bear the risk, he can replace the value for the next
route node, with its maximum risk-bearing value. After deduction, if any element is negative, the node will return `reservation_failed`.  If the
tuple elements are still positive after the deduction, the reservation will continue, and the decremented values will be passed to the next downstream node.
To hide the difference between the main route and route split, and for the sender to communicate the maximum cost in terms of fees and cltv he is willing to accept,
the payment sender will also send the tuple `max_fees` and `max_cltv_delta` that he accepts for the main payment route. Now, if someone is observing
the lightning network traffic, he wouldn't know if a reservation is for the main payment route or route split. This will lead to increased privacy in payments.

**It doesn't matter if the channels are balanced or not.** Assuming that the node is well-connected and has adequate inbound and outbound liquidity,
the route split reservation will, in most cases, find a reservation route and to the next node. So there is **no need for rebalancing or fee update scripts**
to balance channel liquidity.

It is also worth noting that payment larger than the channel capacity A->B can now be forwarded from node `A` to node `B`.
If nodes `A` and `B` are well connected, in theory, it will be possible to forward, with route split, a maximum payment of
`min(A.outbound_liquidity, B.inbound_liquidity)` amount.

### Trampoline route split reservation

What if `A` and `B` don't have a direct channel, or the sender doesn't know the whole network topology?


                          ┌───┐         .         ┌───┐
                       10 │   │---->   . .   ---->│   │ 10
                     ────►│ A │---->   . .   ---->│ B ├─────►
                          │   │---->    .    ---->│   │
                          └───┘                   └───┘

`A` can still try to reserve payments through different routes to `B`. The idea is similar to what is currently known as Trampoline Routing.
The biggest benefit of using Trampoline route split reservation payment over Trampoline Routing payment is that a sender, before initiating the payment, will
know exactly how much in fees he needs to pay. As well as how big `cltv_delta` will be.


#### How expensive is a route split reservation?

To create a route split reservation, `A` must generate a new reservation onion, and until both reservations propagate to `B`, `B` cannot forward
the reservation downstream. So we increased latency again, but on the other hand, we have also increased reliability substantially.
The estimated reliability for single payments with route split reservations might be in the range of 90-95%. However, to realize the full
the potential of the lightning network, we need to aim for a reliability of 99% and finally address the issue of latency.


## Redundant Multi Route Payment Reservation

The idea is similar to what is now called Multi-Path Payments.


                          ┌───┐       ┌───┐       ┌───┐
                          │   ├──────►│   ├──────►│   │
                     ────►│ Z │       │ T │       │ U ├─────►
                          │   ├──────►│   │   ┌──►│   │
                          └───┘       └───┘   │   └───┘
                                              │
                             .......................

                          ┌───┐       ┌───┐  │    ┌───┐
              S           │   ├──────►│   │  └───►│   │
                     ────►│ D │       │ E │       │ F ├─────►    R
                          │   ├────┐  │   ├──┐    │   │
                          └───┘    │  └───┘  │    └───┘
                                   │         │
                          ┌───┐    │  ┌───┐  │    ┌───┐        
                          │   │    └─►│   │◄─┘    │   │
                     ────►│ A │       │ B │       │ C ├─────►
                          │   ├──────►│   ├─────► │   │
                          └───┘       └───┘       └───┘

The problem with a single route reservation is that reservation fees(or cltv) might be too high, and the sender will have to create a new
reservation route hoping to get a better fee. Also, reservations might take longer than we are willing to wait.
To overcome this issue, single payment reservations can be split into sub-payments over different routes.

The one way to do it is to split the amount into N parts, and then create 2*N(or some other multiplier) reservation routes each
with a payment amount of `payment amount`/N.
Note that splitting the amount into equal parts suffers from [Payment Amount Correlation](./Payment%20Correlation%20Attacks%20and%20Preventions.md#amount-correlation),
but is used here for illustration purposes only.
Now a sender needs to wait for just N out of 2N reservation routes to return `reservation_success` before he can initiate payment. Note also that multiple
reservation routes can pass through the same nodes.

This now creates an interesting market dynamic where the fastest route wins. And the fastest route is with the fastest nodes. Nodes are
incentivized to compete with better internet and better hardware. There would also be a competition between lightning node developers to create
the fastest lightning node.

If a sender is not that interested in the speed of payment, he can wait for all the 2N routes to return reservation results before selecting the
cheapest N routes. This forces nodes to compete on fees as well, as the sender is more likely to select the cheaper routes.

Ultimately, all users on the lightning network will benefit from these developments.

Just because nodes reserved the route doesn't mean they have to route the payment. There is still a probability of payment failure. Payment can fail
because of reservation time out, node going offline, griefing node, etc. To further increase reliability, we can also add stuckles payments[1] on the top.

By implementing Redundant Multi-Route Payment Reservations, payment reliability is expected to reach the 99% range, and latency on average should be significantly
reduced since retries would be mostly eliminated.

[1] https://lists.linuxfoundation.org/pipermail/lightning-dev/2019-June/002029.html

## LN Wallet UX improvements

Not every payment is the same. For example, users may value payment speed over fees when paying for groceries at the supermarket,
while they may prioritize low fees over speed when paying their utility bills at home. For VPN subscription payments, users may
prioritize privacy and cost.

UX can be improved by providing users with more options that fit their specific needs for payment.
With route reservation, we can present the exact user fees he is going to pay, so wallet developers can add the following element to UX:

Radio box: Fast Payment | Cheap Payment | MaxPrivacy

- If a user selects `Fast Payment` the wallet will wait for the first N of the fastest routes and initiate payment automatically(unless fees exceed in
  percents some config value).
- If a user selects the `Cheap Payments` node will wait for all 2N routes to return, pick N the cheapest routes, and then present the user
  with fees and the "Send" button. If fees are too high, the user can press the `Try again` button.
- For `MaxPrivacy` wallet will behave as in `Cheap Payment` mode, but the routing algorithm will prefer longer routes over smaller
  nodes, with preferably at least one Tor node.

_N routes can be N trampoline routes if a wallet doesn't have a whole network topology._


## Gossip messages spam

Currently, a large percentage(97% or more) of Lightning Network gossip messages are related to `channel_update` messages, primarily
due to many nodes running automatic fee adjustment scripts. As the number of nodes and channels continues to grow, this can become unsustainable.

Reservation is not used just to reserve the route but also to inform a sender of fees and cltvs. Thus, there is no need for nodes to publish
their fees and cltv to the whole network. So, information like base_fee, ppm fee, and cltv can be removed from the `channel_update` message.
With route reservation, there is no need for channel rebalancing or fee update scripts. Node operators can still run fee update scripts without
any restrictions, but those updates would not propagate to the network.

If we remove `fee` and `cltv_delta` from the channel update gossip, what is left for the routing algorithm to work with is `channel_capacity`, `htlc_minimum_msat`
and `htlc_maximum_msat`. Out of these `htlc_maximum_msat`, can be a local node config. If the reservation attempts to reserve a greater amount
than the maximum allowed for the channel, a node can try to split the reservation over multiple channels. `htlc_minimum_msat` can remain a global config
to prevent unnecessary reservation failures.


## Routing algorithm and network decentralization

The current Lightning Network can be seen as a decentralized network in terms of network topology, but when it comes to payment routing, it
behaves more like a hub-based network.
A small number of large nodes(~30) handle the majority of payments, while medium and small nodes are mainly used for
channel rebalancing and rarely forward any payment. This is a big issue in terms of fairness(the rich are getting richer faster), privacy,
reliability of the network, cost of a transaction, etc.

The root cause of this problem is not because the rich are rich but because of the routing algorithm. Currently, used routing algorithms calculate the
probability of routing a payment over some channel as a function directly proportional to channel capacity. If the channel has a lot of capacity
probability is higher, small channels will have a lower probability of payment routing. Because all wallets are effectively running in `Fast Payment`
mode, the routing algorithm will almost always favor bigger channel routes than smaller ones and thus create the hub-based payment network we have today.

The use of channel capacity as a primary heuristic can be the biggest misleading factor for Lightning Network's (LN) routing algorithms. For instance,
someone can create a node and then buy 100 channels with 1BTC each of inbound liquidity and set all channel
fees to 0. Every routing algorithm will try to route payments through this node, but the node couldn't route a single satoshi.
The most popular nodes on the LN network currently most likely have way more inbound than outbound liquidity, making channel capacity
an unreliable factor for LN routing algorithms.

### Capacity-unaware Routing Algorithm

One potential solution to the problem of using misleading channel capacity as routing heuristics is for nodes to publish their combined
outbound liquidity across all channels. By knowing this information, other nodes can easily calculate cumulative inbound liquidity,
as the capacity of each channel in the Lightning Network (LN) is already known. Recall that maximum payment between nodes A and B in
theory is equal to `min(A.outbound_liquidity, B.inbound_liquidity)` using route split reservation. Publishing cumulative outbound liquidity
would also signal to the rest of the network where liquidity is needed and where there is an excess of liquidity. This approach would eliminate
liquidity sinks and allow for better deployment of network liquidity. Automatic channel opening would also become a trivial task.

A capacity-unaware routing algorithm can be designed to be simple yet effective. The algorithm makes routing decisions just on randomness, hop count from our node,
and node inbound/outbound liquidity. Randomness is the most important as it helps distribute the load across the network and makes payment more reliable.
If a route fails, it can be difficult to pinpoint which node is responsible. In the worst case, it can be any node in
the route. Nodes may try to shift blame to each other, making blacklisting an unreliable solution. Instead, the routing algorithm does not
track blame information about individual nodes or blacklist them. If the node is well-connected and the routing algorithm is random enough,
we should be able to find enough routes to the sender. If there is a faulty, griefing, or just slow node in the network, we let nodes handle
disputes themselves. Each node will track the local reputation of each of its neighbors. If there are a lot of failed payments from
one neighbor, the node operator might contact that neighbor for an explanation or just close the channel to that neighbor node.

**But wouldn't the nodes lie about his outbound liquid?**
If the node lies, a lot of reservations will fail on its node, and if there is a lot of reservation failure, neighbor nodes
might close the channel with a misleading node. Also, while processing huge payment reservations, which will eventually fail,
a node could have routed a smaller payment and earned fees. So nodes will be incentives to report the real outbound liquidity or lower if privacy
is the issue.

For privacy-focused cryptocurrencies, such as Monero, not publishing channel capacity is essential to maintain users' privacy.
These users do not want to disclose any information about their transaction amounts or balances. Therefore, a Lightning Network for
privacy currencies can avoid publishing channel capacity and outbound node liquidity, and make routing decisions based solely on hop
counts and randomness. Although this approach may result in more reservation failures, especially for larger amounts, the benefits of
preserving privacy are likely to outweigh the drawbacks.

### Capacity-aware Routing Algorithm

The rationale for why we would still want to use channel capacity in the routing algorithm is its impact on transaction fees. Disregarding individual
channel capacity and relying solely on inbound/outbound node liquidity in the routing algorithm can result in a large number of Route Split Reservations.
Because more channels will be involved cumulative fees and cltv might be suboptimal.

One potential way to design a routing algorithm that takes into account channel capacity but still avoids centralization around large nodes is as follows:

1. Routing algorithm selects hops for a route using only randomness and hop count from our node.
2. For each route, the smallest channel capacity and smallest `htlc_minimum_msat` of all channels on the route are tracked. If two nodes are
   connected through multiple channels, their channel capacity sum is used.
3. Upon completition routing algorithm will return a route, smallest channel capacity, and smallest `htlc_minimum_msat`. Now we can safely apply the channel
   capacity heuristic without the risk of favoring big channels over smaller ones. Attempting to route a payment equal to the smallest channel capacity has
   a high probability of resulting in at least one route split and thus higher fees. If we try to route, half of the minimum channel
   capacity probability is significantly lower. So for instance we can
   use a heuristic that says for a route with `min_channel_capacity` we will attempt a maximum payment reservation of `min_channel_capacity/3`. The routing
   algorithm can produce multiple routes by conducting parallel route searches. From the returned routes, the wallet will choose some of them in such
   a way that it avoids [Amount Correlation](./Payment%20Correlation%20Attacks%20and%20Preventions.md#amount-correlation).


## Fast spam prevention

Fast spam is a situation on the network when malicious nodes start creating a lot of payments to the random nodes, which will never be resolved because
the receiver does not know the payment preimage. The receiver can only fail the payment. This is not possible with a reservation anymore because
receivers will reject a reservation, and there will be no payment attempted. Now, this is a lot better because
the reservation does not commit to a new channel state, and no DB operations are involved.

#### But how to prevent fast reservation spam?

If a node observes an unusually high number of reservation errors per minute from a particular channel, the node can throttle reservations coming from
that channel. Eventually, if spamming continues despite throttling, the node operator can inform the neighbor node
operator to investigate the cause. The neighbor node will do the same. If the problem persists even after informing the neighbor
node operator, the node operator can decide to close the channel to prevent further spam.

#### But what if an attacker controls both nodes(receiver and sender) or if the attacker creates a circular route?

The malicious receiver node will accept the reservation, but the HTLC will fail upon arrival. This would be basically fast spam again,
with a bit more work to be done by the attacker. As proposed before[2], we can demand a prepayment. If a node detects a high volume
of failed payments per minute on any of its channels, it can begin to demand a prepayment to commit to the HTLC on those channels.

The challenge is how much to charge. If we charge too little, spam will continue. If we charge too much, this will affect regular micro-payments.
To address this, we can extend the `reservation_success` tuple to include a prepayment fee, which nodes can adjust as needed. Note that there is no need
to propagate this fee to the whole network. Initially, nodes can choose to charge no prepayment fees, as the routing algorithm will prioritize
routes without prepayment. Now if a node observes a large number of payment failures per minute, he can start aggressively increasing his
prepayment fees, thus making the attack very costly for the attacker.

There has been discussion surrounding the issue of sending a prepayment and the potential for the first node in the route to steal the
prepayment and then fail the payment. There are two possible scenarios to consider:

* If the prepayment is less than or equal to the fees the node is going to earn for routing the payment, then it is in the node's
  best interest to route the payment.
* If the prepayment amount is greater than the node's payment fees, there are a couple of solutions to consider:
    - Allow the first node to steal the prepayment. This should be acceptable because if the prepayment becomes so large,
      it is likely that the attacker is the one being robbed. Honest users will opt for routes without a prepayment, especially for large prepayments.
    - Send multiple keysend prepayments. However, privacy concerns arise because the length of the route is revealed to the first node,
      and the last node in the route will discover the identity of the receiver. Another option is to send prepayments for different parts
      of the route, for example, one keysend for the first half of the route, and the second keysend for the second half of the route.


[2] https://lists.linuxfoundation.org/pipermail/lightning-dev/2020-February/002547.html


## Channel liquidity probing

With a route reservation split, inspecting channel liquidity from other nodes becomes a more challenging task. This is due to the fact
that reservations are less likely to fail. Now observer will need to deduce the channel liquidity from the reservation
response, which includes information such as fees, cltv delta, and pre-payment.
When a reservation split occurs, fees and cltv delta values are likely to be higher, making it more difficult for potential channel
observers to estimate the channel liquidity accurately.
To further complicate matters for potential observers, each node can dynamically adjust its fees and cltv delta values, making the
task of accurately estimating channel liquidity even more difficult. By randomizing these values with each return, an attacker's ability
to probe the channel liquidity is diminished.

Additionally, if the attacker starts sending a lot of reservation requests, those requests might be seen as a DOS attempt, and the node might start throttling
the reservation requests.


## Eliminating the need for Splice-In and Splice-Out (partially)

One of the ideas behind the splicing is to increase the capacity of the channel, so that channel can route bigger payments. As a reminder, the current
LN routing algorithm tends to favor larger channels. However, with the introduction of route split reservations, this is not necessary anymore.

For instance, if nodes `A` and `B` have N channels between them, during the reservation process, node `A` can reserve a route through all N channels,
effectively treating them as one large channel. If nodes `A` and `B` have N channels between them, closing one channel basically becomes a splice-out operation.

Although there are other reasons why splicing in and out may be necessary, the introduction of Payment Route Reservations would largely
eliminate the need for such a feature.
