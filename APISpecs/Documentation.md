# PEC Workflow and Campaign Initiation API

**Version:** 1.3.0

**Description:** API interaction to generate new PEC campaigns and add applicants.

**Contact:** API Support Team (centralrecruitproj@cabinetoffice.gov.uk)

## 🌍 Environments

* **Production:** `[https://xxx.production.gov.uk/v2](https://xxx.production.gov.uk/v2)` (Production environment).


* **Testing:** `[https://xxx.test.gov.uk/v2](https://xxx.test.gov.uk/v2)` (Testing and sandbox environment).



## 🔒 Authentication

This API utilizes a modern OAuth 2.0 Client Credentials flow designed for system-to-system authentication.

* **Token URL:** `[https://auth.production.gov.uk/oauth/token](https://auth.production.gov.uk/oauth/token)`


* **Required Scope:** `pec:write` (Grants permission to submit PEC requests).



## 🔑 Global Parameters

* **`Idempotency-Key` (Header):** A uniquely generated client string, such as a UUID, required on all `POST` requests to prevent duplicate operations during network retries.



---

## 🚀 Endpoints

### 1. Create a New PEC Campaign

`POST /campaign`

Initializes a new pre-employment check (PEC) campaign. Must be supplied with an `Idempotency-Key` for safe retries.

#### Request Payload

Accepts a JSON body matching the `NewCampaign` schema.

#### Responses

* **201 Created:** Successfully created the PEC campaign, returning a `SuccessResponseCampaign` object.


* **400 Bad Request:** Validation Error, returning RFC 7807 `ProblemDetails`.


* **401 Unauthorized:** Authentication failure, returning `ProblemDetails`.


* **409 Conflict:** Idempotency Error, returning `ProblemDetails`.


* **500 Server Error:** Internal Server Error, returning `ProblemDetails`.



---

### 2. Add Applicants to a PEC Campaign

`POST /campaign/{pecSystemCampaignId}/applicants`

Creates new PEC workflows for campaign candidates.

#### Path Parameters

* **`pecSystemCampaignId`:** The unique system identifier of the campaign (Required, String).



#### Request Payload

Accepts a JSON body matching the `NewApplicants` schema.

#### Responses

* **201 Created:** Successfully created the PEC workflows, returning a `SuccessResponseApplicants` object.


* **400 Bad Request:** Validation Error, returning `ProblemDetails`.


* **401 Unauthorized:** Authentication failure, returning `ProblemDetails`.


* **409 Conflict:** Applicant already exists, returning `ProblemDetails`.


* **500 Server Error:** Internal Server Error, returning `ProblemDetails`.



---

## 📦 Data Models

### NewCampaign

* **`consumerCampaignId`:** String (Max: 100). The human readable campaign ID from the consumer system.


* **`consumerSystemCampaignId`:** String (Max: 100). The guuid for the campaign in the consumer system.


* **`ssfgCluster`:** String (Enum: `Defense`, `Matrix`, `Non-Clustered`, `Overseas`, `Synergy`, `Unity`). The cluster from which the request is coming from.


* **`employingDepartment`:** String (Enum: `Ministry of Defence`, `Home Office`, `HM Revenue and Customs`, `Department for Work and Pensions`, `Cabinet Office`, `Foreign, Commonwealth & Development Office`). The employing department for the campaign.


* **`title`:** String (Max: 255). Campaign Title.


* **`csrfApproach`:** String (Enum: `Internal`, `Across Government`, `External`, `None`).


* **`crcRequirement`:** Object. Refers to the `CRCRequirement` schema.


* **`nsvRequirement`:** Object. Refers to the `NSVRequirement` schema.


* **`requestOwners`:** Array of `RequestOwner` objects (Minimum items: 1).


* **`specificCheckOverrides`:** Array of `specificCheckOverridesObject` schemas.



### RequestOwner

* **`name`:** String. The name of the request owner.


* **`email`:** String (Format: email, Length: 3-320). The email address of the request owner, which must match the one registered with the PEC system.


* **`phoneNumber`:** String (Length: 10-16). Contact phone number for the request owner.


* **`isLead`:** Boolean. This marker determines if this request owner will be the lead on the campaign.



### specificCheckOverridesObject

An object to hold specific check overrides by employee type.

* **`applicantType`:** String (Enum: `Internal`, `External`, `Other Goverment Department`, `Non-Departmental Public Body`).


* **`CheckTypes`:** Array of `CheckTypeEnum` strings.



### CheckTypeEnum

Enum options for check types:

* `Employment History`


* `Criminal Record Check`


* `Right to work`


* `Social Media`


* `Internal Fraud`


* `Overseas`



### CRCRequirement

* **`isCrcRequired`:** Boolean. Set to true if a CRC is required.


* **`jurisdiction`:** String (Enum: `Scotland Only`, `Other Jurisdiction Only`, `Both`). Required if `isCrcRequired` is true.


* **`scotlandCrcType`:** String (Enum: `Level 1`, `Level 2`, `Level 2 with barred list`, `PVG`). Required if jurisdiction is `Scotland Only` or `Both`.


* **`otherJurisdictionCrcLevel`:** String (Enum: `Basic`, `Standard`, `Enhanced`, `Enhanced with barred list`). Required if jurisdiction is `Other Jurisdiction Only` or `Both`.



### NSVRequirement

Defines whether National Security Vetting is required, and the corresponding level.

* **`isNsvRequired`:** Boolean. Set to true if a National Security Vetting check is required.


* **`level`:** String (Enum: `CTC`, `SC`, `DV`, `eDV`). The level of vetting record check required (Mandatory if `isNsvRequired` is true).



### NewApplicants

An object containing an array of new applicants to be attached to a pre-existing campaign.

* **`consumerSystemCampaignId`:** String.


* **`pecSystemCampaignId`:** String.


* **`applicants`:** Array of `Applicant` objects (Min items: 1, Max items: 100).



### Applicant

An object containing the required and optional parameters of an applicant.

* **`consumerApplicantId`:** String. The applicant ID from the consumer system.


* **`firstName`:** String. The legal first name of the applicant.


* **`lastName`:** String. The legal last name of the applicant.


* **`email`:** String (Format: email, Length: 3-320). The email address of the applicant, used to generate their account and invite them to complete checking.


* **`telephoneNumber`:** String (Pattern: `^\+?[1-9]\d{1,14}$`, Length: 10-16). A contact telephone number for the applicant.


* **`dateOfBirth`:** String (Pattern: `^\d{2}-\d{2}-\d{4}$`, Length: 10). The date of birth in the format dd-mm-yyyy.


* **`nino`:** String (Pattern: `^[A-CEGHJ-PR-TW-Z]{1}[A-CEGHJ-NPR-TW-Z]{1} \d{2} \d{2} \d{2} [A-D]{1}$`, Length: 13). National insurance number of the applicant.


* **`crcLocation`:** String (Enum: `England`, `Wales`, `Scotland`, `Northern Ireland`). The crc location that the applicant will be working from.


* **`applicantType`:** String (Enum: `Internal`, `External`, `Other Government Department`, `Non-Departmental Public Body`). The type of applicant at the time of their application.



### SuccessResponseCampaign

* **`consumerSystemCampaignId`:** String. The unique system identifier of the campaign in the requestor's system.


* **`pecSystemCampaignId`:** String. The unique system identifier of the campaign in the PEC system.


* **`message`:** String. Status note regarding the request creation.


* **`ssfgCluster`:** String. The name of the organizational cluster the request is coming from.


* **`campaignPECStatus`:** String. The status of the campaign in the PEC system.



### SuccessResponseApplicants

Shares fields with `SuccessResponseCampaign` but adds:

* **`workflowConfirmations`:** Array of `WorkflowConfirmation` objects.



### WorkflowConfirmation

* **`consumerApplicantId`:** String.


* **`pecApplicantId`:** String.


* **`workflowStatus`:** String.



### ProblemDetails (RFC 7807)

A machine-readable format for specifying errors in HTTP API responses.

* **`type`:** String (URI format). Reference identifying the problem type, defaulting to `about:blank`.


* **`title`:** String. A short, human-readable summary of the problem type.


* **`status`:** Integer (Int32). The HTTP status code.


* **`detail`:** String. A human-readable explanation specific to this occurrence of the problem.


* **`instance`:** String (URI format). A URI reference identifying the specific occurrence of the problem.


* **`errors`:** Array of Strings. Specific validation errors (e.g., failed field rules).
