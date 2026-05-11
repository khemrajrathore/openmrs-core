# Processing Activity: Clinical Data Collection (PA‑002)

## 1. Overview
| Field | Value |
|---|---|
| **PA Name** | Clinical Data Collection (PA‑002) |
| **Business Function** | Capture and store clinical data generated during patient encounters (observations, orders, encounter metadata). |
| **Processing Purpose** | To maintain a complete, searchable electronic medical record that supports clinical care, reporting, and continuity of care in resource‑constrained health‑care settings. |
| **Legal Basis** | **GDPR Art. 9(2)(h)** – processing is necessary for the provision of health or social care services and the management of health‑care systems. |
| **Controller** | OpenMRS Core Project (maintained by the OpenMRS community; the legal controller is the health‑care organisation that deploys the OpenMRS instance). |

## 2. Data Subjects
| Category | Description |
|---|---|
| **Patients** | Individuals receiving health‑care services (vulnerable data subjects). |
| **Healthcare Providers** | Clinicians, nurses, pharmacists, and other staff who create or view encounter data. |
| **System Administrators** | Personnel who manage the OpenMRS platform (access to audit logs, configuration, but not to clinical content unless required for support). |

## 3. Personal Data Elements
| Data Element | Source (Domain Class / Table) | Sensitivity (GDPR) | Retention (Typical) |
|---|---|---|---|
| Encounter ID | `org.openmrs.Encounter` → `encounter` table (PK `encounter_id`) | Medium (identifiable) | Retained as long as the patient record is kept (often ≥ 10 years, per national health‑record statutes). |
| Patient ID | `org.openmrs.Patient` → `patient` table (PK `patient_id`) | High (special‑category health data) | Same as patient record retention. |
| Encounter Date/Time | `Encounter.encounterDatetime` → `encounter` table (`encounter_datetime`) | Medium | Same as patient record retention. |
| Obs ID | `org.openmrs.Obs` → `obs` table (PK `obs_id`) | Medium | Same as patient record retention. |
| Observation Value | `Obs.value*` fields (e.g., `value_text`, `value_numeric`, `value_coded`) → `obs` table | High (clinical measurement) | Same as patient record retention. |
| Order ID | `org.openmrs.Order` → `orders` table (PK `order_id`) | Medium | Same as patient record retention. |

## 4. Data Flows
1. **Ingress** – Data entered via:  
   * **Web UI** (HTML/JSP forms) → `EncounterService.saveEncounter(...)` / `ObsService.saveObs(...)`.  
   * **REST API** (`/ws/rest/v1/encounter`, `/ws/rest/v1/obs`) → same service methods.  
   * **Scheduled Jobs** (e.g., batch imports) → internal calls to the services.  

2. **Processing & Persistence** –  
   * `EncounterService` validates the `Encounter` object, applies `@Authorized` checks, and persists it via `EncounterDAO` → `encounter` table (MySQL/MariaDB).  
   * `ObsService` validates each `Obs`, links it to the parent `Encounter`, and persists via `ObsDAO` → `obs` table.  
   * `OrderService` (used indirectly when an order is attached to an encounter) persists to `orders` table.  

3. **Indexing** – After commit, Hibernate Search triggers indexing of the persisted entities into **Elasticsearch** (or Lucene) for fast retrieval (`EncounterDocument`, `ObsDocument`).  

4. **Egress** – Data is returned to callers via REST responses, UI pages, or exported HL7 messages; no external third‑party recipients are involved in this PA.  

```
UI / REST  →  EncounterService / ObsService  →  MySQL/MariaDB (encounter, obs, orders)  →  Elasticsearch/Lucene (search index)
```

## 5. Third Parties / Processors
| Vendor / Processor | Role | Data Shared | Hosting |
|---|---|---|---|
| **MySQL / MariaDB** (open‑source RDBMS) | Database engine (processor) | Full clinical dataset (all tables listed above) | On‑premises or cloud VM owned by the controller. |
| **Elasticsearch** (optional) | Search index provider (processor) | Indexed copies of encounter, obs, and order data (no raw PHI beyond what is indexed) | On‑premises or cloud VM owned by the controller. |
| **OpenMRS Community** (software maintainer) | Platform development & support (processor for bug‑fixes, updates) | Source code only; no patient data is transferred. | N/A – no data transfer. |

*No external SaaS or data‑sharing agreements are used for this specific processing activity.*

## 6. Security Measures
| Control | Implementation Detail |
|---|---|
| **Access Control** | `@Authorized` AOP annotations on service methods; role‑based permissions (`privilege.view.encounters`, `privilege.edit.obs`, etc.). |
| **Authentication** | Spring Security with BCrypt password hashing for user credentials (`User.password`). |
| **Transport Security** | All HTTP traffic is required to use TLS 1.2+ (HTTPS). |
| **Data-at‑Rest Encryption** | Optional DB‑level encryption (e.g., MySQL InnoDB tablespace encryption) configured by the deploying organisation. |
| **Audit Logging** | Hibernate Envers tracks every change to `Encounter`, `Obs`, and `Order` entities (audit tables `encounter_audit`, `obs_audit`, `orders_audit`). |
| **Transaction Management** | Spring `@Transactional` ensures atomic writes; rollback on validation or security failures. |
| **Search Index Protection** | Elasticsearch is secured with HTTP basic auth / TLS; index data mirrors the DB and is not exposed publicly. |
| **Backup & Recovery** | Regular encrypted backups of MySQL and Elasticsearch snapshots stored in a secure, access‑controlled location. |
| **Server Hardening** | Tomcat 9 with security manager, OS‑level firewalls, SELinux/AppArmor profiles. |

## 7. Cross‑Border Transfers
No cross‑border transfers are performed by this processing activity. All data stores (MySQL, Elasticsearch) are hosted within the same jurisdiction as the controller, unless the deploying organisation explicitly provisions cloud resources in another country – in which case a separate assessment is required.

## 8. DPIA Trigger Assessment
| Factor | Applicable? | Rationale |
|---|---|---|
| Large‑scale processing | **Yes** | Potentially thousands of encounters per month across multiple facilities. |
| Processing of special‑category data | **Yes** | Health observations and diagnoses are GDPR Art. 9 data. |
| Automated decision‑making (significant effect) | **No** | No profiling or automated treatment decisions are made by this core module. |
| Systematic monitoring of individuals | **Yes** | Continuous collection of clinical data for each patient. |
| Vulnerable data subjects | **Yes** | Patients are considered vulnerable under GDPR. |
| Use of new or emerging technologies | **No** | Uses mature Java/Spring/Hibernate stack. |
| Cross‑border transfers | **No** | Data stays within the controller’s jurisdiction. |
| **DPIA Recommended** | **Yes** | Because the activity processes large volumes of high‑sensitivity health data of vulnerable individuals, a DPIA is mandatory under GDPR Art. 35. |

**Conclusion:** A Data Protection Impact Assessment must be carried out before deployment (or when any significant change to the processing, technology stack, or data volume occurs). The DPIA should address risk mitigation for unauthorized access, data integrity, and retention compliance.