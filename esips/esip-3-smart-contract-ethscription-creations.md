# ESIP-3: Smart Contract Ethscription Creations

<mark style="background-color:orange;">This ESIP is a Draft!</mark> [<mark style="background-color:orange;">Discuss it in this GitHub Issue</mark>](https://github.com/ethscriptions-protocol/ESIPs/issues/6)<mark style="background-color:orange;">.</mark>

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

As with EOA-initiated ethscription creations, ESIP-3 ethscription creations are only valid if `contentURI` is both unique and a syntactically valid dataURI.

#### Ethscriptions and Ethereum Transactions remain 1-1

ESIP-3 does **not** change the fact that each Ethereum transaction may have only one corresponding ethscription. If multiple aspects of a transaction constitute valid ethscription creations, calldata will be prioritized over events, and events with lower log indices will be prioritized over those with higher indices. For example:

1. Calldata: valid creation
2. Event Log Index 1: valid creation
3. Event Log Index 2: valid creation

In this case, an ethscription will be created according to the calldata and Event 1 will be ignored. Were the calldata invalid, Event 1 would create the ethscription and Event 2 would be ignored.

## Rationale

Contracts must have the same powers as EOAs and this is the simplest way to do it.

We propose maintaining the 1-1 correspondence between ethscriptions and Ethereum transactions because the convention that `ethscriptionId` = `transactionHash` has proven very useful and would be difficult to change.

Instead of allowing people to create more ethscriptions in a single transaction, we propose making individual ethscription creations more powerful, as we will see in ESIP-4.

#### File size

Unlike calldata, events have no size limit (aside from the 30M block gas limit). Practically this means that ESIP-3 expands ethscriptions' file size limit to beyond 3.5MB.

















