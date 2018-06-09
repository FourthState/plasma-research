This is a potential solution to enabling smart contract functionality on Plasma.

First we need to look at how regular monetary transactions work on Plasma MVP.

With Plasma MVP, we "lock up" funds in the rootchain smart contract.

We then update balances on the sidechain, and when we want to exit we enter an interactive game where we claim to exit the latest account 
state while other actors may prove that the exiting state is not really the latest valid state 
(e.g. by proving that exiting UTXO was already spent)

One can create a similar construction with smart contracts.

In this scenario, a smart contract locks either its entire state or some isolated subset of its state to the 
Plasma rootchain smart contract.

Example:

Lock state: Smart contract allows user to send cryptokitty to a plasma rootchain contract. 
Smart contract will then accept any updated state from rootchain contract

Once the state is locked in the plasma rootchain contract, future state transitions can only happen in the plasma sidechain until the state 
has successfully exited back to the original smart contract.

Consider possible state transitions on the sidechain once state "deposited" at state A:

Valid transitions:

A -> B -> C -> D -> E

In this case, a user exits E. No one can challenge this state, so when the challenge period elapses the rootchain contract can unlock the state by sending a message to the original smart contract.
Following our example, we send the updated cryptokitty(ies) back to the smart contract.

Valid transitions:

A -> B -> C -> D -> E -> F

In this case a user tries to exit E even though F is the latest state. Here a simple proof of inclusion of the state transition E -> F 
is all that is needed to successfully challenge the exit. The challenger wins the bond that the exiter puts up.

Invalid transitions:

A -> B -/-> C -> D -> E

Here the state transition from B to C is invalid. 

