# ESIP-9 Enable Facet Contracts to Transfer Ethscriptions

This is for bridging.

Ethscriptions indexers would listen to Facet contract events using a Facet RPC.

For every ESIP-1 or ESIP-2 compliant Ethscriptions transfer event emitted by a Facet contract, indexers would register a valid Ethscriptions transfer provided the Ethscription in question is currently owned by the _unaliased_ address of the emitting contract.

Unaliasing means this:

```solidity
library AddressAliasHelper {
    uint160 constant offset = uint160(0x1111000000000000000000000000000000001111);

    function undoL1ToL2Alias(address l2Address) internal pure returns (address l1Address) {
        unchecked {
            l1Address = address(uint160(l2Address) - offset);
        }
    }
}
```

This is required because a given address might have both an L1 and L2 contract associated with it and we must disambiguate what actually owns it.

When users want to send an Ethscription to a Facet contract they calculate the recipient address by applying undoL1ToL2Alias(addressOnFacet).

Example: User wants to bridge an Ethscription into Facet using a Facet contract bridge at address(1234) (on Facet).

1. User sends Ethscription to undoL1ToL2Alias(address(1234))
2. User gets a trusted signature to prove that the Facet bridge owns the Ethscription (unfortunately trust is still required at this step)
3. Facet contract mints an L2 asset to bridging user
4. User wants to bridge out, tells Facet contract
5. Facet contract burns L2 asset and emits Ethscription transfer event
6. Indexer identifies the transfer event is coming from Facet and computes undoL1ToL2Alias(address(1234)). Because the user sent the Ethscription there in step (1) this new transfer is valid under the protocol.
