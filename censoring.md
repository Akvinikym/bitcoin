# Bitcoin Censoring

## Introduction

This document covers the basic architecture of Bitcoin and this particular full-node implementation, and how they can be exploited in order to censor transactions or peers. As an addition, it will be shown how to transform this implementation into an extremely light node to be able to poison the Bitcoin network or its part.

## Basics of Bitcoin Architecture

Let's quickly put together the parts which are relevant for our cause.

Bitcoin is a PoW protocol, which means there is a set of peers which share some pool of pending transactions; technically, *every node has its own memory pool of unconfirmed transactions*. The peers build blocks from those transactions and propagate them to the network after solving the computational problem. Roughly, we can name two kinds of peers.

`Light node` is a type of peer which does not participate in the PoW algorithm, neither it keeps the state of the ledger (though it can, of course). Consider a Bitcoin wallet or some web-service, for instance. Its purposes may include:
- transaction validation, by itself or with the help of the full node
- transaction submission, sending it to the full node(s)
- tracking states of transaction(s) of interest, again, asking the statuses from the full node(s)

`Full node`, on the other hand, is the working member of the Bitcoin network. It *must* keep the full state of the ledger from the genesis block to the current one (roughly 500Gb as of now), even though it's usually enough to keep the latest N blocks to be able to validate the incoming transactions. Its purposes may include:
- transaction validation against the known ledger state
- forming the blocks of transaction and solving the computational problem (mining)
- propagating the blocks it has formed
- relaying the valid transactions and blocks submitted by other peers (both full and light nodes) to the set of known peers

To finish up the section, we also should mention how the nodes communicate with each other. For this purpose they use the P2P protocol. From this point of view the typical life cycle of the full node would be:
1. Start up and get the list of known, trusted peers (seed)
2. Try to establish connections with those peers by exchanging `version` and `verack` messages
3. Once the connection is established, the full node will request block information it misses judging from the current height value provided by other peers
4. When the initial synchronization finished, the node is ready to accept transactions and block, validate and relay them, etc.

## Censoring the Transactions

Now, let's see how we can exploit the architecture described in the previous section in order to censor the transactions - not allow some set of transactions into the network based on some predefined or heuristic data.

What allows us to do so is the fact that every node has its own memory pool, contents of which are not necessarily shared between the nodes. Of course, the honest nodes will try to propagate them into the network, but we're not talking about them. So, let's assume some peer P wants to send us a valid transaction T. According to the protocol, we must do the following:
1. Accept a connection from `P` exchanging the `version-verack` messages; necessary only once per peer unless the connection dies
2. Receive the `T`
3. Validate the `T` against the ledger state
4. Add `T` to our mempool
5. Respond to `P` that `T` was accepted
6. Relay `T `to other nodes we know

Now, what we *actually* can do if we try to implement censoring:
1. Accept a connection from `P`:
    - we can start our censoring at this stage already, retrieving IP address of the peer and version of the Bitcoin protocol:
      - `IP address` can be checked against our blacklist
      - we can also get the `hostname` of the peer via reverse DNS lookup and check it against the blacklist
      - `version of Bitcoin protocol` sometimes may contain hints of the application used; for instance, during experiments we've encountered `/btcwire:0.5.0/btcd:0.23.3/` string which hints us that it's a Go implementation of the full node
    - the mechanism of blacklisting the nodes is already implemented in this project, but it's a bit different from what we're proposing:
      - the implemented mechanism is "honest", thus it just rejects connections of the peers in the blacklist, so the peer would not even try to send us messages
      - we are going with "not-so-honest" approach, silently marking this peer as blacklisted, but responding to him that everything's OK
2. Receive the `T`
    - we check it against some blacklist conditions which may include:
      - `public key` with which the transaction was signed
      - peer it has come from
3. Validate the `T` against the ledger state
    - blacklisted transaction is not verified
4. Add `T` to our mempool
    - blacklisted transaction isn't added to the mempool
5. Respond to `P` that `T` was accepted
6. Relay `T `to other nodes we know
    - blacklisted transaction is never relayed to other peers

By following this algorithm we have prevented the transaction from being validated, accepted and relayed to other peers. If we are the only one who received this transaction, it'll be forever lost. One can take a look at the [example](https://github.com/Akvinikym/bitcoin/blob/censoring/src/net_processing.cpp#L3903) of censoring implementation to see how easy it is.

In some blockchains the concept of "slashing" is implemented to punish the nodes not complying with the protocol. In Bitcoin such a concept exists, but it's implemented on P2P level, so that if malicious behaviour of one peer is detected by some other peer, it will disconnect that peer from itself in the worst scenario.

However, our node is not subject to such slashing as it never does actions to be banned for. Such actions include:
- providing (relaying) incorrect block: [link](https://github.com/Akvinikym/bitcoin/blob/censoring/src/net_processing.cpp#L1634)
- providing (relaying) incorrect transaction [link](https://github.com/Akvinikym/bitcoin/blob/censoring/src/net_processing.cpp#L1689)

So, the only thing we need to make sure is that we're relaying blocks and transactions only from the set of reliable peers. Complete silence could also be an option, but some heuristics can detect such cases as well, and the nodes would still ban us.

## Deploying the Evil Nodes

Let's assume we would like to poison the network with our evil nodes. In the worst case (for us) the honest peers will spend resources such as computational power, network bandwidth, etc. to relay transactions and blocks to us, thinking that we'll propagate them further. In the best case sometimes our nodes will be the only ones knowing about some transactions, and we can easily "loose" them, preventing the network from functioning properly.

As it was said, starting up an honest full node required ~500Gb of storage plus some power to validate the transaction and, of course, mine the blocks. However, with some modifications of the code those requirements are no longer necessary:
1. Do not synchronize with the ledger state, removing the storage requirement
    - if we are asked to validate the transaction, we can either timeout, or even ask another reliable node (or our own "honest" full node instance) to validate it for us and return the result to the one who asked if we want to play the role of an honest node completely
    - we can also be asked for our mempool; again, asking another node for its pool and returning the result is not a problem
2. Mining can also be disabled

So, the only requirements left are some computational power (to participate in P2P protocol) and network bandwidth. Both are pretty much negligible if we're talking about one node. This means that we can easily deploy hundreds of such evil nodes, make them acquainted with some set of known honest peers and then start wasting their resources or even making the network itself slower and less reliable.

## Detecting the Evil Nodes

Last topic to be discussed is detecting such evil nodes.

Let's return to the example of `P` sending `T` to our evil node. After we have returned confirmation of `T` being accepted to our mempool, `P` can actually ask us for the contents of our mempool and check if `id(T)` is present there. If it's not, it means that we lied about the acceptance, and `P` can mark us as unreliable.

In order to solve this problem, we can still try to include the `T` into mempool, but stripping it out of the `getrawmempool` request for any other node except for the `P` who sent it. However, if `P` works in pair with some other peer, they can notice that contents of mempool changes depending on the one who asks, and that's a red flag.

Another way to detect such an evil node could be to ask it to validate an invalid transaction. If a naive approach is taken when our evil node always responds with "success", this would mean that it performs no validation, and can be marked as unreliable.

## Conclusion

In this document we have explored the ways of building the evil node which can censor transactions or even peers without being notices unless you know what you are looking for. This can be dangerous for Bitcoin network making it slower and less reliable.
