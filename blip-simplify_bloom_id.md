---
blip: <to be assigned>
title: Simplify BloomID
author: Isaac Patka (@ipatka)
status: Draft
created: 2018-11-19
---

<!--You can leave these HTML comments in your merged BLIP and delete the visible duplicate text guides, they will not appear and may be helpful to refer to if you edit it again. This is the suggested template for new BLIPs. Note that a BLIP number will be assigned by an editor. When opening a pull request to submit your BLIP, please use an abbreviated title in the filename, `blip-draft_title_abbrev.md`. The title should be 44 characters or less.-->

## Simple Summary
<!--"If you can't explain it simply, you don't understand it well enough." Provide a simplified and layman-accessible explanation of the BLIP.-->
Remove numeric BloomIDs for every account. Instead only use the account registry to track address links. All current contracts that consume account registry will instead reference the subject address.

## Abstract
<!--A short (~200 word) description of the technical issue being addressed.-->
Bloom's account registry requires a BloomID be created for each address participating in the protocol. The BloomID is a simple incrementing uint. When a user links an additional address both addresses point to the same numeric BloomID. Attestations, Voting and Token Escrow Marketplace reference the numeric BloomID. AttestationLogic stores attestations by BloomID. Polls emit the BloomID of the user in a vote event. TokenEscrowMarketplace stores balances by BloomID. Creating a BloomID is most of the time a centralized process. Bloom's admin submits a createAccount transaction to the AttestationLogic contract. If the address is not taken the account is created and the counter is incremented. This account creation transaction is a prerequisite to all other protocol activity including voting and completing attestations. This frequently leads to race conditions where someone signs up and tries to complete an attestation or vote right away. The transactions have to be ordered, queued and broadcast. The centralization problem posed by this design pattern is addressed in BLIP0. This BLIP aims to reduce the complexity of the protocol's account handling. There are simpler design patterns to achieve more efficient account handling and user onboarding with the added benefit of being more decentralized.

```
  // Counter to generate unique account Ids
  uint256 numAccounts;
  mapping(address => uint256) public accountByAddress;
  ```

## Motivation
<!--The motivation is critical for BLIPs that want to change the Bloom protocol. It should clearly explain why the existing protocol specification is inadequate to address the problem that the BLIP solves. BLIP submissions without sufficient motivation may be rejected outright.-->
Bloom frequently encounters race conditions surrounding account creation and other protocol activities like voting and attestations. This complexity is unnecessary and provides little value. Any user should be able to use the smart contracts regardless of whether they have had a numeric BloomID generated.

## Specification
<!--The technical specification should describe the syntax and semantics of any new feature. The specification should be detailed enough to allow competing, interoperable implementations for any of the current Bloom platforms.-->
BloomIDs are currently stored as a mapping of an address to a uint.

```
  // Counter to generate unique account Ids
  uint256 numAccounts;
  mapping(address => uint256) public accountByAddress;
```

When Bloom onboards a new user the account admin generates a `createAccount` transaction to claim that address for a user. When a user wishes to link another address to their BloomID a transcation is submitted to point the new address to the same numeric BloomID.

Instead, the `accountByAddress` mapping should be removed altogether. Users should be identified by an ethereum address. The AccountRegistry contract should instead maintain a registry of address links. If someone wishes to demonstrate ownership of multiple addresses they have each address sign a message demonstrating intent to be linked.

```

const newAddressLinkSig = ethSigUtil.signTypedDataLegacy(unclaimedPrivkey, {
  data: getFormattedTypedDataAddAddress(alice, nonce)
})

const currentAddressLinkSig = ethSigUtil.signTypedDataLegacy(alicePrivkey, {
  data: getFormattedTypedDataAddAddress(unclaimed, nonce)
})

export const getFormattedTypedDataAddAddress = (
  addressToAdd: string,
  nonce: string,
): IFormattedTypedData => {
  return {
    types: {
      EIP712Domain: [
          { name: 'name', type: 'string' },
          { name: 'version', type: 'string' },
      ],
      AddAddress: [
        { name: 'addressToAdd', type: 'address'},
        { name: 'nonce', type: 'bytes32'},
      ]
    },
    primaryType: 'AddAddress',
    domain: {
      name: 'Bloom',
      version: '1',
    },
    message: {
      sender: addressToAdd,
      nonce: nonce
    }
  }
}
```

An address can be linked to any number of addresses as long as the new address is unclaimed. Links can be broken by signing a message demonstrating intent to either unlink the signer or an address linked to the signer.

```
export const getFormattedTypedDataRemoveAddress = (
  addressToRemove: string,
  nonce: string,
): IFormattedTypedData => {
  return {
    types: {
      EIP712Domain: [
          { name: 'name', type: 'string' },
          { name: 'version', type: 'string' },
      ],
      AddAddress: [
        { name: 'addressToRemove', type: 'address'},
        { name: 'nonce', type: 'bytes32'},
      ]
    },
    primaryType: 'RemoveAddress',
    domain: {
      name: 'Bloom',
      version: '1',
    },
    message: {
      sender: addressToRemove,
      nonce: nonce
    }
  }
}
```

The new data structure to maintain a registry of these links is identical to the old data structure. However a transaction only needs to be processed to link addresses rather than create and link addresses. Linking happens far less often.
```
  // Counter to generate unique link Ids
  uint256 linkCounter;
  mapping(address => uint256) public linkIds;
```

### Attestations
Instead of an attestation referring to a numeric BloomID it will refer to the address of the subject. When a user wishes to share attested data with a 3rd party they will sign a challenge from the recipient with the same private key associated with the subject address. If they have linked an address to the attested address they can sign with the linked private key and the recipient can check the Account Regsitry to verify the two addresses are linked.

### Initialization
There should be an initialization period during which Bloom can migrate linked accounts to the new registry. This period will be temporary and disabled once all links are migrated. Bloom will have no ongoing privileges over the contract logic after initialization.

## Rationale
<!--The rationale fleshes out the specification by describing what motivated the design and why particular design decisions were made. It should describe alternate designs that were considered and related work, e.g. how the feature is supported in other languages. The rationale may also provide evidence of consensus within the community, and should discuss important objections or concerns raised during discussion.-->
Bloom spends a lot of time, ETH and resources carefully ordering account creation/ linking/ attestation transactions to deal with race conditions. This complexity adds little value to the protocol and can be simplified by converting the Account Registry to a Link Registry. There are two features impacted by this change.

If a user completes an attestation from one address then unlinks that address from their other addresses they may not be able to demonstrate ownership of attested data using the remaining addresses up to the requirements of a third party. Previously the attestation referenced a numeric BloomID which would also be referenced by the linked address. With the new system the attestation references the old address. The recipient may be willing to accept the fact that the addresses were linked at one point. However in the rare event that an address must be unlinked from a BloomID it would be prudent to just re-complete the relevant attestation.

TokenEscrowMarketplace currently stores tokens by BloomID and allows any linked address to withdraw or pay tokens. With this new design tokens will be stored by address and can only be released by the sender address. It is possible to implement a TokenEscrowMarketplace that checks for account links in the Account Registry. However the increased complexity is probably not worth the tradeoff in security.

## Backwards Compatibility
<!--All BLIPs that introduce backwards incompatibilities must include a section describing these incompatibilities and their severity. The BLIP must explain how the author proposes to deal with these incompatibilities. BLIP submissions without a sufficient backwards compatibility treatise may be rejected outright.-->
This is a breaking change that requires moderate modifications and rewrites of each contract that consumes AccountRegistry. This change can be incorporated into the BLIP0 Trustless Contracts change.

## Test Cases
<!--Test cases for an implementation are mandatory for BLIPs that are affecting governance changes. Other BLIPs can choose to include links to test cases if applicable.-->
Tests are covered in the core contracts repo tests.

## Implementation
<!--The implementations must be completed before any BLIP is given status "Final", but it need not be completed before the BLIP is accepted. While there is merit to the approach of reaching consensus on the specification and rationale before writing code, the principle of "rough consensus and running code" is still useful when it comes to resolving many discussions of API details.-->
Implementation is a WIP on the trustlessContracts branch of the core repo [Branch Link](https://github.com/hellobloom/core/tree/trustlessContracts).

## Copyright
Copyright and related rights waived via [CC0](https://creativecommons.org/publicdomain/zero/1.0/). Based on the Ethereum Improvement Proposal template.
