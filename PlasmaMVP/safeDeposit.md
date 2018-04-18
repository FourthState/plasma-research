## Safe Deposits on Plasma

A naive implementation of deposit allows a malicious validator to steal a user's deposit.

To deposit on a child chain:

1. The user validates the child chain up to latest block n
2. The user submits a deposit transaction to the Ethereum rootchain

A malicious validator want to steal the user's deposit. The validator sees the user's deposit transaction in the transaction mempool, and can steal the deposit by doing the following:

1. Create an invalid block of size n + 1 that contains a UTXO created out of thin air that is equal in denomination to the user's deposit.
2. Submit the invalid submitBlock transaction to Ethereum rootchain with a very high gasPrice
3. There is some probability that the Ethereum rootchain processes invalid submitBlock transaction before user's deposit transaction. Everyone on the child chain including the validator submits a startExit transaction after seeing invalid block.

Since the deposit transaction occurs in a block after the invalid block, it will have higher priority than the invalid block and will be processed later.

The validator can successfully withdraw his invalid UTXO, and the smart contract is drained before it can process the user's deposit.

Solution #1:

Assign all deposits a priority = 0. This does not work because it will be easy to DOS a child chain by simply depositing and exiting repeatedly. The attacker loses gas, but for a relatively low price he can indefinitely stall exits on the child chain.

Solution #2:

Change deposit function to ``deposit(validatorBlockNum, txBytes)`` so that user can commit to the latest validatorBlockNum they have seen. If a new validator block gets added before the deposit transaction gets processed, the deposit reverts and the user may resubmit the transaction if the new block is valid.

This may require users to deposit multiple times before succeeding.

Solution #3:

Have validator-submitted blocks increment in multiples of 1000. Fill in deposits at positions starting from 1 that are not multiples of 1000. This allows the deposit to have a much lower priority than any recent submitted block however the deposit amount is limited to ~1000*(number of submitted blocks). For the vast majority of child chains this will not be an issue.
