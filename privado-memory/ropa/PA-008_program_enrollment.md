# Record of Processing Activities (ROPA) – OpenMRS Core  
**Processing Activity:** **Program Enrollment (PA‑008)**  

---

## 1. Overview  

| **PA Name** | **Business Function** | **Processing Purpose** | **Legal Basis (GDPR)** | **Controller** |
|-------------|----------------------|------------------------|------------------------|----------------|
| PA‑008 | Program Enrollment | Enroll patients into health programs, track their progress through defined workflows and generate program‑specific reports. | **Art. 9(2)(h) – processing of special category data is necessary for the provision of health or social care** (plus Art. 6(1)(f) for legitimate interests of the controller in managing programmes). | **OpenMRS Community** (the open‑source project) – operated by the hosting health‑care organisation that runs the OpenMRS instance. |

---

## 2. Data Subjects  

- **Patients** – individuals receiving care (vulnerable data subjects; health data is a special‑category).  
- **Healthcare providers** – clinicians, nurses, community health workers who enrol patients and view programme status.  
- **System administrators / programme managers** – staff that configure programmes and run reports.

---

## 3. Personal Data Elements  

| **Data Element** | **Source (Domain Class / Table)** | **Sensitivity** | **Retention** |
|------------------|-----------------------------------|-----------------|----------------|
| **Enrollment ID** | `ProgramEnrollment` class → `program_enrollment` table (PK `program_enrollment_id`) | High – uniquely identifies a health‑program participation record. | Kept for the lifetime of the patient record or until the patient is formally disenrolled and the retention period for clinical data expires (minimum 10 years per national health‑care law). |
| **Patient ID** | `Patient` class → `patient` table (PK `patient_id`) | High – links to full patient health record (special‑category). | Same as patient record retention (≥ 10 years after last contact). |
| **Program ID** | `Program` class → `program` table (PK `program_id`) | Low – identifies the programme definition (non‑personal). | Retained as long as the programme definition exists (typically indefinite). |
| **ProgramWorkflow ID** | `ProgramWorkflow` class → `program_workflow` table (PK `program_workflow_id`) | Low – identifies the workflow steps (non‑personal). | Same as programme definition. |
| **Enrollment Date** | `ProgramEnrollment.enrollmentDate` (column `date_enrolled` in `program_enrollment`) | Low – date of enrolment (personal but not highly sensitive). | Same as the enrolment record (see *Enrollment ID*). |
| **Completed Date** (optional) | `ProgramEnrollment.completionDate` (column `date_completed`) | Low – indicates programme completion. | Same as enrolment record. |
| **Outcome / State** (e.g., ACTIVE, COMPLETED, WITHDRAWN) | `ProgramEnrollment.state` (column `state`) | Low – operational status. | Same as enrolment record. |

*All fields are persisted via **Hibernate** ORM and are versioned by **Hibernate Envers** for audit trails.*

---

## 4. Data Flows  

### 4.1 ASCII Sequence Diagram (Ingress → Core → Persistence → Egress)

```
Patient / Clinician                     OpenMRS Web UI / REST API
        |                                         |
        |  POST /openmrs/ws/rest/v1/programenrollment
        |---------------------------------------->|
        |                                         |
        |   ProgramWorkflowService.enrollPatient(
        |       Patient, Program, ProgramWorkflow, Date)
        |                                         |
        |   ↳ ProgramEnrollmentService (implementation)
        |                                         |
        |   ↳ DAO → Hibernate → INSERT into program_enrollment
        |                                         |
        |   ←--- Success (Enrollment ID, status)---|
        |                                         |
        |   (Optional) Notification via MessageService → Email / UI alert
```

### 4.2 Internal Component Flow (simplified)

```
+-------------------+      +----------------------+      +-------------------+
|  UI / REST Layer  | ---> |  Service Layer       | ---> |  DAO / Hibernate  |
| (ProgramEnrollmentController) | (ProgramWorkflowService) | (ProgramEnrollmentDAO) |
+-------------------+      +----------------------+      +-------------------+
                                   |
                                   v
                           +-----------------+
                           |  MySQL / MariaDB|
                           |  (program_*,   )|
                           +-----------------+
                                   |
                                   v
                           +-----------------+
                           |  Audit (Envers) |
                           +-----------------+
```

### 4.3 Egress (notifications, reports)

```
ProgramEnrollment → MessageService → Email / SMS
ProgramEnrollment → REST response (JSON) → Calling system
ProgramEnrollment → Reporting module → PDF/CSV export (file system)
```

---

## 5. Third Parties / Processors  

| **Vendor / Organisation** | **Role** | **Data Shared** | **Hosting / Location** |
|---------------------------|----------|-----------------|------------------------|
| *None* (as per current core) | – | – | – |
| (If a custom module is installed, e.g., a national health‑information exchange) | Processor | Enrollment data (Patient ID, Programme ID, dates) | Cloud region specified by the module’s SLA (e.g., EU‑based data centre) |

*The baseline OpenMRS Core does not export enrolment data to external third‑party services. Any such sharing would be governed by a separate Data Processing Agreement (DPA).*

---

## 6. Security Measures  

**Technical & organisational safeguards implemented in OpenMRS Core (relevant to PA‑008):**

| Measure | Description | Status / Comment |
|---------|-------------|------------------|
| **Authentication** | Spring Security with BCrypt password hashing for all user accounts. | Enabled |
| **Authorization** | `@Authorized` AOP annotations on service methods (e.g., `ProgramWorkflowService.enrollPatient`) enforce role‑based access (only users with `Manage Programs` privilege). | Enabled |
| **Transport Security** | All HTTP endpoints are expected to be served over TLS 1.2+ (configured at the servlet container – Tomcat). | Recommended |
| **Database Encryption** | At‑rest encryption can be configured for MySQL/MariaDB (e.g., InnoDB tablespace encryption). Not enforced by core but recommended. | Optional |
| **Audit Logging** | Hibernate Envers records every INSERT/UPDATE/DELETE on `program_enrollment` with user, timestamp, and previous values. | Enabled |
| **Transaction Management** | Spring `@Transactional` ensures atomic enrolment creation; rollback on validation failure. | Enabled |
| **Input Validation** | Service layer validates programme, workflow, and patient existence before persisting. | Enabled |
| **Backup & Recovery** | Regular encrypted backups of the MySQL database; backups stored in the same jurisdiction unless a cross‑border transfer is explicitly configured. | Recommended |
| **File‑system Protection** | Exported reports are written to a protected directory with OS‑level ACLs; no public read access. | Enabled |
| **Security Patches** | OpenMRS releases monthly security patches; Docker images are rebuilt with the latest patches. | Ongoing |
| **Vulnerability Scanning** | CI pipeline runs OWASP Dependency‑Check on Maven dependencies. | Enabled |

**Open Issues / Considerations**

- No built‑in encryption for Elasticsearch indices – if the optional search backend is used, enable TLS and encryption‑at‑rest.  
- Ensure that any custom modules that expose enrolment data over APIs also implement the same `@Authorized` checks.  

---

## 7. Cross‑Border Transfers  

| **Transfer Destination** | **Data Sent** | **Legal Mechanism** | **Justification** |
|--------------------------|---------------|---------------------|-------------------|
| *None* (default OpenMRS Core) | – | – | All data remains on the local MySQL instance. |
| (If a backup is stored in a cloud provider outside the EU, e.g., AWS US‑East) | Full database dump containing `program_enrollment` and related tables | **Standard Contractual Clauses (SCCs)** under Art. 46 GDPR, supplemented by a **Transfer Impact Assessment** (TIA) | Required for disaster‑recovery; ensure SCCs are signed with the cloud provider and that the provider offers equivalent data‑protection guarantees. |

*If the organisation never stores data outside the EU, the “Cross‑Border Transfers” section can be left as “None”.*

---

## 8. DPIA Trigger Assessment  

| **Factor** | **Assessment** | **Risk Level** |
|------------|----------------|----------------|
| **Special‑category health data** (Art. 9) | Processing of enrolment ties a patient to a health programme – clearly special‑category. | **High** |
| **Volume & scope** | Enrolments are created for every patient entering a programme; potentially thousands of records. | Medium |
| **Data subject rights impact** | Patients can request rectification, restriction, or erasure of enrolment data; system supports these via the `ProgramWorkflowService` APIs. | Medium |
| **Cross‑border transfer** | Only if backups are stored abroad; mitigated by SCCs. | Low‑Medium |
| **Technical safeguards** | Strong authentication, authorisation, audit logging, encryption‑at‑rest (optional). | Low |
| **Potential for unauthorised access** | Mis‑configuration of roles could expose enrolment data. | Medium |
| **Overall DPIA necessity** | Because special‑category data is processed at scale and the activity is systematic, a **Data Protection Impact Assessment (DPIA) is required** under Art. 35 GDPR. | **Required** |

**Final Recommendation:** Conduct a full DPIA, document the lawful basis, retention schedule, and mitigation measures, and obtain sign‑off from the Data Protection Officer (DPO) before the enrolment feature is deployed in production.  

---  

*Document generated on 2026‑05‑11 for OpenMRS Core (PA‑008 – Program Enrollment).*