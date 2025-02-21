# ESIP-XX: Extending Ethscriptions to L2 Networks

## Abstract

This ESIP transforms ethscriptions into a novel cross-chain asset—one that exists "above" individual networks and can be manipulated from any qualified L2 without the need for bridging.

Until now, Ethscriptions have relied on a strictly ordered sequence of creations and transfers derived solely from Ethereum L1. Under this proposal, any L2 meeting certain trust-minimized criteria can create and transfer Ethscriptions, giving users access to far cheaper smart-contract interactions (marketplaces, NFT wrappers, etc.). The indexers still maintain a single, canonical timeline of ownership spanning both L1 and L2. Facet is the first L2 to be admitted because it satisfies all of the requirements below, but these requirements are general enough to allow additional L2 integrations in the future.

[Discuss this draft ESIP here](https://github.com/ethscriptions-protocol/ESIP-Discussion/issues/24).

## Specification

### L2 Integration Criteria

To preserve Ethscriptions’ trustless ethos, L2s must meet these criteria:

1. **Ethereum Rollup**
   1. The L2 must use Ethereum for data availability. Otherwise, it would be impossible to reconstruct Ethscription state solely from Ethereum L1 data.
2. **Based Sequencing**
   1. Each L2 block must be **deterministically associated** with a corresponding Ethereum L1 block (via block number and hash).
   2. All L2 blocks referencing L1 block _N_ must be discoverable by the time the indexer processes L1 block _N_.
   3. Multiple L2 blocks per L1 block are allowed, as long as they have a clear final ordering.
   4. This ensures a **unified Ethscriptions state** across all networks—actions on the L2 are recognized globally once the corresponding L1 block is processed.
   5. **Why Based Sequencing?**
      1. It prevents a privileged sequencer from indefinitely delaying or reordering L2 transactions, preserving a fair ordering for **all** users.
      2. It guarantees that a user’s L2 transaction (e.g., a marketplace purchase) is recognized by the Ethscriptions Protocol as soon as it is confirmed on the L2.
3. **Permissionless and Unstoppable**
   1. The L2 must have no privileged roles or administrative controls that can pause, shut down, or censor transactions.
   2. This prevents any single party from exerting disproportionate power over the protocol’s operation.
4. **Open Source Node**

### Integration Data

Each L2 must provide:

| Field              | Description                        | Example    |
| ------------------ | ---------------------------------- | ---------- |
| name               | Lowercase ASCII, no spaces, unique | `facet`    |
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

This ESIP makes ethscriptions interoperable across multiple networks, with no explicit bridge required. Users choose where to transact (L1 or a qualifying L2), and the Ethscriptions indexer logic ensures a single, canonical timeline of ownership.

### Facet as the First Implementation

Facet is the **initial L2** integrated under this ESIP because:

* It uses **Ethereum L1** for data availability.
* Implements **based sequencing** by tying each L2 block to a specific Ethereum L1 block.
* Lacks **privileged roles** or kill switches.
* Has an [open source node](https://github.com/0xFacet/facet-node).

Indexers must add a Facet RPC endpoint and interleave Facet transactions with L1 transactions according to the rules above.

## Rationale

### Reduced Marketplace Costs

Moving contract-heavy operations to an L2 yields 5x–50x cost savings for marketplaces and bulk purchase flows.

### Practical NFT Wrapping

Ethscriptions remain their own asset format on L2, but can be wrapped cheaply in an L2-based NFT contract, enabling full NFT marketplace interoperability.

### Security and Trust

* **Permissionless Rollups**\
  Only L2s without privileged roles qualify, preserving Ethscriptions’ trustless nature.
* **Single Source of Truth**\
  Merging L1 and L2 histories ensures one canonical record of ethscription creation and transfer.

### Conclusion

By adopting this ESIP, Ethscriptions become a **cross-chain** protocol, retaining a single, unified view of asset ownership while allowing users to transact on whichever trust-minimized L2(s) they prefer. **Facet** serves as a proof of concept for how L2 integration delivers cheaper, more flexible ethscription operations, setting the stage for additional L2s that meet these robust requirements.
