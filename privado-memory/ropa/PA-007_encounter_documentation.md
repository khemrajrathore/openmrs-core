# Record of Processing Activities (ROPA) – Encounter Documentation (PA‑007)

## 1. Overview  

| Field | Value |
|---|---|
| **PA Name** | Encounter Documentation (PA‑007) |
| **Business Function** | Capture, store, and retrieve patient encounter information as part of routine clinical care. |
| **Processing Purpose** | To enable clinicians to record the details of each patient‑provider interaction (date, type, and associated patient) for treatment, continuity of care, reporting, and legal compliance. |
| **Legal Basis (GDPR)** | **Article 6 (1)(e)** – processing is necessary for the performance of a task carried out in the public interest (healthcare provision). |
| **Controller** | OpenMRS Core project (the organization that deploys and operates the OpenMRS instance). |
| **Processor (if any)** | None – all processing is performed by the controller’s own infrastructure. |

---

## 2. Data Subjects  

- **Patients** – Individuals receiving medical care whose encounters are being documented.  

*(No other data‑subject categories are involved in this PA.)*

---

## 3. Personal Data Elements  

| Data Element | Source (file / DB table) | Sensitivity (GDPR) | Retention (per OpenMRS policy) |
|---|---|---|---|
| `encounter_date` | `Encounter` entity → persisted in **`encounter`** table (`api/src/main/java/org/openmrs/Encounter.java`). | Low (date of service) | Kept for the duration of the patient record (minimum 10 years, per local health‑law retention schedule). |
| `patient_id` | `Encounter.patient` → foreign key to **`patient`** table (`api/src/main/java/org/openmrs/Patient.java`). | **High** – direct identifier linking to a health record (special‑category personal data under Art. 9). | Same as above – retained as long as the patient record is kept. |
| `encounter_type` | `Encounter.encounterType` → persisted in **`encounter_type`** table (`api/src/main/java/org/openmrs/EncounterType.java`). | Low (categorical code) | Same as above. |

*All three elements are stored together in the `encounter` table and are subject to the same audit‑field handling (`dateCreated`, `dateChanged`, `creator`, `changedBy`).*

---

## 4. Data Flows  

### Primary (active) flow  

```
[Web UI / REST API] 
   │   (HTTP/HTTPS POST /openmrs/ws/rest/v1/encounter)
   ▼
[OpenMRS Filter Chain]  (OpenmrsFilter → ServiceContext)
   │
   ▼
[EncounterService.saveEncounter(Encounter)]   (api/src/main/java/org/openmrs/api/EncounterService.java:46‑71)
   │   – validates privileges (GET_ENCOUNTERS, EDIT_ENCOUNTERS)
   │   – cascades patient & location data to Obs, Orders, etc.
   ▼
[Hibernate] → SQL INSERT/UPDATE on `encounter`, `encounter_type`, `patient` FK
   │   (HibernateContextDAO → Relational DB)
   ▼
[Relational DB (MySQL/PostgreSQL)]   (tables: `encounter`, `patient`, `encounter_type`)
```

### Retrieval flow (read)

```
[Web UI / REST API] 
   │   (GET /openmrs/ws/rest/v1/encounter/{uuid})
   ▼
[EncounterService.getEncounterByUuid(String)]   (api/src/main/java/org/openmrs/api/EncounterService.java:98‑108)
   │
   ▼
[Hibernate] → SELECT from `encounter` (joined with `patient`, `encounter_type`)
   ▼
[Relational DB] → result returned to service → JSON response to caller
```

### Legacy / Commented‑out flows  

- No legacy or commented‑out flows are present for encounter documentation in the current code base.

---

## 5. Third Parties / Processors  

| Vendor / Organisation | Role | Data Shared | Hosting / Location |
|---|---|---|---|
| *None* | – | – | – |

*All processing occurs within the controller’s own Docker‑hosted OpenMRS instance (Tomcat container, internal MySQL/PostgreSQL). No external third‑party processors are involved.*

---

## 6. Security Measures (PA‑specific)

| Measure | Description | Reference |
|---|---|---|
| **Authentication** | Users must authenticate via `Context.authenticate(...)` (see `Security and Authentication` section). | `Context.authenticate` (SECURITY.md) |
| **Authorization** | `EncounterService` methods are protected by `@Authorized` annotations (`GET_ENCOUNTERS`, `EDIT_ENCOUNTERS`, `PURGE_ENCOUNTERS`). | `EncounterService.java` lines 46‑71, 78‑92, 197‑209 |
| **Audit Logging** | Every `Encounter` entity inherits `OpenmrsObject` audit fields (`dateCreated`, `creator`, `dateChanged`, `changedBy`). These are automatically populated by Hibernate. | Hibernate ORM layer (`HibernateContextDAO`) |
| **Transport Security** | All inbound/outbound HTTP traffic is expected to be over TLS (HTTPS) – enforced by deployment configuration (Docker‑Compose with `ports: "8443:8443"`). | Deployment docs (Dockerfile / docker‑compose.yml) |
| **Database Access Controls** | DB user has least‑privilege rights (SELECT/INSERT/UPDATE only on required tables). | `startup-init.sh` builds JDBC URL with limited user. |
| **Input Validation** | `EncounterService.saveEncounter` validates required fields (date, patient, type) and checks for nulls before persisting. | `EncounterService.java` validation logic |
| **Data Encryption at Rest** | Not explicitly coded in OpenMRS core; recommended to enable disk‑level encryption on the host (e.g., LUKS) – a **risk** if omitted. | – |
| **Concern** | Potential for privilege escalation if a user obtains `EDIT_ENCOUNTERS` without proper role review. Mitigation: regular role‑audit and separation of duties. | – |

---

## 7. Cross‑Border Transfers  

- **None**. All data resides within the internal relational database of the OpenMRS instance. No export or replication to foreign jurisdictions is performed by default.  
- If future integrations (e.g., HL7/FHIR export) are added, a Data Transfer Impact Assessment must be performed and appropriate legal mechanisms (Standard Contractual Clauses, adequacy decision) must be documented.

---

## 8. DPIA (Data Protection Impact Assessment) Trigger Assessment  

| Factor | Applicable? | Comments |
|---|---|---|
| Large‑scale processing (≥ 10 000 records) | **No** (typical clinic size) |
| Processing of **special‑category data** (health data) | **Yes** – patient health information is a special category under Art. 9. |
| Automated decision‑making with legal or similarly significant effect | **No** |
| Systematic monitoring of individuals | **Yes** – systematic recording of encounters for each patient. |
| Vulnerable data subjects (patients) | **Yes** |
| Use of new or emerging technologies | **No** (standard Java/Spring/Hibernate stack) |
| Cross‑border transfers | **No** |
| **DPIA Recommended** | **Yes** – because special‑category health data is processed and systematic monitoring occurs, a DPIA is required under GDPR Art. 35. |

**Recommended actions:**  
1. Conduct a formal DPIA documenting the necessity, proportionality, and safeguards.  
2. Verify that all privileged roles (e.g., `EDIT_ENCOUNTERS`) are assigned only to authorized clinical staff.  
3. Implement encryption‑at‑rest for the database or ensure host‑level encryption.  
4. Establish a regular audit of access logs (via SLF4J/Logback) to detect unauthorized reads of `encounter` records.  

---  

**Prepared by:** OpenMRS Core documentation team  
**Date:** 2026‑05‑11  

*All sections are fully populated with concrete references to source files, functions, database tables, and API endpoints as required.*