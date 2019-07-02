---
blip: <to be assigned>
title: Trustless Bloom Protocol Smart Contracts
author: Isaac Patka (@ipatka)
discussions-to: https://github.com/hellobloom/BLIPs/issues/5
status: Draft
created: 2018-11-19
---

<!--You can leave these HTML comments in your merged BLIP and delete the visible duplicate text guides, they will not appear and may be helpful to refer to if you edit it again. This is the suggested template for new BLIPs. Note that a BLIP number will be assigned by an editor. When opening a pull request to submit your BLIP, please use an abbreviated title in the filename, `blip-draft_title_abbrev.md`. The title should be 44 characters or less.-->

## Simple Summary
<!--"If you can't explain it simply, you don't understand it well enough." Provide a simplified and layman-accessible explanation of the BLIP.-->
Remove owner and admin entities from the Bloom Protocol smart contracts. The owner currently has the ability to swap out all contract logic. The admin is the only account with privileges to submit delegated transactions. This architecture is not sufficiently decentralized. 

## Abstract
<!--A short (~200 word) description of the technical issue being addressed.-->
Bloom's contracts have an owner and an admin. The owner currently has the ability to swap out all contract logic. The admin is the only account with privileges to submit delegated transactions. While useful for quick prototyping and bug fixes during the early stages of the protocol this is not a sufficiently decentralized architecture. The Bloom Company should have no higher privileges in the smart contracts than anyone else. To address this issue the onlyAdmin modifiers should be removed so anyone can submit delegated transactions. Any logic that relies on a trusted admin should be refactored to a trustless design. The contracts should not be ownable (see OpenZeppelin Ownable.sol contract currently inherited by all Bloom contracts). The contract logic should be immutable once deployed and configured. Protocol upgrades should involve a fork to newly deployed contracts rather than modifying mutable logic in currently deployed contracts.

## Motivation
<!--The motivation is critical for BLIPs that want to change the Bloom protocol. It should clearly explain why the existing protocol specification is inadequate to address the problem that the BLIP solves. BLIP submissions without sufficient motivation may be rejected outright.-->
Make Bloom's smart contracts trustless and decentralized in support of the company's mission.

## Specification
<!--The technical specification should describe the syntax and semantics of any new feature. The specification should be detailed enough to allow competing, interoperable implementations for any of the current Bloom platforms.-->

### Initializable.sol
Introduce Initializable.sol contract to allow for a temporary configuration and data migration period. During deployment the deployer sets the address of the initializer. The initializer can be an EOA or a smart contract. If an EOA it is recommended to be a hardware wallet address so the initializer can migrate data without having to copy private keys into a script. The initializer can be a smart contract to enable batching of migrations.

The initializer can be given privileges to configure interface contract addresses or migrate data in a trusted way. The `onlyDuringInitialization` modifier is used on these configuration and migration functions. The initializer calls `endInitialization` to conclude the initialization period.

```
pragma solidity 0.4.24;

/**
 * @title Initializable
 * @dev The Initializable contract has an initializer address, and provides basic authorization control
 * only while in initialization mode. Once changed to production mode the inializer loses authority
 */
contract Initializable {
  address public initializer;
  bool public initializing;

  event InitializationEnded();

  /**
   * @dev The Initializable constructor sets the initializer to the provided address
   */
  constructor(address _initializer) public {
    initializer = _initializer;
    initializing = true;
  }

  /**
   * @dev Throws if called by any account other than the owner.
   */
  modifier onlyDuringInitialization() {
    require(msg.sender == initializer, 'Method can only be called by initializer');
    require(initializing, 'Method can only be called during initialization');
    _;
  }

  /**
   * @dev Allows the initializer to end the initialization period
   */
  function endInitialization() public onlyDuringInitialization {
    initializing = false;
    emit InitializationEnded();
  }

}
```

### Remove onlyAdmin modifiers
All delegated transactions in the Bloom smart contracts currently have an `onlyAdmin` modifier restricting privileges to a Bloom owned address. The contract owner has privileges to set the admin address. These admin modifiers unnecessarily restrict the ability to submit delegated transactions. Anyone who gathers valid signatures from users should be allowed to submit the transactions and pay the gas costs. This change may increase the chances of a phishing attack by fraudulently obtaining a signature from an end user and submitting it on their behalf. 

Example from AccountRegistryLogic.sol

```
/**
   * @dev Restricted to registryAdmin
   */
  modifier onlyRegistryAdmin {
    require(msg.sender == registryAdmin);
    _;
  }

/**
  * @notice Change the address of the registryAdmin, who has the privilege to create new accounts
  * @dev Restricted to AccountRegistry owner and new admin address cannot be 0x0
  * @param _newRegistryAdmin Address of new registryAdmin
  */
function setRegistryAdmin(address _newRegistryAdmin) public onlyOwner nonZero(_newRegistryAdmin) {
  address _oldRegistryAdmin = registryAdmin;
  registryAdmin = _newRegistryAdmin;
  emit RegistryAdminChanged(_oldRegistryAdmin, registryAdmin);
}

/**
  * @notice Add an address to an existing id on behalf of a user to pay the gas costs
  * @param _newAddress Address to add to account
  * @param _newAddressSig Signed message from new address confirming ownership by the sender
  * @param _senderSig Signed message from address currently associated with account confirming intention
  * @param _sender User requesting this action
  * @param _nonce uuid used when generating sigs to make them one time use
  */
function addAddressToAccountFor(
  address _newAddress,
  bytes _newAddressSig,
  bytes _senderSig,
  address _sender,
  bytes32 _nonce
  ) public onlyRegistryAdmin {
  addAddressToAccountForUser(_newAddress, _newAddressSig, _senderSig, _sender, _nonce);
}
```

The admin also has privileges to generate BloomIDs for addresses. Without the admin there is a risk that anyone could generate BloomIDs for any number of addresses. They could not use the BloomIDs since they do not have the private keys associated with the address but they could introduce a race condition to interfere with account linking transactions causing them to fail.

There are a few options to deal with this vulnerability. One is to make it so anyone can generate a BloomID for an address but only if they have a valid signature from that address. The other is to refactor AccountRegistry so account creation transactions are no longer needed. The numeric BloomID may be completely unnecessary and instead we would only need on chain transactions to link/ unlink accounts. Attestations, Escrow, Voting would all reference an address instead of a BloomID uint.

This is covered more in a supplemental BLIP (TODO)

### Remove onlyOwner modifiers and Ownable.sol parent contract
All Bloom contracts currently inherit Ownable.sol from OpenZeppelin. The owner has the ability to change almost everything about the contracts. It can modify the Signing Logic, the Storage or Logic contract, set the admin address and more. The contracts should have no upgradeable components so Ownable is no longer needed.

This change would make it impossible to upgrade the signing logic contract. The signing logic contract was made upgradeable due to the immature state of EIP712. EIP712 has since been finalized and is in the process of being supported by various Web3 wallets. Signing logic will no longer be modifiable expect by hard forking the entire contract. This adds to the trustless nature of these new contracts so no one cannot change signing logic to a compromised contract.

### Join storage and logic contracts
Attestations and Account Registry were separated into two contrats each. One holds the storage variables and the other holds the logic to modify the storage. The rationale for this was that it is expensive to migrate storage on the blockchain so we should have an easy way to swap out logic. As the protocol has matured it is now apparent that the cost to migrate storage is worth the value we gain in decentralization/ trustlessness.

## Rationale
<!--The rationale fleshes out the specification by describing what motivated the design and why particular design decisions were made. It should describe alternate designs that were considered and related work, e.g. how the feature is supported in other languages. The rationale may also provide evidence of consensus within the community, and should discuss important objections or concerns raised during discussion.-->
This change was prompted by a close look at Bloom from a decentralization perspective. We identified what parts of the Bloom ecosystem belong to the centralized Bloom Company vs the decentralized Bloom Protocol. After analyzing the control we have over the smart contracts we decided they were far too centralized.

We also considered a recent report from Trail of Bits on [smart contract upgradeability](https://blog.trailofbits.com/2018/09/05/contract-upgrade-anti-patterns/). In summary all contract upgradeability patterns introduce vulnerabilites that are unnacceptable in decentralized systems. Instead smart contracts should use the [data migration pattern](https://blog.trailofbits.com/2018/09/05/contract-upgrade-anti-patterns/).

## Backwards Compatibility
<!--All BLIPs that introduce backwards incompatibilities must include a section describing these incompatibilities and their severity. The BLIP must explain how the author proposes to deal with these incompatibilities. BLIP submissions without a sufficient backwards compatibility treatise may be rejected outright.-->
This change involves deploying a new set of smart contracts. Anything that relies on current contract addresses has to be updated. This change alone will not change the public contract interfaces.

## Test Cases
<!--Test cases for an implementation are mandatory for BLIPs that are affecting governance changes. Other BLIPs can choose to include links to test cases if applicable.-->
Tests are covered in the core contracts repo tests.

## Implementation
<!--The implementations must be completed before any BLIP is given status "Final", but it need not be completed before the BLIP is accepted. While there is merit to the approach of reaching consensus on the specification and rationale before writing code, the principle of "rough consensus and running code" is still useful when it comes to resolving many discussions of API details.-->
Implementation is a WIP on the trustlessContracts branch of the core repo [Branch Link](https://github.com/hellobloom/core/tree/trustlessContracts).

## Copyright
Copyright and related rights waived via [CC0](https://creativecommons.org/publicdomain/zero/1.0/). Based on the Ethereum Improvement Proposal template.
