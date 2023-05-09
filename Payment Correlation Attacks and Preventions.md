
# Payment Correlation Attacks and Preventions


Using payment correlation attacks adversary can try to link the sender and receiver of payment by observing traffic from
the potential sender to the potential receiver. Such observations can be made by the adversary nodes if they are present
on the payment path or if the adversary is able to monitor the network traffic of the potential sender and receiver. In some
circumstances, the adversary can detect not only his presence on the payment path, but also if the monitored nodes are the
sender and the receiver.

        ___    ____                   ____    ___
       |   |  |    |                 |    |  |   |
     --| S |->| A1 |--> ......... -->| A2 |->| R |--
       |___|  |____|                 |____|  |___|

* `S` - potential sender
* `R` - potential receiver
* `A1`, `A2` - adversary surveillance node

For big well-distributed networks, these forms of attacks are very costly and can be economically applied only on a small set
of nodes. However, if a network is centralized, with the majority of traffic passing through a small number of big nodes, an
adversary's job is much easier. An adversary can monitor traffic on those nodes, or in the case of a state-funded surveillance
adversary, an adversary can acquire a court order to get complete access to big nodes routing information.

The adversary job can be simplified if:
- `A1` and `A2` are the same nodes. The sender and receiver are connected through a single node, the adversary node.
- `A2` and `R` are the same nodes. The receiver is some form of custodial wallet, directly controlled or in collusion with the adversary.
  The adversary is going to be aware of all income transactions. The only thing left is to find out who the sender is.
- `S` and `A1` are the same nodes. If the sender is some form of custodial wallet, directly controlled or in collusion with the adversary,
  the sender has no privacy, so correlation attacks are unnecessary.
- S->A1 is an unpublished channel. An adversary can identify `S` as the sender for all payments originating from `S` and passing through `A1`.
- A2->R is an unpublished channel. An adversary can identify `R` as the receiver for all payments destined for `R` and passing through `A2`.


The most notable LN payment correlations in order of severity are:

* Hash correlation
* Amount correlation
* CLTV correlation
* Timing correlation

## Hash correlation

Hash correlation is the most straightforward to detect for surveillance nodes. If adversary nodes `A1` and `A2` observe a payment with the
same hash, they can confidently conclude that they are on the payment path. However, the adversary cannot yet determine with enough
certainty whether `S` is the sender and `R` is the receiver. Yet when combined with other correlation attacks or by examining network topology,
the adversary can establish such a conclusion with enough probabilities.

**Prevention:**

Fortunately, payment hash correlation is soon expected to be fixed with point time lock contracts (PTLCs)[1]. Each payment hop will use a
unique lock contract point, so there will be no information that can correlate different payments.

[1] https://bitcoinops.org/en/topics/ptlc/

## Amount correlation

Payment amount correlation is only slightly better than hash correlation in terms of privacy because the receiver amount on each hop
is mixed with the fees of all the downstream nodes. Fees on LN are just a tiny fraction of the amount, so for the attacker fees are
not an issue, especially in combination with timing correlation attacks.

Single-path payments are the most vulnerable to amount correlation attacks. Besides the fact that nodes `A1` and `A2` will see a payment
with roughly the same amount, node `A2` depending on the payment amount, can conclude that `R` is a receiver. For instance, if the
receiver is a shop that sells some product for X satoshis, and if the attacker sees a payment of around X satoshis, he can be sure
that this payment goes to that shop node.

Multi-path payments have better privacy because the amount is now split into multiple parts. The attacker can not easily find out
what product the sender is buying. But there is still a potential correlation factor, depending on how we split the payment amount.
If we split the payment into equal parts, the attacker still can find out if a partial payment is multiple of the price of some of
the shop products. Also, those sub-payment paths will be easily distinguishable by the amount, just like in the case of single-path payment.

**Prevention:**

So, what can be done to de-correlate sub-path payment amounts?

Rather than splitting the payment amount into equal parts, we split it into predefined values. For instance: 10k, 20k, 50k, 100k,
200k, 500k, 1000k, ... satoshis. Just like physical cash. By doing so, every individual payment is part of a much larger anonymity
set consisting of all the payments at that moment. Using this approach, we can split a payment into as many paths as needed until
we get to the exact number of satoshis. Splitting the payment amount into enough sub-paths to get an exact amount might be an overhead
if the current satoshi value, fees, and latencies are taken into account. To limit the number of routes, we can opt to overpay slightly,
however, a more efficient approach can be implemented. As previously discussed in the context of [Payment Route Reservation](./Payment%20Route%20Reservation.md),
it is imperative for the receiver to return a non-zero fee,
so that his neighbor node does not find out he is the receiver. The sender will use these non-zero fee values to pass an exact change
to the receiver, eliminating the need for the overpayment.

For instance, if there is a payment of 460,645 satoshis, the sender can split the payment into four different sub-payments:
200k, 200k, 50k, and 10k, with the change being 645 satoshis. Sender will then create four reservation routes for 200k, 200k, 50k, and 10k.
However, the onion information intended for the recipient will contain different values, such as 200251, 200214, 50108, and 10072.
These added amounts (251, 214, 108, and 72) instruct the recipient to employ them as a `fees_sum` for the `reservation_success` call.

Now all sub-payments of all users are mixed together using pre-defined payment reservation amounts. Thus, the anonymity set is increased substantially,
especially if LN is processing a huge number of transactions per second.

The drawback in splitting payment amounts into predefined values is that we might need to create more
[redundant payment routes](./Payment%20Route%20Reservation.md#redundant-multi-route-payment-reservation) to match
the reliability of sub-payments split into equal parts. When we split into equal parts, every redundant payment path can be used to
replace any other failed paths. But in the case of predefined values, redundant sub-payment can replace only the sub-path with the
same payment amount.

## CLTV correlation

CLTV correlation is not as serious as hash or amount correlation because it doesn't explicitly connect payment routes through
adversary nodes. CLTV value does give a sense of closeness to either sender or receiver. However, if hop CLTV delta values
used are exactly as the one nodes gossiped, then an attacker can potentially determine the payment path as well. This is
especially true when CLTV deltas are used in combination with timing correlation, allowing the attacker to calculate all
cltv path combinations between `A1` and `A2` and deduce if they are on the payment path.

**Prevention:**

With payment route reservations, CLTV delta gets shuffled thanks to the payment route split. If there is no route split
node can return a random CLTV value around some predefined value. Thus, for the attacker's job to correlate payment with
payment route reservation, using only CLTV gets much harder.


## Timing correlation

Every low-latency network is susceptible to timing correlation attacks. The adversary observes the network traffic between
the potential sender and receiver, and the time of the transactions is used to make correlations. This type of attack can be
carried out even without an LN node if the adversary can monitor the surveilled node's network traffic. Low-traffic networks are
more vulnerable to timing attacks than high-traffic networks. As the number of LN users and payments continues to grow, the
potential payment set will increase, making it increasingly difficult to correlate payments using only timing analysis.

**Prevention:**

To mitigate timing correlation attacks, a possible solution is for each node on the route to introduce a small random delay
for privacy-oriented payments. This approach can make the attacker's job somewhat harder.


## Conclusion

What the attacker would most likely do is use a combination of amount, cltv, and timing correlation attacks. Each correlation
attack will give some probability, and cumulative probability might reveal the payment route and, in the worst case, the sender
and receiver. Therefore, it is crucial to minimize the probability of success for each attack to ensure the highest possible payment privacy.
