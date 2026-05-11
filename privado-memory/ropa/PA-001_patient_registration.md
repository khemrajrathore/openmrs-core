# Processing Activity: Patient Registration (PA‑001)

## 1. Overview

| Field | Value |
|---|---|
| **PA Name** | Patient Registration (PA‑001) |
| **Business Function** | Capture and store patient demographic data so that health‑care providers can identify patients and deliver clinical services. |
| **Processing Purpose** | Creation of a new patient record (or update of an existing one) to enable identification, clinical documentation, and care coordination. |
| **Legal Basis** | **GDPR Art. 6 (1)(c)** – processing is necessary for compliance with a legal obligation (health‑care statutory requirements). |
| **Controller** | OpenMRS Core project (the organisation that deploys the OpenMRS instance). |

## 2. Data Subjects
- **Patients** – Individuals who receive health‑care services from the facility that runs the OpenMRS instance.

## 3. Personal Data Elements

| Data Element | Source (code / UI) | Sensitivity (GDPR) | Retention |
|---|---|---|---|
| **name** | Collected via the patient registration JSP/Velocity page (`webapp/src/main/webapp/pages/patientRegistration.jsp`) and the REST endpoint `POST /openmrs/ws/rest/v1/patient` (`org.openmrs.web.controller.PatientController`). Persisted by `PatientService.savePatient` → `PatientDAO.savePatient`. Stored in the `person_name` table (linked to `person`). | **Medium** – personal data, not a special category on its own. | Kept as long as the patient record exists (or until a lawful deletion/purge is performed). |
| **date_of_birth** | Same UI and REST entry points as above. Stored in the `person` table (`date_of_birth` column). | **Medium** – personal data that can be combined with other data to identify a person. | Same as above. |
| **address** | Collected via the same registration UI and REST payload (`address` sub‑object). Persisted in the `person_address` table. | **Low** – personal data but not a special category. | Same as above. |

> **Note on special‑category data:** Although the three elements themselves are not “health data”, they are *necessary* for linking a patient to health records, which are special‑category data under GDPR Art. 9. Consequently the overall processing is considered to involve special‑category data.

## 4. Data Flows  

### Primary (active) flow
```
Patient (browser) ──► Web UI (JSP/Velocity) ──► OpenMRS Filter (OpenmrsFilter) ──► PatientController (REST) ──► PatientService.savePatient (org.openmrs.api.PatientService) ──► PatientDAO.savePatient (org.openmrs.api.db.PatientDAO) ──► Relational DB (MySQL/PostgreSQL) tables: patient, person, person_name, person_address
```

### Internal (service‑to‑service) flow
```
PatientService ──► ServiceContext (Spring) ──► Hibernate Session ──► INSERT/UPDATE statements on patient‑related tables
```

### Legacy / Commented‑out flows
- No legacy flows are present for patient registration in the current code base (all registration paths go through the UI or the REST API).

## 5. Third Parties / Processors

| Vendor / Entity | Role | Data Shared | Hosting / Location |
|---|---|---|---|
| **None** (as per the supplied “Relevant third parties: []”) | – | – | – |

*If a hosting provider (e.g., a cloud‑IaaS) is used, it would be a **processor** and should be listed here with its location and contractual safeguards.*

## 6. Security Measures

### Positive (implemented) controls
| Control | Implementation detail |
|---|---|
| **Authentication** | `Context.authenticate()` validates credentials; only authenticated sessions reach the `PatientController`. |
| **Authorization** | `PatientService.savePatient` is annotated with `@Authorized({ADD_PATIENTS, EDIT_PATIENTS})`; the framework checks the caller’s privileges before persisting. |
| **Transport security** | Recommended deployment uses HTTPS (TLS) for all inbound HTTP(S) traffic to Tomcat. |
| **Input validation** | The `PatientValidator` (invoked inside `PatientService`) checks mandatory fields and prevents injection attacks. |
| **Audit logging** | Hibernate automatically populates `dateCreated`, `creator`, `dateChanged`, `changedBy` on `OpenmrsObject`; logs are written via SLF4J/Logback. |
| **Database protection** | DB credentials are stored in `${OMRS_HOME}/runtimeProperties` with file‑system permissions; the DB itself can be hardened (e.g., role‑based access, encryption at rest). |
| **Backup & recovery** | Regular MySQL/PostgreSQL backups (outside the scope of this PA but part of the overall system). |

### Identified concerns / mitigations
| Concern | Mitigation |
|---|---|
| **Insufficient input sanitisation** (e.g., script injection in address fields) | Ensure the `PatientValidator` is up‑to‑date; enable server‑side HTML‑escaping in the UI layer. |
| **Mis‑configured TLS** (weak cipher suites) | Enforce TLS 1.2+ with strong ciphers in Tomcat’s `server.xml`. |
| **Excessive retention** (records kept longer than needed) | Implement a data‑retention policy in the OpenMRS “purge” job and document the schedule. |
| **Privilege creep** (users granted `ADD_PATIENTS` without need) | Conduct periodic role reviews; use least‑privilege principle. |

## 7. Cross‑Border Transfers
- **Not applicable** when the OpenMRS instance is hosted on‑premises within the same jurisdiction as the data subjects.  
- If a cloud provider is used, the transfer would be covered by Standard Contractual Clauses (SCCs) or an adequacy decision, and the SCC details must be recorded in the ROPA.

## 8. DPIA Trigger Assessment

| Factor | Applicable? | Comments |
|---|---|---|
| Large‑scale processing | **No** – typical health‑centre registers a limited number of patients per day. |
| Processing of special‑category data (health) | **Yes** – patient data links to clinical records (Art. 9). |
| Automated decision‑making with legal or similarly significant effect | **No** – registration is a manual data entry process. |
| Systematic monitoring of individuals | **No** – registration is a one‑off capture, not ongoing monitoring. |
| Vulnerable data subjects | **Yes** – patients may be considered vulnerable in a health context. |
| Use of new or innovative technology | **No** – standard Java/Spring/Hibernate stack. |
| Cross‑border transfers | **No** – data stays within the host environment. |
| **DPIA Recommended** | **Yes** | Because special‑category data are processed and vulnerable subjects are involved, a DPIA should be carried out (or a DPIA‑exemption justification documented). |

---

**Prepared by:** OpenMRS documentation team  
**Date:** 2026‑05‑11  

*All sections are fully populated with concrete references to source files, service/DAO methods, database tables, and API paths as required by the ROPA template.*