# Page

## ESIP-9: L2 Support for Ethscriptions

### Abstract

This ESIP transforms ethscriptions into a novel cross-chain asset—one that exists "above" individual networks and can be manipulated from any qualified L2 without bridging.

Ethscriptions rely on an ordered sequence of creations and transfers to determine existence and ownership—currently derived from Ethereum L1 alone. This ESIP generalizes Ethscriptions to include any qualifying L2 network, provided it is permissionless, unstoppable, and uses Ethereum for data availability. By allowing ethscriptions to be created and transferred on L2, users gain access to far cheaper smart-contract interactions (e.g. marketplaces, NFT wrapping) compared to L1. Facet is the first supported L2, as it meets these requirements and provides a direct, trustless link to Ethereum blocks.

### Specification

#### L2 Integration Criteria

An L2 can be admitted under this ESIP if it satisfies all of the following:

1. **Ethereum Rollup**
   * Must use Ethereum for data availability
2. **Based Sequencing**
   * Each L2 block is deterministically associated with an L1 block
   * All L2 blocks referencing L1 block N must be discoverable when processing block N
   * Multiple L2 blocks per L1 block are allowed with clear ordering
   * Each L2 block must reference L1 block hash for reorg detection
3. **Permissionless and Unstoppable**
   * No privileged roles or admin keys
   * No ability to pause, shut down, or censor transactions
4. **Standard RPC Interface**
   * Must provide reliable transaction and block data access

#### Transaction Processing

The indexer maintains a canonical sequence of creations and transfers across L1 and L2s:

1. Process L1 block N transactions
2. For each admitted L2:
   * Find all L2 blocks referencing L1 block N
   * Process these in ascending L2 block order
3. Move to L1 block N+1

Note: With multiple L2s, indexers must use a deterministic ordering method (e.g., by chain ID or L2 name). As Facet is currently the only L2, specific multi-L2 ordering is left for future specification.

#### State Changes

Each ethscription tracks:

* `current_owner`: Address (EOA or contract) that owns the ethscription
* `owned_on_network`: Network name where ownership was last updated (e.g., "ethereum" or L2 name)

#### Transfer Rules

1. **Creation**
   * Set `current_owner` and `owned_on_network` to creating network
   * EOA creates: `current_owner` is transaction's "to" address
   * Contract creates: `current_owner` per contract logic (unchanged)
2. **Transfer Requirements**
   * Must come from `current_owner`
   * EOAs can transfer from any network
   * Contracts can only transfer if on same network as `owned_on_network`
3. **Transfer Effects**
   * Update `current_owner` to recipient
   * Set `owned_on_network` to processing network

### Implementation

#### Facet Integration

Facet will be the first supported L2 as it meets all criteria. Indexers must:

* Add Facet RPC endpoint alongside L1
* Apply ownership rules across both networks
* Process blocks in canonical order

### Rationale

#### Cost Reduction

* L2 transactions are 5x-50x cheaper
* Enables economical marketplace operations
* Makes NFT wrapping practical

#### Security Model

* Only permissionless L2s qualify
* Unified transaction ordering
* No privileged roles or shutdown mechanisms

### Conclusion

This ESIP enables true cross-chain functionality for ethscriptions while maintaining their core trustless properties. Starting with Facet, it creates a framework for future L2 integrations that meet the same stringent requirements.
