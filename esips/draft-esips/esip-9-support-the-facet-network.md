# ESIP-9 Support the Facet Network

### Abstract

The Ethscriptions Protocol relies on an ordered sequence of ethscriptions creations and transfers to determine which ethscriptions exist and who owns them.

Currently this ordered sequence is derived from Ethereum L1 transaction history. This ESIP proposes adding Facet transaction history as a source for ethscription creations and transfers.

In other words, if a certain L1 Ethereum transaction would create or transfer and ethscription, then the equivalent Facet transaction will do the same.

The purpose of this addition is to enable cheaper ethscriptions-smart contract interactions as using L1 contracts for this is expensive. This has a few applications:

* **Ethscriptions marketplaces.** This ESIP should make marketplace transactions 2x–50x cheaper, with the greatest savings being in purchasing multiple ethscriptions in a single transaction.
* **Ethscriptions-NFT wrappers**. Wrapping an ethscription into an NFT unlocks functionality such a non-escrow based marketplaces. However wrapping with L1 smart contracts (and then selling on L1 NFT marketplaces) is prohibitively expensive.

### Specification

Every Facet block is associated with a single L1 block via the L1Block.sol contract. To integrate Facet transactions, an indexer must:

1. Process a given L1 Block, just as it does today. Say the number is #1234.
2. Then, find all Facet blocks corresponding to L1 block #1234 and process them in order of their Facet block number just as if they were L1 blocks, with the exception of contract address aliasing.

#### Facet Contract Address Aliasing

EOA addresses can be treated normally in transactions initiated in Facet because the same person / private key controls the address on both Facet and the L1.

Contract addresses work differently, however. This is because contracts on Facet and the L1 can share addresses but can contain different code and have different owners. Therefore, a request to transfer an ethscription from `address(0x1234)` initiated on the L1 does not necessarily mean the same thing as the request to transfer an ethscription from the same `address(0x1234)` on Facet.

To eliminate this ambiguity, Facet contract addresses will be transformed so that, in the eyes of the Ethscriptions Protocol, no Facet contract will share an address with an L1 contract.

This is the transformation (it is borrowed from the OP Stack):

```solidity
uint160 constant offset = uint160(0x1111000000000000000000000000000000001111);

function convertL2addresstoL1(address l2Address) internal pure returns (address l1Address) {
    unchecked {
        l1Address = address(uint160(l2Address) - offset);
    }
}
```

Example:

* Facet contract address: `0x0000000000000000000000000000000000001234`
* Transformed address: `0xeEef000000000000000000000000000000000123`

If an ethscriptions indexer sees an event emitted by the Facet contract with address `0x0000000000000000000000000000000000001234` it should "pretend" that the event was actually emitted by address `0xeEef000000000000000000000000000000000123`.

Users must take care to not send assets to un-converted Facet contract addresses as they will be lost.

#### FAQ

Does this make Facet a dependency for Ethscriptions? Yes, after this ESIP all ethscriptions indexers will  need access to both an L1 RPC and a Facet RPC. They can also run a Facet node, which is permissionless.

Can this compromise the security of Ethscriptions? No, because Facet is permissionless, owned by no one, and cannot be turned off.

Why was Facet chosen over any other L2? Facet is the only Ethereum rollup that cannot be shut down and has no privileged roles.

Will ethscriptions be NFTs in the Facet protocol? No, they will be ethscriptions! It will work just like it does on the L1. However they can be wrapped into NFTs.
