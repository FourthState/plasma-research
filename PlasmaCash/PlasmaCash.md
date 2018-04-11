# Plasma Cash 

The original spec for plasma cash was posted [here](https://ethresear.ch/t/plasma-cash-plasma-with-much-less-per-user-data-checking/1298) on ethresearch by Vitalik.
A simple spec written by Karl can be found [here](https://karl.tech/plasma-cash-simple-spec/)

### Review:
Plasma Cash introduces non-fungible assets onto the plasma chain to allow for the sharded client-side validation. Users are no longer required to validate an entire block, but rather only the tokens they own. This is done by using a merkle patricia tree where the transaction index of a block is the token id. See the simple spec for the functionality of the withdrawal and challenge mechanism's. 

### Improvements:
* Confirmation Signatures are no longer required
* Sharded client-side validation
* Mass exits are no longer needed

Confirmation Signatures are not required since the exact location of a token transaction within a block is known. Furthermore since tokens are non-fungible, users would only be forced to withdraw if a validator attempts to spend their tokens by creating an invalid transaction and including it in a block. Unlike Plasma MVP where the presence of a malicious validator causes all participants to exit, only the tokens being attacked by the validator would be forced to withdraw. 

### Concerns:
* Tokens are non-fungible
* There should not be bonding for exits as a validator could decieve a user into attempting to withdraw a token that has already been spent 

### Potential:
Plasma Cash and Plasma MVP make different trade offs for scalability and security. Plasma Cash makes a major improvement by allowing for sharded client-side validation, but its use cases are likely limited. Solutions to the indivisibility of tokens such as through mergning and splitting or state channels would greatly increase the functionality of Plasma Cash.  

### Implementation using Cosmos-sdk:
Unlike Plasma MVP, the underlying IAVL tree could be used to implement Plasma Cash as the transaction index would be the hash of the token ID. Proofs of token inclusion/non-inclusion would need to be stored for every block. The majority of the changes based on our current implementation of Plasma MVP would be done on the root contract. These changes include adjusting merkle proof validation, challengeExit functionality and adding a mapping from deposit to token id.

### Fees:
1. Fees associated with a token ID. Overtime the total fee will approach the value of the token
2. Fee balance on the root chain for each user. Users will spend from their fee balance and this would be included in their transactions. 

An implementation of merging and splitting on the Plasma Cash chain could also provide a solution to fees. 

## State Channels on Plasma Cash
This section is inspired by the State Channel proposal on ethresearch originally posted [here](https://ethresear.ch/t/state-channels-and-plasma-cash/1515)

### Purpose: 
State Channels on Plasma Cash allow for the exchange of portions of a non-fungibile token within a State Channel as well as decreased time to finality. Other commonly associated advantages with State Channels (scalability, privacy, light-clients) would also be beneficial to the Plasma Cash Chain. 

### Concept: 
A Plasma Cash chain will allow for the deployment and closure of cooperative state channels. The closure of cooperative state channels **cannot** allow for the partial withdrawal of a token unless the Plasma Cash chain implements the splitting and merging of tokens. 

A state channel address A on the root chain is responsible for resolving the disputes for:
1. uncooperative closures
2. cooperative but partial closures

A cooperative but partial closure is where participants agree on the state of the channel and all agree to close it, but the state of the channel will not disburse tokens in the denomination of the Plasma Cash tokens locked on the channel. In this case, the participants could withdraw off of the Plasma Cash chain and recieve their partial disbursements from state channel address A on the root chain. This situation is unlikely to occur as the participants would be forced to wait at least 2 weeks (Plasma Cash withdrawal time) to recieve their funds even though the closure is cooperative. A valid method for splitting and merging of tokens could resolve this issue by allowing for the participants to split the locked Plasma Cash tokens. 

### How it works:
On the root chain, a state channel contract would need to be deployed at address A. In the case of counterfactual state channels, one could simply deploy a bond manager at address A and a channel registry at address B. A channel with N participants, p1, p2, ..., pN, would execute a multi sig transaction on the Plasma Chain to lock up the non-fungible tokens. Each participant must have a copy of the exit transaction sending the tokens to address A on the root contract. The state channel would now be considered deployed. In the case of a cooperative non-partial closure, the participants would simply broadcast the updated exit transactions to the Plasma Cash validator sending the non-fungible tokens to their new owners. In this case, the state channel was created and closed all off chain. 

Uncooperative Closure procedure:
1. A participant attempts to withdraw the locked tokens to address A  
2. The challenge period goes by (2 weeks)
3. The root contract sending the tokens to address A
4. Address A resolves the uncooperative dispute on the root chain

In the case of a counterfactual state channel, the root contract would be required to execute the logic to lookup the counterfactual address C in the channel registry and then provide the necessary information to address A to resolve the uncooperative closure. 

### Implementation:
The plasma chain is only required to support multi sig transactions in order to support state channels. 
In order to implement counterfactual state channels with plasma cash, adjustments to the root contract would need to be made to allow the lookup of counterfactual address C in the registry and deployment any necessary smart contracts. Existing counterfactual state channel research found [here](https://github.com/SpankChain/general-state-channels) can be used to implement all other necessary features. 

### Uses:
* Chess, Poker, or any game where the winners will be awarded non-fungible tokens within the state channel. 
* Unidirectional payments over time, for example buying coffee over time until the value of the non fungible tokens is reached. 

Notes:
Large entities or organizations with liquidity could take advantage of state channels by opening a new channel when the other participant would like to withdraw only a portion of the non fungible token. 

Example:
Alice and Bob open a state channel with 1 eth each
Alice spends .5 eth
Alice would like to withdraw her 1 eth
Bob has a of various size tokens
Bob opens a new state channel in which he deposits .5 eth and alice deposits 0 eth
Bob and Alice agree to close state channel 1 by sending the originally tokens to Bob. They also agree to close state channel 2 by sending the tokens to Alice. 
Alice now has .5 eth and Bob has 2 eth