---
description: An overview of the protocol
---

# How Ethscriptions Work

## High Level

### Creating Ethscriptions

#### From an EOA

Any successful Ethereum transaction whose input data (when interpreted as UTF-8, see algorithm below for doing this) is a valid data URI (see spec below) and who has a "to" (i.e., is not a contract creation) creates an Ethscription, provided the data URI is unique.

For the URI to be unique, no Ethscription from a previous block or a transaction earlier in the block can have a dataURI with the same sha256. The sha is taken of the UTF-8 version of the dataURI.

The transaction hash of the transaction in which an ethscription was created is that ethscription's id. The recipient of the creation transaction is the Ethscription’s initial owner. The sender of the creation transaction is the Ethscription's creator.

#### From a Smart Contract

See details in [ESIP-3](https://docs.ethscriptions.com/esips/accepted-esips/esip-3-smart-contract-ethscription-creations).

### Transferring Ethscriptions

#### The Basics

The protocol defines for each ethscription a list of valid ethscription transfers. This list is ordered first by block number, then transaction index, then log index (in the case the transfer was triggered by an event).

The "from" in the first valid transfer is the ethscription's creator and the "to" in the final transfer is the ethscriptions current owner.

An ethscription's "previous owner" is the address that is in the "from" of the most recent valid transfer.

**Transferring Upon Ethscription Creation**

The creation of a new ethscription counts as a valid transfer from the "from" on the ethscription creation's transaction to the "to" on this same transaction. If I create an ethscription in a transaction with you as the "to," you are the current owner and I am the previous owner.

#### Transferring From EOAs

Any Ethereum transaction whose input data is an ethscription id as defined above is a valid Ethscription transfer, provided the transaction sender is the Ethscription’s owner. Because internal transactions from smart contracts do not have input data, this method only works for EOAs.

#### Transferring From Smart Contracts, Under ESIP-1

If a contract emits `ethscriptions_protocol_TransferEthscription`(signature below), the protocol should register a valid ethscription transfer from the emitting contract to `recipient` of the `ethscription` with id `ethscriptionId`, provided the emitting contract owns that ethscription when emitting the event, and the event is emitted in block `17672762` or a later block.

#### Transferring From Smart Contracts, Under ESIP-2

If a contract emits `ethscriptions_protocol_TransferEthscriptionForPreviousOwner`, the protocol should register a valid ethscription transfer from the emitting contract to `recipient` of the ethscription with id `ethscriptionId`, provided:

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

```ruby
class DataUri
  REGEXP = %r{
    data:
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
