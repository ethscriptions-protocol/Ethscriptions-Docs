# ESIP-XX Extending Ethscriptions to L2 Networks

## Abstract

This ESIP transforms ethscriptions into a novel cross-chain asset—one that exists "above" individual networks and can be manipulated from any qualified L2 without bridging.

Ethscriptions rely on an ordered sequence of creations and transfers to determine existence and ownership—currently derived from Ethereum L1 alone. This ESIP generalizes Ethscriptions to include **any** qualifying L2 network, provided it is permissionless, unstoppable, and uses Ethereum for data availability. By allowing ethscriptions to be created and transferred on L2, users gain access to far cheaper smart-contract interactions (e.g. marketplaces, NFT wrapping) compared to L1. Facet is the first supported L2, as it meets these requirements and provides a direct, trustless link to Ethereum blocks.

## Specification

### L2 Integration Criteria

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

### Integration Data

Each L2 must provide:

| Field              | Description                        | Example    |
| ------------------ | ---------------------------------- | ---------- |
| name               | Lowercase ASCII, no spaces, unique | `"facet"`  |
| mainnet\_chain\_id | L2's mainnet chain identifier      | `84532`    |
| testnet\_chain\_id | L2's testnet chain identifier      | `84531`    |
| l1\_start\_block   | L1 block where integration begins  | `19000000` |
| l2\_start\_block   | L2 block where integration begins  | `1`        |
| l2\_start\_hash    | Hash of L2 start block             | `0x123...` |

### Transaction Processing

Ethscriptions indexers must consider both L1 and L2 transactions to determine the canonical sequence of ethscriptions events:

1. **Process the L1 Block #N**\
   Parse L1 transactions as normal.
2. **Process Each L2 Referencing L1 Block #N**
   * Identify all blocks from each admitted L2 that reference L1 block #N.
   * Process these L2 blocks in ascending L2 block order.
   * If multiple L2s are admitted, the indexer must use a deterministic ordering method (e.g., by chain ID or alphabetical L2 name).
3. **Repeat for the Next L1 Block**\
   Move on to block #(N+1), and repeat.

### New Field: `owned_on_network`

In order to guard against the edge case of two contracts with different code having the same address in two different networks, under this ESIP Ethscriptions now track:

* **`current_owner`**: An address (EOA or contract) that owns the ethscription. The behavior is the same as today.
* **`owned_on_network`**: A string indicating which network last updated ownership (e.g., `"ethereum"` for L1 or the L2 name for an L2).

### Rules for Updating `owned_on_network`

* **Creation**
  * When an ethscription is minted on L1 or an eligible L2, set `current_owner` and `owned_on_network` to that network’s name.
  * For EOAs, `current_owner` is the “to” address from the transaction.
  * For contract-created ethscriptions, `current_owner` is determined by the contract (just like on L1 today).
* **Transfer Attempt**
  * Must originate from the `current_owner` (unchanged from existing Ethscriptions logic).
  * **EOAs** can initiate a transfer from any network. If successful, `owned_on_network` is set to the network of the successful transfer.
  * **Contracts** can only transfer if the call originates on the same network stored in `owned_on_network`.
    * Example: If `current_owner = 0x1234` and `owned_on_network = "ethereum"`, a Facet-based call from `0x1234` is invalid. This prevents collisions across multiple networks where the same address might correspond to different contract code or ownership.
* **Successful Transfer**
  * Update `current_owner` to the recipient.
  * Update `owned_on_network` to the network that processed the transfer.

Marketplaces, wallets, and explorers may display `owned_on_network` at their discretion.

<figure><img src="../../.gitbook/assets/Zight 2025-01-29 at 10.24.54 AM.png" alt=""><figcaption><p>Ethscription transfer logic flow chart</p></figcaption></figure>

### A New Kind of Cross-Chain Asset

This ESIP makes ethscriptions interoperable across multiple networks, with no explicit “bridge” required. Users choose where to transact (L1 or a qualifying L2), and the Ethscriptions indexer logic ensures a single, canonical timeline of ownership.

### Facet as the First Implementation

Facet will be the first L2 network added under this ESIP because it is the only L2 that meets the above criteria for admission.

Ethscriptions indexers that adopt this ESIP must add a Facet RPC endpoint alongside their existing L1 RPC. This allows them to retrieve both L1 and Facet blocks and apply the ownership update rules described above.

## Rationale

### Reduced Marketplace Costs

Moving contract-heavy operations to an L2 yields 5x–50x cost savings for marketplaces and bulk purchase flows.

### Practical NFT Wrapping

Ethscriptions remain their own asset format on L2, but can be wrapped cheaply in an L2-based NFT contract, enabling full NFT marketplace interoperability.

### Security and Trust

* **Permissionless Rollups**\
  Only L2s without privileged roles qualify, preserving Ethscriptions’ trustless nature.
* **Single Source of Truth**\
  By processing L1 blocks first, then L2 blocks, each indexer arrives at the same final ownership state.

### Conclusion

With this ESIP, Ethscriptions gain a “cross-chain” dimension while retaining a unified, canonical view of creations and transfers, starting with Facet. This paves the way for future L2 integrations that adhere to the same trust-minimized requirements.
