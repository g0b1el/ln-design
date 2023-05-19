
# Reusable Static Invoices

## Proposal Summary

The following proposal is intended to adapt the existing PTLC protocol into the [Payment Route Reservation](./Payment%20Route%20Reservation.md) flow.
By modifying the original PTLC protocol slightly, we get support for reusable static invoices with atomic proof of payment and payer
proofs, that don't require previous communications between sender and receiver. By doing so, we will also reduce the number
of cryptographic calculations the sender needs to perform for each PTCL payment. Then a similar approach is also applied
to HTLC, with the same benefits.

To best understand the proposal, let's first start by presenting the original PTLC protocol and then proceed to describe
the iterative changes made to it.

## PTLC protocol

Let's use Alice, Bob, Carol, and Dave example from this page: https://github.com/BlockstreamResearch/scriptless-scripts/blob/master/md/multi-hop-locks.md

        Z  <--------------------------------------------------------------------- Z = z*G

      Alice  --------------->   Bob  --------------->  Carol  ------------------> Dave
        |                        |                       |                         |
        y0                       y1                      y2                        z
              tab=z+y0                tbc=z+y0+y1                tcd=z+y0+y1+y2
             Tab=Z+y0*G             Tbc=Z+y0*G+y1*G            Tcd=Z+y0*G+y1*G+y2*G         


    * y0, y1, y2 - decorrelation secrets
    * z - proof of payment
    * tij - adaptor secret for the adaptor point between nodes i and j
    * Tij = tij*G adaptor lock point between nodes i and j
    * sig(m,Tij) := psig(i,m,Tij) + psig(j,m,Tij) + tij - is the complete Schnorr signature for the transaction between i and j.


A brief summary of the PTLC protocol:

1. The receiver, Dave, shares proof of payment point `Z`(=z*G) with the sender, Alice. This is typically done by publishing a QR code.
2. Alice generates random decorrelation secrets `y0`, `y1`, and `y2`.
3. For each hop in the route, Alice calculates a tuple `(Li,yi,Ri)` using the formula:

   `Ri <- Li + yi*G` and `Lj <- Ri`.

    * `Li` - left adaptor point
    * `Ri` - right adaptor point
    * `yi` - decorrelation secret

4. Alice generates a payment onion that includes the previously calculated lock information for each hop on the route. For Dave,
   Alice includes `(y0+y1+y2)` decorrelation secret sum(we ignore stuckless payment for now).
5. Forwarding nodes use the lock information to construct a partial 2-of-2 MuSig2 signature.
6. When Dave signs the transaction with Carol, he reveals the adaptor secret `tcd` to Carol.
7. Carol then calculates adaptor `tbc` secret using the formula `tbc = tcd - y2`, which allows her to sign the transaction with Bob.
8. Bob calculates the adaptor secret `tab` using the similar formula `tab = tbc - y1`.
9. Eventually, when Bob signs his transactions with Alice, Alice can calculate the proof of payment `z = tab - y0`.

Note that Alice does a lot of work here, especially in step 3, which requires a lot of elliptic curve additions and multiplications.
This workload increases with the length of the route, resulting in more work for the sender. While this is not typically problematic,
it can become an issue if the sender is a mobile wallet with limited computational resources.


## PTLC with reusable static invoices

We are going to make two modifications to the vanilla PTLC flow:

1) To alleviate Alice of heavy elliptic curve calculations, we will let routing nodes calculate their decorrelation secrets
   and adaptor points independently. But in such a way that the sender (Alice) and routing nodes get to the same shared
   decorrelation secret. This can be easily done because sphinx onion routing already establishes one shared secret with each hop,
   from which we can calculate as much as new shared secrets as we want.
   For instance, we can use this formula:

   `yi = hmac("decorrelation secret", sharedSecret(i))`

2) The second change we are going to make is how the tuple `(Li,yi,Ri)` is calculated for each hop.
   Currently, is made using this formula:

   `Ri <- Li + yi*G`, `Lj <- Ri`.

   and we are going to change it into:

   `Ri + yi*G <- Li`, `Lj <- Ri`.


After applying this formula to the previous example, we get the following:


      Alice  --------------->   Bob  --------------->  Carol  ---------------->  Dave
        |                        |                       |                         |
                                 y1                      y2                      y3, z
           tab=(z+y1+y2+y3)            tbc=(z+y2+y3)               tcd=(z+y3)


During the PTLC resolve phase, everything remains the same, with the exception that intermediate hops will now add their
decorrelation secrets instead of subtracting them:

    tbc = tcd + y2
    tab = tbc + y1.

In addition, Alice calculates the proof of payment `z` slightly differently:

    z = tab - (y1 + y2 + y3)  // tab will get when bob signs the transactionAB, (y1, y2 and y3) she has already calculated using shared secrets.


We will now apply the new PTLC protocol to the Payment Route Reservation flow:


            add_res(amount,onion,...)      add_res(amount,onion,...)        add_res(amount,onion,...)
      Alice -----------------------> Bob  -----------------------> Carol  ------------------------> Dave
        |                             |                              |                               |
        |                             y1                             y2                            y3, z
        |                             |                              |                               |
        |  res_succes(Z+Y3+Y2+Y1,...) |       res_succes(Z+Y3+Y2,...)|      res_succes(Z+Y3,...)     |
        | <--------------------------    <---------------------------   <----------------------------|  
        <---------------------------------------- Z -------------------------------------------------|
    Z+(y1+y2+y3)*G =? Tab
        |                                                                                            |
        |      sign_commitment              sign_commitment                 sign_commitment          |
        |       revoke_and_ack               revoke_and_ack                  revoke_and_ack          |
        | <-------------------------> | <-------------------------> | <----------------------------->|


During `res_success` call, each intermediate hop gets to the expected upstream and downstream adaptor lock point, thereby
obtaining all the necessary information to proceed to the commitment phase.

With just this change, the PTLC protocol would **NOT** be safe. Bob can send Alice a lock point for which he is the one with knowledge
of the secret. To prevent this, Dave needs to "send" `Z` point to Alice. However, Dave cannot include `Z` as part of the `res_success`
response because the same `Z` would appear on every node, reintroducing payment correlation. For the proposal on how the sender
and receiver can exchange messages check [Message Passing](./todo) proposal. Message passing action will be denoted as "send" in this proposal.

There are two benefits of doing PTLC this way:

1) Smaller hop onion and less work to be done by the sender. Alice has delegated all the heavy curve multiplications work
   to the routing nodes. Alice still has to calculate shared decorations secrets(simple hashing operation) and do one curve
   multiplications `(y1+y2+y3)*G`. As a result, this protocol should have an overall positive impact on mobile clients.
2) The biggest improvement is how Alice gets to proof of payment point `Z`. Now, new unique proof of payment `z` can be generated
   with every payment reservation. As a result, we can generate a single static invoice that can be paid simultaneously by
   multiple parties multiple times without any previous communication between the sender and receiver.


## PTLC with reusable static invoices and stuckless payments[1]


Note that the adaptor lock between Carol and Dave is built solely by Dave. This means stuckless payments would not work in
such a PTLC protocol. We can already think of reservations as a weaker form of stuckless payment. When Dave sends `resservation_success`
it can be seen as `ACK`(from the original stuckless payment proposal), and when Alice signs and revokes commitment with the
first hop(Bob) it can be seen as `key` send. But there is still a chance for payment to get stuck during `sign_commitment` phase.

If we want to add support for stuckless payments, or if we want to increase the safety of the new PTLC protocol, we can create
a third PTLC protocol by combining the previous two.

Each hop will now generate two decorrelation secrets, `fi` and `bi`:

    fi = hmac("forward decorrelation secret", sharedSecret(i))   - used during add_reservation
    bi = hmac("backward decorrelation secret", sharedSecret(i))  - used during res_success

Lock adaptor points are now calculated using this formula:

    Ri + bi*G <- Li + fi*G,  Lj <- Ri.

When applied to the Payment Route Reservation flow:

             add_res(amount,F0,onion,...)      add_res(amount,F0+F1,onion,...)     add_res(amount,F0+F1+F2,onion,...)  
        Alice ---------------------------> Bob  --------------------------> Carol  ----------------------------------> Dave
          |                                 |                                 |                                      |
          f0                              f1,b1                             f2,b2                                 f3,b3,z
          |                                 |                                 |                                      |
          |  res_succes(Z+B3+B2+B1,...)     |       res_succes(Z+B3+B2,...)   |      res_succes(Z+B3,...)            |
          | <-----------------------------    <------------------------------   <------------------------------------|
          |     Tab=(F0)+(Z+B3+B2+B1)               Tbc=(F0+F1)+(Z+B3+B2)             Tcd=(F0+F1+F2)+(Z+B3)          | 
          |<------------------------------------------- Z -----------------------------------------------------------|
    Z+(f0+b1+b2+b3)*G =? Tab


Alice will pass `(f0+f1+f2)*G` point inside the onion to Dave as a precautionary measure, just to make sure routing nodes
don't do anything out of order. Dave will check if `add_res` call from Carol contains the expected forward decorrelation point sum.

During the PTLC resolve phase, routing nodes would calculate the upstream lock secret by subtracting `fi` and adding `bi`:

    li = ri - fi + bi

Alice will calculate proof of payment z, using this formula:

    z = tab - (b1 + b2 + b3) - (f0)

This PTLC protocol now supports both reusable static invoices and stuckless payments, where Alice's knowledge of `f0+f1+f2`
secret serves as a stuckless key.


## PTLC with atomic Proof of Payment and Payer Proof

The issue with the above construct is that proof of payment is known by Alice and Dave. Dave can give `z`, for instance,
to Oscar, and if Alice comes for a refund, Dave can say that she wasn't the one that paid. Dave now can claim it was
Oscar because Oscar knows proof of payment as well.

To fix this, we will borrow the idea of a `payer` key from bolt12[2] and pay for the signature[3] proposals. Alice will
generate a unique `payer` secret, and then pass `payer_id`(=payer*G) point to Dave as part of the onion. This time Dave
will not generate proof of payment `z` randomly, but calculates `z` as Schnor's signature of the message.

    z = r + h(m|R)*d

    * d - Dave's private key
    * r - random secret, R = r*G
    * m - the message.

The signature message can be "`payer_id` send Dave `amount` satoshis", or it can be a hash of a Merkle Tree as described
in Bolt12[2]. To avoid passing the message between the payer and the sender, the message format should be defined on the
protocol level. Alice can use this `payer` secret later to sign some message to prove she is the payer.

As described above, Alice will find out Z(=z*G) when `res_success` reaches her node. But she still can't validate that
this public key commits to the signature of the message "`payer_id` send Dave `amount` satoshis", because she doesn't know R.
To fix this Dave will "send" R instead Z to Alice. There is no need to send Z anymore because Z can be calculated.

Assuming the payment reservation was successful, and Dave has "send" R to Alice, Alice can now validate:

    Z =? R + h(m|R)D, where Z = Tab-(f0+b1+b2+b3)*G.
    
    * D - Dave's public key

If the equation holds, Alice can safely sign and revoke commitment with first hop Bob(and then "send" the stuckless key
in case of stuckless payments to Dave), knowing she'll get both proofs if the receiver accepts the payment by revealing `z` secret.

### Atomic Multipath Payments

Everything remains the same as in the original PTLC proposal.


## HTLC with reusable static invoices with payment and payer proof

A similar procedure can be implemented for HTLC payments. Now `add_res` will not contain the payment hash. Payment hash `H`
will be generated by the receiver(Dave) upon `add_reservaion` call and sent back to the sender and forwarding nodes as part
of `res_success` response. Dave will also need to "send" the payment hash to Alice, to make sure routing nodes don't replace
the payment hash with their hash value.


         add_res(amount,onion,...)      add_res(amount,onion,...)      add_res(amount,onion,...)
    Alice -----------------------> Bob  -----------------------> Carol  --------------------> Dave
      |                             |                              |                            |
      |                             |                              |                            |
      |  res_succes(H,...) |        |         res_succes(H,...)|   |         res_succes(H,...)  |
      | <--------------------------    <---------------------------   <-------------------------|  
      <-------------------------------------- H,z,R --------------------------------------------|

This construction gives us support for reusable invoices with atomic proof of payment. For payer proof, Alice sends `payer_id`
in the onion, and Dave can "send" back a signed proof message (z, R), where a message can be "`payer_id` paid Dave `amount`
satoshis only if `payer_id` knows the primage of `H`. This way, we effectively link the atomic proof of payment with the payer
proof in a cryptographically secure manner.

As it is currently, stuckles payments are not possible with HTLC unless a second hash is added into HTLC.


## Different Types of Payments


Using the new PTLC/HTLC protocol, we can do a quick sketch of how different payment types will work. Here we will cover some
basic use cases, though there may be many more:

### Donations/Transfer/Key send/Spontaneous Payments/AMP...


There are multiple names for this payment type in circulation, such as donations, transfers, spontaneous payments, key sends, AMP, etc.
An example would be Alice wants to donate or transfer a certain amount of BTC to Dave. Alice also wants to receive proof
of payment and payer proof.

Dave will share his `node_id` public key, either by publishing a static QR or plain text invoice. Dave can post some additional
information in the invoice, but the only mandatory element is his public key. Alice scans the QR code and finds Dave's
public key, picks the amount, and makes the payment. Payment proceeds as described above for PTLC/HTLC payments.

Side note for wallet developers, payer proof, might be undesirable for some donations. For instance, if we want to donate
to Wikileaks, and our government is not a fan of Wikileaks, the last thing we want to have incriminating cryptographic
proof saying we "paid x amount to Wikileaks". If payer proof is undesirable, the sender can omit `payer_id`, and the
receiver will generate proof z randomly. Alternatively, the payer wallet can generate a random `payer_id` point. In either
case, the donation transaction shouldn't appear in the payment history.

### Purchases denominated in BTC

#### One static invoice per article

An example would be a vending machine. Each article in the vending machine would have a small static QR code invoice next to it.
The payer scans the invoice and then picks the quantity on his phone. If the vending machine supports multi-article purchases,
the payer repeats this process for each article. Upon completion, click "send" to pay for the entire cart. If the payment
succeeds, the vending machine dispenses the selected articles through a slot.

The mandatory fields in each static invoice are now `node_id`, `article_id`, and `price`. QR invoices can contain additional
information like article name, article picture, etc..., which can be presented to the user on his mobile wallet.
The payment flow is similar to donations, with the exception that the buyer must provide additional information to the
vending machine LN node so that the sender and receiver nodes can calculate the same proof message. The sender generates
a new unique `shopping_id` and includes this id with an array of tuples (`article_id`, `count`) as shopping information
for Dave. Note that in this case maximum size of the article list is limited by the unused payment onion space. For very
large purchases wallet can split the shopping cart into multiple sub-carts with different payments. Receiver Dave will
now check that a payment amount matches the article price sum. The signed proof of payment message now has to contain
`payer_id`, `shopping_id`, and a list of tuples (`article_id`, `count`, `price_per_article`).

Given the volatility of BTC prices, the article price on a static invoice may require frequent updates by vending machine
operators. Even if prices are denominated in fiat currency, these invoices would require periodic updates. To address this
issue, we can remove the article price from the invoice. When the user scans the first article in the vending machine,
the wallet informs the user that they need to retrieve the articles' prices to proceed. This can be implemented as a new
service, or a vending machine LN could be configured to "send" prices. Once the user receives the pricing information,
the user can proceed with the purchase as before. This way, vending machine merchants would be able to update all article
prices on their vending machines dynamically.

Lightning-based vending machines would be more cost-effective as they will not require a front-facing LCD, touch screen,
or physical buttons. Furthermore, multiple customers would be able to make purchases simultaneously, though a single
dispensing slot might result in conflicts about who bought which item. To address this issue, the vending machine could
serialize item dispensing by delaying the dispensing of new shopping carts until the previous one is collected.

#### One dynamic invoice per shopping cart

An example would be web shopping. The user adds articles one by one into the shopping cart, clicks checkout, and then sends
the payment for the whole cart.

Shopping chart invoices are dynamically created after each checkout and will contain information like a `shopping_id`, and
an array of tuples (`article_id`, `count`, `price_per_article`). The payment flow is similar to single article purchases,
except we are now sending just `payer_id` and `shopping_id` to the merchant node. We assume the merchant LN node can contact
some other merchant service to get hashed signature message using `shopping_id`. The signed proof of payment message needs
to contain `payer_id`, `shopping_id`, and a list of tuples (`article_id`, `count`, `price_per_article`). A merchant service
will check if payment for this `shopping_id` hasn't been paid yet and if the payment amount corresponds to the whole cart amount.

Very large shopping carts might not fit into QR code invoices, which are limited in size. Also, larger QR code phones with
weaker cameras could have difficulties scanning the code. This is an issue because the customer's wallet needs a whole
shopping cart so he can validate that payer proof includes every item bought. For situations like this, merchants' QR
can contain a link to the API call, which will return the whole shopping cart. Or the whole shopping chart can be "send"
later during the settlement phase before the stuckless key is sent to the receiver.


### Purchases denominated in fiat

Purchases denominated in fiat can be seen as extensions of Purchases denominated in BTC, with potentially one more additional step.

Invoice now contains amounts denominated in some of the fiat currencies. Upon scanning the QR invoice, the sender will first
convert the fiat amount into the BTC amount. The sender's wallet needs to track the BTC price, which most wallets already
do today. Then the sender makes a payment as it is denominated in BTC using the converted amount. Onion information for
the receiver should also contain the `original amount` and `original fiat currency`. Receiver checks if the reservation
amount is greater or equal to his expected conversion amount. If it is, everything continues as before. The signed proof
of payment message should now include `original amount`, `original currency`, and `BTC amount`.

If the received amount is lower than the anticipated amount, the recipient has two options. If the shortfall is less than
the `add_reservation` `max_fee`, the recipient can demand the missing amount through fees. However, if the shortfall exceeds
the `max_fee`, the recipient will accept the payment reservation, but will also "send" a message to a sender with information
about BTC missing amount for payment. The sender can decide if the receiver conversion is fair. If it is not, the sender
can cancel the payment reservation. If additional payment is acceptable, the sender will create a new route with a missing
amount. Unless the Bitcoin price drops significantly in the meantime, the receiver will accept a new reservation route,
and payment will continue as before, but this time over one additional payment route.

### Refund/ Partial Refund

Left out of this proposal, as refund requires additional constructs from Randevouz routing.


Conclusion
==========

This proposal extends the payment route reservation protocol, enabling the dynamic agreement of not only route fees, prepayment
fees, and cltv deltas but also proof of payment and payer proof between the sender and receiver. Compared to Bolt12 offers
proposal, there is no need for an additional round trip between sender and receiver to fetch the invoice, resulting in lower latency.


[1] Stuckless Payments - https://lists.linuxfoundation.org/pipermail/lightning-dev/2019-June/002029.html  
[2] Bolt12 - https://github.com/rustyrussell/lightning-rfc/blob/guilt/offers/12-offer-encoding.md   
[3] Selling Signatures - https://lists.linuxfoundation.org/pipermail/lightning-dev/2019-July/002077.html  
