# ESIP-7: Support gzipped Calldata in Ethscription Creation

## Abstract

For EOAs, the cost of creating an ethscription is determined by the size of the calldata payload. Therefore, to save gas, it is crucial to make this payload as small as possible.

Often this happens "for free" via the ethscribing of file formats that implement native compression, such as PNG.

However most ethscription content is JSON, a format that has no native compression. Because of this, a protocol-level compression solution has the potential to unlock massive savings.

Specifically, the total size of all EOA-created ethscriptions is about 1.5gb. Gzipping these ethscriptions would reduce size by more than 500mb, **a massive 35% reduction**!

This is somewhat skewed by outliers but the median ethscription would be reduced in size by 14%, which is still significant.

For fun, [here is the most-compressible ethscription that currently exists](https://ethscriptions.com/ethscriptions/0xf50ca8aa758b9ff524b7f7805703beae28c505c6a1fc513370b1390c9ba62c4c). Were it gzipped it would be 150x smaller! Though as we'll see below, it is not a candidate for this ESIP.

## Specification

Users may gzip calldata payloads for ethscriptions created via transaction inputs. This ESIP does not apply to contract-created ethscriptions or ethscription transfers.

Indexers should recognize gzipped ethscriptions via the magic leading byte sequence `0x1F8B`. When such an ethscription is recognized it should be unzipped and then processed normally.

When queried, ethscriptions should be returned in their uncompressed form so that users and API consumers do not have to change behavior.

**Compression Ratio Limit**

To avoid [zip bomb attacks](https://en.wikipedia.org/wiki/Zip\_bomb), gzipped calldata will only be valid if the compression ratio is less than or equal to 5x. For example, if the calldata is 10kb, it cannot decompress to more than 50kb, otherwise it is considered invalid.

When compressed, 99% of current ethscriptions would have a compression ratio of 3.85x or less, so a 5x limit should be plenty for all realistic use-cases.

**Reference Implementation**

Find a complete reference implementation for this ESIP in [this pull request](https://github.com/0xFacet/ethscriptions-indexer/pull/56). Here is the implementation for the compression ratio limit:

```ruby
def decompress_with_ratio_limit(data, max_ratio)
  original_size = data.bytesize
  decompressed = StringIO.new

  Zlib::GzipReader.wrap(StringIO.new(data)) do |gz|
    while chunk = gz.read(16.kilobytes) # Read in chunks
      decompressed.write(chunk)
      if decompressed.length > original_size * max_ratio
        return nil # Exceeds compression ratio limit
      end
    end
  end

  decompressed.string
rescue Zlib::Error
  nil
end
```

## Rationale

UTF-8 dataURIs are a clear and intuitive transport mechanism for ethscription content. For example, this approach allows anyone to consume the Ethscriptions Protocol using only Etherscan.

However, we have matured as a protocol to the point that minimizing expense is a more dominant concern.

Gzipping is a backwards compatible approach that will be completely transparent to the end user and requires only localized changes to indexers. Even with a "slow" language like Ruby, unzipping is extremely fast: on the order of 1ms for typical ethscription payloads.
