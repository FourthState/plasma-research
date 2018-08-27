## Anchor: Finality Gadget on Tendermint

### Motivation
In order for Plasma to work, the sidechain must be aware of the rootchain smart contract state. The sidechain must know what has been deposited so that they can be spent, and must also know what positions have exited in order to prevent them from being spent on the sidechain. The main difficulty in acheiving this is the fact that Ethereum has probabilistic finality while Tendermint has instant finality. Thus a Tendermint chain communicating with a PoW chain may commit something on the basis of unfinalized state in the PoW chain.

Having a single node act as an oracle for the sidechain would cause a central point of failure. Including the anchor mechanism in the sidechain consensus state introduces a lot of complexity and is not very modular. Anchor is a seperate process that will run Tendermint consensus to determine the latest rootchain state that a quorum of validators deem finalized. This will be used to communicate to the sidechain the latest finalized rootchain state to process deposits and exits.

The sidechain must track `Deposits` in the rootchain so that users can spend finalized deposits on the sidechain. The sidechain should also track `Exits` in order to delete positions that are no longer spendable (space-saving) and prevent users from spamming the chain by spending exitted positions.

### App State of Anchor
The app state of anchor contains the following data in substores:

Rootchain Blocks: `Map{Number -> BlockHash}`

Deposits: `Map{Number -> Deposit}`

where `Deposit` is:

```go
type Deposit struct {
    Owner  ethcmn.Address
    Amount uint64
}
```

and Exits: `Maps{Position -> Exit}`

where `Exit` is:

```go
type Exit struct {
    Owner    ethcm.Address
    Position [4]uint64
}
```

**Note:** RootChain Blocks and Deposits are continuous from 1 to the current objectively finalized value, thus both are simply lists represented in an sdk store.

Let the current objectively finalized block be `B` and the currently objectively finalized deposit be `N`.

### Msgs

Msgs in anchor are submitted by each sidechain validator once per block

```go
// Implement sdk.Msg interface
type AnchorMsg struct {
    // Msg contains all subjectively finalized data 
    // since last objectively finalized data
    RootchainBlockHash [][]byte
    Deposits           []Deposit
    Exits              []Exit
}
```

For validator A with subjectively finalized block `bA` and subjectively finalized deposit `nA`.

Then ValidatorA would send `AnchorMsg` with RootchainBlockHash including BlockHashes `B+1` through `bA`, Deposits `N+1` through `nA`, and all new subjectively finalized exits that are not objectively finalized.

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

Thus, `endBlocker` calculates the Maximum Continuous Intersection of BlockHashes and Deposits independently and adds them to their respective stores.

NOTE: Exits have no notion of being continuous so it is just the Intersection of Exits among a 2/3 quorum of validators.

Since there are a limited number of sidechain validators and the lists are limited to being at most the length of new data that has been added to rootchain since last objectively finalized checkpoint, this isn't an infeasibly expensive calculation to happen in `endBlocker`.

### Subjective Finality

In order to preserve the notion of subjective finality, validators define a parameter in their anchor node's `config.toml` file called `subjective-finality`. This is how many blocks the validator believes need to be added after a piece of data before that data is presumed finalized by the validator. The Anchor node will then poll the rootchain node continuously and pull all relevant data that has been subjectively finalized according to the above parameter but not objectively finalized by Anchor and construct an `AnchorMsg` submitting that data to Anchor chain.

To tune their `subjective-finality` parameter, validators simply stop their anchor node, update their `config.toml` with their preferred `subjective-finality` bound, and restart the anchor node.

### Catastrophic Failure

In the current design there is no recovery mode if the anchor chain diverges from the eventually finalized rootchain. If this happens the sidechain is irreparably compromised, thus everyone should exit. To prevent this, validators should set sufficiently high `subjective-finality` bounds.

### Key Terms
Objectively Finalized: Data on the rootchain is objectively finalized if the anchor chain has checkpointed that piece of rootchain data

Subjectively Finalized: Rootchain data is subjectively finalized for a validator if the validator considers that data sufficiently final. For example: Validator A considers rootchain data final if she sees that data on her rootchain node 6 blocks deep. Thus any data 6 blocks deep is considered subjectively final by A even if the anchor chain has not checkpointed yet.

Maximum Continuous Intersection: the longest continuous intersection starting from the latest objectively finalized block agreed upon by 2/3 of validators. Examples are rootchain block hashes, but deposits behave identically.

Ex 1:

A: `{1 -> A43..., 2 -> B32..., 3 -> C67...}` // Latest blocks seen by A since last objectively finalized blocks

B: `{1 -> A43..., 2 -> B32..., 3 -> D43...}` // B sees a different block 3

C: `{1 -> A43..., 2 -> B32...}` // C only sees 2 blocks since last objectively finalized blocks

D: `{1 -> C54..., 2 -> E13..., 3 -> D52...}` // D currently sees an entirely different version of the blockchain

MCI: `{1 -> A43..., 2 -> B32...}`

With Exits we simply calculate the intersection among 2/3 quorum without notion of being "continuous".