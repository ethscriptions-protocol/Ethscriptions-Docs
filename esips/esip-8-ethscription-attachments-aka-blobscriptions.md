# ESIP-8: Ethscription Attachments aka "BlobScriptions"

### Links <a href="#abstract" id="abstract"></a>

* [Reference Implementation](https://github.com/0xFacet/ethscriptions-indexer/pull/60)
* [ESIP-8 Discussion](https://github.com/ethscriptions-protocol/ESIP-Discussion/issues/17)

### Abstract <a href="#abstract" id="abstract"></a>

The introduction of blobs in EIP-4844 enables anyone to store data on Ethereum for 10x to 100x cheaper than calldata. This comes at a cost, however: the Ethereum protocol doesn't guarantee the availability of blob data for more than 18 days.

However, on a practical level it is not clear how burdensome this limitation will be. Because L2s use blobs to store transaction data there will be strong incentives to create publicly accessible archives of blob data to enhance the transparency and auditability of Layer 2s.

Also, like IPFS, blob data is completely decentralized—as long as one person has blob data it can be verified and used by anyone.

This ESIP proposes using blobs to store data within the Ethscriptions Protocol. We presuppose the ready availability of blob data and require indexers to store or find user blob data along with the other blockchain data the Ethscriptions Protocol currently uses.

Specifically, ESIP-8 proposes a new "sidecar" **attachment** field for Ethscriptions that is composed from the data in one or more blobs. This field is in addition to the existing **content** field.

The name "Ethscription Attachment" is preferred over "Ethscription Blob" (or similar) because transactions can have multiple blobs, but ethscriptions can only have one attachment (that is composed of all the blobs together).

### An Example <a href="#specification" id="specification"></a>

Consider the ethscription created by [this Sepolia transaction](https://sepolia.etherscan.io/tx/0x5d04d632d3affef95b0ae141f2b5b5af474ab80a24925672bdd551637990054b). The transaction's calldata contains the hex data `0x646174613a2c68656c6c6f2066726f6d20457468736372697074696f6e2063616c6c6461746121` which corresponds to the dataURI "data:,hello from Ethscription calldata!" which becomes the ethscription's content.

[The transaction's blobs](https://sepolia.etherscan.io/tx/0x5d04d632d3affef95b0ae141f2b5b5af474ab80a24925672bdd551637990054b#blobs), when interpreted according to the rules described below, contains the data for this image which becomes the ethscription's attachment:

<div align="center">

<figure><img src="../.gitbook/assets/starroom.gif" alt="" width="375"><figcaption></figcaption></figure>

</div>

### Specification <a href="#specification" id="specification"></a>

All new ethscriptions have an optional `attachment` field. If an ethscription is created in a transaction with no blobs this field will be `null`.&#x20;

If an ethscription's creation transaction does include blobs _and_ the ethscription was created via calldata (i.e., not via an event emission), its blobs are concatenated and interpreted as an untagged [CBOR](https://cbor.io/) object (as defined by [RFC 8949](https://datatracker.ietf.org/doc/html/rfc8949)) that decodes into a hash with _exactly_ these keys:

* `content`
* `contentType`

If the concatenated data is a valid CBOR object, and that object decodes into a hash with exactly those two fields, an attachment for the ethscription is created. Note:

* There is no uniqueness requirement for the attachment's content and/or contentType.
* Attachment content, contentType, and the container CBOR object itself can each be optionally gzipped.
* The attachment is **not** valid if:
  * If the CBOR object has a tag
  * If the decoded object his not a hash
  * If the decoded hash's keys aren't exactly `content` and `contentType`. There cannot be extra keys.
  * The values of `content` and `contentType` aren't both strings (either binary or UTF-8).

When such an attachment exists, the indexer's API must include the path for retrieving it in an `attachment_path` field in the JSON representation of an ethscription with at most a one block delay between ethscription creation and inclusion of the URL. For example, if an ethscription is created in block 15, the attachment\_url must appear no later than block 17.

The attachment\_url field will be available _in addition_ to the `content_uri` field.

#### Creating an Ethscription Attachment in Javascript

You can use the `cbor` package and Viem's `toBlobs`:

```typescript
const { toBlobs } = require('viem');
const fs = require('fs');
const cbor = require('cbor');

const imagePath = '/whatever.gif'
const imageData = fs.readFileSync(imagePath);

const dataObject = {
  contentType: 'image/gif',
  content: imageData
};

const cborData = cbor.encode(dataObject);
const blobs = toBlobs({ data: cborData });
```

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

#### Converting Blob Content to an Attachment

At a high-level we use normalize the blob data, concatenate it, and CBOR-decode it. However blobs have a few interesting quirks that make this more challenging:

* Currently blobs have a minimum length of 128kb. If your data is smaller than that you'll have to pad it (probably will null bytes) to the full length.
* Blobs are composed of "segments" of 32 bytes, none of which, when interpreted as an integer, can exceed the value of the cryptography-related "BLS modulus", which is 52435875175126190479447740508185965837690552500527637822603658699938581184513.

So if you want to use blobs you need a protocol for communicating where the data ends and a mechanism for ensuring no 32 byte segment is too large.

Here Ethscriptions will follow [Viem's approach](https://github.com/wevm/viem/blob/main/src/utils/blob/toBlobs.ts):

* Left-pad each segment with a null byte. A `0x00` in the most significant byte ensures no segment can be larger than the BLS modulus.
* End the content of every blob with `0x80`, which, when combined with the rule above, provides an unambiguous way to determine the length of the data in the blob.

When a blob creator follows these rules, we decode the attachment using this class.

[`HexDataProcessor`](https://github.com/0xFacet/ethscriptions-indexer/blob/main/lib/hex\_data\_processor.rb) is the same class one we use for processing calldata. Here we use CBOR instead of DataURIs because CBOR is a more efficient way of representing binary data.

```ruby
class EthscriptionAttachment < ApplicationRecord
  class InvalidInputError < StandardError; end
  
  has_many :ethscriptions,
    foreign_key: :attachment_sha,
    primary_key: :sha,
    inverse_of: :attachment
  
  def self.from_cbor(cbor_encoded_data)
    cbor_encoded_data = ungzip_if_necessary!(cbor_encoded_data)
    
    decoded_data = CBOR.decode(cbor_encoded_data)
    validate_input!(decoded_data)
    
    content = ungzip_if_necessary!(decoded_data['content'])
    contentType = ungzip_if_necessary!(decoded_data['contentType'])
    
    is_text = content.encoding.name == 'UTF-8'
    
    sha_input = {
      contentType: contentType,
      content: content,
    }.to_canonical_cbor
    sha = "0x" + Digest::SHA256.hexdigest(sha_input)
    
    new(
      content: content,
      is_text: is_text,
      sha: sha,
      contentType: contentType,
      size: content.bytesize,
    )
  rescue EOFError, CBOR::MalformedFormatError => e
    raise InvalidInputError, "Failed to decode CBOR: #{e.message}"
  end
  
  def self.from_blobs(blobs)
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
        last_non_empty_section.gsub!(/80(00)*\z/, '')
      end
      
      non_empty_sections = non_empty_sections.map do |section|
        unless section.start_with?('00')
          raise InvalidInputError, "Expected the first byte to be zero"
        end
        
        section.delete_prefix("00")
      end
      
      non_empty_sections.join
    end.join
    
    cbor = [concatenated_hex].pack("H*")
    
    from_cbor(cbor)
  end
  
  def create_unless_exists!
    save! unless self.class.exists?(sha: sha)
  end
  
  def prepared_content
    is_text ? HexDataProcessor.clean_utf8(content) : content
  end
  
  def self.ungzip_if_necessary!(binary)
    HexDataProcessor.ungzip_if_necessary(binary).tap do |res|
      if res.nil?
        raise InvalidInputError, "Failed to decompress content"
      end
    end
  end
  
  def self.validate_input!(decoded_data)
    if decoded_data['content'].nil? || decoded_data['contentType'].nil?
      raise InvalidInputError, "Missing required fields: content, contentType"
    end
  end
end
```
