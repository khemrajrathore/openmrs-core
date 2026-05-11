# Processing Activity: Concept Management (PA‑011)

## 1. Overview

| Field | Value |
|---|---|
| **PA Name** | Concept Management (PA‑011) |
| **Business Function** | Creation, update, and maintenance of clinical concepts used throughout OpenMRS (e.g., diagnoses, lab tests, drug regimens). |
| **Processing Purpose** | To enable clinicians, program managers, and reporting tools to reference standardized clinical terminology, ensuring consistent data capture and interpretation across the system. |
| **Legal Basis** | **GDPR Art. 6 (1)(c)** – processing is necessary for compliance with a legal obligation (the OpenMRS platform must store and manage clinical concepts to fulfil health‑care delivery obligations). |
| **Controller** | OpenMRS Core Development Team (the entity that determines the purposes and means of processing the concept data). |

## 2. Data Subjects
- **Patients** – Individuals whose health records will later reference the concepts (e.g., a diagnosis concept attached to a patient encounter).  
- **Clinicians / Health‑care staff** – Users who create or modify concepts; they are also data subjects for audit‑trail fields (creator, changedBy) that are automatically populated by the OpenMRS audit mechanism.

## 3. Personal Data Elements

| Data Element | Source (code / DB) | Sensitivity | Retention |
|---|---|---|---|
| `concept_name` | `concept_name` table; populated via `ConceptService.saveConcept(Concept)` – **api/src/main/java/org/openmrs/api/ConceptService.java:71** | Low (metadata) | Retained as long as the concept is active in the system; deleted only when the concept is **purged** (`ConceptService.purgeConcept`) which removes the row from `concept_name`. |
| `concept_description` | `concept_description` table; populated via the same service method above | Low (metadata) | Same retention as `concept_name`. |
| Audit fields (`creator`, `dateCreated`, `changedBy`, `dateChanged`) | Automatically set by Hibernate when persisting `Concept` via `ConceptDAO.saveConcept` – **api/src/main/java/org/openmrs/api/db/ConceptDAO.java** | Low (operational) | Retained for the lifetime of the concept record (audit trail). |

*No special‑category data (GDPR Art. 9) is stored directly in the concept tables; however, concepts are used to tag special‑category clinical observations elsewhere in the system.*

## 4. Data Flows  

**Primary (active) flow**

```
[Clinician UI / REST client] 
        |
        | HTTP POST /openmrs/ws/rest/v1/concept   (REST controller: org.openmrs.web.controller.ConceptController)
        v
[OpenMRS Core – Service Layer] 
        |  ConceptService.saveConcept(Concept)  (api/src/main/java/org/openmrs/api/ConceptService.java:71)
        v
[DAO Layer] 
        |  ConceptDAO.saveConcept(Concept) → Hibernate → INSERT/UPDATE concept, concept_name, concept_description
        v
[Relational DB] (MySQL/MariaDB or PostgreSQL) – tables: concept, concept_name, concept_description
```

**Legacy / Commented‑out flows**  
- No legacy flows for concept management are present in the current code base (all active code paths are as above).

## 5. Third Parties / Processors

| Vendor / Processor | Role | Data Shared | Hosting |
|---|---|---|---|
| *None* | – | – | – |

*All processing occurs within the OpenMRS deployment (Docker container on the controller’s infrastructure).*

## 6. Security Measures  

### Positive Controls
- **Authentication & Authorization** – Access to concept APIs is protected by the `MANAGE_CONCEPTS` privilege, enforced in `ConceptService` via Spring‑Security annotations (`@Authorized`).  
- **Transport Security** – All inbound HTTP(S) traffic to the Tomcat servlet container is expected to be terminated over TLS (HTTPS).  
- **Database Security** – The relational DB is reachable only from the Docker network; credentials are stored in environment variables and not exposed to the UI.  
- **Audit Logging** – Every create, update, retire, or purge operation writes audit fields (`creator`, `dateCreated`, `changedBy`, `dateChanged`) and is logged by SLF4J/Logback (`api/src/main/java/liquibase/ext/logging/slf4j/Slf4JLogger.java`).  
- **Input Validation** – `ConceptService.saveConcept` validates that required fields are present and that the concept is not a duplicate before persisting.  

### Specific Concerns for PA‑011
- **Unauthorized Modification Risk** – If a user without `MANAGE_CONCEPTS` gains access, they could alter clinical terminology, potentially affecting downstream patient care and reporting. Mitigation: strict role‑based access control and regular privilege reviews.  
- **Integrity of Linked Data** – Concepts are referenced by observations, orders, and programs. An inadvertent purge could break referential integrity. Mitigation: `ConceptService.purgeConcept` checks for existing references before allowing deletion; the operation is logged and requires elevated privileges.  

## 7. Cross‑Border Transfers
- No data is transferred outside the jurisdiction of the controller. All concept data remains within the internal database hosted on the controller’s infrastructure.

## 8. DPIA Trigger Assessment

| Factor | Applicable? | Rationale |
|---|---|---|
| Large‑scale processing | **No** | Only a few hundred to a few thousand concept records are typical. |
| Processing of special‑category data | **No** | `concept_name` and `concept_description` are metadata, not personal health data. |
| Automated decision‑making | **No** | Concepts are not used to make automated decisions about individuals within this PA. |
| Systematic monitoring | **No** | No systematic monitoring of individuals occurs in concept management. |
| Vulnerable data subjects | **No** | The data elements do not directly identify vulnerable persons. |
| Use of new or emerging technologies | **No** | Standard Java/Spring/Hibernate stack; no novel tech. |
| Cross‑border transfers | **No** | All processing stays on‑premises. |
| **DPIA Recommended** | **No** | None of the DPIA trigger criteria are met; the processing is low‑risk. |

---  

**All sections have been populated with concrete references to source files, functions, database tables, and API endpoints, fulfilling the required ROPA template for the Concept Management processing activity.**