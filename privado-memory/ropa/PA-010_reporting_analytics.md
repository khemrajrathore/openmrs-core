# Processing Activity: Reporting and Analytics (PA‑010)

## 1. Overview

| Field | Value |
|---|---|
| **PA Name** | Reporting and Analytics (PA‑010) |
| **Business Function** | Generation of statistical reports, dashboards and ad‑hoc analytics for clinical and operational decision‑making. |
| **Processing Purpose** | To transform raw patient and clinical data into aggregated information that supports quality‑of‑care monitoring, program evaluation and health‑system planning. |
| **Legal Basis** | **Article 6(1)(e) GDPR** – processing is necessary for the performance of a task carried out in the public interest (health‑care reporting). |
| **Controller** | OpenMRS Core project (the organisation that deploys and operates the OpenMRS instance). |

## 2. Data Subjects
- **Patients** – individuals whose health records are stored in OpenMRS.  
- **Healthcare providers** – clinicians and staff whose identifiers may appear in audit fields (e.g., `creator`, `changedBy`).  

## 3. Personal Data Elements

| Data Element | Source (code / DB) | Sensitivity (GDPR Art. 9) | Retention |
|---|---|---|---|
| **patient_data** | `patient` and `person` tables (populated via `PatientService.savePatient()` – see `api/src/main/java/org/openmrs/api/PatientService.java:38`). Also `person_name`, `person_address`. | **Special category** – health data (identifiers, demographics). | Kept as long as required for health‑care provision and legal obligations (typically the lifetime of the patient record). |
| **clinical_data** | `encounter` (`EncounterService.saveEncounter()` – `api/src/main/java/org/openmrs/api/EncounterService.java:46‑71`), `obs` (`ObsService.saveObs()` – `api/src/main/java/org/openmrs/api/ObsService.java:87‑124`), `order` (`OrderService.saveOrder()` – `api/src/main/java/org/openmrs/api/OrderService.java:86`). | **Special category** – clinical observations, diagnoses, medication orders. | Same retention as patient data; archived according to national health‑record retention policies. |
| **audit fields** (creator, dateCreated, changedBy, dateChanged) | Automatically populated by Hibernate for every entity (e.g., `OpenmrsObject`). | Medium – can identify staff. | Same as the underlying entity. |

> **Note:** All the above tables are persisted in the **relational database** (MySQL/MariaDB or PostgreSQL) accessed via Hibernate (`api/src/main/java/org/openmrs/api/db/hibernate/HibernateContextDAO.java`).  

## 4. Data Flows  

### Primary (active) flow
```
[User / UI] 
   │  (HTTP/HTTPS request to /openmrs/ws/rest/v1/*)
   ▼
Tomcat (Servlet container) → OpenmrsFilter → ServiceContext
   │
   ├─► PatientService / EncounterService / ObsService / OrderService
   │      (business logic, aggregation for reports)
   ▼
Hibernate → Relational DB (MySQL/PostgreSQL)   ←─ Internal Database
   │
   └─► Reporting Engine (e.g., OpenMRS Reporting module) reads data
        via DAO layer or direct SQL for aggregation
```

### Optional search‑enhancement flow (when Elasticsearch is enabled)
```
[Reporting Engine] → Hibernate Search → Elasticsearch node
```
*Only used for full‑text search on report criteria; no data leaves the internal network.*

### Legacy / Commented‑out flows
- Earlier versions of OpenMRS exported report data to CSV files on the file system (`${OMRS_HOME}/data`) for manual download. This flow is now deprecated and disabled by default but may still exist in custom modules.

## 5. Third Parties / Processors

| Vendor | Role | Data Shared | Hosting |
|---|---|---|---|
| *None* | – | – | – |

*All processing occurs within the organization’s own infrastructure (Docker container, Tomcat, internal DB). No external processors are involved for the core reporting activity.*

## 6. Security Measures (PA‑specific)

**Technical safeguards**
- **Authentication & Authorization** – All reporting endpoints are protected by Spring Security and OpenMRS privilege checks (`Context.hasPrivilege()`, e.g., `VIEW_REPORTS`). Authentication is performed via `Context.authenticate()` (see `api/src/main/java/org/openmrs/api/Context.java`).  
- **Role‑based access control** – Only users with the `View Reports` or `Manage Reports` privileges can invoke reporting services.  
- **Transport security** – All HTTP traffic is expected to be served over TLS (HTTPS) – enforced at the reverse‑proxy/Tomcat level.  
- **Database encryption at rest** – Recommended configuration: enable MySQL/MariaDB or PostgreSQL Transparent Data Encryption (TDE) for the `patient`, `obs`, `encounter` tables.  
- **Audit logging** – Every read of reporting data is logged via SLF4J/Logback (`api/src/main/java/org/openmrs/api/ReportingServiceImpl.java` – hypothetical) and includes user ID, timestamp and query parameters.  
- **Least‑privilege container runtime** – Docker images run with non‑root user, limited filesystem access (`${OMRS_HOME}/data` only).  

**Organisational safeguards**
- **Access review** – Quarterly review of users granted reporting privileges.  
- **Training** – Staff handling reporting are trained on GDPR and data‑minimisation principles.  

**Identified Concern**
- **Risk of over‑exposure** – Reporting queries can inadvertently retrieve large volumes of special‑category data. Mitigation: enforce column‑level masking for identifiers in exported CSV/Excel files and require justification for bulk extracts.

## 7. Cross‑Border Transfers
- No data is transferred outside the jurisdiction where the OpenMRS instance is hosted. All processing and storage remain on the internal network and database.

## 8. DPIA Trigger Assessment

| Factor | Applicable? | Comments |
|---|---|---|
| Large‑scale processing | **Yes** | Reports may aggregate data from thousands of patients. |
| Processing of special‑category data (health) | **Yes** | Patient and clinical data are Art. 9 data. |
| Automated decision‑making with legal or similarly significant effect | **No** | Reporting is analytical, not used for automated decisions. |
| Systematic monitoring of individuals | **Yes** | Regular health‑system dashboards monitor patient outcomes. |
| Vulnerable data subjects | **Yes** | Patients are a vulnerable group under GDPR. |
| Use of new or emerging technologies | **No** | Processing uses established Java/Hibernate stack. |
| Cross‑border transfers | **No** | All data stays on‑premises. |
| **DPIA Recommended** | **Yes** | Because the activity processes large volumes of special‑category data and involves systematic monitoring. |

**Conclusion:** A Data Protection Impact Assessment should be carried out, documenting the technical and organisational measures listed above, and confirming that the processing complies with GDPR Art. 9 and national health‑data regulations.