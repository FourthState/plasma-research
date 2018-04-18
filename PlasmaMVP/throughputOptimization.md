### Increase Throughput of Plasma

Rather than submitting a block to the Ethereum rootchain and waiting for presumed finality before submitting the next block; we propose changes to the submitBlock function that allow blocks to be added before the previous blocks get finalized. We effectively allow the blocks to be added "asynchronously".

The submitBlock function changes to:
``submitBlock(blockNum, root, prevRoot, sig)``

blockNum: The blocknumber that is being submitted

root: Merkle root of transaction tree in block

prevRoot: Merkle root of transaction tree in previous block (blockNum - 1)

sig: Signature by the authority over the hash of blockNum, root, prevRoot.

The contract keeps track of the largest blocknumber seen, call this ``latestBlock`` as well as a childchain mapping from ``(uint blocknumber => Block struct)``

For a given call of submitBlock:

#### Case 1: Submitting a new Block
If the blockNum is ``1 + latestBlock``, we simply check that ``ChildChain[latestBlock].root == prevRoot``

#### Case 2: Filling in reorged blocks in the middle
If a block (with blocknumber k < latestBlock) in the middle gets reorged out of existence, one can fill in the gap by creating ``submitBlock(k, kth_block_root, k-1th_block_root, sig)``

Here we check that ``ChildChain[k + 1].prevRoot == root``, this prevents a malicious validator from submitting a block at height k that they did not originally submit in the case where kth block gets reorged.
If multiple blocks are reorged in a row, we simply resubmit the missing block with highest missing height and move towards the lowest missing height.

#### Case 3: n latest blocks get reorged out of existence
Lastly, there is the case where the last n blocks get reorged out of existence. This is analogous to have one very big block getting reorged out of the existence, and the blocks must be added back in order from lowest height to highest height since latestBlock has been changed to ``latestBlock - n``

In all cases, we check that ``sig`` is the authority's signature over ``keccak(blockNum, root, prevRoot)``. The reason we include this signature is so that anyone can submit blocks to the rootchain so long as they have the authority's signature. This prevents validators from refusing to fill in gaps in the case of a reorg.
In all cases, we prevent a submitBlock from replacing an already existing key-value pair in the chilchain mapping.

Pros:

This approach allows the childchain can continue creating blocks at full speed without waiting for presumed finality on the rootchain.
The latency for a final transaction hasn't changed because one must wait for the block the transaction was included in to be submitted on to the rootchain and then become presumed final on the rootchain before sending their confirm signatures to the recipient.
However, the throughput has vastly increased because the child chain does not have to wait for the root chain. Instead, the child chain creates and propagates blocks at full speed and fills in gaps on the rootchain in the case of a reorg.

Cons:

We must add a new way to challenge an exit. If a user tries to exit at block k, one can challenge the exit by proving that block j (j < k) has been reorged out of existence.
This is because the exiting transaction in block k may be invalid given block j.

Currently, we implement safe deposits by queuing pending deposits and upon a submitBlock transaction the pending deposits are added to the childchain and then the validator-submitted block is added after.
Since there is no more waiting between validator-submitted blocks (in fact it is possible to have multiple submitBlock transactions in the same Ethereum block), we can no longer expect that the deposit gets added to a pending queue before the next submitBlock transaction comes in.

Possible Solution:

Number validator blocks in multiples of 1000, fill in deposits at all heights that are not divisible by 1000.

