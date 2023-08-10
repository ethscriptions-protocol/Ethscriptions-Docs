---
description: This ESIP is LIVE
---

# ESIP-1: Smart Contract Ethscription Transfers

#### Version History

* June 29: Changed event name to be more explicit and to reduce changes of collision.
* June 29: Added spec for case in which there are multiple transfers in a given transaction.

#### Specification

Incorporate one new smart contract event into the Ethscriptions Protocol:

```solidity
ethscriptions_protocol_TransferEthscription(
  address indexed recipient,
  bytes32 indexed ethscriptionId
)
```

Event signature:

```solidity
// "0xf30861289185032f511ff94a8127e470f3d0e6230be4925cb6fad33f3436dffb"
keccak256("ethscriptions_protocol_TransferEthscription(address,bytes32)")
```

When a contract emits `ethscriptions_protocol_TransferEthscription`, the protocol should register a valid ethscription transfer from the emitting contract to `recipient` of the `ethscription` with id `ethscriptionId`, provided the emitting contract owns that ethscription when emitting the event, and the event is emitted in `17672762` or a later block.

If there are multiple valid events they should be processed in the order of their log index.

If the input data of the transaction also represents a valid transfer, this transfer will be processed before all event-based transfers.

#### Rationale

Ethscriptions can be transferred to any address, which means smart contracts can own them. However, smart contracts cannot currently transfer or create ethscriptions themselves.

This inhibits the creation of protocol-native apps that require smart contracts, such as marketplaces. Further, it makes the protocol difficult to use for smart contract wallet users.

This proposal lays out a simple and low gas mechanism for enabling smart contracts to transfer ethscriptions, with ethscription creation to follow soon.

Indexing these events across all contracts increases the burden of operating an indexer, but this extra cost is incremental given that indexers must inspect the calldata of every transaction anyway.

#### Notes

A previous version of this proposal included an additional smart contract event for Ethscription creation:

```solidity
CreateEthscription(
  address indexed initialOwner,
  string dataURI
)
```

However I think the need for this event is smaller and will consider it in a different proposal so as to maintain as much simplicity as possible and being as deliberate as possible in making changes
