# ESIP-8: Ethscription Attachments aka "BlobScriptions"

### Links <a href="#abstract" id="abstract"></a>

* [Reference Implementation](https://github.com/0xFacet/ethscriptions-indexer/pull/60)
* [ESIP-8 Discussion](https://github.com/ethscriptions-protocol/ESIP-Discussion/issues/16)

### Abstract <a href="#abstract" id="abstract"></a>

The introduction of blobs in EIP-4844 enables anyone to store data on Ethereum for 10x to 100x cheaper than calldata. This comes at a cost, however: the Ethereum protocol doesn't guarantee the availability of blob data for more than 18 days.

However, on a practical level it is not clear how burdensome this limitation will be. Because L2s use blobs to store transaction data there will be strong incentives to create publicly accessible archives of blob data to enhance the transparency and auditability of Layer 2s.

Like IPFS, blob data is completely decentralized—as long as one person has blob data it can be verified and used by anyone.

This ESIP proposes a change to the Ethscriptions Protocol that will enable the Ethscriptions Community to harness the benefits of blob storage while mitigating the downsides.

It does this not by allowing users to create ethscriptions through blobs, as this would jeopardize Ethscriptions-based protocols that rely on full availability, but rather by following EIP-4844's "sidecar" approach. ESIP-8 proposes a new "sidecar" `attachment_uri` field on Ethscriptions that is composed from the data in one or more blobs.

The name "Ethscription Attachment" is preferred over "Ethscription Blob" (or similar) because transactions can have multiple blobs, but ethscriptions can only have one attachment (that is composed of all the blobs together).

### Specification <a href="#specification" id="specification"></a>

All new ethscriptions have an optional new `attachment_uri` field. If an ethscription is created in a transaction with no blobs this field will be `null`.&#x20;

If an ethscription's creation transaction _does_ include blobs, its blobs are concatenated and interpreted as UTF-8. If the resulting string is valid dataURI then it will be set as the `attachment_uri` for the ethscription. There is no uniqueness requirement.

This new field will be available in all API responses _in addition_ to the `content_uri` field. However, unlike the `content_uri` field, the protocol doesn't guarantee the `attachment_uri` field will be populated when an ethscription has an attachemnt.

Instead, users will rely on mechanisms outside of the protocol to enforce this; for example by choosing an ethscriptions indexer that commits to making attachments available. Clients can also provide their own blob data and verify they generate the correct `attachment_uri`.

In this way, you can think of the `attachment_uri` field as functioning like IPFS. However, because of the social factors mentioned above, the assumption is that, by default, blobs will be more broadly available than IPFS data.

**Getting Blob Data**

Blob data is available on a block-level through the `blob_sidecars` API endpoint available on Ethereum Beacon nodes. If you don't want to run a node yourself, [you can use Quicknode](https://www.quicknode.com/docs/ethereum/eth-v1-beacon-blob\_sidecars-id).

The input to this function is a "block id," which is a slot number (not block number) or block root. Block roots are available on normal Ethereum API requests, but only for the _previous_ block (the field is `parentBeaconBlockRoot`).

This means that attachments must be populated on a one block delay.

**Associating Blobs with Ethereum Transactions**

Because the Beacon API only provides blob information on a block level, it requires some additional logic to match blobs to the transactions that created them. Fortunately, transactions now have a `blobVersionedHashes` field that can be computed from the `kzg_commitments` field on the block-level blob data.

Here's an example implementation (it's `O(n^2)` but the numbers involved are small)

```ruby
def blob_from_version_hash(version_hash)
  block_blob_sidecars.find do |blob|
    kzg_commitment = blob["kzg_commitment"].sub(/\A0x/, '')
    binary_kzg_commitment = [kzg_commitment].pack("H*")
    sha256_hash = Digest::SHA256.hexdigest(binary_kzg_commitment)
    modified_hash = "0x01" + sha256_hash[2..-1]
    
    version_hash == modified_hash
  end
end

def transaction_blobs
  blob_versioned_hashes.map do |version_hash|
    blob_from_version_hash(version_hash)
  end
end
```

#### Converting Blob Content to a UTF-8 dataURI

At a high-level we use the same logic to convert blobs to UTF-8 that we use for calldata. However blobs have a few interesting quirks that make this more challenging:

* Currently blobs have a minimum length of 128kb. If your data is smaller than that you'll have to pad it (probably will null bytes) to the full length.
* Blobs are composed of "segments" of 32 bytes, none of which, when interpreted as an integer, can exceed the value of the cryptography-related "BLS modulus", which is 52435875175126190479447740508185965837690552500527637822603658699938581184513.

So if you want to use blobs you need a protocol for communicating where the data ends and a mechanism for ensuring no 32 byte segment is too large.

Here Ethscriptions will follow [Viem's approach](https://github.com/wevm/viem/blob/main/src/utils/blob/toBlobs.ts):

* Left-pad each segment with a null byte. A `0x00` in the most significant byte ensures no segment can be larger than the BLS modulus.
* End the content of every blob with `0x80`, which, when combined with the rule above, provides an unambiguous way to determine the length of the data in the blob.

When a blob creator follows these rules, the Ethscriptions Protocol can convert the list of blobs to a dataURI like this:

```ruby
def compute_attachment_uri
  return if blobs.blank?
  
  concatenated_hex = blobs.map do |blob|
    hex_blob = blob["blob"].sub(/\A0x/, '')
    
    sections = hex_blob.scan(/.{64}/m)
    
    last_non_empty_section_index = sections.rindex { |section| section != '00' * 32 }
    non_empty_sections = sections.take(last_non_empty_section_index + 1)
    
    last_non_empty_section = non_empty_sections.last
    
    if last_non_empty_section == "0080" + "00" * 30
      non_empty_sections.pop
    else
      last_non_empty_section.gsub!(/80(00)+\z/, '')
    end
    
    non_empty_sections.map do |section|
      unless section.start_with?('00')
        raise "Expected the first byte to be zero"
      end
      
      section.delete_prefix("00")
    end
    
    non_empty_sections.join
  end.join
  
  HexDataProcessor.hex_to_utf8(concatenated_hex, support_gzip: true)
end
```

Where [`HexDataProcessor`](https://github.com/0xFacet/ethscriptions-indexer/blob/main/lib/hex\_data\_processor.rb) is the one we use for processing calldata. Here's [an example transaction that uses this format](https://sepolia.etherscan.io/tx/0xc6032f926842060d5556eccecd7cd5cec49cdfd3cad03a6caa7a6efe5e248b9b).

### Rationale <a href="#rationale" id="rationale"></a>

It's all about the right tool for the job. When data is part of a stateful protocol, for example information about who owns an NFT, Attachments are the wrong choice because the protocol cannot function if ownership information is lost for even a short time.

However, temporarily losing the _image_ of an NFT doesn't break the protocol, so NFT images could be a good choice for attachments. Of course it's preferable for NFT images (and everything we do) to be on-chain and available at all times, sometimes this is prohibitively expensive.

Attachments fill an empty "sweet spot" between calldata and IPFS and will enhance the overall utility of the Ethscriptions Protocol significantly.
