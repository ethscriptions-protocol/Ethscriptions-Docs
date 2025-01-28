# ESIP-9 Support the Facet Network

### Abstract

Ethscriptions determine existence and ownership via an ordered sequence of creations and transfers. Currently, that sequence is based only on Ethereum L1 transaction history. This ESIP extends that sequence to include Facet (L2) transactions, so that if an L1 transaction would create or transfer an ethscription, the corresponding Facet transaction can do the same. The main goal is to reduce gas costs for ethscriptions-smart contract interactions (marketplaces, wrappers, etc.) that would otherwise be prohibitively expensive on L1.

### Specification

#### Transaction Processing

1. Every Facet block is linked to one Ethereum L1 block
2. Indexers process both:
   * First, process the L1 block (#1234)
   * Then, process all Facet blocks linked to that L1 block.
     * Currently there will be one such Facet block per L1 block
     * This requires Ethscriptions indexers to have access to a Facet RPC endpoint

#### New Field: `owned_on_network`

Every ethscription now stores:

* `current_owner`: The address (EOA or contract) that currently owns it.
* `owned_on_network`: Either `L1` or `L2`, depending on which network last updated ownership.

#### Rules for Updating `owned_on_network`

1. **Creation**: When an ethscription is created (on L1 or L2), set `owned_on_network` to the network where it was minted.
2. **Transfer Attempt**:
   * Transfers must always originate from the current owner (as today).
   * **EOAs** can transfer from either L1 or L2. If successful, `owned_on_network` is set to the network where the transfer occurred.
   * **Contracts** can only transfer if they are doing so from the same network recorded in `owned_on_network`.
     * Example: If `current_owner = 0x1234` (a contract) and `owned_on_network = L1`, only an L1 transaction from that contract can transfer it.
3. **Successful Transfer**
   1. Update `current_owner` to the recipient.
   2. Update `owned_on_network` to the network that processed the transfer.

Marketplaces, wallets, and explorers can display the owned\_on\_network data in any way they choose.

Here is a flowchart of the block processing logic:

<figure><img src="../../.gitbook/assets/Zight 2025-01-28 at 3.01.53 PM (1).png" alt=""><figcaption><p>Block processing flowchart</p></figcaption></figure>

### Rationale

#### Key Benefits

* **Reduced Marketplace Costs**: Marketplace transactions can be 5x–50x cheaper when performed on Facet, especially when buying multiple ethscriptions in one transaction.
* **Practical NFT Wrapping**: Ethscriptions remain Ethscriptions on both networks—they do not become ERC-721 NFTs just by existing on Facet. However, they can be wrapped in an NFT contract if desired, unlocking more complex features without the high gas costs of L1 interactions.

#### Security and Trust

* **Immutable L2**: Facet is permissionless, has no privileged roles, and cannot be shut down, preserving Ethscriptions’ trustlessness.
* **Consistent State**: Indexers maintain a single source of truth by processing L1 blocks, then Facet blocks, preserving a definitive transaction order.

#### Future Extensions

* Further L2 support can be added later (using similar logic) if additional networks without privileged roles emerge.
* `owned_on_network` can be extended or replaced by more granular identifiers if needed.
