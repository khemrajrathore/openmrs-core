# Processing Activity: Allergy Management (PA‑005)

## 1. Overview
| Field | Value |
|---|---|
| **PA Name** | Allergy Management (PA‑005) |
| **Business Function** | Capture, store, and expose patient allergy information so that safety alerts can be generated during clinical encounters. |
| **Processing Purpose** | To protect patients from adverse drug or environmental reactions by (i) recording known allergens, reactions and severity; (ii) making the data available to the **Encounter Service** for real‑time validation and alerting; (iii) supporting reporting and audit. |
| **Legal Basis** | **GDPR Art. 9(2)(h)** – processing is necessary for the provision of health or social care services (preventive medicine & safety alerts). |
| **Controller** | OpenMRS Core Project (the OpenMRS community acts as the data controller for deployments that use the core code‑base). |
| **Relevant Modules / Classes** | `org.openmrs.Allergy` (entity), `org.openmrs.api.AllergyService` (service), `org.openmrs.api.dao.AllergyDAO` (DAO), `org.openmrs.api.EncounterService` (references allergies during encounter validation). |
| **Datastores** | Primary relational store – **MySQL / MariaDB** (`allergy` table). Optional search index – **Elasticsearch** (populated by Hibernate‑Search for fast lookup). |
| **Runtime Environment** | Java 8, Spring Core, Hibernate 5, Tomcat 9, Docker containers (as described in the OpenMRS engineering overview). |

## 2. Data Subjects
| Role | Description |
|---|---|
| **Patients** | Vulnerable data subjects whose health information (allergy data) is recorded. |
| **Healthcare Providers** | Users who view allergy data to make clinical decisions (doctors, nurses, pharmacists). |
| **System Administrators** | May have indirect access for maintenance but are not primary data subjects. |

## 3. Personal Data Elements
| Data Element | Source (Domain Class / Table) | Sensitivity | Retention |
|---|---|---|---|
| Allergy ID | `Allergy.id` – column `allergy_id` in `allergy` table | High (technical identifier) | Retained as long as the patient record is active; deleted only when the patient record is purged in accordance with the institution’s data‑retention policy. |
| Patient ID | `Allergy.patient` → `Patient.patientId` – column `patient_id` in `allergy` table | High (links health data to a specific person) | Same as patient record lifecycle (typically the lifetime of the medical record). |
| Allergen Concept | `Allergy.allergen` → `Concept` – column `allergen_concept_id` in `allergy` table | High (clinical concept) | Same as patient record lifecycle. |
| Reaction | `Allergy.reaction` – column `reaction` in `allergy` table (free‑text or coded) | High (clinical detail) | Same as patient record lifecycle. |
| Severity | `Allergy.severity` – column `severity` in `allergy` table (coded) | High (clinical detail) | Same as patient record lifecycle. |

*All elements are **special‑category data** under GDPR Art. 9 because they constitute health information.*

## 4. Data Flows
1. **Ingress** – Allergy data is entered via:  
   * Web UI forms (Spring MVC controllers in `web` module).  
   * REST API (`AllergyResource` under the OpenMRS‑REST module).  
   * HL7 inbound messages parsed by `org.openmrs.hl7.HL7Service` (if a lab or external system sends an allergy update).  

2. **Processing** –  
   * The UI/REST layer calls `AllergyService.saveAllergy(Allergy)` → `AllergyDAO.save(Allergy)`.  
   * Hibernate persists the entity to the `allergy` table (MySQL/MariaDB).  
   * Hibernate Envers creates an audit entry in `allergy_audit` for every create/update/delete.  
   * Spring AOP `@Authorized` annotations enforce that only users with the `Manage Allergies` privilege can create or modify records.  

3. **Consumption** –  
   * During an encounter, `EncounterService.validateAllergies(Encounter)` loads all active `Allergy` rows for the patient and checks them against prescribed drugs/observations.  
   * If a match is found, the `MessageService` sends an on‑screen alert and optionally an email notification (`MailMessageSender`).  

4. **Egress** –  
   * No external third‑party transmission is defined for this PA (third‑party list is empty).  
   * Data may be exported via the OpenMRS‑REST API to internal reporting tools, but such exports remain within the same organisational boundary.  

5. **Search Index (optional)** –  
   * Hibernate‑Search indexes the `Allergy` entity into Elasticsearch for fast lookup in UI auto‑complete fields. The index contains the same fields as the relational table but does not store raw patient identifiers (they are hashed or omitted in the index).  

**Mermaid diagram (simplified)**  

```mermaid
flowchart TD
    UI[Web UI / REST] -->|Create/Update| AllergySvc[AllergyService]
    HL7[HL7 inbound] -->|Parse| AllergySvc
    AllergySvc -->|Persist| DB[(MySQL/MariaDB – allergy table)]
    DB -->|Audit| Envers[Hibernate Envers]
    AllergySvc -->|Index| ES[(Elasticsearch – allergy index)]
    EncounterSvc[EncounterService] -->|Load| DB
    EncounterSvc -->|Validate| Alert[Safety Alert (MessageService)]
```

## 5. Third Parties / Processors
| Vendor | Role | Data Shared | Hosting |
|---|---|---|---|
| *None* | No external processors are used for this PA. All processing occurs on the organisation’s own infrastructure (application server, database, optional Elasticsearch node). |

## 6. Security Measures
| Layer | Measure | Description |
|---|---|---|
| **Application** | `@Authorized` AOP | Enforces role‑based access; only users with `Manage Allergies` or `View Allergies` can read/write. |
| **Authentication** | Spring Security + BCrypt password hashing | All user passwords are stored as salted BCrypt hashes. |
| **Transport** | TLS 1.2+ enforced on all HTTP/HTTPS endpoints (Tomcat configuration). |
| **Persistence** | Hibernate Envers audit logging | Every change to an `Allergy` record is recorded with user, timestamp, and before/after values. |
| **Database** | MySQL/MariaDB with at‑rest encryption (optional via Docker‑compose secrets) and row‑level access controls. |
| **Search Index** | Elasticsearch secured with X‑Pack security (TLS, role‑based access) – only the OpenMRS service account can query the index. |
| **Logging** | SLF4J + Logback with log sanitisation – no PII is written to application logs. |
| **Backup & Recovery** | Encrypted backups stored in the same jurisdiction; retention aligned with medical‑record policies. |
| **Session Management** | Short‑lived HTTP sessions, CSRF tokens, SameSite cookies. |
| **Code Quality** | Static analysis (SpotBugs, OWASP Dependency‑Check) and regular dependency updates. |

## 7. Cross‑Border Transfers
- The OpenMRS core deployment stores and processes allergy data **solely within the host organisation’s data centre** (or cloud region) that is located in the same legal jurisdiction as the data subjects.  
- No automatic replication or export to foreign jurisdictions occurs.  
- If a deployment chooses to replicate Elasticsearch to another region, that would constitute a cross‑border transfer and would require a separate contractual and adequacy assessment.

## 8. DPIA Trigger Assessment
| Factor | Applicable? | Comments |
|---|---|---|
| Large‑scale processing | **Yes** | Potentially thousands of patients per hospital; each patient may have multiple allergy records. |
| Sensitive data (special‑category) | **Yes** | Health data (allergies) falls under GDPR Art. 9. |
| Automated decision‑making | **No** | Alerts are rule‑based, not automated profiling. |
| Systematic monitoring | **Yes** | Allergy data is systematically consulted on every encounter. |
| Vulnerable data subjects | **Yes** | Patients are considered vulnerable under GDPR Recital 33. |
| New technologies | **No** | Uses established Java/Spring/Hibernate stack. |
| Cross‑border transfers | **No** (by default) | Data stays within the same jurisdiction. |
| **DPIA Recommended** | **Yes** | Because the activity processes special‑category health data on a large scale and involves vulnerable subjects, a Data Protection Impact Assessment is mandatory. |

---  

**Summary:**  
The Allergy Management PA records special‑category health data (allergies) for each patient, stores it in the `allergy` table (MySQL/MariaDB) and optionally indexes it in Elasticsearch, and uses it during encounter validation to generate safety alerts. All processing is performed internally, protected by a layered security model (role‑based access, TLS, audit logging, encrypted storage). Given the nature of the data, the scale, and the involvement of vulnerable subjects, a DPIA is required and should be documented as part of the organisation’s GDPR compliance programme.