# LazyTower: An O(1) Replacement for Incremental Merkle Trees

(This is a draft version)

LazyTower is a data structure. Its purpose is the same as an Incremental Merkle Tree (IMT): to allow users to incrementally append items, and to support zero-knowledge proofs of membership.

Appending an item to LazyTower has an amortized cost of O(1).

The circuit complexity of the proof is O(log N). The verification cost is O(1).

## Core Concepts

Items are appended starting from the bottom-most level. When a level is full, its entire contents are digested and appended to the level above, freeing space for new items on the lower level. For example, consider a tower with a width of 4; please watch the video:

https://www.youtube.com/watch?v=3QeJgxB9ZiQ

For instance, by storing
```text
digest([0, 1, 2, 3])
```
we can later prove that items 0, 1, 2, and 3 have been appended.

By storing
```text
digest([digest([0, 1, 2, 3]),
        digest([4, 5, 6, 7]),
        digest([8, 9, 10, 11]),
        digest([12, 13, 14, 15])])
```
we can later prove that items 0 through 15 have been appended.

We can observe that each cell in the tower is a Merkle root, which fixes 4^0, 4^1, 4^2, ... items respectively.

This is explained more clearly in the following video:

https://www.youtube.com/watch?v=7MsGTO6CuqI#t=4m45s

## Cost

On average, the lowest level is modified once per append.<br>
The next level is modified once for every 4 appends.<br>
The level above that is modified once for every 16 appends.

We can find a constant C such that the cost of any single-level modification does not exceed C.

Thus, on average, the cost of a single append is no more than:
```text
1 * C  +  1/4 * C  +  1/16 * C  + ...
= 1.333 C
```

## Privacy

When we want to prove that an item has been appended without revealing the item itself, we can use zero-knowledge proofs to demonstrate its membership.

We can use a traditional Merkle proof to prove that an item belongs to a particular root in the tower. For example: "My item is hidden in the second root of level 10, which covers 4^10 items."

If we wish to maintain complete privacy, we would need to load all the roots in the tower to prove membership, which incurs an O(log N) cost – not ideal.

Is it possible to use a single value to fix these roots so that only one value needs to be loaded when proving membership?

We improve this in two stages: horizontal and vertical.

## Improvement 1: Level Digests (Horizontal)

As mentioned earlier, when a level is full, we digest it and store the result in the upper level. If we choose a digest function that can be computed incrementally, we only need to store the latest digest instead of an array of roots.

For example, we can use a Merkle-Damgård construction combined with a ZK-friendly hash function (such as Poseidon hash):

```text
digest([a]) = a
digest([a, b]) = H(a, b)
digest([a, b, c]) = H(H(a, b), c)
digest([a, b, c, d]) = H(H(H(a, b), c), d)
digest([a, b, c, d, e]) = H(H(H(H(a, b), c), d), e)
...
```

Since each level only appends data at the end, every operation requires a fixed set of steps — loading, hashing, saving, and updating the length — without the need to recompute everything from scratch.

<img src="https://lcamel.github.io/LCamel-Notes/LazyTower/level_digests.png" width="431px">

Thus, during the proof, each level only needs to load a single level digest.

However, this still requires O(log N) values to be loaded into the circuit. Can this be further improved?

## Improvement 2: Digest of Digests (Vertical)

For the digests of each level, we can compute a vertical digest of digests from top to bottom, thereby fixing all the digests.

By placing the frequently changing lower levels at the end, we can perform localized updates by storing the prefix result, without recomputing from scratch.

That is, we store the following values:
```text
digest([d4]) = d4
digest([d4, d3]) = H(d4, d3)
digest([d4, d3, d2]) = H(H(d4, d3), d2)
digest([d4, d3, d2, d1]) = H(H(H(d4, d3), d2), d1)
digest([d4, d3, d2, d1, d0]) = H(H(H(H(d4, d3), d2), d1), d0)
```

<img src="https://lcamel.github.io/LCamel-Notes/LazyTower/digest_of_digests.png" width="350px">

When the lowest level d0 changes, we can load H(H(H(d4, d3), d2), d1) and update d0 locally.

When the lower levels d1 and d0 change, we can load H(H(d4, d3), d2) and update d1 and d0 locally.

The cost of updating a single level remains constant (load/hash/save).

## One to Rule Them All

Thus, we obtain a digest of digests that fixes each level's digest. Each level digest, in turn, fixes the roots of that level; and each root fixes the leaves of that tree, i.e., the items that were initially appended.

This means that during the proof, we only need to load O(1) data.

<img src="https://lcamel.github.io/LCamel-Notes/LazyTower/proof.png" width="751px">

Because:
1. The cost at each level remains bounded by a constant.
2. The frequency of modifications decreases exponentially at higher levels.

The amortized cost for appending an item remains O(1).

## Implementation

Below, we observe the results of the implementation.

We can see that the average gas cost for appending an item quickly converges to a constant (21000 included).

Moreover, a tower with a larger width results in a lower average gas usage, though with limits.

<img src="https://lcamel.github.io/LCamel-Notes/LazyTower/average_gas_usage.png" width="728px">

Compared to the Incremental Merkle Tree, even when accommodating a large number of items, users do not have to worry about rising gas costs, nor do they need to determine a capacity limit from the start.

<img src="https://lcamel.github.io/LCamel-Notes/LazyTower/IMT_LazyTower.png" width="625px">

Although increasing the width can slightly reduce gas usage, it also increases circuit complexity during the proof.

<img src="https://lcamel.github.io/LCamel-Notes/LazyTower/circuit_complexity.png" width="625px">

During deployment, both gas cost and circuit complexity are considered. A width of 4 to 7 is a reasonable choice.

## Conclusion

LazyTower has an average gas cost of O(1), a circuit complexity of O(log N), and does not require determining a capacity limit upfront. Projects that use Merkle Trees may consider using LazyTower.

## Acknowledgement

I conceived this idea in early 2023. Many thanks to the Ethereum Foundation for its grant, which enabled the implementation, and to the reviewers for their invaluable help!

The implementation is currently maintained by the PSE team and released under an open source license.
* Javascript: https://github.com/privacy-scaling-explorations/zk-kit
* Solidity: https://github.com/privacy-scaling-explorations/zk-kit.solidity
* Circom: https://github.com/privacy-scaling-explorations/zk-kit.circom
