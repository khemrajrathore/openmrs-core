# Record of Processing Activities (ROPA) – Visit Tracking (PA‑007)

**Processing Activity (PA):** **Visit Tracking (PA‑007)**  
**System:** OpenMRS Core (open‑source electronic medical records)  

---

## 1. Overview  

| **PA Name** | **Business Function** | **Processing Purpose** | **Legal Basis (GDPR article)** | **Controller** |
|-------------|----------------------|------------------------|--------------------------------|----------------|
| Visit Tracking (PA‑007) | Managing patient visits & location of care | Record when a patient starts and ends a care episode, and at which facility/location the care occurs. | **Art. 9(2)(h)** – processing of special‑category data is necessary for the provision of health or social care services. | OpenMRS Core Development Team (acting on behalf of the health‑care organisation that deploys the instance). |

---

## 2. Data Subjects  

- **Patients** – individuals receiving health‑care (vulnerable data subjects).  
- **Healthcare providers** – clinicians, nurses, or other staff who create or update visit records.  
- **System administrators / auditors** – personnel who may view visit logs for operational or compliance purposes.

---

## 3. Personal Data Elements  

| **Data Element** | **Source (Domain class / DB table)** | **Sensitivity** | **Retention** |
|------------------|--------------------------------------|-----------------|----------------|
| Visit ID | `org.openmrs.Visit` → `visit` table (primary key `visit_id`) | Low (technical identifier) | Retained as long as the patient record is retained (minimum 10 years per national health‑record retention rules, or until anonymisation). |
| Patient ID | `org.openmrs.Patient` → `patient` table (`patient_id`) | **High – special‑category health data** (links visit to a patient) | Same as patient record (minimum 10 years, or until patient request for deletion where lawful). |
| Location ID | `org.openmrs.Location` → `location` table (`location_id`) | Low (facility identifier) | Retained while the location exists in the system (typically the lifetime of the health‑care provider). |
| Visit Start Date | `Visit.startDatetime` column (`visit_start_datetime`) | Low (date‑time) | Same as Visit ID. |
| Visit End Date | `Visit.stopDatetime` column (`visit_end_datetime`) | Low (date‑time) | Same as Visit ID. |

*All fields are stored in MySQL/MariaDB; optional search indexes are kept in Elasticsearch (no additional personal data stored).*

---

## 4. Data Flows  

### 4.1 ASCII Sequence Diagram (Ingress → Persistence → Egress)

```
+----------------+      +----------------+      +-------------------+
|  UI / REST API | ---> | VisitService   | ---> | MySQL / MariaDB   |
| (webapp, API)  |      | (saveVisit())  |      | (visit, patient, |
+----------------+      +----------------+      |  location tables)|
                                                +-------------------+
```

### 4.2 Detailed Flow (textual)

1. **Ingress** – A clinician or integration (REST call, HL7 inbound, UI form) invokes `VisitService.saveVisit(Visit visit)`.  
2. **Validation & Enrichment** – `VisitService` checks that:  
   * `visit.getPatient()` exists (`PatientService.getPatientById`).  
   * `visit.getLocation()` exists (`LocationService.getLocationById`).  
   * `startDatetime` ≤ `stopDatetime`.  
3. **Persistence** – Hibernate writes a row to the `visit` table (`visit_id`, `patient_id`, `location_id`, `visit_start_datetime`, `visit_end_datetime`). Foreign‑key constraints enforce linkage to `patient` and `location`.  
4. **Auditing** – Hibernate Envers creates an audit entry in `visit_audit` for every INSERT/UPDATE/DELETE.  
5. **Search Index (optional)** – If Elasticsearch is enabled, the `Visit` entity is indexed via Hibernate Search for fast lookup; the index contains only the same identifiers (no new personal data).  
6. **Egress** – Data may be returned to the caller as JSON/XML via the OpenMRS REST module, or used internally for reporting (e.g., `VisitReportService`). No external transmission to third parties occurs.

---

## 5. Third Parties / Processors  

| **Vendor** | **Role** | **Data Shared** | **Hosting** |
|------------|----------|-----------------|-------------|
| *None* | – | – | – |

*All processing is performed within the deploying organisation’s infrastructure (MySQL/MariaDB, optional Elasticsearch, file‑system storage).*

---

## 6. Security Measures  

**Positive controls (implemented):**  

- **Authentication & Password Protection** – User passwords are hashed with BCrypt (`BCryptPasswordEncoder`).  
- **Authorization** – Method‑level security via `@Authorized` AOP annotations; only users with `View Visits`, `Edit Visits`, etc., can access the service.  
- **Transport Security** – All HTTP endpoints are expected to be served over TLS 1.2+ (HTTPS).  
- **Database Security** – MySQL/MariaDB runs with least‑privilege accounts; tables are protected by role‑based grants.  
- **Hibernate Envers Auditing** – Immutable audit trail of every change to `Visit` records.  
- **Spring Transaction Management** – Guarantees atomicity of create/update operations.  
- **Input Validation** – Bean Validation (`@NotNull`, `@PastOrPresent`) on `Visit` fields.  
- **File‑system protection** – Sensitive configuration files (e.g., `openmrs-runtime.properties`) are stored with OS‑level permissions (600).  

**Remaining concerns / mitigations:**  

- **Encryption at rest** – MySQL data‑at‑rest encryption is recommended (e.g., InnoDB tablespace encryption).  
- **Backup security** – Backups must be encrypted and stored securely; access limited to backup administrators.  
- **Logging** – Ensure audit logs do not expose full patient identifiers in clear text; mask where appropriate.  
- **Elasticsearch exposure** – If enabled, restrict access to the ES cluster via firewall/VPN and enable TLS.

---

## 7. Cross‑Border Transfers  

- **Current state:** No data is transferred outside the country of deployment. All components (MySQL, Elasticsearch, file system) reside on servers located within the EU/EEA (or the jurisdiction of the health‑care provider).  
- **Legal mechanism (if future transfer needed):** Standard Contractual Clauses (SCCs) or an adequacy decision would be used, together with a supplementary controller‑processor agreement that includes GDPR Art. 9‑specific clauses.

---

## 8. DPIA Trigger Assessment  

| **Factor** | **Description** | **Risk Level** |
|------------|-----------------|----------------|
| **Data Subject Rights** | Patients can request access, rectification, or erasure of their visit data. | **High** |
| **Data Sensitivity** | Visits are linked to health‑care episodes → special‑category data (Art. 9). | **High** |
| **Systematic Monitoring** | Continuous creation of visit records for every patient interaction. | **High** |
| **Innovative Technology** | Use of Elasticsearch for search indexing (adds a technical layer). | **Medium** |
| **Scale of Processing** | Potentially thousands of visits per day in large facilities. | **Medium** |
| **Retention Period** | Long‑term retention (≥10 years) increases exposure risk. | **Medium** |

**Overall DPIA Recommendation:** **A Data Protection Impact Assessment is required** before the system goes live (or when the feature is first enabled). The DPIA should address:  

- Lawful basis verification (Art. 9(2)(h)).  
- Detailed retention schedule and deletion procedures.  
- Encryption‑at‑rest and backup protection.  
- Access‑control review (role‑based permissions, audit log monitoring).  
- Impact of optional Elasticsearch indexing on data confidentiality.

--- 

*Document prepared on 2026‑05‑11 for the OpenMRS Core “Visit Tracking” processing activity.*