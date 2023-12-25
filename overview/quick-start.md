# Quick Start

### Create an Ethscription in 60 Seconds

[Ethscriptions.com](https://ethscriptions.com) has an [easy creation tool](https://ethscriptions.com/create), but if you want to go step-by-step:

1. Convert an image (max size: \~90KB) to a Base 64-encoded data URI (`data:image/png;base64,...`) using a service like [base64-image.de](https://www.base64-image.de/). The Ethscriptions protocol supports all data URIs but images work best.
2. Convert the data URI to hex using an online tool like [hexhero](https://www.hexhero.com/converters/utf8-to-hex).
3. Send a 0 eth transaction _to the person you want to own the Ethscription_ with the hex data from (2) in the "Hex data" field
4. After a few moments it should appear on this site, _provided someone hasn’t already Ethscribed the same data_. Duplicate content is ignored!

### How to Transfer Ethscriptions

1. Find the id of the Ethscription you want to transfer. An Ethscription’s id is the transaction hash of the transaction that created the Ethscription. It looks like this: `0xcdb372580242c1c1bbcd2914ddbdb609b33d2e2e163c6595e164cb4dc6665153`. You can get this from Etherscan or from this site.
2. Send a 0 ETH transaction to the new proposed owner, including the Ethscription ID in the "Hex data" field.
3. After a few moments the Ethscription’s owner should update on this site, _provided you were the owner of the Ethscription when you sent the transfer transaction_. Unauthorized transfers are ignored.

### How to Track Ethscriptions

You can use [ethscriptions.com](https://ethscriptions.com)! However if you don’t want to rely on me you can track things yourself as all the necessary data is publicly available and uncensorable.

You will need an indexer, and soon the indexer that powers this website will be open sourced. In the meantime it's possible to build your own indexer that follows [the rules of the protocol](how-ethscriptions-work.md).
