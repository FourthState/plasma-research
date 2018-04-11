# Plamsa MVP Fee Structures

## Problem:
A Plasma validator includes a transaction in a block and corresponding confirm sigs must be added in future transactions spending the original utxo or the utxo must be withdrawn before he can actually collect transaction fees.

## Potential Solutions:

### Fee Budget:
There is a mapping from address to max amount of fee you're willing to pay. Validator checks to see you haven't exceeded your fee budget before adding your transaction to the block. A validator can withdraw by showing user signed over a fee and that transaction was included in a block that the validator submitted. The validator has to submit an expensive transaction each time he wants to exit a single fee of a single transaction. This may be solved with aggregate signatures.

### Claim Fee:
When a Plasma validator would like to withdraw his fees, he submits the amount he would like to withdraw, and his exit is added to the exitQueue with the lowest possible priority of the current block (currBlockNum, 2^16, 1). This way, users can verify that the amount he would like to withdraw doesn't exceed the amount of fees he has earned.
* In the case of a valid fee withdrawal, his exit will not be challenged and his fee will be sent after the 1 week period.
* In the case of an invalid fee withdrawal, users will see that and initiate a mass withdrawal. Since the validator has the lowest priority, all other users' exits will be processed before his, leaving him with the true fee amount.

### Fees Stored in Utxos
Fees are included in the transaction. When a validator creates a block, he reserves a utxo at the last leaf (to give himself the lowest priority in the block) of the block where he can aggregate all the fees of this block. This utxo will look like a deposit utxo. Users can check that the amount stored in this fee utxo is correct. If the amount or transaction is found to be invalid, users will mass exit.
