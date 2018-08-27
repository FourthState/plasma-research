## Sidechain Communication with Anchor

The sidechain will be modified to expect a separate subroutine or process to provide it the latest checkpointed deposits and exits. This subroutine can have arbitrary implementation so long as it provides the expected data in the expected format.

Ideally, the sidechain does not have to pull from this subroutine and wait for a response. Rather the sidechain has an open channel with this checkpointing routine, and whenever the routine updates the checkpointed state, it pushes the relevant data to the sidechain channel. The sidechain can then pick up this data instantly by just grabbing the current data on the channel during processing.

## Single Validator

With a single validator, this checkpointing routine can be simple glue code between the validating service and a geth node. This glue code is responsible for submitting deposits and exits from the geth node that the validator has deemed final.

## Multiple Validators with Anchor

In the case of multiple validators, we don't want to place our trust in a single rootchain oracle. Thus, we use Anchor to provide this information for the sidechain. After each anchor block, the validator's anchor node pushes the latest checkpointed state (only deposits and exits) to the channel it maintains with the validator's sidechain node.

Thus in this setup, the validator maintains 3 processes. A geth node (possibly light node), an anchor validator node, and a sidechain validator node.

## Sidechain Updates

Regardless of the process used to checkpoint state, the sidechain will grab the latest checkpointed state from the channel during `beginBlocker`.

It will then add all checkpointed deposits to a store and delete all utxo positions associated with a checkpointed exit.

After this point, it can approve any spends of the checkpointed deposits on the sidechain and reject any spends of the checkpointed exits.