# Superbolt Proposal

## A Power Grid for the Lightning Network

### Introduction

Currently, the Lightning Network (LN) user experience is far from retail ready. Inbound/outbound channel liquidity issues and nodes dropping out sporadically mean that many payment attempts will not succeed.

A BOLT specification which would enforce a stricter set of rules for nodes to follow and which would ensure uptime, connectivity, throughput (sufficient liquidity), and channel management and rebalancing would move the needle greatly in the direction of a LN UX which can go mainstream. If LN is currently resulting in many “gutter balls” in terms of payment routing failures, Superbolt is designed to be like bowling with bumpers.

### The Problem

In [Christian Decker’s talk](https://www.youtube.com/watch?time_continue=1&v=HtU7ZlxvLL4&feature=emb_logo) at The Lightning Conference (Berlin, October 2019) he presented some frustrating statistics from a study he conducted to test payment routing success or failure on LN using payment probes. Some of the salient points:

1. 48% of payment probes failed to find a payment path to the targeted node. This is likely because either the node itself was offline or a connecting node along the path was offline.
2. Ignoring the 48% of unreachable nodes, payment success rate was ~66% on the first payment attempt. With multiple retries for the payment, success rates reach ~80%. This means that even for nodes which are available and reachable, 20% of payments are not able to complete. This is not good.
3. Stuck payments (initiated but not completed) because a node died along the path occurred at approximately 0.19%.

It should go without saying that a payment network which works less than 50% of the time presents a user experience which will never catch on with the vast majority of the total addressable market. If you are flipping a coin every time you attempt to use a payment method, you will quickly abandon this method for one which works reliably.

To make matters worse, Christian was attempting to route micropayments for his testing of the network, so the above numbers may actually be optimistic when you consider routing larger payments (say a few hundred dollars worth of BTC). For example, a route may be found for the desired payment but if there is insufficient liquidity on one of the hops (either due to insufficient channel capacity or because of inbound/outbound liquidity issues), then the payment will fail.

In summary, there are several fundamental problems with LN as it is currently functioning:

1. **Uptime:** Nodes are not sufficiently incentivized to guarantee they remain online to route payments.
2. **Connectivity:** Node connectivity to LN is entirely left to the discretion of node operators who may be underconnected or have connections with poorly operated nodes.
3. **Throughput:** Node channel capacity is frequently insufficient due to low total capacity and/or inbound/outbound liquidity snags.
4. **Channel Rebalancing:** Channel inbound/outbound capacity disruptions are a point of failure in routing payments and currently must be dealt with manually.

### Proposal

We are proposing a LN BOLT, called Superbolt Network (SBN). Conceptually, this might be analogous to an “electrical grid” for LN. An electrical grid is carefully engineered with standarization of elements such as the voltage of different classes of electrical lines (Extra High Voltage, High Voltage, Low Voltage), electrical substations for stepping voltage up for transmission or down for distribution, and so on. With these standards in place, it is possible to quantify grid capacity, provide consistent service, and plan for the future. While LN is not an electric grid, it has naturally started to develop a similar hub and spoke topology. It is expected that standardization of LN node capacity and operation may bring similar benefits in performance and reliability. To that end, SBN would enforce and/or automate the following:

1. **Uptime:** Nodes would be required to maintain uptime of at least 99%. It has been proposed to use [OpenTimeStamps](https://opentimestamps.org/) so that nodes can attest to their uptime in a censorship-free way.
   - Nodes that want to declare themselves as members of SBN sign the current Bitcoin block.
   - These nodes put their signatures in OpenTimeStamps for the _next_ block.
   - When another node asks, they show the signatures they have that are attested on OpenTimeStamps.
   - The asker will disbelieve them unless the set of signatures attested on OpenTimeStamps is large enough.
   - To ensure that nodes have strong incentive to maintain high uptime, a node would be considered a valid SBN member if it can show 998 signatures within the last 1008 blocks (7 days).
   - In addition, optionally, nodes could also monitor connections using [chanfitness](https://github.com/lightningnetwork/lnd/pull/3332) from [lnd v0.9.0-beta](https://blog.lightning.engineering/announcement/2020/01/22/lnd-v0.9.html) and remove connections to nodes which fall below their own more stringent standards.
2. **Connectivity/Liquidity:** Distinct and uniform LN node classes with commensurate total node and per channel liquidity requirements. To begin, two node classes are proposed with minimum capacity requirements.
   - **Routing Node (RN):** 4 BTC total node capacity, 4 x 1 BTC channels (0.5 BTC per side) to other RNs, 8 x 0.5 BTC (0.25 BTC per side) channels to ANs. 3 of 4 RN connections should be with shared peers (i.e. A => B => C => A) while the 4th connection should be with an RN without shared peers to ensure the network is sufficiently connected (so that a group of nodes are not sequestered away from the larger network).
   - **Access Node (AN):** 1 BTC total node capacity, 2 x 0.5 BTC channels (0.25 BTC per side) to RNs. 10 x 0.1 BTC channels (0.05 BTC per side) to regular LN wallets/individual users/etc. RNs should be peers to allow off chain rebalancing via circular payments.
   - **Please note:** Additional node classes (larger or smaller) may be beneficial to network performance. However, to maintain sufficient decentralization, it may be beneficial to have a maximum node capitalization limit.
3. **Channel balancing:** To ensure that channels do not become stuck from inbound/outbound liquidity snags, the protocol would include some scripting to automate channel rebalancing.

   - [Circular payments](https://github.com/lightningnetwork/lnd/pull/3736) should be feasable given the connectivity requirements outlined above. A naive algorithm to utilize circular rebalancing opportunities in Python:
     ```python
     for channel in channels:
         if channel.local_balance >= (channel.total_capacity * 0.75):
             lowest_balance_channel = get_lowest_local_balance_channel()
             # Rebalance exactly to 50/50 local/remote balances
             payment_amount = channel.local_balance - (channel.total_capacity * 0.5)
             # Args: From channel, to channel, payment amount
             circular_payment(channel, lowest_balance_channel, payment_amount)
     ```
   - [Loop](https://github.com/lightninglabs/loop) would be used only if circular payments are not feasable and channels become eggresiously imbalanced (perhaps > 90%).

   It is in the best interest of nodes to ensure they are able to route payments, and automation of this process will make it more manageable for node operators to route payments with a very high degree of success.

4. **Uniform fee:** Ideally, payments sent on the network would be subject to a uniform float or percentage fee regardless of hops involved in routing the payment. Would it be possible to have the payer find the best route, count the number of hops between payer and payee, and then split the predetermined fee by that many nodes? A naive Python function to achieve this end might look like:
   ```python
   def split_fee_by_total_hops(payer, payee, amount, fee_percentage):
       best_route = get_best_route(payer, payee, amount)
       fee_amount = amount * fee_percentage
       per_hop_fee = fee_amount/(best_route.hops - 1)
       return per_hop_fee
   ```
5. **Self Attestation:** To be accepted as an SBN member, a node needs only to give these proofs to anyone who asks:
   - **Proof of uptime** --- by the above **Uptime** mechanism.
   - **Proof of connectivity** ---- need to prove peer/non-peer node connection requirements are met and that these nodes are SBN members.
   - **Proof of liquidity** --- which can be attested to by the Bitcoin blockchain and the LN gossip system.
   - **Proof of channel management** --- is there a way to self-attest to keeping channels balanced? Perhaps this is not necessary, but would be nice to have.

### Why?

Given the above, anyone using SBN/LN/BTC would have a close to 100% guarantee that their payment would be successfully routed from any given SBN Access Node to any other SBN Access Node up to a reasonable network-defined maximum (perhaps 0.025 BTC ~= $215 with BTC at $8600). We can be confident of this because:

1. Channel capacity is sufficient such that any one payment is an order of magnitude smaller than the nearest chokepoint (0.25 BTC outbound from AN to RN while maximum payment would be 0.025 BTC). We still need to do the hard math on this, but intuition tells us that the probability of all participants connected to any given AN attempting to route payments from said AN at the same time would approach 0%.
2. In the event that an AN or RN node channel capacity becomes unbalanced (i.e. Node A = 0 BTC, Node B = 1 BTC in given channel), channels should typically be able to use [circular](https://github.com/lightningnetwork/lnd/pull/3736) payments to unstick capacity given that nodes are sufficiently connected with common piers. In the event that off-chain rebalancing is impossible, [Loop](https://github.com/lightninglabs/loop) may be used. Ideally, both approaches would be automated such that rebalancing occurs if node liquidity is stuck in either direction beyond a reasonable threshold (<25% of total channel capacity).
3. Nodes are incentivized to stay online ready to route payments and ostracized for insufficient uptime.

The user experience envisioned with this protocol would be one where a user would go to pay with Lightning, look for the Superbolt logo, and know with near certainty that their payment will be successful. This is the experience today with processors like Visa and Mastercard and it seems unlikely that LN will achieve similar levels of reliability unless some additional requirements such as those proposed here are added to the LN/BTC stack.
