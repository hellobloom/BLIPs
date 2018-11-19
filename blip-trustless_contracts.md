---
blip: <to be assigned>
title: Trustless Bloom Protocol Smart Contracts
author: Isaac Patka (@ipatka)
status: Draft
created: 2018-11-19
---

<!--You can leave these HTML comments in your merged BLIP and delete the visible duplicate text guides, they will not appear and may be helpful to refer to if you edit it again. This is the suggested template for new BLIPs. Note that a BLIP number will be assigned by an editor. When opening a pull request to submit your BLIP, please use an abbreviated title in the filename, `blip-draft_title_abbrev.md`. The title should be 44 characters or less.-->

## Simple Summary
<!--"If you can't explain it simply, you don't understand it well enough." Provide a simplified and layman-accessible explanation of the BLIP.-->
Remove owner and admin entities from the Bloom Protocol smart contracts. The owner currently has the ability to swap out all contract logic. The admin is the only account with privileges to submit delegated transactions. This architecture is not sufficiently decentralized. 

## Abstract
<!--A short (~200 word) description of the technical issue being addressed.-->
Bloom's contracts have and owner and an admin. The owner currently has the ability to swap out all contract logic. The admin is the only account with privileges to submit delegated transactions. While useful for quick prototyping and bug fixes during the early stages of the protocol this is not a sufficiently decentralized architecture. The Bloom Company should have no higher privileges in the smart contracts than anyone else. To address this issue the onlyAdmin modifiers should be removed so anyone can submit delegated transactions. Any logic that relies on a trusted admin should be refactored to a trustless design. The contracts should not be ownable (see Open Zeppelin Ownable.sol contract currently inherited by all Bloom contracts). The contract logic should be immutable once deployed and configured. Protocol upgrades should involve a fork to newly deployed contracts rather than modifying mutable logic in currently deployed contracts.

## Motivation
<!--The motivation is critical for BLIPs that want to change the Bloom protocol. It should clearly explain why the existing protocol specification is inadequate to address the problem that the BLIP solves. BLIP submissions without sufficient motivation may be rejected outright.-->
Make Bloom's smart contracts trustless and decentralized in support of the company's mission.

## Specification
<!--The technical specification should describe the syntax and semantics of any new feature. The specification should be detailed enough to allow competing, interoperable implementations for any of the current Bloom platforms.-->
WIP

Introduce Initializable.sol contract to allow for a temporary configuration and data migration period

Remove onlyAdmin modifiers

Remove onlyOwner modifiers and Ownable.sol parent contract

Join storage and logic contracts

## Rationale
<!--The rationale fleshes out the specification by describing what motivated the design and why particular design decisions were made. It should describe alternate designs that were considered and related work, e.g. how the feature is supported in other languages. The rationale may also provide evidence of consensus within the community, and should discuss important objections or concerns raised during discussion.-->
The rationale fleshes out the specification by describing what motivated the design and why particular design decisions were made. It should describe alternate designs that were considered and related work, e.g. how the feature is supported in other languages. The rationale may also provide evidence of consensus within the community, and should discuss important objections or concerns raised during discussion.

## Backwards Compatibility
<!--All BLIPs that introduce backwards incompatibilities must include a section describing these incompatibilities and their severity. The BLIP must explain how the author proposes to deal with these incompatibilities. BLIP submissions without a sufficient backwards compatibility treatise may be rejected outright.-->
This change involves deploying a new set of smart contracts. Anything that relies on current contract addresses has to be updated. This change alone will not change the public contract interfaces.

## Test Cases
<!--Test cases for an implementation are mandatory for BLIPs that are affecting governance changes. Other BLIPs can choose to include links to test cases if applicable.-->
Tests are covered in the core contracts repo tests. TODO add link

## Implementation
<!--The implementations must be completed before any BLIP is given status "Final", but it need not be completed before the BLIP is accepted. While there is merit to the approach of reaching consensus on the specification and rationale before writing code, the principle of "rough consensus and running code" is still useful when it comes to resolving many discussions of API details.-->
Implementation is a WIP on the trustlessContracts branch of the core repo [Branch Link](https://github.com/hellobloom/core/tree/trustlessContracts).

## Copyright
Copyright and related rights waived via [CC0](https://creativecommons.org/publicdomain/zero/1.0/). Based on the Ethereum Improvement Proposal template.
