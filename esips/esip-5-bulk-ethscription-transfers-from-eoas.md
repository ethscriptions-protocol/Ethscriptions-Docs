# ESIP-5: Bulk Ethscription Transfers from EOAs

<mark style="background-color:orange;">This ESIP is a Draft!</mark> [<mark style="background-color:orange;">Discuss it in this GitHub Issue</mark>](https://github.com/ethscriptions-protocol/ESIPs/issues/9)<mark style="background-color:orange;">.</mark>

## Abstract

Bulk transferring means transferring more than one ethscription in a single Ethereum transaction.

ESIP-1 and ESIP-2 gave Smart Contracts the ability to transfer ethscriptions through events. Because multiple events can be emitted in a single transaction, ESIP-1 and ESIP-2 also gave Smart Contracts the ability to bulk transfer ethscriptions.

ESIP-5 brings EOAs to the level of Smart Contracts by introducing a mechanism for EOAs to bulk transfer ethscriptions.

## Specification

Pre-ESIP-5, this was the rule for EOAs transferring an ethscription:

> Any Ethereum transaction whose input data is an ethscription id \[...] is a valid Ethscription transfer, provided the transaction sender is the Ethscription’s owner.

ESIP-5 retains the spirit of this rule, but allows users to add multiple ordered ethscription ids as input data.

If the input data of a transaction (without its leading `0x`) is a sequence of 1 or more valid ethscription ids (without their leading `0x`), that transaction will constitute a valid transfer for each ethscription that is owned by the transaction's creator.

#### An Example

Suppose a transaction has the input data:

{% code overflow="wrap" %}
```
0x8ad5dc6c7a6133eb1c42b2a1443125b57913c9d63376825e310e4d1222a91e24533c5e38d1b8bf75166bd6443a443cd25bd36c087e1a5b8b0881b388fa1a942c
```
{% endcode %}

We first remove the leading `0x`:

{% code overflow="wrap" %}
```
8ad5dc6c7a6133eb1c42b2a1443125b57913c9d63376825e310e4d1222a91e24533c5e38d1b8bf75166bd6443a443cd25bd36c087e1a5b8b0881b388fa1a942c
```
{% endcode %}

Now we observe that this hex string's length is 128, which is an even multiple of 64, which is the length of an ethscription id with its leading `0x` removed.

Now we split the hex string into two chunks of 64 characters and determine whether these chunks are valid ethscription ids. We prepend the `0x` and check for ethscriptions. Now we find:

1. `0x8ad5dc6c7a6133eb1c42b2a1443125b57913c9d63376825e310e4d1222a91e24` is [Ethscription #2](https://ethscriptions.com/ethscriptions/0x8ad5dc6c7a6133eb1c42b2a1443125b57913c9d63376825e310e4d1222a91e24).
2. `0x533c5e38d1b8bf75166bd6443a443cd25bd36c087e1a5b8b0881b388fa1a942c` is [Ethscription #1](https://ethscriptions.com/ethscriptions/0x533c5e38d1b8bf75166bd6443a443cd25bd36c087e1a5b8b0881b388fa1a942c).

Because both ids correspond to valid ethscriptions, we proceed. If one or more weren't valid ethscriptions, we would stop processing at this point.

Now we look at the ids in the order they were listed in calldata, and register in order:

1. A valid transfer for `0x8ad5dc6c7a6133eb1c42b2a1443125b57913c9d63376825e310e4d1222a91e24` if the "from" of the transaction is the owner of this ethscription as of this moment.
2. A valid transfer for `0x533c5e38d1b8bf75166bd6443a443cd25bd36c087e1a5b8b0881b388fa1a942c` is "from" of the transaction is the owner of this ethscription as of this moment.

If the "from" of the transaction is _not_ the owner of the ethscription, we skip that transfer and continue processing. This means that if the "from" is the owner on some, but not all, ethscriptions some of the transfers will be valid and some will not.

## Rationale

The goal is to reduce the question of bulk transferring to a sequence of individual transfers. This is why there can be partially valid bulk transfers—creating a notion of bulk validity is additional complexity.

However, we must enforce a notion of global validity for all ethscription ids, otherwise we introduce too much potential for unintentional transfers and confusion.
