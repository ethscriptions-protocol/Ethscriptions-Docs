# ESIP-7: Support Gzipped Calldata in Ethscription Creation

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

To avoid [zip bomb attacks](https://en.wikipedia.org/wiki/Zip\_bomb), gzipped calldata will only be valid if the compression ratio is less than or equal to 10x. For example, if the calldata is 10kb, it cannot decompress to more than 50kb, otherwise it is considered invalid.

When compressed, 99% of current ethscriptions would have a compression ratio of 3.85x or less, so a 10x limit should be plenty for all realistic use-cases.

**Reference Implementation**

Find a complete reference implementation for this ESIP in [this pull request](https://github.com/0xFacet/ethscriptions-indexer/pull/58). Here is the implementation for the compression ratio limit:

```ruby
module HexDataProcessor
  def self.hex_to_utf8(hex_string, support_gzip:)
    clean_hex_string = hex_string.gsub(/\A0x/, '')
    binary_data = hex_string_to_binary(clean_hex_string)
    
    if support_gzip && gzip_compressed?(binary_data)
      decompressed_data = decompress_with_ratio_limit(binary_data, 10)
    else
      decompressed_data = binary_data
    end
  
    return nil unless decompressed_data
    
    clean_utf8(decompressed_data)
  end

  def self.hex_string_to_binary(hex_string)
    ary = hex_string.scan(/../).map { |pair| pair.to_i(16) }
    ary.pack('C*')
  end

  def self.gzip_compressed?(data)
    data[0..1].bytes == [0x1F, 0x8B]
  end

  def self.decompress_with_ratio_limit(data, max_ratio)
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

  def self.clean_utf8(binary_data)
    utf8_string = binary_data.force_encoding('UTF-8')
    
    unless utf8_string.valid_encoding?
      utf8_string = utf8_string.encode('UTF-8', invalid: :replace, undef: :replace, replace: "\uFFFD")
    end
    
    utf8_string.delete("\u0000")
  end
end
```

## Rationale

UTF-8 dataURIs are a clear and intuitive transport mechanism for ethscription content. For example, this approach allows anyone to consume the Ethscriptions Protocol using only Etherscan.

However, we have matured as a protocol to the point that minimizing expense is a more dominant concern.

Gzipping is a backwards compatible approach that will be completely transparent to the end user and requires only localized changes to indexers. Even with a "slow" language like Ruby, unzipping is extremely fast: on the order of 1ms for typical ethscription payloads.
