# Pre-Employment Check Case Management System

This document outlines the proposed architecture and data model for a campaign-based pre-employment check system, including dynamic data collection via schema-driven webforms.

## 1. General Architecture Diagram

The system is designed with a microservices-oriented approach. To support dynamic forms, we introduce a **Candidate Portal** and a **Form Schema Engine** within the Candidate Service.

When a candidate is added, the system determines the required checks (Template + Exceptions), merges the data requirements (JSON Schemas) of those checks, and serves a unified dynamic form to the candidate.

```mermaid
graph TD
    %% External Interfaces
    AdminClient[HR / Admin Portal]
    CandidatePortal[Candidate Portal / Webform]
    Gateway[API Gateway]

    %% Core System Components
    subgraph Case Management System
        CampaignSvc[Campaign Service]
        CandidateSvc[Candidate Service & Form Engine]
        WorkflowSvc[Workflow Engine / Orchestrator]
        TaskSvc[Task Management Service]
        IntegrationSvc[Integration Service]
    end

    %% External Systems / Workers
    subgraph External Agents
        RPA[RPA Workers]
        ExtAPI[External APIs e.g., Background Check]
    end

    %% Data Stores
    subgraph Data Tier
        DB[(Primary Relational Database)]
        DocStore[(Document Storage e.g., S3)]
        MessageQueue[[Message Queue / Event Bus]]
    end

    %% Connections
    AdminClient -->|Manage Campaigns/Tasks| Gateway
    CandidatePortal -->|Request Form / Submit Data| Gateway
    Gateway --> CampaignSvc
    Gateway --> CandidateSvc
    
    CampaignSvc -->|Read/Write| DB
    CandidateSvc -->|Read/Write| DB
    
    CandidateSvc -->|Trigger Workflow| WorkflowSvc
    WorkflowSvc -->|Create Tasks\nEvaluate Exceptions| TaskSvc
    WorkflowSvc -->|Publish Events| MessageQueue
    
    TaskSvc -->|Read/Write| DB
    TaskSvc -->|Store Docs| DocStore
    
    %% Task Execution Flow
    MessageQueue -.->|Consume Events| IntegrationSvc
    MessageQueue -.->|Consume Events| RPA
    
    IntegrationSvc <--> ExtAPI
    IntegrationSvc -->|Update Task Status| TaskSvc
    
    RPA -->|Update Task Status| TaskSvc
```

## 2. Dynamic Form Flow & Data Architecture

To make the webform dynamic:
1.  **Define Requirements:** Every `Check Type` (e.g., "Right to Work", "Criminal Record") defines its required fields using a JSON Schema (e.g., requires `passport_number`, `5_year_address_history`).
2.  **Schema Merging:** The Form Engine retrieves all checks needed for the candidate, extracts their schemas, and merges them (deduplicating overlapping fields like `date_of_birth`).
3.  **Rendering:** The Candidate Portal uses a library (like React JSON Schema Form) to render the UI automatically from this merged schema.
4.  **Storage:** The submitted data is saved to a single JSONB column on the Candidate profile, which the Workflow Engine later uses to feed individual tasks.

```mermaid
graph LR
    %% Data Flow for Dynamic Forms
    Check1[Right to Work Check\nNeeds: Passport, DOB]
    Check2[Credit Check\nNeeds: Address, DOB, SSN]
    
    Merge[Form Schema Engine\n(Merges & Deduplicates)]
    
    UI[Candidate Portal\nRenders Dynamic Webform]
    
    Storage[(Candidate Profile\nJSONB Data Store)]
    
    TaskDist[Workflow Engine\nDistributes Data to Tasks]

    Check1 --> Merge
    Check2 --> Merge
    Merge -->|Unified JSON Schema| UI
    UI -->|Candidate submits JSON| Storage
    Storage --> TaskDist
    TaskDist -->|Passport, DOB| RPA_Task[RPA: Right to Work]
    TaskDist -->|Address, SSN| API_Task[API: Credit Agency]
```

## 3. Data Models

We have updated the models to include `InputSchema` on the Check Type (to define the form) and `ProfileData` on the Candidate (to store the form answers).

### 3.1 Conceptual Data Model

```mermaid
erDiagram
    CAMPAIGN ||--o{ CANDIDATE : "contains"
    CAMPAIGN ||--o{ CAMPAIGN_EXCEPTION : "dictates"
    CAMPAIGN }|--|| WORKFLOW_TEMPLATE : "uses default"
    
    WORKFLOW_TEMPLATE ||--o{ WORKFLOW_STEP : "defines"
    WORKFLOW_STEP }|--|| CHECK_TYPE : "specifies"
    CAMPAIGN_EXCEPTION }|--|| CHECK_TYPE : "modifies requirement for"

    CANDIDATE ||--o{ CANDIDATE_WORKFLOW : "undergoes"
    CANDIDATE ||--|| SUBMITTED_DATA : "provides"
    CANDIDATE_WORKFLOW ||--o{ TASK : "consists of"
    
    CHECK_TYPE ||--|| DATA_SCHEMA : "defines required fields for form"
    TASK }|--|| CHECK_TYPE : "is an instance of"
    TASK ||--o{ DOCUMENT : "generates/requires"
```

### 3.2 Logical Data Model

*Note the addition of `InputSchema` to `CHECK_TYPE` and `ProfileData` to `CANDIDATE`.*

```mermaid
erDiagram
    CAMPAIGN {
        ID Identifier PK
        DefaultWorkflowTemplateID Identifier FK
        Name String
        ClientName String
    }

    CAMPAIGN_EXCEPTION {
        ID Identifier PK
        CampaignID Identifier FK
        CheckTypeID Identifier FK
        ExceptionAction String
    }

    CANDIDATE {
        ID Identifier PK
        CampaignID Identifier FK
        Email String
        ProfileData JSON "Stores all dynamic form answers"
        FormStatus String "e.g., PENDING, SUBMITTED"
    }

    WORKFLOW_TEMPLATE {
        ID Identifier PK
        Name String
    }

    WORKFLOW_STEP {
        ID Identifier PK
        WorkflowTemplateID Identifier FK
        CheckTypeID Identifier FK
    }

    CHECK_TYPE {
        ID Identifier PK
        Name String
        ExecutionMethod String
        InputSchema JSON "JSON Schema defining required UI fields"
        ConfigSchema JSON "Execution config for RPA/API"
    }

    CANDIDATE_WORKFLOW {
        ID Identifier PK
        CandidateID Identifier FK
    }

    TASK {
        ID Identifier PK
        CandidateWorkflowID Identifier FK
        CheckTypeID Identifier FK
        InputData JSON "Subset of Candidate ProfileData needed for this specific check"
        ResultData JSON
    }
```

### 3.3 Physical Data Model (PostgreSQL)

```mermaid
erDiagram
    campaign {
        uuid id PK
        uuid default_workflow_template_id FK
        varchar(255) name
        varchar(255) client_name
        timestamp created_at
    }

    campaign_exception {
        uuid id PK
        uuid campaign_id FK
        uuid check_type_id FK
        varchar(50) exception_action "Enum: 'EXCLUDE', 'ADD'"
    }

    candidate {
        uuid id PK
        uuid campaign_id FK
        varchar(100) first_name
        varchar(100) last_name
        varchar(255) email
        jsonb profile_data "Stores aggregated dynamic webform submission"
        varchar(50) form_status "Enum: 'PENDING', 'SUBMITTED'"
        timestamp created_at
    }

    workflow_template {
        uuid id PK
        varchar(255) name
    }

    workflow_step {
        uuid id PK
        uuid workflow_template_id FK
        uuid check_type_id FK
        integer sequence_order
    }

    check_type {
        uuid id PK
        varchar(100) name
        varchar(50) execution_method
        jsonb input_schema "JSON Schema (Draft 7/2020-12) for UI generation"
        jsonb config_schema "Configuration for execution agents"
    }

    candidate_workflow {
        uuid id PK
        uuid candidate_id FK
        varchar(50) status
    }

    task {
        uuid id PK
        uuid candidate_workflow_id FK
        uuid check_type_id FK
        varchar(50) status
        jsonb input_data "The specific data passed to the worker/API"
        jsonb result_data
        timestamp started_at
    }

    %% Relationships
    campaign ||--o{ candidate : "campaign_id"
    campaign ||--o{ campaign_exception : "campaign_id"
    campaign }|--|| workflow_template : "default_workflow_template_id"
    campaign_exception }|--|| check_type : "check_type_id"
    workflow_template ||--o{ workflow_step : "workflow_template_id"
    workflow_step }|--|| check_type : "check_type_id"
    candidate ||--o{ candidate_workflow : "candidate_id"
    candidate_workflow ||--o{ task : "candidate_workflow_id"
    task }|--|| check_type : "check_type_id"
```