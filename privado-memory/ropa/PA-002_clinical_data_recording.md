# Processing Activity: Clinical Data Recording (PA‑002)

## 1. Overview

| Field | Value |
|---|---|
| **PA Name** | PA‑002 – Clinical Data Recording |
| **Business Function** | Capture and store clinical observations (medical history, diagnoses, medications) entered by clinicians during patient encounters |
| **Processing Purpose** | To enable diagnosis, treatment planning, continuity of care, reporting and analytics for health‑care delivery |
| **Legal Basis** | **GDPR Art. 6(1)(e)** – processing is necessary for the performance of a task carried out in the public interest (health‑care) and **GDPR Art. 9(2)(h)** – processing of special‑category health data is allowed for the provision of health or social care services |
| **Controller** | OpenMRS Core project (the organisation that deploys and operates the OpenMRS instance) |

## 2. Data Subjects
- **Patients** – individuals receiving health‑care services whose clinical observations are recorded.

## 3. Personal Data Elements

| Data Element | Source (code / DB) | Sensitivity (GDPR) | Retention |
|---|---|---|---|
| medical_history | `Obs` objects created via `ObsService.saveObs()` (source file: `api/src/main/java/org/openmrs/api/ObsService.java:87‑124`); persisted in the **`obs`** table (MySQL/PostgreSQL) | **Special category** (health data) – Art. 9 | Kept for the duration required by national health‑care regulations (typically the patient’s lifetime or as defined by the health‑system retention policy) |
| diagnoses | Same as above – stored as coded `Obs` rows (concept = diagnosis) in **`obs`** table; created through the same service method | **Special category** – Art. 9 | Same retention as medical_history |
| medications | Same as above – stored as `Obs` rows with medication concepts; created via `ObsService.saveObs()` | **Special category** – Art. 9 | Same retention as medical_history |

*All three elements are created/updated by the **ObsService** (or indirectly via **EncounterService.saveEncounter()**, which cascades Obs persistence).*

## 4. Data Flows  

### Primary (active) flow
```
[Clinician UI / REST client] 
   → HTTP(S) POST /openmrs/ws/rest/v1/obs   (REST API)
   → OpenmrsFilter → ServiceContext
   → ObsService.saveObs(Obs, String)        (api/src/main/java/org/openmrs/api/ObsService.java)
   → ObsDAO.saveObs(Obs)                    (api/src/main/java/org/openmrs/api/db/ObsDAO.java)
   → Hibernate → Relational DB (obs table)  (MySQL / PostgreSQL)
   → Data stored in internal database
```

### Supporting internal flows
- **Encounter creation** (`EncounterService.saveEncounter()`) automatically persists any Obs attached to the encounter, using the same DAO chain.
- **Search / retrieval** (`ObsService.getObservations(...)`) reads from the **`obs`** table via `ObsDAO.getObservations`.

### Legacy / Commented‑out flows
- No legacy or commented‑out data‑flow paths are present for this PA in the current OpenMRS code base.

## 5. Third Parties / Processors

| Vendor / Processor | Role | Data Shared | Hosting |
|---|---|---|---|
| *None* | – | – | – |

*All processing occurs inside the OpenMRS container and the internal relational database.*

## 6. Security Measures

### Positive (implemented) controls
| Control | Description | Technical artefact |
|---|---|---|
| **Authentication** | Users must authenticate via `Context.authenticate()` (file: `api/src/main/java/org/openmrs/api/Context.java:191‑199`). | Session‑based `UserContext` stored in a `ThreadLocal`. |
| **Authorization** | Each Obs‑related operation requires the `ADD_OBS` / `EDIT_OBS` privileges, enforced by `@Authorized` annotations on `ObsService` methods. | `ObsService.saveObs` is annotated with `@Authorized({ADD_OBS, EDIT_OBS})`. |
| **Transport security** | All inbound HTTP traffic is expected to be protected by TLS (HTTPS) at the load‑balancer / reverse‑proxy level. | Not coded in OpenMRS but part of deployment best‑practice. |
| **Data‑at‑rest encryption** | Database disks are typically encrypted (e.g., LUKS, Azure Disk Encryption) as part of the Docker host hardening guide. | Deployment‑specific, not in source code. |
| **Audit logging** | Every create / update of an `Obs` records `dateCreated`, `creator`, `dateChanged`, `changedBy` automatically via `OpenmrsObject` audit fields (Hibernate). | `Obs` inherits from `BaseOpenmrsObject`. |
| **Input validation** | `ObsService.saveObs` validates required fields (concept, person, encounter) and checks for null values before persisting. | `ObsService.saveObs` implementation. |
| **Database access control** | The relational DB user has least‑privilege rights (only `SELECT`, `INSERT`, `UPDATE`, `DELETE` on the `obs` schema). | Configured in `docker‑compose.yml` / `startup‑init.sh`. |

### Identified concerns for this PA
| Concern | Impact | Mitigation |
|---|---|---|
| **Privilege escalation** – If a user is granted `ADD_OBS` without proper role review, they could insert falsified clinical data. | Compromises data integrity and patient safety. | Enforce strict role‑based access control (RBAC) and periodic privilege reviews. |
| **Insufficient encryption of backups** – Database backups may be stored unencrypted on the host file system. | Risk of data breach if backup media are accessed. | Apply encryption to backup files and restrict filesystem permissions. |
| **Lack of granular consent tracking** – GDPR requires evidence of lawful basis per data subject; OpenMRS does not store explicit consent flags per Obs. | Potential non‑compliance in jurisdictions requiring explicit consent. | Implement a consent module that records consent status linked to patient records. |

## 7. Cross‑Border Transfers
- No data is transmitted outside the jurisdiction where the OpenMRS instance is hosted. All flows remain within the internal network and the local relational database.

## 8. DPIA Trigger Assessment

| Factor | Applicable? | Comments |
|---|---|---|
| Large‑scale processing | **Yes** | Potentially thousands of Obs per day across many facilities. |
| Processing of special‑category data | **Yes** | Medical history, diagnoses, and medication data are health data (GDPR Art. 9). |
| Automated decision‑making with legal or similarly significant effects | **No** | Obs are stored for human‑driven clinical decisions; no automated profiling. |
| Systematic monitoring of individuals | **Yes** | Continuous clinical monitoring via Obs collection. |
| Vulnerable data subjects | **Yes** | Patients are a vulnerable group under GDPR. |
| Use of new or emerging technologies | **No** | Standard Java/Spring/Hibernate stack, no novel tech. |
| Cross‑border transfers | **No** | All processing stays on‑premises. |
| **DPIA Recommended** | **Yes** | Because the activity processes large volumes of special‑category health data on a systematic basis, a DPIA is required under GDPR Art. 35. |

---  

**Prepared by:** OpenMRS compliance analyst  
**Date:** 2026‑05‑11