---
description: 'Dumb Contracts: "So dumb, they''re smart"'
---

# DEPRECATED: ESIP-4: The Ethscriptions Virtual Machine

### ESIP-4 is now the Facet Protocol. Read the docs here: [https://docs.facet.org/](https://docs.facet.org/)

Dumb Contracts will not be integrated into the Ethscriptions protocol, but rather into Facet, which builts on top of Ethscriptions.

The Facet docs supersede everything in this document, which is left purely for historical interest.



###

## Abstract

ESIP-4 introduces the Ethscriptions Virtual Machine (ESC VM), a new protocol built on top of Ethscriptions. The ESC VM enhances the functionality of the Ethscriptions Protocol by enabling ethscriptions to function as computer commands in addition to digital artifacts. These computer commands allow users to interact with special computer programs called Dumb Contracts.

The ESC VM provides Dumb Contracts with an environment similar to the EVM, enabling Dumb Contract functionality to parallel that of Smart Contracts. However Dumb Contracts are much cheaper than Smart Contracts because they bypass EVM execution and storage costs.

The Ethscriptions Virtual Machine is not yet a true VM in that it currently executes only curated Dumb Contracts. This will change in the future as we progress toward bringing general purpose computation to Ethscriptions.

## Specification

Add two "special" ethscription formats to Ethscriptions. Ethscriptions made using these formats will count as normal ethscriptions but they will also be recognized by the ESC VM.

Specifically, these ethscriptions are to be interpreted not only as digital artifacts, but as "computer commands" as well. These commands are "deploy" and "call." Deploy creates a new Dumb Contract and Call calls a state-changing function on an existing Dumb Contract.

#### Deploy

Anyone can issue a "deploy" command by creating a new ethscription with `0x0000000000000000000000000000000000000000` as the "to" / initial owner and with JSON content in this format:

```json
data:application/vnd.esc.contract.deploy+json,{
  "protocol": "SimpleToken",
  "constructorArgs": {
    "name": "My Fun Token",
    "symbol": "FUN",
    "maxSupply": "21000000",
    "perMintLimit": "1000"
  }
}
```

This ethscription represents the deployment of a Dumb Contract with the provided constructor arguments. The created contract's id is the ethscription's id (i.e., the transaction hash of the deployment transaction).

The "protocol" field contains the protocol name in traditional Smart Contract camel case. Each named protocol will be specified in an ESIP. Protocols are defined (but not necessarily executed) with Solidity code, as we will see.

#### Call

Once a Dumb Contract is deployed, its functions can be called. Anyone can call any Dumb Contract by sending an ethscription in this format to `address(0)`:

```json
data:application/vnd.esc.contract.call+json,{
  "contractId": "0xb1bdb91f010c154dd04e5c11a6298e91472c27a347b770684981873a6408c11c",
  "functionName": "mint",
  "args": {
    "amount": 1000
  }
}
```

* `contractId` refers to the called contract's id, which is the transaction hash of the ethscription that deployed the Dumb Contract.
* `functionName` and `args` are instructions for which Dumb Contract function to call and with what arguments.

These "calls" are _write_ operations and, because they are initiated via Ethereum transaction, the result of the call is not available synchronously.

However, as we will see below, conforming indexers must make "call receipts" available as well as a "staticCall" endpoint for synchronous read-only data access.

#### Writing Dumb Contracts

Dumb Contracts are specified using Solidity code because programming languages communicate protocol logic more efficiently than English prose and Solidity is the most widely used and understood language in blockchain development.

However everyone implementing the Dumb Contracts protocol can use whatever language they want as long as their implementation matches the behavior of the Solidity specification.

Here is an example specification for the contract we deployed above:

```solidity
// SPDX-License-Identifier: MIT
pragma solidity 0.8.20;

contract SimpleToken {
    event Transfer(address indexed from, address indexed to, uint256 value);
    
    string public name;
    string public symbol;
    
    uint256 immutable public maxSupply;
    uint256 immutable public perMintLimit;
    
    uint256 public totalSupply;
    
    mapping(address => uint256) public balanceOf;
    
    constructor(
        string memory _name,
        string memory _symbol,
        uint256 _maxSupply,
        uint256 _perMintLimit
    ) {
        name = _name;
        symbol = _symbol;
        maxSupply = _maxSupply;
        perMintLimit = _perMintLimit;
    }

    function mint(uint256 amount) public {
        require(amount > 0, "amount must be positive);
        require(amount <= perMintLimit, "amount exceeds perMintLimit");
        require(amount + totalSupply <= maxSupply, "amount exceeds maxSupply");
        
        totalSupply += amount;
        balanceOf[msg.sender] += amount;
        
        emit Transfer(address(0), msg.sender, amount);
    }

    function transfer(address to, uint256 amount) public {
        require(balanceOf[msg.sender] >= amount, "insufficient balance");
      
        balanceOf[msg.sender] -= amount;
        balanceOf[to] += amount;
        
        emit Transfer(msg.sender, to, amount);
    }
}
```

Remember, this code is just meant to communicate how the `SimpleToken` works!

This Solidity code will _not_ be deployed to mainnet. Protocol implementations will be validated by their behavior, not by what language they use or their execution environment.

#### Rubidity: An Example Execution Environment

In developing the Dumb Contracts protocol we built out an approach to executing them using a Ruby DSL we call "Rubidity." The goal of Rubidity is to create a line-by-line port of the Solidity language that can run on any computer.

Here is how the `SimpleToken` Dumb Contract might be implemented in Rubidity:

```solidity
class Contracts::SimpleToken < ContractImplementation
  event :Transfer, { from: :addressOrDumbContract, to: :addressOrDumbContract, amount: :uint256 }
  
  string :public, :name
  string :public, :symbol
  
  uint256 :public, :maxSupply
  uint256 :public, :perMintLimit
  
  uint256 :public, :totalSupply
  
  mapping ({ addressOrDumbContract: :uint256 }), :public, :balanceOf
  
  constructor(
    name: :string,
    symbol: :string,
    maxSupply: :uint256,
    perMintLimit: :uint256,
  ) {
    s.name = name
    s.symbol = symbol
    s.maxSupply = maxSupply
    s.perMintLimit = perMintLimit
  }
  
  function :mint, { amount: :uint256 }, :public do
    require(amount > 0, 'Amount must be positive')
    require(amount <= s.perMintLimit, 'Exceeded mint limit')
    
    require(s.totalSupply + amount <= s.maxSupply, 'Exceeded max supply')
    
    s.totalSupply += amount
    s.balanceOf[msg.sender] += amount
    
    emit :Transfer, from: address(0), to: msg.sender, amount: amount
  end

  function :transfer, { to: :addressOrDumbContract, amount: :uint256 }, :public do
    require(s.balanceOf[msg.sender] >= amount, 'Insufficient balance')
    
    s.balanceOf[msg.sender] -= amount
    s.balanceOf[to] += amount

    emit :Transfer, from: msg.sender, to: to, amount: amount
    
    return true
  end
end
```

Rubidity Dumb Contracts are then executed by a "Contract Controller" that manages state, nested calls, receipt generation and so forth. The goal of the system, which we will soon open source, is to simulate the EVM environment so Contract developers can write Dumb Contracts  with their existing knowledge.

#### Dumb Contract Globals

Dumb Contracts have access to some global blockchain state, and this access is represented with Solidity global variables, though the meaning is a bit different.

Specifically the block-related variables refer to the block in which the `call` ethscription was included, meaning they refer to a past block, unlike in the EVM where they refer to an "in-progress" block.

* `blockhash(uint blockNumber) returns (bytes32)`: hash of the given block when `blocknumber` is one of the 256 most recent blocks; otherwise returns zero
* `block.number` (`uint`): the number of the block in which the `call` ethscription was included
* `block.timestamp` (`uint`): the timestamp of the block in which the `call` ethscription was included
* `msg.sender` (`address`): creator of the `call` ethscription

#### Accessing the Ethscriptions Ecosystem

It is useful for Dumb Contracts to interact with Ethscriptions, for example to determine who owns an ethscription or what its content is. Because Solidity has no concept of Ethscriptions, we include an `esc` global in our specification language with the following functions:

```solidity
esc.findEthscriptionById(bytes32 id) internal returns (Ethscription memory)

esc.anyOwnedBy(address owner, bytes32[] memory ids) internal returns (bool)

esc.allOwnedBy(address owner, bytes32[] memory ids) internal returns (bool)

esc.anyOwnedByWithPreviousOwner(
    bytes32[] memory ids,
    address owner,
    address previousOwner
) internal returns (bool)

esc.allOwnedByWithPreviousOwner(
    bytes32[] memory ids,
    address owner,
    address previousOwner
) internal returns (bool)
```

`findEthscriptionById` returns instances of the following struct:

```solidity
struct Ethscription {
    bytes32 ethscriptionId;
    address owner;
    address creator;
    address previousOwner;
    uint256 creationTimestamp;
    uint256 blockNumber;
    uint256 transactionIndex;
    string mimetype;
    string contentSha256;
    string dataURI;
}
```

#### Reading Dumb Contract State

Without a way to query contract state, all of the above is moot. You could submit all the "call" ethscriptions you want but would have no way of verifying whether they did anything!

To read contract state, you make an `esc://` call to an Ethscriptions indexer that looks like this:

`esc://contracts/:contract_id/static-call/:function_name?args={args}&env={env}`

To convert this to an HTTP call you should replace `esc://` with the indexer's base API URI, for example `https://api.ethscriptions.com/api/`.

`args` and `env` should be JSON-encoded objects. The former specifying the function's arguments by name, and the latter specifying the environment or calling context of the static call. The user must specify the environment because ordinary Solidity globals don't have meaning in the static call context.

Currently the only supported `env` parameter is `msgSender`, which populate references to `msg.sender`.

The api response should either be&#x20;

{% code fullWidth="false" %}
```json
{
  "status": "error",
  "message": error_message
}
```
{% endcode %}

or

```json
{
  "status": "success",
  "result": result_data
}
```

#### Dumb Contract Call Receipts

Because the results of calls to Dumb Contracts are not available synchronously, there must be a separate mechanism for users to determine whether their calls succeeded, see emitted events if they did, and error messages if they failed.

These call receipts are available at the following URI:

`esc://contracts/call-receipts/:transactionHashOfCall`

Call receipts for successful calls look like this:

```json
{
  "callTransactionHash": "0x1234",
  "contractId": "0x1234",
  "status": "success",
  "functionName": "mint",
  "args": {
    "amount": 1000
  },
  "events": [{
    "name": "Transfer",
    "args": {
      "from": "0x0000000000000000000000000000000000000000",
      "to": "0x1234",
      "amount": 1000
    }
  }]
}
```

Call receipts for failed calls look like this:

```json
{
  "callTransactionHash": "0x1234",
  "contractId": "0x1234",
  "status": "failure",
  "functionName": "transfer",
  "args": {
    "to": "0x1234",
    "amount": 1000
  },
  "errors": [
    "insufficient balance"
  ]
}
```

#### Reorgs

Dumb Contracts have state, but their state is a theoretical construct that must always match Ethereum transaction history. This means that if there is a blockchain reorg, Dumb Contract state must be rolled back accordingly. This is true of all Ethscriptions state, but worth emphasizing here as Dumb Contract state can become arbitrarily complicated.

#### Limitations and Future Features

Dumb Contracts are the first step toward bringing general purpose computation to Ethscriptions. As they develop, we will work to overcome their two primary limitations.

First, Dumb Contract functions cannot be payable. Ether as a concept does not exist in "Dumb Contract Land" and there is no way to transfer it to a Dumb Contract. However, it is possible to "bridge" ether into Dumb Contracts with the following flow:

1. User sends ether to Smart Contract.
2. Smart Contract creates an ethscription that makes a call to a Dumb Contract, informing the Dumb Contract that the Smart Contract received ether.
3. The Dumb Contract verifies that the Smart Contract is a registry of trusted Smart Contracts and then sends the user an asset.

Conceptually, bridging out would be the same in reverse: the user sends the asset to the Dumb Contract and the Dumb Contract notifies the Smart Contract to release the ether.

However, while Smart Contracts can communicate with Dumb Contracts, the reverse is not possible. This means that a third party is required to verify that the Smart Contract is allowed to release the ether. This is no different from L2 bridge scenarios, but requires care to implement correctly.

The second and largest Dumb Contracts limitation is that this ESIP does not allow for arbitrary Dumb Contract creation. Dumb Contracts can be deployed and executed trustlessly, but the code itself is defined along with the rest of Ethscriptions Protocol rules in the ESIP process.

To lift this restriction we must build a true sandboxed virtual machine that protects Dumb Contracts against the malicious or careless actions of other Dumb Contracts. This is an enormous project, but well worth it.

## Rationale

Ethscriptions were designed to model the creation, ownership, and transfer of non-fungible digital artifacts. Despite the tremendous potential this functionality shows, it is fundamentally limited. It is not surprising that since launching we have heard the constant request: "Make Ethscriptions do more things!"

Just as Ethscriptions are a reinterpretation of ordinarily inert Ethereum calldata, we can add new functionality to Ethscriptions by reinterpreting the ethscriptions themselves.

For example, as in the BRC-20 standard on Bitcoin, a properly-formatted ethscription could represent not a digital artifact to be held or transferred, but rather a one-time command like "deploy this new token!"

#### The BRC-20 Approach

BRC-20 works like this. If a participating indexer sees a Bitcoin inscription with the below format, it knows that the inscription is "really" a BRC-20 deployment and updates its internal state to indicate that subsequent mints of this ticker are valid.

```json
{ 
  "p": "brc-20",
  "op": "deploy",
  "tick": "ordi",
  "max": "21000000",
  "lim": "1000"
}
```

Using this same approach, we can create as many new protocols as we want. For example, this inscription could be part of a ENS-like domain name registration protocol:

```json
{ 
  "p": "bns",
  "op": "register",
  "name": "middlemarch",
  "expiry_timestamp": "1720437604",
}
```

#### Building on BRC-20

BRC-20s are a groundbreaking technology and without them Ethscriptions and the ESC VM could not exist. We aim to build on BRC-20 by adapting its approach for general protocol creation. The ESC VM does this by:

1. Defining a clear familiar paradigm and domain language for talking about protocols (contracts, functions, arguments, etc).
2. Define a paradigm for protocols to communicate (contracts calling other contracts' public functions).
3. Providing a clear and consistent methodology and domain language (Solidity) for defining protocols.
4. Giving end-users a consistent API for accessing protocol state in the same way across protocols.
5. Clearly segmenting Dumb Contract ethscriptions from normal ethscriptions such that Dumb Contract ethscriptions cannot be mistakenly traded and a user will never accidentally make a Dumb Contract ethscription.

#### All Non-Buggy Indexers Must Behave the Same Way

The above improvements are nice, but are they merely "quality of life" improvements? Or does the ESC VM enable something fundamentally more powerful than what is possible with the BRC-20 approach?

The ESC VM implement these quality of life improvements in order to demand a fundamental improvement over BRC-20s: all protocols invented according to the Dumb Contracts standard are approved according to the ESIP process and are part of the Ethscriptions Protocol itself.

This means that all valid Ethscriptions indexers must implement all approved Dumb Contracts. Unlike in the BRC-20 case, where some Inscriptions indexers do not recognize BRC-20s, in Ethscriptions, all valid indexers will behave exactly the same way.

#### Is the ESC VM an L2?

The ESC VM is not an L2. One way to understand this is to consider the two notions of consensus that exist on Ethereum:

1. Consensus over what transactions are included in each block and in what order.
2. Consensus over the aggregate impact (1) has on the state of the EVM.

The main idea behind Ethscriptions is that you can build a fully decentralized system by focusing on (1) because the state of the blockchain unambiguously and deterministically specifies the state of the EVM. Given the blockchain alone, anyone can verify EVM state independently and with complete certainty.

On the other hand, it is impossible to verify the "truth" of (1) because it is a non-deterministic process with no "right answer."

Having (1) and (2) together as in the Ethereum protocol is ideal. However the combination is too expensive for most applications. Ethscriptions sacrifices part (2) of the Ethereum Protocol and builds tools to make the deterministic computation of state convenient.

L2s, by contrast, take the opposite approach. Because L2 state is managed in the context of a blockchain, it more convenient to verify than the state of the Ethscriptions ecosystem.

However L2 verification is conditional. It says _given_ X transactions were included in a block with ordering Y, we can infer the state of the blockchain should change to Z. But within the system of an L2 there is no way to verify that X and Y are correct.

And in the general case X and Y will only be fair when making them fair aligns with the goals of the organization that operates the L2. Corporations that operate L2s bear a fiduciary responsibility to value the interests of shareholders over the interests of L2 users. In the limit case, if the L2 no longer serves the corporation's interests, the L2 will be shut down.

Ethscriptions stand for the ideal that without decentralized consensus over non-deterministic questions like block inclusion and transaction ordering, a blockchain can never be considered secure.

Our goal with the ESC VM is to pair decentralization and security with functionality that approaches that of the EVM.

#### Conclusion

There is no one-size-fits-all solution for blockchain development. The goal of the ESC VM is not to replace Smart Contracts or L2s, but rather to provide lost cost computation when decentralization is a priority.

Today our goal is to release something simple and extensible as we work toward a world of general Ethscription computability.
