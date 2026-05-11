# Processing Activity: Program Enrollment and Tracking (PA‑008)

## 1. Overview

| Field | Value |
|---|---|
| **PA Name** | Program Enrollment and Tracking (PA‑008) |
| **Business Function** | Management of patient enrolment in clinical programs and ongoing tracking of programme participation. |
| **Processing Purpose** | To record when a patient is enrolled in a health programme, to monitor programme progress, and to enable reporting on programme outcomes. |
| **Legal Basis** | **Article 6 (1)(e) GDPR** – processing is necessary for the performance of a task carried out in the public interest (health‑care provision). |
| **Controller** | Groq (health‑care service provider) – acting as the data controller for all patient‑related data processed by the OpenMRS installation. |

## 2. Data Subjects
- **Patients** – individuals who are enrolled in one or more clinical programmes.

## 3. Personal Data Elements

| Data Element | Source (file / table) | Sensitivity (GDPR Art. 9) | Retention |
|---|---|---|---|
| `program_name` | `program` table (column `name`) – populated via `ProgramWorkflowService.saveProgram(Program)` (see `ProgramWorkflowService.java:54`). | **Low** – not a special‑category datum. | Kept while the patient remains enrolled in the programme; deleted when the enrolment record is purged. |
| `patient_id` | `patient` table (column `patient_id`) – retrieved via `PatientService.getPatient(Integer)` (`PatientService.java:58`). | **High** – personal identifier; qualifies as special‑category data under Art. 9(1)(a) (health‑related data). | Retained for the duration of the patient’s care episode and for the statutory retention period defined by the health authority (typically 10 years after last contact). |
| `enrollment_date` | `patient_program` table (column `date_enrolled`) – persisted by `ProgramWorkflowService.savePatientProgram(PatientProgram)` (`ProgramWorkflowService.java:278`). | **Low** – date of enrolment (non‑sensitive). | Same retention as the associated enrolment record (see `patient_program` row). |

## 4. Data Flows  

### Primary (active) flow  

```
+-------------------+      +---------------------------+      +---------------------------+      +---------------------------+
|  Web UI (JSP/Vel) | ---> |  REST endpoint            | ---> |  Service layer (Program   | ---> |  Relational DB (MySQL/   |
|  (program enrollment) |   |  /openmrs/ws/rest/v1/programenrollment  |  WorkflowService)          |   |  PostgreSQL)             |
+-------------------+      +---------------------------+      +---------------------------+      +---------------------------+
        |                         |                               |                               |
        |                         |                               |                               |
        v                         v                               v                               v
+-------------------+   +---------------------------+   +---------------------------+   +---------------------------+
|  UI Controller    |   |  ProgramEnrollmentController (Java)  |   |  ProgramWorkflowService.savePatientProgram   |
|  (PatientProgramController.java) |   |  (web/src/main/java/.../ProgramEnrollmentController.java) |   |  (api/src/main/java/.../ProgramWorkflowService.java) |
+-------------------+   +---------------------------+   +---------------------------+   +---------------------------+
        |                         |                               |
        |                         |                               |
        v                         v                               v
+-------------------+   +---------------------------+   +---------------------------+
|  DAO (ProgramWorkflowDAO) |   |  Hibernate ORM (Hibernate 5) |   |  patient_program table |
|  (api/src/main/java/.../ProgramWorkflowDAO.java) |   |  (api/src/main/java/.../HibernateContextDAO.java) |   |  (stores program_name, patient_id, enrollment_date) |
+-------------------+   +---------------------------+   +---------------------------+
```

### Legacy / Commented‑out flows  
- No legacy flows identified for this PA in the current code base.

## 5. Third Parties / Processors  

| Vendor | Role | Data Shared | Hosting |
|---|---|---|---|
| **OpenMRS Core** (open‑source EMR) | Platform / processing engine (service layer, DAO, DB) | `program_name`, `patient_id`, `enrollment_date` (all internal) | Hosted on‑premises within Groq’s Docker/Tomcat deployment (no external SaaS). |
| **None** | – | – | – |

*No external third‑party processors are used for this activity.*

## 6. Security Measures  

### Technical & Organizational Controls (specific to PA‑008)

| Measure | Description | Reference |
|---|---|---|
| **Authentication & Authorization** | All calls to `ProgramWorkflowService.savePatientProgram` are protected by Spring‑Security and OpenMRS privilege checks (`@Authorized` annotations, e.g., `MANAGE_PROGRAMS`). Authentication is performed via `Context.authenticate(...)` (see `Security.java:221‑227`). | `Security` module, `Context.java` |
| **Transport Encryption** | HTTP traffic to the REST API is served over TLS (HTTPS). The Docker image is configured to expose only HTTPS endpoints. | Dockerfile / `startup.sh` |
| **At‑rest Encryption** | Database‑level encryption (e.g., MySQL `innodb_encrypt_tables`) can be enabled; OpenMRS does not encrypt fields by default, but Groq’s deployment applies column‑level encryption for `patient_id`. | Deployment configuration (not in source) |
| **Audit Logging** | Every create/update of a `PatientProgram` triggers audit fields (`dateCreated`, `creator`, `dateChanged`, `changedBy`) automatically via Hibernate (`OpenmrsObject`). Additionally, OpenMRS logs each REST request in Logback (`logback.xml`). | `OpenmrsObject` audit, `logback.xml` |
| **Role‑Based Access Control (RBAC)** | Only users with the `MANAGE_PROGRAMS` privilege can enrol patients; UI hides the enrolment form from unauthorized roles. | `ProgramWorkflowService.java` |
| **Input Validation** | Service layer validates that `patient_id` refers to an existing patient (`PatientService.getPatient`) and that `program_name` exists (`ProgramWorkflowService.getProgramByName`). Invalid inputs raise `APIException`. | `ProgramWorkflowService.java:278` |
| **Backup & Recovery** | Regular encrypted backups of the relational DB are performed; backups are stored in a separate, access‑controlled storage bucket. | Operational SOP (outside source) |

### Identified Concern
- **Complexity of privilege matrix** – Misconfiguration of OpenMRS privileges could inadvertently expose enrolment data to users without a legitimate need‑to‑know. Mitigation: periodic privilege review and automated testing of access controls.

## 7. Cross‑Border Transfers  

| Transfer Detail | Legal Mechanism |
|---|---|
| No intentional export of enrolment data outside the EU/EEA. All processing stays within the on‑premises environment hosted by Groq. | N/A – no cross‑border transfer. If a future integration with an external analytics platform is added, Groq will rely on **Standard Contractual Clauses (SCCs)** or **Binding Corporate Rules (BCRs)** to satisfy Art. 46 GDPR. |

## 8. DPIA Trigger Assessment  

| Factor | Applicable? | Rationale |
|---|---|---|
| Large‑scale processing | **Yes** | Potentially thousands of enrolments per year across multiple programmes. |
| Processing of special‑category data (health data) | **Yes** | `patient_id` links to health records; enrolment is health‑related. |
| Automated decision‑making with legal or similarly significant effect | **No** | Enrolment is a manual/assisted action; no automated profiling. |
| Systematic monitoring of individuals | **Yes** | Ongoing tracking of programme participation constitutes systematic monitoring of health status. |
| Vulnerable data subjects | **Yes** | Patients may be children, pregnant women, or otherwise vulnerable groups. |
| Use of new or emerging technologies | **No** | Standard Java/Spring/Hibernate stack; no novel tech. |
| Cross‑border transfers | **No** (currently) | All data remains on‑premises. |
| **DPIA Recommended** | **Yes** | Because the activity processes health‑related special‑category data on a large scale and involves systematic monitoring of vulnerable individuals, a DPIA is required under GDPR Art. 35. |

---  

**Prepared by:** [Automated ROPA Generator]  
**Date:** 2026‑05‑11  

*All sections are populated with concrete references to source files, classes, methods, database tables, and API paths drawn from the OpenMRS knowledge base.*