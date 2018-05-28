## Rootchain Exit Ordering

Since any utxo being exited must wait for a specified challenge period, an efficient way to order exits is necessary.

It is obvious that exits should be added to a priority queue, but special consideration must be made when determining the priority of an exit.

### Naive Solution
Each position of an utxo is unique and therefore we can convert the position into a uint256 as follows:

` 1000000000*blkNum + 10000*txIndex + oindex`

Max(oindex) == 1

Max(txIndex*10000) == 655350000 

Since 1 < 655350000 < 1000000000, this uint256 value will always be unique as no two utxos can have the same position.

While the priority of an utxo being exited is unique, we find that it is possible for an attacker to exploit the ordering of these exits. 

The attack:

Assume probabilistic finality occurs in **x** blocks.

- Deposit a small amount of eth
- Every **x** blocks submit a transaction creating 2 utxos
- Repeat step 2 51 times with 1 of the created utxos

To prevent a majority of withdrawals for up to a year, submit an exit for your youngest utxo. Right before 1 week is up, submit a withdrawal for your next youngest utxo. In each case, the newer withdrawals will have a better priority and its 1 week challenge period will start fresh. The priority queue will continue to wait to process any exit until the top exit finishes its challenge period. 

### Solution \#1

Allow users to finalize an exit by passing in their priority. If the corresponding exit has waited longer than the challenge period, finalize it and add a flag to remove it from the priority queue once it becomes the next exit to be popped off the queue.

This solution works, but is less efficient than the following solution

### Solution \#2

As described in the [mvp spec](https://ethresear.ch/t/minimal-viable-plasma/426),  limit priorities to reference block numbers no older than the oldest block less than 1 week old. Therefore an exit that is almost done with its challenge period can never be put behind an exit that just started its challenge period.

This can be implemented in the same way as [OmiseGo](https://github.com/omisego/plasma-mvp/blob/master/plasma/root_chain/contracts/RootChain/RootChain.sol):

Priority is represented by a uint256:

- Left 128 Bits: `Math.max(childChain[txPos[0]].created_at + 2 weeks, block.timestamp + 1 weeks)`

- Right 128 Bits: `1000000000*txPos[0] + 10000*txPos[1] + txPos[2]`

txPos represents the position of the utxo being exited

txPos[0]: block number  
txPos[1]: transaction index   
txPos[2]: output index  

We have already shown the right 128 bits to be unique and they will not overflow into the left 128 bit until 10^29 blocks have been created. 

The left 128 bits do not need to be unique since the right 128 bits will always be unique. Instead they represent when an utxo will become exitable. 

This combination allows us to order utxos older than 1 week by the time they call startExit() and order utxos younger than 1 week by when their utxo was created.

In regards to the [FourthState implementation](https://github.com/FourthState/plasma-mvp-rootchain), the exits mapping is from Right 128 bits -> Exit and the full 256 bits of the priority will be inserted into the priority queue. 
