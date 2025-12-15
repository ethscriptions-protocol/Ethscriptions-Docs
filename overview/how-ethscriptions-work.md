---
description: An overview of the protocol
---

# Protocol Specification

## High Level

### Indexing Mechanics

Ethscriptions protocol state is determined by indexing all Ethereum transactions in order, starting with transactions in the Ethscriptions "genesis block" and proceeding from there sequentially in order of block number and transaction index within the block. The genesis blocks are:

<pre class="language-ruby"><code class="lang-ruby"><strong>ethscriptionsGenesisBlocks = [
</strong><strong>    1608625, 3369985, 3981254, 5873780, 8205613,
</strong><strong>    9046950, 9046974, 9239285, 9430552, 10548855, 10711341, 15437996, 17478950
</strong><strong>]
</strong></code></pre>

The Ethscriptions protocol only considers successful transaction. If a transaction has `status == 0` it must be ignored—i.e., transaction status must be either `1` or `null` as in the case of transactions with `blockNumber <= 4370000`.

### Creating Ethscriptions

#### From an EOA

Any successful Ethereum transaction whose input data (when interpreted as UTF-8, see algorithm below for doing this) is a valid data URI (see spec below) and who has a "to" (i.e., is not a contract creation) creates an Ethscription, provided the data URI is unique _or_ the data uri has the parameter `rule=esip6`. [Read more](https://docs.ethscriptions.com/esips/accepted-esips/esip-6-opt-in-ethscription-non-uniqueness).

For the URI to be unique, no Ethscription from a previous block or a transaction earlier in the block can have a dataURI with the same sha256. The sha is taken of the UTF-8 version of the dataURI.

The transaction hash of the transaction in which an ethscription was created is that ethscription's id. The recipient of the creation transaction is the Ethscription’s initial owner. The sender of the creation transaction is the Ethscription's creator.

DataURIs can be gzipped per ESIP-7 starting in block `19376500`. [Read more](https://docs.ethscriptions.com/esips/accepted-esips/esip-7-support-gzipped-calldata-in-ethscription-creation).

#### From a Smart Contract

See details in [ESIP-3](https://docs.ethscriptions.com/esips/accepted-esips/esip-3-smart-contract-ethscription-creations). The start block for ESIP-3 is `18130000`.

#### Ethscription Attachments

Per [ESIP-8](https://docs.ethscriptions.com/esips/esip-8-ethscription-attachments-aka-blobscriptions), starting in block `19526000` you can add attachments to an ethscription using EIP-4844 blobs.

### Transferring Ethscriptions

#### The Basics

The protocol defines for each ethscription a list of valid ethscription transfers. This list is ordered first by block number, then transaction index, then log index (in the case the transfer was triggered by an event).

The "from" in the first valid transfer is the ethscription's creator and the "to" in the final transfer is the ethscriptions current owner.

An ethscription's "previous owner" is the address that is in the "from" of the most recent valid transfer.

**Transferring Upon Ethscription Creation**

The creation of a new ethscription counts as a valid transfer from the "from" on the ethscription creation's transaction to the "to" on this same transaction. If I create an ethscription in a transaction with you as the "to," you are the current owner and I am the previous owner.

#### Transferring From EOAs

Any Ethereum transaction whose input data is an ethscription id as defined above is a valid Ethscription transfer, provided the transaction sender is the Ethscription’s owner. Because internal transactions from smart contracts do not have input data, this method only works for EOAs.

#### Transferring From EOAs (Under ESIP-5)

If the input data of a transaction (without its leading `0x`) is a sequence of 1 or more valid ethscription ids (without their leading `0x`), that transaction will constitute a valid transfer for each ethscription that is owned by the transaction's creator. The transaction must have occurred in `18330000` or a later block. [Read more](https://docs.ethscriptions.com/esips/accepted-esips/esip-5-bulk-ethscription-transfers-from-eoas).

#### Transferring From Smart Contracts, Under ESIP-1

If a contract emits `ethscriptions_protocol_TransferEthscription`(signature below), the protocol should register a valid ethscription transfer from the emitting contract to `recipient` of the `ethscription` with id `ethscriptionId`, provided the emitting contract owns that ethscription when emitting the event, and the event is emitted in block `17672762` or a later block.

#### Transferring From Smart Contracts, Under ESIP-2

If a contract emits `ethscriptions_protocol_TransferEthscriptionForPreviousOwner`, the protocol should register a valid ethscription transfer from the emitting contract to `recipient` of the `ethscription` with id `ethscriptionId`, provided:

1. The emitting contract owns the ethscription with id `ethscriptionId` when it emits the event.
2. The ethscription's previous owner was `previousOwner`.
3. The event is emitted in block `17764910` or a later block.

## The Details

#### How to interpret hex input data as UTF-8

Any method functionally equivalent to this code will work. Note that null bytes are removed even though they are valid UTF-8. This is a pragmatic choice based around the special behavior of these characters in postgres string columns.

```javascript
function hexToUTF8(hexString) {
  if (hexString.indexOf('0x') === 0) {
    hexString = hexString.slice(2);
  }

  const bytes = new Uint8Array(hexString.length / 2);

  for (let index = 0; index < bytes.length; index++) {
    const start = index * 2;
    const hexByte = hexString.slice(start, start + 2);
    const byte = Number.parseInt(hexByte, 16);
    if (Number.isNaN(byte) || byte < 0)
      throw new Error(
        `Invalid byte sequence ("${hexByte}" in "${hexString}").`
      );
    bytes[index] = byte;
  }

  let result = new TextDecoder().decode(bytes);
  return result.replace(/\0/g, '');
}
```

#### How to validate a dataURI

Any method functionally equivalent to this Ruby class will work. Note that any syntactically valid mimetype is allowed.

Base64 decoding is done according to RFC 4648 and is "strict." We do not attempt to recover from any encoding issues, meaning that `encode(decode(b64_string)) == b64_string` must be true.

A good test case is the string `str = "bD5="`. Lenient decoders will decode this to `l>`, but the correct encoding  of `l>` is `bD4=` and so `encode(decode(str)) != str`, which means str is not valid Base64 for our purposes.

```ruby
class DataUri
  REGEXP = %r{
    \Adata:
    (?<mediatype>
      (?<mimetype> .+? / .+? )?
      (?<parameters> (?: ; .+? = .+? )* )
    )?
    (?<extension>;base64)?
    ,
    (?<data>.*)
  }x.freeze

  def self.valid?(uri)
    match = REGEXP.match(uri)

    match && valid_base64_content?(match[:data], match[:extension])
  end

  private 

  def self.valid_base64_content?(data, extension)
    if extension
      begin
        Base64.strict_decode64(data)
        true
      rescue ArgumentError
        false
      end
    else
      true
    end
  end
end

```

#### ESIP-1 Transfer Event Signature

```solidity
ethscriptions_protocol_TransferEthscription(
  address indexed recipient,
  bytes32 indexed ethscriptionId
)
```

#### ESIP-2 Transfer Event Signature

```solidity
event ethscriptions_protocol_TransferEthscriptionForPreviousOwner(
    address indexed previousOwner,
    address indexed recipient,
    bytes32 indexed ethscriptionId
);
```
