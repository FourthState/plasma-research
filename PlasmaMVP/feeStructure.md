# Plamsa MVP Fee Structures

## Problem:
There needs to be an incentive for validators to run a full node on the plasma side chain. The obvious economic incentive is to charge fees on any spent utxo. Due to the high volume of transactions per block, 2^16 slots, the fees per transactions will remain relatively low.

**GOAL**: Due to the role of confirm signatures, we need to ensure that the proposer responsible for a block can securely withdraw their rightfully earned fees in the scenario of a mass exit.  

## Potential Solutions:

### Fee Budget:
We can require all participants of the side chain to allocate and bond a fee budget of Ether on the rootchain. In addition to the previous constraints on what constitutes a valid transaction, we impose that any users attempting to spend a transcation must specify a fee denomination in the txBytes that is checked against the rootchain. Per every transaction processed, the validator for a current round will deduct from all corresponding participant's fee budget. However, this opens up another order of complexity that we would like to avoid. The main purpose of Plasma is to scale ethereum by maximizing activity off chain but this solution introduces too many transcation calls from the validator to the rootchain.  

Does this change introduce a new challenge mechansim? Are there checks to ensure that the validator only deducts the fee amount as specified in the txBytes? How much code will this change/introduce in the rootchain?  

  For these questions alone, this solution is not sufficient


### Fees Stored in Utxos
In our earlier discussion of a plasma side chain vulnerability, [Ethereum Research Post](https://ethresear.ch/t/plasma-vulnerabiltity-sybil-txs-drained-contract/1654), regarding the plasma mvp spec not specifying the requirement of storing confirm sigs on-chain, we can exploit this to solve our issues with fees!  

By requiring correct confirm signatures be included in a transaction, spending a utxo on the child chain allows anyone watching to successfully challenge an invalid attempt to exit any ancestor of that utxo. We claim that that this requirement makes it very unlikely to exit a invalid uxto successfully.  

We can take advantage of this by having a validator collect fees corresponding to the **inputs** of all utxos in his/her block. As specified in the txBytes, the fees must be deducted from the inputs. A valid spend message on the child chain follows this equality:  

    Amount1 + Amount2 - Fee = Output1 + Output2  

The fees are deducted from the first input and determined by the participants and not validators. This allows validators to choose which spend messages to include from the mempool. However due to the high volume, the market will most likely settle at an average low transaction fee  

While processing each tx, the validator aggregates all input fees, and creates a new utxo with no history owned by them as the very last utxo in the block. By placing this new utxo last, all watchers of this child chain can verify the validity of the denomination of the new utxo by aggregating the fees for each tx themselves. Malicious activity of this last tx is grounds for a mass exit of the child chain. Placing the new utxo last also ensures that the validator has the lowest priority to exit with that block as the current head.  

There are two different cases in which we assure that the validator is guarunteed their fee utxo for a given block.
1. For every transaction included in a block, any spend of the new utxos allows the validator to challenge an invalid exit that would steal the validator fees
2. For transactions included in a block but not spent, perhaps the confirmsig was never broadcasted, the previous owner retains the right to withdraw their previous utxo. In this scenario, we uphold the right for the previous owner to exit but they must commit to any fee payed by broadcasting a spend message by including the fee amount in startExit. The validator can challenge an invalid exit that does not commit to fees payed from the first input by submitting a merkle proof that the user had payed a different fee amount than described in the startExit. In a successful challenge, the validator keeps their earned fees and also the startExit bond.  

Note: The validator has the ability to steal all current fees in the mempool by submitting invalid blocks. However, this is not an issue because there is zero incentive to steal fees. By mining honest blocks, the validators will eventually claim all fees in the mempool. Additionally, by attaching fees to spend messages in the network, this incentivies users to only broadcast transaction in which they will eventually send confirm signatures.  

**BONUS**: Because the brand new utxo is owned by the proposer of the current block, this solutions scales to any consensus mechanism with more than 1 validator. All without changing much of the root contract code at all!

