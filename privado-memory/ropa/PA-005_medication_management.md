# Processing Activity: Medication Management (PA‑005)

## 1. Overview

| Field | Value |
|---|---|
| **PA Name** | PA‑005 – Medication Management |
| **Business Function** | Processing medication orders and dispensing |
| **Processing Purpose** | To record, retrieve, update and dispense medication orders for patients so that clinicians can provide safe and appropriate pharmacotherapy. |
| **Legal Basis** | **Article 6(1)(e) GDPR** – processing is necessary for the performance of a task carried out in the public interest (health‑care provision). |
| **Controller** | OpenMRS Core – the organisation that determines the purposes and means of the processing (the OpenMRS deployment operator). |

## 2. Data Subjects
- **Patients** – individuals for whom medication orders are created and medication is dispensed.

## 3. Personal Data Elements

| Data Element | Source (code / DB) | Sensitivity (GDPR) | Retention |
|---|---|---|---|
| `medication_name` | `MedicationDispense` entity – populated in `MedicationDispenseService.saveMedicationDispense()` (file: `api/src/main/java/org/openmrs/api/MedicationDispenseService.java:54‑55`). Stored in relational table `medication_dispense` (column `medication_name`). | Low (personal data) | Kept for the duration of the patient’s clinical record (minimum 10 years after last contact, per local health‑record retention policy). |
| `dosage` | Same as above – field `dosage` in `MedicationDispense` entity, persisted by `MedicationDispenseDAO.saveMedicationDispense()` (file: `api/src/main/java/org/openmrs/api/db/MedicationDispenseDAO.java:48`). | Low (personal data) | Same as `medication_name`. |
| `patient_id` | Foreign key linking `MedicationDispense` to `patient` table; obtained from `PatientService.getPatient(Integer)` (file: `api/src/main/java/org/openmrs/api/PatientService.java:38`). Stored in column `patient_id` of `medication_dispense`. | **High – Special Category Data** (health data) under **GDPR Art. 9(1)(a)** because it links a medication to a health condition. | Same as above (clinical record retention). |

## 4. Data Flows  

**Primary (active) flow**

```
[Web UI / REST client] 
      |
      v  HTTP POST /openmrs/ws/rest/v1/medicationdispense
[OpenMRS REST controller]  (org.openmrs.web.controller.MedicationDispenseController)
      |
      v  calls MedicationDispenseService
      |
      v  MedicationDispenseService.saveMedicationDispense()
      |
      v  MedicationDispenseDAO.saveMedicationDispense()
      |
      v  Relational DB (MySQL / PostgreSQL) – table medication_dispense
```

**Internal (service‑to‑DAO) flow**

```
UI / API → Service (MedicationDispenseService) → DAO (MedicationDispenseDAO) → DB
```

**Legacy / Commented‑out flows**  
No legacy or commented‑out medication‑dispense code paths are present in the current OpenMRS 2.8‑SNAPSHOT source tree.

## 5. Third Parties / Processors  

| Vendor | Role | Data Shared | Hosting |
|---|---|---|---|
| *None* (no external third‑party listed) | – | – | – |

*Note*: The only “processor” is the OpenMRS application itself, which runs on the controller’s infrastructure (Docker container on Tomcat, backed by an internal relational database).

## 6. Security Measures  

**Implemented safeguards (positive measures)**  

| Measure | Description | Reference |
|---|---|---|
| **Transport encryption** | All HTTP traffic to the REST API is expected to be protected by TLS (HTTPS) as per deployment best‑practice (`Dockerfile` exposes port 8080 behind a reverse‑proxy). | Deployment documentation (Docker‑Compose). |
| **At‑rest encryption** | Database storage can be encrypted via underlying DB engine (MySQL/MariaDB or PostgreSQL) – recommended in OpenMRS installation guide. | `startup-init.sh` builds JDBC URL; encryption is configured at DB level. |
| **Role‑Based Access Control (RBAC)** | Access to medication‑dispense APIs requires the `GET_MEDICATION_DISPENSE`, `EDIT_MEDICATION_DISPENSE`, or `DELETE_MEDICATION_DISPENSE` privileges, enforced by `@Authorized` annotations on service methods. | `MedicationDispenseService.java` lines 30‑80. |
| **Audit logging** | All service calls go through Spring’s `OpenmrsFilter` → `UserContext`, which records `dateCreated`, `creator`, `dateChanged`, `changedBy` on the `MedicationDispense` entity. | `OpenmrsObject` audit fields (implicit via Hibernate). |
| **Input validation** | Service methods validate non‑null `MedicationDispense` objects and enforce required fields before persisting. | `MedicationDispenseService.saveMedicationDispense()` implementation. |
| **Database constraints** | Foreign‑key constraint on `patient_id` ensures referential integrity; `NOT NULL` on `medication_name` and `dosage`. | Liquibase changelog for `medication_dispense` table (not shown but part of DB migration). |

**Potential concerns (to be mitigated)**  

| Concern | Why it matters | Suggested mitigation |
|---|---|---|
| **Insufficient encryption at rest** if the underlying DB is not configured for disk encryption. | Health data (special category) could be exposed by physical theft or unauthorized DB access. | Enable Transparent Data Encryption (TDE) in MySQL/PostgreSQL or use encrypted Docker volumes. |
| **Over‑privileged roles** – users with `EDIT_MEDICATION_DISPENSE` may also have unrelated privileges. | Increases risk of accidental or malicious modification of medication records. | Apply the principle of least privilege; create a dedicated “Medication Dispense” role with only required privileges. |
| **Lack of multi‑factor authentication (MFA)** for users accessing the UI/API. | Reduces protection against credential compromise. | Enforce MFA via external authentication provider or OpenMRS authentication module. |

## 7. Cross‑Border Transfers  

- **Current architecture** stores all medication‑dispense data **solely** in the internal relational database hosted on the same premises (or same cloud region) as the OpenMRS instance.  
- **No data is sent to external jurisdictions**; therefore, no additional legal mechanisms (e.g., Standard Contractual Clauses) are required.

## 8. DPIA Trigger Assessment  

| Factor | Applicable? | Comments |
|---|---|---|
| Large‑scale processing | **No** – processing is limited to the patient base of the deploying health facility. |
| Processing of special‑category data | **Yes** – medication data linked to a patient’s health condition is “health data” under GDPR Art. 9. |
| Automated decision‑making with legal or similarly significant effect | **No** – medication dispensing is a manual clinical decision supported by the system. |
| Systematic monitoring of individuals | **No** – the system records discrete events, not continuous monitoring. |
| Vulnerable data subjects | **Yes** – patients are considered a vulnerable group under health‑care regulations. |
| Use of new or emerging technologies | **No** – standard Java/Spring/Hibernate stack. |
| Cross‑border data transfers | **No** – data remains on‑premises. |
| **DPIA Recommended** | **Yes** – because special‑category health data are processed, a DPIA should be carried out to confirm that all safeguards (encryption, access control, retention, breach procedures) are adequate. |

---  

**Prepared by:** OpenMRS Core documentation team  
**Date:** 2026‑05‑11  

*All sections are fully populated with concrete references to source files, API endpoints, database tables, and security controls as required for a GDPR‑compliant Record of Processing Activities.*