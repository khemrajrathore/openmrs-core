# Processing Activity: Diagnosis & Condition Recording (PA‑004)

## 1. Overview
| Field | Value |
|---|---|
| **PA Name** | Diagnosis & Condition Recording (PA‑004) |
| **Business Function** | Capture, store, and retrieve patient diagnoses and chronic conditions as part of the clinical record. |
| **Processing Purpose** | To enable clinicians to document, view, and analyse a patient’s diagnoses and conditions for safe, effective, and continuous healthcare delivery. |
| **Legal Basis** | **GDPR Art. 9(2)(h)** – processing is necessary for the provision of health or social care services. |
| **Controller** | OpenMRS Core project (maintained by the OpenMRS community; the legal controller is the health‑care organisation that deploys the OpenMRS instance). |
| **Processor (if any)** | None declared by the core code; infrastructure providers (e.g., MySQL, Elasticsearch, cloud‑host) act as sub‑processors under the controller’s contracts. |

## 2. Data Subjects
| Category | Description |
|---|---|
| **Patients** | Primary data subjects; their health information is recorded as diagnoses and conditions. |
| **Healthcare providers** | Appear in audit logs (e.g., “created by” fields) and may be referenced in the `Diagnosis` entity (e.g., encounter‑linked provider). |
| **System administrators** | Appear in system‑level audit logs (e.g., who voided a record). |

## 3. Personal Data Elements
| Data Element | Source (Domain Class / Table) | Sensitivity* | Retention (Typical) |
|---|---|---|---|
| Diagnosis ID | `Diagnosis` entity → `diagnosis` table (PK) | High (identifiable health record) | Retained as long as the patient record is kept (minimum 10 years after last contact, per local health‑care regulations). |
| Condition ID | `Condition` entity → `condition` table (PK) | High | Same as Diagnosis ID. |
| Patient ID | `Patient` entity → `patient` table (FK) | High | Same as Diagnosis/Condition. |
| Concept (diagnosis) | `Concept` entity (referenced by `Diagnosis`/`Condition`) → `concept` table | High (clinical concept linked to a patient) | Same as Diagnosis/Condition. |
| Onset Date | `Condition` entity → `condition` table (column `onset_date`) | Medium (date of health event) | Same as Diagnosis/Condition. |
| Status | `Condition`/`Diagnosis` entity → `condition`/`diagnosis` table (column `status`) | Medium (e.g., active, resolved) | Same as Diagnosis/Condition. |

\*Sensitivity classification follows GDPR guidance: **High** = special‑category health data; **Medium** = health‑related data that is still personal but less granular.

## 4. Data Flows
| Step | Description | Technical Artefacts |
|---|---|---|
| **Ingress – UI / REST** | Clinician enters a diagnosis/condition via the web UI or REST API. | `webapp` → Spring MVC controller → `ConditionService` / `DiagnosisService`. |
| **Ingress – HL7 inbound** | External systems can push diagnoses via HL7 messages parsed by `HL7Service`. | `HL7Service` → `ConditionService` / `DiagnosisService`. |
| **Processing** | Service layer validates, applies business rules, and persists the entity. | Spring AOP `@Authorized`, Spring transactions, Hibernate ORM, Hibernate Envers (audit). |
| **Storage** | Persistent storage in MySQL/MariaDB. | Tables: `condition`, `diagnosis`, `patient`, `concept`. |
| **Search Index** | Data is indexed for fast lookup via Hibernate Search (Lucene or Elasticsearch backend). | `Hibernate Search` → Elasticsearch (optional) or embedded Lucene index. |
| **Egress – UI / REST** | Clinicians retrieve diagnoses/conditions for display or reporting. | Controllers → Service → DAO → DB → JSON/XML response. |
| **Egress – HL7 outbound** | Diagnoses can be exported to external systems. | `HL7Service` → HL7 message generation. |
| **Audit / Logging** | All create/update/void actions are logged and versioned. | Hibernate Envers tables (`condition_aud`, `diagnosis_aud`), `log_event` table, SLF4J/Logback. |

## 5. Third Parties / Processors
| Vendor / Platform | Role | Data Shared | Hosting / Location |
|---|---|---|---|
| **MySQL / MariaDB** | Database engine (sub‑processor) | Full patient‑diagnosis data (encrypted at rest) | On‑premises or cloud (as configured by the controller). |
| **Elasticsearch (optional)** | Search index (sub‑processor) | Indexed copy of diagnosis/condition data (same fields as stored) | On‑premises or cloud (as configured). |
| **Cloud / Hosting Provider** (e.g., AWS, Azure) | Infrastructure (sub‑processor) | All system data (including health data) | Data centre location defined by the controller’s contract. |
| **OpenMRS Community** | Development & support (processor only when the controller outsources support) | May receive anonymised logs for debugging (no PHI) | N/A (only if a support contract exists). |

*No direct third‑party data sharing is defined in the core code; all external interactions are limited to the infrastructure components listed above.*

## 6. Security Measures
| Control | Implementation Detail |
|---|---|
| **Encryption at Rest** | MySQL/MariaDB tables encrypted via InnoDB tablespace encryption; optional disk‑level encryption (LUKS). |
| **Encryption in Transit** | All HTTP endpoints exposed over TLS 1.2+; HL7 over MLLP with TLS when configured. |
| **Access Control** | `@Authorized` annotations enforce role‑based permissions (e.g., `ROLE_CLINICIAN`, `ROLE_ADMIN`). |
| **Authentication** | Passwords stored using bcrypt (Spring Security `PasswordEncoder`). |
| **Auditing** | Hibernate Envers automatically creates audit tables (`*_aud`) for every change; OpenMRS `log_event` captures user actions. |
| **Transaction Management** | Spring `@Transactional` ensures atomic writes; rollback on validation failures. |
| **Input Validation** | Bean Validation (`@NotNull`, `@PastOrPresent` for dates) prevents malformed data. |
| **Secure Configuration** | Default passwords disabled; secret keys stored in environment variables; Tomcat configured with HTTPOnly and Secure cookies. |
| **Backup & Recovery** | Encrypted backups of MySQL and Elasticsearch indices; regular restore testing. |
| **Monitoring & Alerting** | Logback + ELK stack monitors for suspicious activity; alerts on failed login attempts. |

## 7. Cross‑Border Transfers
- **Potential Transfer**: If the MySQL or Elasticsearch instance is hosted in a cloud region outside the EU/EEA, data will cross borders.
- **Safeguards**: Standard Contractual Clauses (SCCs) or Binding Corporate Rules (BCRs) are required. The controller must document the exact location of each storage component and ensure adequacy decisions or appropriate safeguards are in place.

## 8. DPIA Trigger Assessment
| Factor | Applicable? | Comments |
|---|---|---|
| **Large‑scale processing** | Yes | Potentially thousands of diagnoses per month in a national deployment. |
| **Special‑category (sensitive) data** | Yes | Health data falls under GDPR Art. 9. |
| **Automated decision‑making** | No | No profiling or automated decisions are performed on this data. |
| **Systematic monitoring** | Yes | Continuous recording of health status for each patient. |
| **Vulnerable data subjects** | Yes | Patients are considered vulnerable under GDPR Recital 34. |
| **Use of new technologies** | Potentially (e.g., Elasticsearch, Hibernate Search) | Requires assessment of search‑index exposure. |
| **Cross‑border transfers** | Potentially (depends on hosting) | Triggers additional DPIA considerations. |
| **Overall DPIA Recommendation** | **Yes** | Given the processing of special‑category data at scale, a Data Protection Impact Assessment is mandatory. |

---

### References to OpenMRS Code & Architecture
- **Domain classes**: `org.openmrs.Condition`, `org.openmrs.Diagnosis` (persisted in `condition` and `diagnosis` tables).  
- **Service interfaces**: `org.openmrs.api.ConditionService`, `org.openmrs.api.DiagnosisService`.  
- **Persistence**: Hibernate 5 ORM with Envers for audit; Liquibase migrations create the tables.  
- **Search**: Hibernate Search (Lucene default, optional Elasticsearch backend) indexes `Condition` and `Diagnosis` for fast lookup.  
- **Ingress/Egress**: Defined in `webapp/src/main/webapp/WEB-INF/web.xml` (servlet mapping) and REST controllers under `api/src/main/java/org/openmrs/api/rest/…`.  
- **Security**: `@Authorized` AOP, Spring Security, password hashing (`BCryptPasswordEncoder`), transaction management (`@Transactional`).  

---  

**Conclusion**: The Diagnosis & Condition Recording activity processes high‑sensitivity health data at a scale that triggers a DPIA. All required GDPR elements (legal basis, data subject rights, security, cross‑border safeguards) are documented, and the technical implementation in OpenMRS Core provides robust controls aligned with best‑practice privacy‑by‑design.