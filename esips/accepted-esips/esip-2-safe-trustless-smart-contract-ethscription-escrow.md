# ESIP-2: Safe Trustless Smart Contract Ethscription Escrow

Discuss and offer feedback [in this GitHub Issue](https://github.com/ethscriptions-protocol/ESIPs/issues/3).

## Abstract

This proposal introduces ESIP-2, an enhancement to the Ethscriptions Protocol that enables smart contracts to safely and trustlessly escrow Ethscriptions.

ESIP-2 accomplishes this by offering a mechanism for conditional transfers, relieving contracts from the requirement to identify the depositor of a given Ethscription.

## Specification

Add a new smart contract event into the Ethscriptions Protocol:

```solidity
event ethscriptions_protocol_TransferEthscriptionForPreviousOwner(
    address indexed previousOwner,
    address indexed recipient,
    bytes32 indexed ethscriptionId
);
```

When a contract emits this event, the protocol should register a valid ethscription transfer from the emitting contract to `recipient` of the ethscription with id `ethscriptionId`, provided:

1. The emitting contract owns the ethscription with id `ethscriptionId` when it emits the event.
2. The ethscription's previous owner was `previousOwner` as defined below.

An ethscription's "current owner" is the address that is in the "to" of the most recent valid transfer of that ethscription.

An ethscription's "previous owner" is the address that is in the "from" of the most recent valid transfer.

"Previous owner" doesn't necessarily mean "previous unique owner." For example, if you transfer an ethscription to me and then I transfer it to myself, I will be both the "current owner" and the "previous owner."

#### Implementation Guidelines

After ESIP-2, a valid ethscription transfer must have two properties:

1. Its "from" must equal the "to" of the previous valid transfer
2. Transfers sent under ESIP-2 will have an "enforced previous owner." In the case one exists, the enforced previous owner must equal the "from" of the previous valid transfer.

Below is an example of how an Ethscription transfers could be validated after the implementation of ESIP-2:

```javascript
const _ = require('lodash');

function validTransfers(ethscriptionTransfers) {
  const sorted = _.sortBy(
    ethscriptionTransfers,
    ['blockNumber', 'transactionIndex', 'transferIndex']
  );

  const valid = [];
  
  for (const transfer of sorted) {
    const lastValid = valid[valid.length - 1];
    const basicRulePasses = valid.length === 0 || transfer.from === lastValid.to;
    const previousOwnerRulePasses =
      transfer.enforcedPreviousOwner === null || 
      transfer.enforcedPreviousOwner === (lastValid?.from || null);

    if (basicRulePasses && previousOwnerRulePasses) {
      valid.push(transfer);
    }
  }

  return valid;
}
```

## Rationale

ESIP-2 is formulated primarily to enable smart contracts to safely escrow ethscriptions.

The idea of the smart contract escrow is that you send an ethscription to a smart contract, and, though that ethscription is owned by the smart contract, you retain some power over it—typically the ability to withdraw it and the ability to instruct the smart contract to send it to someone else.

Marketplaces are a common use-case for smart contract ethscription escrow. Because it is currently not possible for people to give smart contracts approval to transfer their ethscriptions, in order to list an ethscription for sale it must be transferred to the marketplace contract first.

With the introduction of `ethscriptions_protocol_TransferEthscription` in ESIP-1, smart contracts have the capability send and receive ethscriptions and function as marketplaces / escrows. However with just ESIP-1, smart contracts cannot obtain the information required to function as **safe** escrows without additional help.

The purpose of ESIP-2 is to enable smart contracts to overcome this limitation.

#### Who is the Depositor?

As an escrow, a smart contract should act to the benefit of the depositor of a given ethscription. However, because smart contracts cannot access ethscription ownership information, contracts cannot determine who deposited a given ethscription.

For example, if Alice and Bob both send a transaction to a smart contract with calldata `0xb1bdb91f010c154dd04e5c11a6298e91472c27a347b770684981873a6408c11c`, the smart contract can recognize this as a potential deposit, but it cannot know which (if either) of Alice or Bob's transactions is a legitimate deposit.

Because the contract can't determine the ethscription's depositor, it cannot determine who should have the power to control the ethscription once deposited. For example, the smart contract cannot determine who should have the power to withdraw the ethscription.

Because a smart contract cannot distinguish between Alice and Bob's deposits, it might treat them equally, leading to this exploit:

1. Alice "Deposits" id 0x123
2. Bob Deposits id 0x123
3. (Bob's deposit is real, Alice's isn't)
4. Alice requests a withdraw
5. Contract emits `ethscriptions_protocol_TransferEthscription(Alice, 0x123)`
6. Bob requests a withdraw
7. Contract emits `ethscriptions_protocol_TransferEthscription(Bob, 0x123)`

Alice owns the ethscription after (5) and the transfer in (7) fails.

#### Giving Contracts More Information

The most straightforward way to avoid this exploit is to require a trusted third party to confirm which deposits are valid.

For example, after you deposited id 0x123 you could go to the trusted party, ask them to verify it was your deposit that caused the contract to own id 0x123, and create a signed message memorializing this information.

Then you could present this signed message to the escrow contract to prove your deposit was legitimate. The contract would know to believe your message by comparing the signer of the message to the address of the trusted party.

Finally, when deposits are pending confirmation, they cannot be withdrawn, because the contract doesn’t know who should have the ability to do so.

Here's how the exploit would be foiled using this approach:

1. Alice "Deposits" id 0x123
2. Bob Deposits id 0x123
3. Smart Contract Freezes Assets
4. Third Party informs "Bob is real depositor"
5. Alice requests a withdraw
6. Contracts does nothing
7. Bob requests a withdraw
8. Contract emits `ethscriptions_protocol_TransferEthscription(Bob, 0x123)`

This solution works, but if the third party is not available it will be impossible for anyone to withdraw their assets. It would be preferable to have a decentralized alternative.

#### Reducing Contract Informational Needs

If a contract doesn't itself have a piece of information, it is not possible to deliver that information to the contract in a trustless fashion. Because of this, trustless solutions for contract escrow involve reducing the information a contract requires to make correct decisions, rather than supplying the contract with inaccessible information.

Specifically, ESIP-2 creates a mechanism for smart contracts to act in the interests of a depositor without having to know who that depositor is. Contracts achieve this through conditional transfers. Instead absolute transfers like "Send 0x123 to Alice," contracts can say "Send 0x123 to Alice, _if and only if_ Alice deposited 0x123."

Now the potential exploit looks like this:

1. Alice "Deposits" id 0x123
2. Bob Deposits id 0x123
3. (Bob's deposit is real, Alice's isn't)
4. Alice requests a withdraw
5. Contracts emits `ethscriptions_protocol_TransferEthscriptionForPreviousOwner(Alice, Alice, 0x123)`
6. Bob requests a withdraw
7. Contract emits `ethscriptions_protocol_TransferEthscriptionForPreviousOwner(Bob, Bob, 0x123)`

With ESIP-2 the contract doesn't have to gather the information necessary to determine which of Alice and Bob's withdrawal requests are legitimate and to change its behavior accordingly.

Instead, the contract does the same thing for Alice's withdraw as it does for Bob's. However, because `TransferEthscriptionForPreviousOwner` is only valid when Alice is the legitimate previous owner—which she cannot be here as her deposit is invalid—this transfer is invalid under the protocol and, like all invalid transfers, will be ignored by indexers.

The goal is to make smart contracts "dumber." Instead of smart contracts having to decide which user requests to ignore based on different user permissions, the smart contract can treat all user requests the same, knowing that the invalid requests will be filtered out at the protocol level.

## Example Smart Contract

This is an example implementation of an `EthscriptionsEscrower` base contract that a marketplace can inherit from.

For example, a marketplace would call something like `_transferEthscription(seller, msg.sender, ethscriptionId)` in the "buy" function.

In addition to ESIP-2 it contains an additional best practice of an enforced 5 block cooldown period between transfers to account for potential indexer delays and reorgs.

```solidity
// SPDX-License-Identifier: MIT
pragma solidity 0.8.17;

library EthscriptionsEscrowerStorage {
    struct Layout {
        mapping(address => mapping(bytes32 => uint256)) ethscriptionReceivedOnBlockNumber;
    }

    bytes32 internal constant STORAGE_SLOT =
        keccak256('ethscriptions.contracts.storage.EthscriptionsEscrowerStorage');

    function s() internal pure returns (Layout storage l) {
        bytes32 slot = STORAGE_SLOT;
        assembly {
            l.slot := slot
        }
    }
}

contract EthscriptionsEscrower {
    error EthscriptionNotDeposited();
    error EthscriptionAlreadyReceivedFromSender();
    error InvalidEthscriptionLength();
    error AdditionalCooldownRequired(uint256 additionalBlocksNeeded);
    
    event ethscriptions_protocol_TransferEthscriptionForPreviousOwner(
        address indexed previousOwner,
        address indexed recipient,
        bytes32 indexed id
    );
    
    event PotentialEthscriptionDeposited(
        address indexed owner,
        bytes32 indexed potentialEthscriptionId
    );
    
    event PotentialEthscriptionWithdrawn(
        address indexed owner,
        bytes32 indexed potentialEthscriptionId
    );
    
    uint256 public constant ETHSCRIPTION_TRANSFER_COOLDOWN_BLOCKS = 5;
    
    function _transferEthscription(address previousOwner, address to, bytes32 ethscriptionId) internal virtual {
        _validateTransferEthscription(previousOwner, to, ethscriptionId);
        
        emit ethscriptions_protocol_TransferEthscriptionForPreviousOwner(previousOwner, to, ethscriptionId);
        
        _afterTransferEthscription(previousOwner, to, ethscriptionId);
    }
    
    function withdrawEthscription(bytes32 ethscriptionId) public virtual {
        _transferEthscription(msg.sender, msg.sender, ethscriptionId);
        
        emit PotentialEthscriptionWithdrawn(msg.sender, ethscriptionId);
    }
    
    function _onPotentialEthscriptionDeposit(address previousOwner, bytes memory userCalldata) internal virtual {
        if (userCalldata.length != 32) revert InvalidEthscriptionLength();
        
        bytes32 potentialEthscriptionId = abi.decode(userCalldata, (bytes32));
        
        if (userEthscriptionPossiblyStored(previousOwner, potentialEthscriptionId)) {
            revert EthscriptionAlreadyReceivedFromSender();
        }

        EthscriptionsEscrowerStorage.s().ethscriptionReceivedOnBlockNumber[previousOwner][potentialEthscriptionId] = block.number;
        
        emit PotentialEthscriptionDeposited(previousOwner, potentialEthscriptionId);
    }
    
    function _validateTransferEthscription(
        address previousOwner,
        address to,
        bytes32 ethscriptionId
    ) internal view virtual {
        if (userEthscriptionDefinitelyNotStored(previousOwner, ethscriptionId)) {
            revert EthscriptionNotDeposited();
        }
        
        uint256 blocksRemaining = blocksRemainingUntilValidTransfer(previousOwner, ethscriptionId);
        
        if (blocksRemaining != 0) {
            revert AdditionalCooldownRequired(blocksRemaining);
        }
    }
    
    function _afterTransferEthscription(
        address previousOwner,
        address to,
        bytes32 ethscriptionId
    ) internal virtual {
        delete EthscriptionsEscrowerStorage.s().ethscriptionReceivedOnBlockNumber[previousOwner][ethscriptionId];
    }
    
    function blocksRemainingUntilValidTransfer(
        address previousOwner,
        bytes32 ethscriptionId
    ) public view virtual returns (uint256) {
        uint256 receivedBlockNumber = EthscriptionsEscrowerStorage.s().ethscriptionReceivedOnBlockNumber[previousOwner][ethscriptionId];
        
        if (receivedBlockNumber == 0) {
            revert EthscriptionNotDeposited();
        }
        
        uint256 blocksPassed = block.number - receivedBlockNumber;
        
        return blocksPassed < ETHSCRIPTION_TRANSFER_COOLDOWN_BLOCKS ?
            ETHSCRIPTION_TRANSFER_COOLDOWN_BLOCKS - blocksPassed :
            0;
    }
    
    function userEthscriptionDefinitelyNotStored(
        address owner,
        bytes32 ethscriptionId
    ) public view virtual returns (bool) {
        return EthscriptionsEscrowerStorage.s().ethscriptionReceivedOnBlockNumber[owner][ethscriptionId] == 0;
    }
    
    function userEthscriptionPossiblyStored(
        address owner,
        bytes32 ethscriptionId
    ) public view virtual returns (bool) {
        return !userEthscriptionDefinitelyNotStored(owner, ethscriptionId);
    }
    
    fallback() external virtual {
        _onPotentialEthscriptionDeposit(msg.sender, msg.data);
    }
}
```
