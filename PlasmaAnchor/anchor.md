## Anchor: Finality Gadget on Tendermint

### Motivation
In order for Plasma to work, the sidechain must be aware of the rootchain smart contract state. The sidechain must know what has been deposited so that they can be spent, and must also know what positions have exited in order to prevent them from being spent on the sidechain. The main difficulty in acheiving this is the fact that Ethereum has probabilistic finality while Tendermint has instant finality. Thus a Tendermint chain communicating with a PoW chain may commit something on the basis of unfinalized state in the PoW chain.

Dealing with rootchain finality is much harder when we have multiple sidechain validators that may have differing views on the roochain. Having a single node act as an oracle for the sidechain validators would cause a central point of failure. Including the anchor mechanism in the sidechain consensus state introduces a lot of complexity and is not very modular. Anchor is a seperate process that will run Tendermint consensus to determine the latest rootchain state that a quorum of validators deem finalized. This will be used to communicate to the sidechain the latest finalized rootchain state to process deposits and exits.

The sidechain must track `Deposits` in the rootchain so that users can spend finalized deposits on the sidechain. The sidechain should also track `Exits` in order to delete positions that are no longer spendable (space-saving) and prevent users from spamming the chain by spending exited positions.

### App State of Anchor
The app state of anchor contains the following data in substores:

Rootchain Blocks: `Map{Number -> BlockHash}`

**Note:** RootChain Blocks are continuous from 1 to the current objectively finalized value, thus it is simply a list represented in an sdk store.

Let the current objectively finalized block be `B` and the currently objectively finalized deposit be `N`.

### Msgs

Msgs in anchor are submitted by each sidechain validator once per block

```go
// Implement sdk.Msg interface
type AnchorMsg struct {
    // Msg contains all subjectively finalized data 
    // since last objectively finalized data
    RootchainBlockHash [][]byte
}
```

For validator A with subjectively finalized block `bA` and subjectively finalized deposit `nA`.

Then ValidatorA would send `AnchorMsg` with RootchainBlockHash including BlockHashes `B+1` through `bA`, Deposits `N+1` through `nA`, and all new subjectively finalized exits that are not objectively finalized.

The length of each list should have a maximum size. If the validator has more subjectively final data then the maximum list size, she should split that data in smaller chunks. Thus if the max size is 50, and the validator has 60 new subjectively final blocks. She will submit the first 50 blocks in her message; and once at least the first 10 are objectively finalized, she will include the latest 10 subjectively final blocks in the next message.

If the Validator does not agree with the Anchor Chain
i.e., there exists some blocknumber k < N s.t. Anchor's kth block has a different blockhash then the validator rootchain node's kth blockhash; then the validator does not send an `AnchorMsg` until his rootchain view converges with the AnchorChain.

### Handling

##### Ante
Antehandler makes sure msg sender is a sidechain validator and validator has not already sent `AnchorMsg` in this block

##### Handler

The AnchorMsg Handler then stores all the above data into a transient store called `Subjective`.

##### EndBlocker

At the end of the block, `endBlocker` loads all the lists validators inputted which is stored in subjective

`endBlocker` then starts with the first element of each list and checks if more than 2/3 of validators agree on a single value. If they do, it is added to the appropriate objectively finalized store. Then endBlocker continues through each element of the list objectively finalizing it by adding the data to the checkpointed stores until there isn't a 2/3 quorum on the next value.

Thus, `endBlocker` calculates the Maximum Continuous Intersection of BlockHashes and adds them to the blockhash store.

At the end of this for loop, `endBlocker` wipes the data in `Subjective` store so it can be reused in next block.

Since there are a limited number of sidechain validators and the lists are capped in length, this isn't an infeasibly expensive calculation to happen in `endBlocker`.

### Subjective Finality

In order to preserve the notion of subjective finality, validators define a parameter in their anchor node's `config.toml` file called `subjective-finality`. This is how many blocks the validator believes need to be added after a piece of data before that data is presumed finalized by the validator. The Anchor node will then poll the rootchain node continuously and pull blockhashes that has been subjectively finalized according to the above parameter but not objectively finalized by Anchor and construct an `AnchorMsg` submitting that data to Anchor chain.

To tune their `subjective-finality` parameter, validators simply stop their anchor node, update their `config.toml` with their preferred `subjective-finality` bound, and restart the anchor node.

### Catastrophic Failure

In the current design there is no recovery mode if the anchor chain diverges from the eventually finalized rootchain. If this happens the sidechain is irreparably compromised, thus everyone should exit. To prevent this, validators should set sufficiently high `subjective-finality` bounds.

### Key Terms
Objectively Finalized: Data on the rootchain is objectively finalized if the anchor chain has checkpointed that piece of rootchain data

Subjectively Finalized: Rootchain data is subjectively finalized for a validator if the validator considers that data sufficiently final. For example: Validator A considers rootchain data final if she sees that data on her rootchain node 6 blocks deep. Thus any data 6 blocks deep is considered subjectively final by A even if the anchor chain has not checkpointed yet.

Maximum Continuous Intersection: the longest continuous intersection starting from the latest objectively finalized block agreed upon by 2/3 of validators.

Ex 1:

A: `{1 -> A43..., 2 -> B32..., 3 -> C67...}` // Latest blocks seen by A since last objectively finalized blocks

B: `{1 -> A43..., 2 -> B32..., 3 -> D43...}` // B sees a different block 3

C: `{1 -> A43..., 2 -> B32...}` // C only sees 2 blocks since last objectively finalized blocks

D: `{1 -> C54..., 2 -> E13..., 3 -> D52...}` // D currently sees an entirely different version of the blockchain

MCI: `{1 -> A43..., 2 -> B32...}`
