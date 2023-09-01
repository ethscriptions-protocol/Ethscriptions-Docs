---
description: >-
  Ethscriptions are a new way of creating and sharing digital artifacts on
  Ethereum using transaction calldata.
---

# Introducing Ethscriptions

### NEW: <mark style="background-color:green;">The first Ethscriptions Virtual Machine implementation is now Open Source!</mark> [<mark style="background-color:green;">Read the docs</mark>](https://docs.ethscriptions.com/v/ethscriptions-vm)

## Overview

Ethscriptions are an alternative to smart contracts—which are prohibitively expensive for most users—and to L2s, which today are centralized.

Ethscriptions is a protocol that allows users to share information and perform computations on Ethereum L1 at a drastically lower cost.

Ethscriptions achieves this by bypassing smart contract storage and execution and instead calculating state by applying deterministic protocol rules to "dumb" Ethereum calldata.

The goal of Ethscriptions is to give ordinary users the ability to perform decentralized computations for a reasonable price.

Today, Ethscriptions primarily function as cheaper NFTs. After the launch of [the Ethscriptions Virtual Machine](https://docs.ethscriptions.com/v/ethscriptions-vm), they will function as a cheaper alternative to the Ethereum Virtual Machine.

### Links

* [Ethscriptions Protocol GitHub](https://github.com/ethscriptions-protocol/)
* [Ethscriptions.com](api-docs/ethscriptions-endpoints.md)
* [Ethscription VM Goerli Testing Site](https://goerli.ethscriptionsvm.com)
* [Ethscriptions VM Docs](https://docs.ethscriptions.com/v/ethscriptions-vm)

### What is Calldata?

Ethscriptions are cheaper because they store data on-chain using Ethereum transaction calldata, not smart contracts.

When you send someone eth via an Ethereum transaction, calldata is the "notes field." Sometimes people write things in the notes field, but typically when you send eth to a person you leave it blank. When you interact with a smart contract, however, you add the information you're passing to the smart contract—the function name and parameters—to the calldata field.

Ethscriptions are similar in that they encode data into calldata, but this information is not directed at smart contracts.

This video breaks it down:

{% embed url="https://www.youtube.com/watch?v=SjVrSihJOkU" %}
What are Ethscriptions? Venmo but you put an image in the "notes" field.
{% endembed %}

Using calldata like this enables ethscriptions to be 100% on-chain, permissionless, and censorship resistant, at a fraction of the cost of NFTs.

## FAQ

### **Are Ethscriptions secure and trustless?**

Absolutely! You can use the Ethscriptions protocol without relying on external parties. While it might be convenient to trust an indexer, like most Ethereum community members do with Etherscan, you can always rebuild and verify the indexer data manually.

### **Are Ethscriptions decentralized?**

Yes, Ethscriptions reinterpret existing Ethereum data, which is decentralized by nature. No one's permission is required to use Ethscriptions and no one can ban you from using it. By contrast, NFTs often rely on data stored in specific contracts that one person might control.

### **Does relying on off-chain indexers as the source of truth make Ethscriptions centralized?**

Ethscriptions doesn't rely on off-chain indexers as the source of truth any more than Ethereum relies on Etherscan as the source of truth. Both types of indexers are tools, and if they report data inconsistent with protocol rules they should be fixed. The key to decentralization is that these kinds of bugs can be discovered and verified by all protocol participants equally.

### Who invented Ethscriptions?

The [first ethscription](https://ethscriptions.com/ethscriptions/0) was created in 2016, but the formal protocol was developed by Tom Lehman aka [Middlemarch](https://twitter.com/dumbnamenumbers). In addition to Bitcoin inscriptions, he was inspired by the famous “proto-Ethscription” from the Poly Network hacker that you can see [in this transaction](https://etherscan.io/tx/0x0ae3d3ce3630b5162484db5f3bdfacdfba33724ffb195ea92a6056beaa169490).&#x20;

The author writes:

> ETHEREUM HAS THE POTENTIAL TO BE A SECURED AND ANONYMOUS COMMUNICATION CHANNEL, BUT ITS NOT FRIENDLY TO AVERAGE USERS. THE EXTRACTION OF MESSAGE REQUIRES SOME THEQUINIES, THE ENCRYPTION OF MESSAGE IS A MORE ADVANCED SKILL. I HAVE NO RESEARCH ON EXISTING PROJECTS. AND THE GAS FEE STOPS MOST USERS, THOUGH IT DOES NOT STOP REFUGEES. IS IT POSSIBLE TO ULTILIZE THE ETH NETWORK FOR FREE BY USING EXTREMELY LOW GAS? A SNAPCHAT ON CHAIN?

### More questions?

Jump into the [Discord](https://discord.gg/ethscriptions)!
