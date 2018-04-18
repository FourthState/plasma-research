# Plamsa MVP Fee Structures

## Problem:
There needs to be an incentive for validators to run a full node on the plasma side chain. The obvious economic incentive is to charge fees on any spend utxo on the child chain. Due to the high volume of transactions per block on the child chain, 2^16 slots, the fees per transactions will remain relatively low.

GOAL: Due to the role of confirm signatures, we need to ensure that the proposer responsible for a block can securely withdraw their rightfully earned fees in the scenario of a mass exit.  

## Potential Solutions:

### Fee Budget:
We can require all participants of the side chain to allocate and bond a fee budget of Ether on the rootchain. In addition to the previous constraints on what constitutes a valid transaction, we impose that any accounts attempting to spend a transcation must specify a fee denomination in the txBytes that is checked against the rootchain. Per every transaction processed, the validator for a current round will deduct from all corresponding participant's budget on the rootchain. However, this opens up another order of complexity that we would like to avoid.  

Does this change introduce a new challenge mechansim? Are there checks to ensure that the validator only deducts the fee amount specified in the txBytes? How much code will this change/introduce in the rootchain?  

  For these questions alone, this solution is not sufficient


### Fees Stored in Utxos
In our earlier discussion of a plasma side chain vulnerability, [Ethereum Research Post](https://ethresear.ch/t/plasma-vulnerabiltity-sybil-txs-drained-contract/1654), regarding the plasma mvp spec not specifying the requirement of storing confirm sigs on-chain, we can exploit this to solve our issues with fees!  

By requiring that a utxo cannot be spent without the correct confirm signatures that now are included in the txBytes stored on-chain, spending a utxo on the child chain allows anyone watching to successfully challenge an invalid attempt to exit any grandparent of that particular utxo. We claim that that this requirement makes it very unlikely to exit a invalid uxto successfully.  

We can take advantage of this by having a validator collect fees corresponding to the inputs of all utxos in his/her block. As specified in the txBytes, the fees must be deducted from the inputs. A valid spend message on the child chain follows this equality:  

    Amount1 + Amount2 + Fee = Output1 + Output2  

The fees are determined by the participants and not the validators. This allows validators to choose which spend messages to include from the mempool. However due to the high volume, the market will most likely settle at an expected low transaction fee

While processing each tx, the validator aggregates all input fees, and creates a new utxo with no history owned by them as the very last utxo in the block. By placing this new utxo last, all watchers of this child chain can verify the validity of the denomination of the new utxo by aggregating the fees for each tx themselves. Malicious activity of this last tx is grounds for a mass exit of the child chain. Placing the new utxo last also ensure that the validator has the lowest priority to exit with that block as the current head.

In the case of an invalid block, all participants that spent their utxo in the invalid block keep the ability to exit before the validator (confirm sig never broadcasted and their lower blockNum will lead to a quicker finalized exit). Additionally, an honest validator can rest assured that they can always withdraw this new uxto due to their ability to challenge any transcation of all grandparent utxo and beyond because of the confirm signatures stored on chain.


BONUS: Because the brand new utxo is owned by the proposer of the current block, this solutions scales to any consensus mechanism with more than 1 validator. All without changing the root contract code at all! Simplicity FTW!

