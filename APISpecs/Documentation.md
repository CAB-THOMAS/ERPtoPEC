# PEC Workflow and Campaign Initiation API

**Version:** 1.2.0
**Description:** API interaction to generate new PEC campaigns and add applicants.
**Contact:** API Support Team (api-support@production.gov.uk)

---

## 🌍 Environments

| Environment | Base URL | Description |
| :--- | :--- | :--- |
| **Production** | `https://xxx.production.gov.uk/v2` | Live production environment |
| **Testing** | `https://xxx.test.gov.uk/v2` | Testing and sandbox environment |

## 🔒 Authentication

This API uses **OAuth 2.0 (Client Credentials Flow)** for system-to-system authentication. 

* **Token URL:** `https://auth.production.gov.uk/oauth/token`
* **Required Scope:** `pec:write` (Grants permission to submit PEC requests)

All requests must include an `Authorization: Bearer <token>` header.

## 🔑 Global Headers

| Header | Type | Required | Description |
| :--- | :--- | :--- | :--- |
| `Idempotency-Key` | String (UUID) | **Yes** | A unique client-generated string (e.g., a UUID) to prevent duplicate operations on network retries. |

---

## 🚀 Endpoints

### 1. Create a New PEC Campaign
`POST /campaign`

Initialize a new pre-employment check (PEC) campaign. Must be supplied with an Idempotency-Key for safe retries.

#### Request Body ( `application/json` )
Creates a new campaign object. 

**Example Payload:**
```json
{
  "consumerCampaignId": "CMP12345",
  "consumerSystemCampaignId": "req-1234-abcd",
  "ssfgCluster": "Unity",
  "employingDepartment": "Ministry of Defence",
  "title": "Senior Developer Recruitment",
  "csrfApproach": "External",
  "crcRequirement": {
    "isCrcRequired": true,
    "jurisdiction": "Both",
    "scotlandCrcType": "PVG",
    "otherJurisdictionCrcLevel": "Enhanced"
  },
  "nsvRequirement": {
    "isNsvRequired": true,
    "level": "SC"
  },
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
| :--- | :--- | :--- |
| **201** | **Created.** Successfully created the PEC campaign. | `SuccessResponseCampaign` |
| **400** | **Bad Request.** Validation Error. | `ProblemDetails` |
| **401** | **Unauthorized.** Authentication failure. | `ProblemDetails` |
| **409** | **Conflict.** Idempotency Error. | `ProblemDetails` |
| **500** | **Server Error.** Internal Server Error. | `ProblemDetails` |

---

### 2. Add Applicants to a PEC Campaign
`POST /campaign/{pecSystemCampaignId}/applicants`

Creates new PEC workflows for campaign candidates.

#### Path Parameters
* `pecSystemCampaignId` *(string, required)*: The unique system identifier of the campaign.

#### Request Body ( `application/json` )
Accepts an array of candidates to process.

**Example Payload:**
```json
{
  "consumerSystemCampaignId": "req-1234-abcd",
  "pecSystemCampaignId": "pec-sys-9999",
  "applicants": [
    {
      "consumerApplicantId": "APP-001",
      "firstName": "Robert",
      "lastName": "Builder",
      "email": "bob@builder.com",
      "telephoneNumber": "+447700900123",
      "dateOfBirth": "21-07-2017",
      "nino": "AB 12 34 56 C",
      "crcLocation": "England",
      "applicantType": "External"
    }
  ]
}
```

#### Responses

| Status | Meaning | Response Schema |
| :--- | :--- | :--- |
| **201** | **Created.** Successfully created the PEC workflows. | `SuccessResponseApplicants` |
| **400** | **Bad Request.** Validation Error. | `ProblemDetails` |
| **401** | **Unauthorized.** Authentication failure. | `ProblemDetails` |
| **409** | **Conflict.** Applicant already exists. | `ProblemDetails` |
| **500** | **Server Error.** Internal Server Error. | `ProblemDetails` |

---

## 📦 Data Models

### NewCampaign
| Property | Type | Constraints | Description |
| :--- | :--- | :--- | :--- |
| `consumerCampaignId` | String | Required, Max Length: 100 | Unique ID in the consumer's system. |
| `consumerSystemCampaignId`| String | Required, Max Length: 100 | Unique system identifier in the consumer's system. |
| `ssfgCluster` | String | Required, Enum | Allowed: `Defense`, `Matrix`, `Non-Clustered`, `Overseas`, `Synergy`, `Unity`. |
| `employingDepartment` | String | Required, Enum | E.g., `Ministry of Defence`, `Home Office`, `HM Revenue and Customs`. |
| `title` | String | Required, Max Length: 255 | Title of the campaign. |
| `csrfApproach` | String | Required, Enum | Allowed: `Internal`, `Across Government`, `External`, `None`. |
| `crcRequirement` | Object | Required | Defines Criminal Record Check logic (see `CRCRequirement`). |
| `nsvRequirement` | Object | Required | Defines National Security Vetting logic (see `NSVRequirement`). |
| `requestOwners` | Array | Required, Min Items: 1 | Array of `RequestOwner` objects. |
| `specificCheckOverrides`| Object | Optional | Overrides grouped by `Internal`, `External`, `Other Government Department`, `Non-Departmental Public Body`. |

### CRCRequirement (Conditional Logic)
| Property | Type | Description | Required When |
| :--- | :--- | :--- | :--- |
| `isCrcRequired` | Boolean | True if a CRC is required. | Always Required. |
| `jurisdiction` | String | Options: `Scotland Only`, `Other Jurisdiction Only`, `Both`. | `isCrcRequired` is true. |
| `scotlandCrcType` | String | Options: `Level 1`, `Level 2`, `Level 2 with barred list`, `PVG`. | `jurisdiction` is `Scotland Only` or `Both`. |
| `otherJurisdictionCrcLevel`| String | Options: `Basic`, `Standard`, `Enhanced`, `Enhanced with barred list`. | `jurisdiction` is `Other Jurisdiction Only` or `Both`. |

### NSVRequirement (Conditional Logic)
| Property | Type | Description | Required When |
| :--- | :--- | :--- | :--- |
| `isNsvRequired` | Boolean | True if National Security Vetting is required. | Always Required. |
| `level` | String | Options: `AC`, `CTC`, `SC`, `eSC`, `DV`, `eDV`. | `isNsvRequired` is true. |

### Applicant
| Property | Type | Constraints | Description |
| :--- | :--- | :--- | :--- |
| `consumerApplicantId` | String | Required | Unique ID assigned by the consuming source system. |
| `firstName` | String | Required | Applicant's given name. |
| `lastName` | String | Required | Applicant's family name. |
| `email` | String | Required, Format: email | Applicant's email address. |
| `telephoneNumber` | String | Optional, Regex | Contact number (`^\+?[1-9]\d{1,14}$`). |
| `dateOfBirth` | String | Optional, Regex | Date of birth (`^\d{2}-\d{2}-\d{4}$`). |
| `nino` | String | Optional, Regex | National Insurance Number formatted with spaces (e.g., `AB 12 34 56 C`). |
| `crcLocation` | String | Optional, Enum | Allowed: `England`, `Wales`, `Scotland`, `Northern Ireland`. |
| `applicantType` | String | Optional, Enum | Allowed: `Internal`, `External`, `Other Government Department`, `Non-Departmental Public Body`. |

### SuccessResponseCampaign
| Property | Type | Description |
| :--- | :--- | :--- |
| `consumerSystemCampaignId`| String | Unique system identifier in the requestor's system. |
| `pecSystemCampaignId` | String | Unique system identifier of the campaign generated in the PEC system. |
| `message` | String | Status note regarding the request creation. |
| `ssfgCluster` | String | The name of the organizational cluster. |
| `campaignPECStatus` | String | Status of the campaign in the PEC system. |

### SuccessResponseApplicants
Extends `SuccessResponseCampaign` with an array of created workflows.
| Property | Type | Description |
| :--- | :--- | :--- |
| `workflowConfirmations` | Array | A list of `WorkflowConfirmation` objects detailing candidate statuses. |

### ProblemDetails (RFC 7807)
| Property | Type | Description |
| :--- | :--- | :--- |
| `type` | String (URI) | A URI reference that identifies the problem type. |
| `title` | String | A short, human-readable summary of the problem type. |
| `status` | Integer | The HTTP status code. |
| `detail` | String | A human-readable explanation specific to this occurrence. |
| `instance` | String (URI) | A URI reference identifying the specific occurrence of the problem. |
| `errors` | Array[String]| Specific validation errors (e.g., failed field rules). |
