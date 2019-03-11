---
blip: <to be assigned>
title: Attestation data string format guidelines
author: Dustin van Schouwen @djvs
status: Draft
created: 2019-03-11
---

<!--You can leave these HTML comments in your merged BLIP and delete the visible duplicate text guides, they will not appear and may be helpful to refer to if you edit it again. This is the suggested template for new BLIPs. Note that a BLIP number will be assigned by an editor. When opening a pull request to submit your BLIP, please use an abbreviated title in the filename, `blip-draft_title_abbrev.md`. The title should be 44 characters or less.-->

# Standardization of attestation data 

## Simple Summary
<!--"If you can't explain it simply, you don't understand it well enough." Provide a simplified and layman-accessible explanation of the BLIP.-->
Provide guidelines and expectations for commonalities in attestation data format

## Abstract
<!--A short (~200 word) description of the technical issue being addressed.-->
Attestation data string format is currently unspecified, leading to ambiguity and variation in the data strings used for different attestations. Guidelines are proposed for structuring JSON objects for consistent attestation data formatting.

## Motivation
<!--The motivation is critical for BLIPs that want to change the Bloom protocol. It should clearly explain why the existing protocol specification is inadequate to address the problem that the BLIP solves. BLIP submissions without sufficient motivation may be rejected outright.-->
To provide loose standardization for attestation data string format.

## Specification/Rationale
<!--The technical specification should describe the syntax and semantics of any new feature. The specification should be detailed enough to allow competing, interoperable implementations for any of the current Bloom platforms.-->
<!--The rationale fleshes out the specification by describing what motivated the design and why particular design decisions were made. It should describe alternate designs that were considered and related work, e.g. how the feature is supported in other languages. The rationale may also provide evidence of consensus within the community, and should discuss important objections or concerns raised during discussion.-->

Attestation data strings should be a string parseable in Javascript as a JSON object, without error.  The minimum format, a "null" attestation, would thus be `"{}"`.  

As attestations are intended to be a completely open-ended data structure, their format cannot be definitively specified - the aim of this proposal is to standardize fields that, when present, should be named according to the specification.

JSON objects were chosen for simplicity, machine readability, extensive language/library support, and ease of use on web applications.  Key names and structures are chosen for simplicity, broad applicability, and intuitiveness.

The following code contains a Typescript interface, which could be "extended" with additional keys, to produce the data structure to be passed through `JSON.stringify` or an equivalent function.  All dates should be written as ISO 8601-compliant date or date and time strings, expressed in UTC time - `YYYY-MM-DD` or `YYYY-MM-DDThh:mm:ssZ`. A question mark denotes an optional field in Typescript.

All attestation types must provide a "@context" field to provide a minimum level of JSON-LD compliance.  This may take fthe formatArray of URLs providing [JSON-LD](https://json-ld.org/) specification for data.  Field is considered mandatory.  If possible, URL should be permanently accessible - the second possible format for context URLs provides an open-ended way for arbitrary protocols to be used as long as they provide a stable reference for the specification - for instance, an IPFS resource locator, a BitTorrent magnet link, or just a hash of the specification (e.g., {type: 'sha256', data: "93c5291f6f047ecf196d08794780a3129a014fde5a4658f3a315dbcd223852ce"}).

```typescript
type TContextField = string | {type: string, data: string}

interface IBaseAttestation {
  "@context": TContextField | Array<TContextField>,
  date?: string, // Date/time attestation was performed
  expiry_date?: string, // Date/time when attestation should be considered void (e.g., for credential expiry)
  summary?: {
    date?: string, // Single date/time during which attestation was applicable
    start_date?: string, // Start date/time of period during which attestation was applicable
    end_date?: string, // End date/time of period during which attestation was applicable
    specificity?: integer, // Different levels of specificity - zero-indexed, with increasing numbers indicating less specificity, to allow for unlimited levels of depth (in practice, 3-5 should be sufficient for most cases).  This allows for arbitrary levels or amounts of specificity within sub-attestations, to promote an attestation subject's ability to partially disclose the amount of data provided in an attestation.
    // Extensible with other fields that summarize the content of the attestation - e.g., a list of addresses, accounts, totals of statistics, etc.
  },
  data?: any 
  // Extensible with other fields.  Other fields explicating general data about the attestation, such as location, shelf life, common units, etc., should be placed here.
}
```

Guidelines for specific attestation types are also provided:

```typescript
// "name" attestation data format.  
interface IAttestationName extends IBaseAttestation {
  data: string | {
    full?: string,
    first?: string,
    middle?: string,
    last?: string,
    title?: string,
    prefix?: string,
    suffix?: string
  }
}
```

```typescript
// "name" attestation phone format.  
interface IAttestationName extends IBaseAttestation {
  data: string | {
    full?: string,
    country?: string,
    subscriber?: string,
    area?: string,
    prefix?: string,
    line?: string,
    ext? :string
  }
}
```

```typescript
// "account" attestation phone format.  
interface IAttestationName extends IBaseAttestation {
  data: {
    id?: number | string,
    email?: string,
    start_date?: string,
    end_date?: string
  }
}
```

## Backwards Compatibility
<!--All BLIPs that introduce backwards incompatibilities must include a section describing these incompatibilities and their severity. The BLIP must explain how the author proposes to deal with these incompatibilities. BLIP submissions without a sufficient backwards compatibility treatise may be rejected outright.-->
Several existing attestation implementations are non-compliant with the specification, and should be updated to reflect the new format to encourage interoperability between protocol participants.  

## Test Cases
<!--Test cases for an implementation are mandatory for BLIPs that are affecting governance changes. Other BLIPs can choose to include links to test cases if applicable.-->
Native Javascript `JSON.parse()` should succeed without error, and producing a Javascript object, on any attestation data string, according to ECMAScript standards (generally, adhering to the JSON v1.0 specification - the use of any present or future features with partial support are discouraged).  Testing for mandatory keys can also provide assurance of specification compliance.

## Implementation
<!--The implementations must be completed before any BLIP is given status "Final", but it need not be completed before the BLIP is accepted. While there is merit to the approach of reaching consensus on the specification and rationale before writing code, the principle of "rough consensus and running code" is still useful when it comes to resolving many discussions of API details.-->
The implementations must be completed before any BLIP is given status "Final", but it need not be completed before the BLIP is accepted. While there is merit to the approach of reaching consensus on the specification and rationale before writing code, the principle of "rough consensus and running code" is still useful when it comes to resolving many discussions of API details.

## Copyright
Copyright and related rights waived via [CC0](https://creativecommons.org/publicdomain/zero/1.0/). Based on the Ethereum Improvement Proposal template.

