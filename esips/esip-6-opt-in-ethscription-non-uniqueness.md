# ESIP-6: Opt-in Ethscription Non-uniqueness

## Abstract

Currently, only the first ethscription with a given content uri is valid.

For example, if there is an existing ethscription with content `data:,1234`, then no future ethscription can be created with this same content.

This mechanic was designed for the digital artifact use-case when it is valuable to know provenance. It also makes Ethscriptions content-addressable, allowing users to look up ownership and other metadata using only ethscription content.

However, uniqueness creates problems for use-cases where guaranteed delivery is necessary. For example, if two people are using Ethscriptions as a messaging protocol, they shouldn't have to worry about making each message globally unique.

This problem is more acute in the case of Smart Contract-created ethscriptions because while people can "try again" if their ethscription is a duplicate, Smart Contracts cannot "revert" in the case of ethscription creation failure.

For example, if a Smart Contract has collected money from a user in exchange for creating an ethscription, the Smart Contract cannot return this money if the creation fails.

ESIP-6 proposes a backwards-compatible mechanism to support all of these use-cases. By default, duplicate ethscriptions will continue to be invalid, as they are today. However, users will be able to modify dataURIs to "opt-in" to potential duplication on ethscriptions they create after this ESIP is live.

## Specification

#### Opt In Non-uniqueness

To opt in to potential duplication, a user must add a special "magic" parameter to their dataURI.

In a dataURI, parameters are strings that appear after the mimetype and before the optional "base64" and the start of the content. The most common use of parameters is to specify a character encoding for the dataURI as in this example:

`data:text/plain;charset=utf-8,hi`

In this dataURI, `charset` is a parameter and it has the value `utf-8`.

We will discuss the choice of magic parameter below, but for now let's assume it is `rule=esip6`.

If a user wants to mark an ethscription "okay to duplicate" they would add the parameter `rule=esip6` to their dataURI. For example:

`data:text/plain;charset=utf-8;rule=esip6,hi`

If there were no other parameter, it would look like this:

`data:text/plain;rule=esip6,hi`

Marking an ethscription "okay to duplicate" also guarantees that it will never be invalidated as a duplicate itself because any potential duplicate would also contain the parameter `rule=esip6` which marks _it_ as "okay to duplicate."

#### Updated Indexer Behavior

To implement this ESIP, indexers must change their behavior. Here is how an indexer should determine if a new ethscription is valid.

1. Determine whether the ethscription's content is a valid dataURI. The rules for dataURI validity are **not** changing in this ESIP. Everything that was a valid dataURI previously is still valid, and everything that wasn't a valid dataURI is still invalid.
   1. If the ethscription has an invalid dataURI then it is an invalid ethscription. If it has a valid dataURI, proceed to step 2.\

2. Does the ethscription contain `rule=esip6` as a dataURI parameter?
   1. If yes, the ethscription is valid. If no, proceed to step 3.\

3. Does another ethscription created in an earlier block, or created in the same block but with an earlier transaction index, have the same content?
   1. If yes, the ethscription is invalid. If no, it is valid.

#### Parsing dataURI parameters

DataURI validity is defined by this Ruby regular expression:

```ruby
%r{
  data:
  (?<mediatype>
    (?<mimetype> .+? / .+? )?
    (?<parameters> (?: ; .+? = .+? )* )
  )?
  (?<extension>;base64)?
  ,
  (?<data>.*)
}x
```

Here is example code you can use to find the correct parameter using this regex:

```ruby
def is_esip6?(uri)
  match = REGEXP.match(uri)
  String(match[:parameters]).split(';').include?('rule=esip6')
end
```

#### Client Behavior

Ethscriptions clients are encouraged to indicate the presence of the `rule=esip6` parameter as well as the number of duplicates that exist for a specific `rule=esip6` ethscription.

Many clients display "Ethscription Numbers" that indicate the order in which a given ethscription was created. Clients are encouraged to continue assigning numbers to all valid ethscriptions, whether or not they include the `rule=esip6` parameter.

#### Smart Contract Behavior

Because Smart Contracts cannot "try again" in the case of duplicates, Smart Contracts should include the `rule=esip6` parameter in any scenario in which ethscription creation failure would lead to loss of funds or ethscriptions.

## Rationale

Ethscriptions cannot succeed as a general protocol without the ability to guarantee message delivery. If we can't rely on our ability to create ethscriptions, we can't rely on the creation of an ethscription to trigger something important, and this limits what we can use ethscriptions to do.

The immediate need for this ESIP comes from the fact that it is impossible to create a secure Ethscriptions VM bridge if Smart Contracts cannot reliably communicate with Dumb Contracts by creating ethscriptions.

However, this proposal is not restricted to Ethscription VM-related ethscriptions because the need for message delivery is more universal.

Why do it this way?

#### Why Not Change the Default to Allow Duplicates?

Even if this were a good change, it is too late to make.

We cannot change the default retroactively because people have relied on protocol rules to make important decisions and invalidating those decisions would irreparably damage trust in the protocol.

We also cannot change the default going forward because as we have seen this will still leave past ethscriptions un-duplicatable.

#### Front Running and Censorship

In the end, ESIP-6 isn't really about the ability to create duplicate ethscriptions.

It has always been possible to create "pseudo" duplicates of ethscriptions by varying parts of images that do not affect pixels but do affect the final bytes of the ethscription. It is also possible to "duplicate" a JSON object by creating a new object that shares keys and values but differs in some respect the JSON parser ignores.

Theoretically users could take advantage of this to ensure message delivery by creating a message that was a "pseudo" duplicate of an existing message but whose bytes were different.

Unfortunately, this fix does not work because of front running. Someone can always observe the ethscription you are creating and create the same one earlier in the same block. This ESIP gives users a method to create duplicates that cannot be censored by front runners and this is absolutely necessary for Ethscriptions to be the uncensorable protocol it was always intended to be.
