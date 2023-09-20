# ESIP-3: Smart Contract Ethscription Creations

## Abstract

ESIP-3 introduces a mechanism for smart contracts to create ethscriptions using Ethereum events.

## Specification

Add a new smart contract event into the Ethscriptions Protocol:

```solidity
event ethscriptions_protocol_CreateEthscription(
    address indexed initialOwner,
    string contentURI
);
```

When a contract emits this event, the protocol should register a valid ethscription creation attempt with:

1. `contentURI` interpreted as the ethscription's utf-8 encoded dataURI.
2. `initialOwner` as the created ethscription's initial owner.
3. The emitting contract as the creator.

Functionally speaking, this event is the equivalent of an EOA hex-encoding `contentURI` and putting it in the calldata of an Ethereum transaction from itself to `initialOwner`.

As with EOA-initiated ethscription creations, ESIP-3 ethscription creations are only valid if `contentURI` is both unique and [a syntactically valid dataURI](https://docs.ethscriptions.com/overview/how-ethscriptions-work#how-to-validate-a-datauri).

#### Example `contentURI` format

`data:,1234`.

Note: it is utf-8 encoded, _not_ hex-encoded. Note also this specific example is a duplicate and would not result in an ethscription creation.

#### Ethscriptions and Ethereum Transactions remain 1-1

ESIP-3 does **not** change the fact that each Ethereum transaction may have only one corresponding ethscription. If multiple aspects of a transaction constitute valid ethscription creations, calldata will be prioritized over events, and events with lower log indices will be prioritized over those with higher indices.

Example 1:

1. Calldata: valid creation
2. Event Log Index 1: valid creation
3. Event Log Index 2: valid creation

In this case, an ethscription will be created according to the calldata and Events 1 and 2 will be ignored.

Example 2:

1. Calldata: empty (i.e., invalid creation)
2. Event Log Index 1: valid creation
3. Event Log Index 2: valid creation

Here, Event 1's log will trigger the ethscription creation. If calldata and Event 1 were both invalid then Event 2's log would trigger the ethscription creation.

## Rationale

Contracts must have the same powers as EOAs and this is the cheapest way to do it.

We propose maintaining the 1-1 correspondence between ethscriptions and Ethereum transactions because the convention that `ethscriptionId` = `transactionHash` has proven useful.

Multiple ethscriptions in a transaction are also an inefficient way of capturing a user's intent. Creating multiple ethscriptions in a transaction will always have an underlying purpose and structure, and we should be capturing this structure using [ESIP-4](https://docs.ethscriptions.com/esips/esip-4-the-ethscriptions-virtual-machine).

For example, instead of forcing a user to bulk create ethscriptions of this form:

```
data:,{"p":"erc-20","op":"mint","tick":"fair","id":"17560","amt":"1000"}
data:,{"p":"erc-20","op":"mint","tick":"fair","id":"17561","amt":"1000"}
data:,{"p":"erc-20","op":"mint","tick":"fair","id":"17562","amt":"1000"}
data:,{"p":"erc-20","op":"mint","tick":"fair","id":"17563","amt":"1000"}
data:,{"p":"erc-20","op":"mint","tick":"fair","id":"17564","amt":"1000"}
data:,{"p":"erc-20","op":"mint","tick":"fair","id":"17565","amt":"1000"}
data:,{"p":"erc-20","op":"mint","tick":"fair","id":"17566","amt":"1000"}
...
```

We should capture the user's intent with a single ethscription containing the command `mint(50)`.

#### File size

Unlike calldata, events have no size limit (aside from the 30M block gas limit). Practically this means that ESIP-3 expands ethscriptions' file size limit to beyond 3.5MB.

















