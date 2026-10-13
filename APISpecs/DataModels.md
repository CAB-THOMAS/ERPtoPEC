##Conceptual Data model
```mermaid
erDiagram
    CAMPAIGN {
    }
    APPLICANT {
    }
    REQUEST_OWNER {
    }
    CHECK_OVERRIDE {
    }

    CAMPAIGN ||--|{ APPLICANT : "generates workflows for"
    CAMPAIGN ||--|{ REQUEST_OWNER : "is managed by"
    CAMPAIGN ||--o{ CHECK_OVERRIDE : "may have specific"
```
