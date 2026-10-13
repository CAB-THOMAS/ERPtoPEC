# PEC Workflow and Campaign Initiation API

**Version:** 0.1.0

**Description:** API interaction to generate a new Pre-Employment Check (PEC) campaign, and subsequently add applicants to generate individual checking workflows.

---

## 🌍 Environments

| Environment | Base URL | Description |
| --- | --- | --- |
| **Production** | `[https://xxx.production.gov.uk/v2](https://xxx.production.gov.uk/v2)` | Live production environment |
| **Testing** | `[https://xxx.test.gov.uk/v2](https://xxx.test.gov.uk/v2)` | Sandbox environment for testing and integration |

## 🔒 Authentication

This API uses **OAuth 2.0 (Client Credentials Flow)** for system-to-system authentication.

* **Token URL:** `[https://auth.production.gov.uk/oauth/token](https://auth.production.gov.uk/oauth/token)`
* **Required Scope:** `pec:write` (Grants permission to submit PEC requests)

All requests must include an `Authorization: Bearer <token>` header.

---

## 🚀 Endpoints

### 1. Create New PEC Campaign

`POST /campaign`

Parses payload data to create a new Pre-Employment Checks (PEC) campaign. This campaign acts as the container for future applicants.

#### Request Body ( `application/json` )

Creates a new campaign object.

**Example Payload:**

```json
{
  "consumerCampaignId": "CMP12345",
  "businessCluster": "Unity",
  "employingDepartment": "Home Office",
  "consumerSystemCampaignId": "asd4-45sd-w4344-wer3",
  "title": "Senior Developer",
  "crcLocations": ["England", "Wales"],
  "crcLevel": "Basic",
  "nsvLevel": "SC",
  "requestOwners": [
    {
      "name": "Jane Doe",
      "email": "jane.doe@gov.uk",
      "phoneNumber": "+44 7700 900077"
    }
  ]
}

```

#### Responses

| Status | Meaning | Response Schema |
| --- | --- | --- |
| **201** | **Created.** Successfully created the PEC campaign. | `SuccessResponseCampaign` |
| **400** | **Bad Request.** Payload failed schema validation. | `ErrorResponse` |
| **401** | **Unauthorized.** Authentication failure or missing scope. | `ErrorResponse` |
| **409** | **Conflict.** Idempotency error (campaign already exists). | `ErrorResponse` |
| **500** | **Server Error.** Internal service failure. | `ErrorResponse` |

---

### 2. Add Applicants to PEC Campaign

`POST /campaign/{pECSystemCampaignId}/applicants`

Parses data to create new PEC workflows for individual candidates under a specific, existing campaign.

#### Path Parameters

* `pECSystemCampaignId` *(string, required)*: The unique system identifier of the campaign within the PEC system.

#### Request Body ( `application/json` )

Accepts an array of candidates (Maximum of 100 per request to prevent payload bloat).

**Example Payload:**

```json
{
  "consumerSystemCampaignId": "asd4-45sd-w4344-wer3",
  "pECSystemCampaignId": "pec-sys-9999",
  "applicants": [
    {
      "firstName": "Robert",
      "lastName": "Builder",
      "email": "bob@builder.com",
      "telephoneNumber": "+447700900123",
      "dateOfBirth": "2017-07-21",
      "consumerApplicantId": "APP-88291",
      "nino": "QQ123456C",
      "crcLocation": "England"
    }
  ]
}

```

#### Responses

| Status | Meaning | Response Schema |
| --- | --- | --- |
| **201** | **Created.** Successfully created applicant PEC workflows. | `SuccessResponseApplicants` |
| **400** | **Bad Request.** Payload failed schema validation. | `ErrorResponse` |
| **401** | **Unauthorized.** Authentication failure or missing scope. | `ErrorResponse` |
| **409** | **Conflict.** Idempotency error. | `ErrorResponse` |
| **500** | **Server Error.** Internal service failure. | `ErrorResponse` |

---

## 📦 Data Models

### NewCampaign

| Property | Type | Constraints | Description |
| --- | --- | --- | --- |
| `consumerCampaignId` | String | Required, Max: 100 | Human-readable unique ID of the recruitment campaign. |
| `businessCluster` | String | Required, Enum | The organizational cluster. Valid: `Defense`, `Matrix`, `Non-Clustered`, `Overseas`, `Synergy`, `Unity`. |
| `employingDepartment` | String | Required, Max: 150 | Department name (e.g., HMRC, Home Office). |
| `consumerSystemCampaignId` | String | Required, Max: 100 | Unique system identifier in the requestor's system. |
| `title` | String | Required, Max: 255 | Campaign title (e.g., "Senior Developer"). |
| `crcLocations` | Array[String] | Required | Allowed: `England`, `Northern Ireland`, `Scotland`, `Wales`, `Overseas`. |
| `crcLevel` | String | Required, Enum | Criminal Record Check level: `None`, `Basic`, `Standard`, `Enhanced`. |
| `nsvLevel` | String | Required, Enum | National Security Vetting level: `None`, `AC`, `CTC`, `SC`, `eSC`, `DV`, `eDV`. |
| `requestOwners` | Array | Required, Min: 1 | Array of `RequestOwner` objects managing the cases. |
| `specificCheckOverrides` | Array | Optional | Array of `CheckOverride` objects. |

### Applicant

| Property | Type | Constraints | Description |
| --- | --- | --- | --- |
| `firstName` | String | Required, Max: 100 | Applicant's given name. |
| `lastName` | String | Required, Max: 100 | Applicant's family name. |
| `email` | String | Required, Format: email | Applicant's email address. |
| `consumerApplicantId` | String | Required, Max: 100 | Unique ID assigned by the consuming source system. |
| `telephoneNumber` | String | Optional, Regex | Contact number (E.164 format: `^\+?[1-9]\d{1,14}$`). |
| `dateOfBirth` | String | Optional, Format: date | Date of birth (YYYY-MM-DD). |
| `nino` | String | Optional, Regex | UK National Insurance Number (`^[A-CEGHJ-PR-TW-Z]{1}[A-CEGHJ-NPR-TW-Z]{1}[0-9]{6}[A-D]{1}$`). |
| `crcLocation` | String | Optional, Enum | The specific location for this applicant's CRC (`England`, `Wales`, `Scotland`, `Northern Ireland`, `Overseas`). |

### RequestOwner

| Property | Type | Constraints | Description |
| --- | --- | --- | --- |
| `name` | String | Required | Full name of the case manager. |
| `email` | String | Required, Format: email | Email address of the case manager. |
| `phoneNumber` | String | Optional | Contact phone number. |

### SuccessResponseCampaign

| Property | Type | Description |
| --- | --- | --- |
| `message` | String | Status note regarding the request creation. |
| `consumerSystemCampaignId` | String | Unique system identifier in the requestor's system. |
| `pECSystemCampaignId` | String | Unique system identifier of the campaign generated in the PEC system. |
| `businessCluster` | String | Organizational cluster. |
| `campaignPECStatus` | String | Status of the campaign in the PEC system (e.g., `active`). |

### SuccessResponseApplicants

Extends `SuccessResponseCampaign` with an array of created workflows.

| Property | Type | Description |
| --- | --- | --- |
| `workflowConfirmations` | Array | A list of `WorkflowConfirmation` objects detailing candidate statuses. |

### WorkflowConfirmation

| Property | Type | Description |
| --- | --- | --- |
| `consumerApplicantId` | String | The unique ID assigned by the consuming source system. |
| `pECApplicantId` | String | The unique ID generated by the PEC provider for this candidate. |
| `workflowStatus` | String | Current status of the created workflow (e.g., `Pending`). |

### ErrorResponse

| Property | Type | Description |
| --- | --- | --- |
| `errorCode` | Integer | Numeric HTTP error code (e.g., `400`). |
| `title` | String | Short title of the error (e.g., `Validation Error`). |
| `message` | String | Detailed description of the error. |
| `errorDetailArray` | Array[String] | List of specific validation failures or detailed error messages. |
