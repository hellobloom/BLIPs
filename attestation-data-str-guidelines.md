---
blip: <to be assigned>
title: Attestation data string format guidelines
author: Dustin van Schouwen @djvs
status: Draft
created: 2019-03-11
---

<!--You can leave these HTML comments in your merged BLIP and delete the visible duplicate text guides, they will not appear and may be helpful to refer to if you edit it again. This is the suggested template for new BLIPs. Note that a BLIP number will be assigned by an editor. When opening a pull request to submit your BLIP, please use an abbreviated title in the filename, `blip-draft_title_abbrev.md`. The title should be 44 characters or less.-->

# Attestation data string format guidelines

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

**Typescript implementations of the IBaseAttestation interface (meant to be extended into interfaces specific to individual attestation types) [are provided in Addendum 1](#addendum-1), as well as implementations of attestations for the majority of currently implemented attestation types.**

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


## Addendum 1

### Typescript implementation of basic types

The current semi-authoritative implementation of the base types and  attestation-specific extension types is provided below, in standard Typescript syntax.  As discussed above, the types are considered to be non-binding and extensible, while being an attempt to provide a harmonized baseline for common attestation types.  Organizations or individuals who wish to extend these types are encouraged to extend them and publish their improvements, or ideally, to create a BLIP-proposal with suggested changes to Bloom's reference implementation.


```typescript
export type TContextField = string | {type: string; data: string}

///////////////////////////////////////////////////
// Base attestation dataStr type
///////////////////////////////////////////////////
export interface IBaseAtt {
  '@context': TContextField | Array<TContextField>
  // Date/time attestation was performed
  date?: TDateOrTime

  // Secondary provider
  provider?: string

  // Unique ID for attestation subject from provider
  id?: number | string

  // Date/time when attestation should be considered void (e.g., for credential expiry)
  expiry_date?: TDateOrTime

  // Optional summary object (often useful for multiple-account types - otherwise, the data field is preferable for most of these fields, for simplicity's sake
  summary?: {
    // Single date/time during which attestation was applicable or ascertained
    date?: TDateOrTime

    // Start date/time of period during which attestation was applicable
    start_date?: TDateOrTime

    // End date/time of period during which attestation was applicable
    end_date?: TDateOrTime

    // Different levels of generality - zero-indexed, with increasing numbers indicating less generality, to allow for unlimited levels of depth (in practice, 3-5 should be sufficient for most cases).  This allows for arbitrary levels or amounts of generality within sub-attestations, to promote an attestation subject's ability to partially disclose the amount of data provided in an attestation.
    generality?: number

    // ...extensible with other fields that summarize the content of the attestation - e.g., a list of addresses, accounts, totals of statistics, etc.
  }

  // Core attestation data, dependent on attestation type
  data?: Array<TBaseAttData> | TBaseAttData
  // ...extensible with other fields.  Other fields explicating general data about the attestation, such as location, shelf life, common units, etc., should be placed here.
}

export interface IBaseAttDataObj {}
export type TBaseAttData = IBaseAttDataObj | string | number

///////////////////////////////////////////////////
// Helper types
///////////////////////////////////////////////////
export type TPersonalName =  // Designed to be flexible - as a rule, a basic {given: "x", middle: "x", family: "x"} is probably the easiest for most Western use cases
  | string
  | {
      full?: string
      given?: string | Array<string>
      middle?: string | Array<string>
      family?: string | Array<string>
      title?: string | Array<string>
      prefix?: string | Array<string>
      suffix?: string | Array<string>
      nickname?: string | Array<string>
      generational?: string | Array<string>

      // For name changes
      start_date?: TDateOrTime
      end_date?: TDateOrTime
    }

export type TDateOrTime = TDate | TDatetime

export type TDate =
  | string // ISO-8601 date in YYYY-MM-DD format
  | {
      year: string // YYYY
      month: string // MM
      day: string // DD
    }

export type TDatetime =
  | string // ISO-8601 datetime in YYYY-MM-DDTHH:MM:SSZ format
  | {
      year: string // YYYY
      month: string // MM
      day: string // DD
      hour: string // HH
      minute: string // MM
      second: string // SS
    }

export type TPhoneNumber =
  | string // Valid internationally-formatted phone number
  | {
      full?: string
      country?: string
      subscriber?: string
      area?: string
      prefix?: string
      line?: string
      ext?: string
    }

export type TGender = string // 'male', 'female', ...

export type TAddress = {
  full: string
  name: string
  street_1: string
  street_2?: string
  street_3?: string
  city: string
  postal_code: string | number
  region_1: string
  region_2?: string
  country?: string
}

///////////////////////////////////////////////////
// Phone attestation dataStr type
///////////////////////////////////////////////////
export interface IBaseAttPhone extends IBaseAtt {
  data: TPhoneNumber | Array<TPhoneNumber>
}

///////////////////////////////////////////////////
// Email attestation dataStr type
///////////////////////////////////////////////////
export interface IBaseAttEmailData extends IBaseAttDataObj {
  email?: string
  start_date?: TDateOrTime
  end_date?: TDateOrTime
}
export interface IBaseAttEmail extends IBaseAtt {
  data: IBaseAttEmailData | Array<IBaseAttEmailData>
}

///////////////////////////////////////////////////
// Name attestation dataStr type
///////////////////////////////////////////////////
export interface IBaseAttName extends IBaseAtt {
  data: TPersonalName | Array<TPersonalName>
}

///////////////////////////////////////////////////
// SSN/government ID # attestation dataStr type
///////////////////////////////////////////////////
export interface IBaseAttSSNData extends IBaseAttDataObj {
  country_code: string
  id_type: string
  id: string | number
}
export interface IBaseAttSSN extends IBaseAtt {
  data: IBaseAttSSNData | Array<IBaseAttSSNData>
}

///////////////////////////////////////////////////
// Date of birth attestation dataStr type
///////////////////////////////////////////////////
export interface IBaseAttDOB extends IBaseAtt {
  data: TDateOrTime
}

///////////////////////////////////////////////////
// Account attestation dataStr type
///////////////////////////////////////////////////
export interface IBaseAttAccountData extends IBaseAttDataObj {
  id?: string | number
  email?: string

  name?: TPersonalName
  start_date?: TDateOrTime
  end_date?: TDateOrTime
}
export interface IBaseAttAccount extends IBaseAtt {
  data: IBaseAttAccountData | Array<IBaseAttAccountData>
}

///////////////////////////////////////////////////
// Sanction screen attestation dataStr type
///////////////////////////////////////////////////
export interface IBaseAttSanctionScreenData extends IBaseAttDataObj {
  name: TPersonalName
  birthday: TDateOrTime
}
export interface IBaseAttSanctionScreen extends IBaseAtt {
  data: IBaseAttSanctionScreenData | Array<IBaseAttSanctionScreenData>
}

///////////////////////////////////////////////////
// PEP screen attestation dataStr type
///////////////////////////////////////////////////
export interface IBaseAttPEPData extends IBaseAttDataObj {
  date: TDateOrTime
  name: TPersonalName
  country: string

  // Primarily modelled after KYC2020 responses, most fields left optional for flexibility
  search_summary: {
    hit_location?: string
    hit_number?: number
    list_name?: string
    list_url?: string
    record_id?: string
    search_reference_id?: string
    score?: string
    hits?: Array<{
      id?: string
      hit_name?: string
    }>
    flag_type?: string
    comment?: string
  }
}
export interface IBaseAttPEP extends IBaseAtt {
  data: IBaseAttPEPData | Array<IBaseAttPEPData>
}

///////////////////////////////////////////////////
// ID document attestation dataStr type
///////////////////////////////////////////////////
export interface IBaseAttIDDocData extends IBaseAttDataObj {
  date: TDateOrTime
  name: TPersonalName
  country: string

  authenticationResult?:
    | 'unknown'
    | 'passed'
    | 'failed'
    | 'skipped'
    | 'caution'
    | 'attention' // IAssureIDResult.AuthenticationResult
  biographic?: {
    age?: number
    dob?: TDateOrTime
    expiration_date?: TDateOrTime
    name?: TPersonalName
    gender?: string
    photo?: string
  } // IAssureIDResult.Biographic,
  facematch_result?: {
    is_match?: boolean
    score?: number
    transaction_id?: string
  } // IFaceMatchResult
}
export interface IBaseAttIDDoc extends IBaseAtt {
  data: IBaseAttIDDocData | Array<IBaseAttIDDocData>
}

///////////////////////////////////////////////////
// Utility bill attestation dataStr type
///////////////////////////////////////////////////
export type TBaseAttUtilitySummary = {
  generality: number
  currency?: string
  total_paid?: number
  account_numbers?: Array<string>
  statement_dates: Array<TDate> | Array<TDatetime>
  addresses?: Array<TAddress>
}
export interface TBaseAttUtilityData extends IBaseAttDataObj {
  account_number?: string | number
  currency?: string
  billing_address: TAddress
  service_address: TAddress
  total_bill?: number
  balance_adjustments?: number
  due_date?: TDateOrTime
  statement_date: TDateOrTime
}
export interface IBaseAttUtility extends IBaseAtt {
  summary?: TBaseAttUtilitySummary
  data: TBaseAttUtilityData | Array<TBaseAttUtilityData>
}

///////////////////////////////////////////////////
// Address attestation dataStr type
///////////////////////////////////////////////////
export interface IBaseAttAddressDataProviderAccount {}
export interface IBaseAttAddressData {
  provider?: {
    name: string
    id?: string
    country?: string
    service_types?: Array<string>
    website?: string
    accounts?: Array<IBaseAttAddressDataProviderAccount>
  }
  addresses?: Array<TAddress>
}
export interface IBaseAttAddress extends IBaseAtt {
  data: IBaseAttAddressData | Array<IBaseAttAddressData>
}

///////////////////////////////////////////////////
// Income attestation dataStr type (total, gross, or expenses)
///////////////////////////////////////////////////
export type TBaseAttIncomeSummary = {
  generality: number
  start_date: TDateOrTime
  end_date: TDateOrTime
  net?: TBaseAttIncomeIncome
  gross?: TBaseAttIncomeIncome
  expenses?: TBaseAttIncomeIncome
}
export type TBaseAttIncomeIncome = {
  total: number
  regular: number
  irregular: number
  num_transactions: number
}
export type TBaseAttIncomeStream = {
  id?: number
  start_date: TDateOrTime
  end_date: TDateOrTime

  cashflow_category?: string
  cashflow_subcategory?: string
  is_payroll_agency?: boolean
  memo?: string
  num_transactions?: number
  length: number // length in days
  payee?: string
  payer?: string
  rank?: string

  frequency?: string // suggested: 'daily', 'weekly' 'biweekly', 'monthly', 'semi_monthly', 'multiple_months', 'irregular'
  periodicity?: number // numerical alternative to above, in days

  stdev_value?: number
  total_value?: number
  mean_value?: number
  median_value?: number

  transactions?: Array<{
    currency?: string
    date: TDateOrTime
    value: number
  }>
}
export interface IBaseAttIncome extends IBaseAtt {
  summary?: TBaseAttIncomeSummary
  data: TBaseAttIncomeStream | Array<TBaseAttIncomeStream>
}

///////////////////////////////////////////////////
// Assets attestation dataStr type (total, gross, or expenses)
///////////////////////////////////////////////////
export interface IBaseAttAssetsSummary {
  date?: TDateOrTime
  value?: number
  num_accounts?: number
}
export interface IBaseAttAssetsAccount {
  category: string
  institution_name: string
  institution_id: number
  owner_type: string
  type: string
  type_confidence: string
  value: number
}
export interface IBaseAttAssets extends IBaseAtt {
  generality: number
  summary?: IBaseAttAssetsSummary
  data: IBaseAttAssetsAccount | Array<IBaseAttAssetsAccount>
}

///////////////////////////////////////////////////
// Gender attestation dataStr type
///////////////////////////////////////////////////
export interface IBaseAttGender extends IBaseAtt {
  data: TGender
}


```
