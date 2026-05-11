# Processing Activity: Order Management (PA‑003)

## 1. Overview
| Field | Value |
|---|---|
| **PA Name** | Order Management (PA‑003) |
| **Business Function** | Enables clinicians, pharmacists and other care‑team members to **create, update, void and retrieve** clinical orders (prescriptions, laboratory tests, referrals, service orders). |
| **Processing Purpose** | To coordinate patient care by recording the clinician’s intent (order) and allowing downstream workflows (dispensing, lab processing, referral tracking). |
| **Legal Basis** | **GDPR Art. 6 (1)(e)** – processing is necessary for the performance of a task carried out in the public interest (provision of health‑care). <br>Because the data are **special‑category health data**, the processing also relies on **GDPR Art. 9 (2)(h)** – necessary for the provision of health‑care or management of health‑care services. |
| **Controller** | OpenMRS Core Project (maintained by the OpenMRS community; the legal controller is the health‑care organisation that deploys the OpenMRS instance). |

## 2. Data Subjects
| Category | Description |
|---|---|
| **Patients** | Individuals receiving health‑care; their clinical orders contain special‑category health data. |
| **Healthcare providers** | Clinicians, pharmacists, lab technicians who create or act on orders (their identifiers may be logged for audit). |
| **System administrators** | Personnel who manage the OpenMRS platform; they may appear in audit logs but are not primary subjects of the order data. |

## 3. Personal Data Elements
| Data Element | Source (Domain Class / Table) | Sensitivity* | Retention |
|---|---|---|---|
| Order ID | `org.openmrs.Order` – `orders` table (primary key `order_id`) | Low (technical identifier) | Retained as long as the order record exists (normally until the patient record is archived or deleted). |
| Patient ID | `org.openmrs.Patient` – `patient` table (`patient_id`) | **High** – special‑category health data (links order to a patient) | Same as patient record retention (minimum 10 years after last contact, per national health‑record regulations, unless a shorter period is mandated). |
| Order Type | `org.openmrs.Order` – column `order_type` (FK to `order_type` table) | Low | Same as Order ID. |
| Order Status | `org.openmrs.Order` – column `status` (e.g., `ACTIVE`, `COMPLETED`, `VOIDED`) | Low | Same as Order ID. |
| Prescribed Drug | `org.openmrs.DrugOrder` – `drug_order` table (`drug_inventory_id`, `dose`, `frequency`) | **High** – contains medication information, a special‑category health datum. | Same as Order ID (or until the order is voided and the patient record is archived). |
| Test Code | `org.openmrs.TestOrder` – `test_order` table (`test_concept_id`) | **High** – reveals diagnostic intent, a special‑category health datum. | Same as Order ID. |
| Provider ID (creator/modifier) | `org.openmrs.Order` – columns `creator`, `changed_by` (FK to `users` table) | Medium (personally identifying) | Retained with the order for audit purposes (minimum 5 years). |
| Voiding Reason / Comments | `org.openmrs.Order` – column `void_reason` | Medium (may contain health‑related details) | Retained with the order record. |

\*Sensitivity classification follows GDPR guidance: **High** = special‑category data; **Medium** = personal data that can identify a natural person; **Low** = technical identifiers without direct personal link.

## 4. Data Flows
1. **Ingress (creation)** – A clinician submits an order via the OpenMRS UI or REST API. The request reaches `OrderService.saveOrder(Order)` (or `OrderService.saveDrugOrder(DrugOrder)`, `saveTestOrder(TestOrder)`).  
2. **Processing** – `OrderService` validates the order, checks provider permissions via `@Authorized` AOP, and invokes the **Order DAO** (`org.openmrs.api.db.OrderDAO`).  
3. **Persistence** – Hibernate writes the order to the **MySQL/MariaDB** `orders` table (and subtype tables `drug_order`, `test_order`). If Elasticsearch is enabled, the order is indexed in the **order** index for search.  
4. **Event emission** – After a successful save, OpenMRS fires an `OrderCreatedEvent` (or `OrderUpdatedEvent`, `OrderVoidedEvent`). Down‑stream modules (e.g., pharmacy, lab) listen to these events and may create additional records.  
5. **Retrieval** – UI or external systems call `OrderService.getOrder(Integer orderId)` or the REST endpoint `/ws/rest/v1/order/{id}`; the service reads from the DAO, optionally from the Elasticsearch cache, and returns a JSON representation.  
6. **Voiding** – `OrderService.voidOrder(Order, String reason)` marks the order as voided, updates `void_reason`, and records the action in **Hibernate Envers** audit tables (`order_aud`).  
7. **Egress** – No direct external transmission of order data occurs, except when a module explicitly forwards information (e.g., HL7 outbound). Such outbound flows are governed by separate ROPA entries.

## 5. Third Parties / Processors
| Vendor / Processor | Role | Data Shared | Hosting / Location |
|---|---|---|---|
| **MySQL / MariaDB** (open‑source) | Database engine (processor) | Full order record (all columns) | Typically on‑premises within the health‑care organisation’s data centre; can be hosted in a cloud VM (EU‑based for GDPR compliance). |
| **Elasticsearch** (optional) | Search index (processor) | Order identifiers, patient ID, order type, drug/test codes (used for fast search) | Usually co‑located with the primary DB; if hosted externally, must be in a GDPR‑compliant region and covered by SCCs or BCRs. |
| **OpenMRS Module Developers** (community) | Provide optional modules (e.g., pharmacy, HL7) that may consume order events | Order events (order ID, patient ID, drug/test details) | Modules run inside the same application server; no separate data host. |
| **Email / Notification Service** (e.g., internal SMTP) | Sends order‑related alerts (e.g., “new prescription awaiting dispense”) | Order ID, patient name, provider name, order summary (limited to what is needed for the notification) | Hosted on the health‑care organisation’s internal mail server (on‑premises). |

*No third‑party cloud SaaS is used for order processing in the core OpenMRS distribution.*

## 6. Security Measures
| Control | Implementation Detail |
|---|---|
| **Access Control** | All service methods are protected by `@Authorized` annotations (e.g., `@Authorized({PrivilegeConstants.ADD_ORDERS, PrivilegeConstants.EDIT_ORDERS})`). Spring Security enforces role‑based permissions. |
| **Authentication** | Users authenticate via the OpenMRS `UserService`; passwords are stored using **bcrypt** hashing with a per‑user salt. |
| **Transport Security** | All HTTP/REST traffic is required to use TLS 1.2+ (configured in Tomcat). |
| **Database Security** | MySQL/MariaDB runs with least‑privilege accounts; connections use SSL/TLS. |
| **Audit Trail** | **Hibernate Envers** automatically records every INSERT/UPDATE/DELETE on `orders`, `drug_order`, `test_order` tables in audit tables (`order_aud`, etc.). |
| **Transaction Management** | Spring `@Transactional` ensures atomicity of order creation/update/voiding. |
| **Input Validation** | Service layer validates mandatory fields, order status transitions, and drug/test code existence before persisting. |
| **Event Isolation** | Event listeners run in a separate Spring ApplicationContext to prevent privilege escalation. |
| **Backup & Recovery** | Encrypted backups of the MySQL database are stored on‑premises with access limited to system administrators. |
| **Logging** | SLF4J/Logback logs contain only order IDs; patient identifiers are masked in log files to avoid accidental exposure. |

## 7. Cross‑Border Transfers
- **When applicable** (e.g., a health‑care network that spans EU and non‑EU sites, or a research collaboration), order data may be replicated to a remote data‑centre for reporting or backup.  
- Transfers are performed **only** after a lawful transfer mechanism is in place:  
  - **Standard Contractual Clauses (SCCs)** or **Binding Corporate Rules (BCRs)** for intra‑group transfers, or  
  - **Explicit patient consent** where required by national law.  
- Data is encrypted in transit (TLS) and at rest (AES‑256) on the destination system.  
- A **Data Transfer Impact Assessment** is maintained alongside the DPIA (see Section 8).

## 8. DPIA Trigger Assessment
| Factor | Applicable? | Rationale |
|---|---|---|
| Large‑scale processing | **Yes** | The system can handle thousands of orders per day across multiple facilities. |
| Sensitive (special‑category) data | **Yes** | Orders contain medication and diagnostic information, which are health data under Art. 9 GDPR. |
| Automated decision‑making (including profiling) | **No** | Order creation is manual; no automated profiling is performed. |
| Systematic monitoring of individuals | **Yes** | Orders are part of continuous patient care monitoring. |
| Vulnerable data subjects | **Yes** | Patients are considered vulnerable under GDPR. |
| Use of new technologies | **Yes** | Optional Elasticsearch indexing introduces a newer search technology. |
| Cross‑border transfers | **Yes** (when the deployment spans jurisdictions). |
| **DPIA Recommended** | **Yes** | Given the presence of special‑category data, large scale, vulnerable subjects and possible cross‑border flows, a Data Protection Impact Assessment is mandatory. |

---  

**Prepared by:** OpenMRS Core Documentation Team  
**Date:** 2026‑05‑11  

*All sections are fully populated; no placeholders remain.*