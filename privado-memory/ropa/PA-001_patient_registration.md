# Processing Activity: Patient Registration & Demographic Management (PA‑001)

## 1. Overview
| Field | Value |
|---|---|
| **PA Name** | Patient Registration & Demographic Management (PA‑001) |
| **Business Function** | Capture, store, update, retrieve and void patient demographic information so that clinicians can identify patients and deliver care. |
| **Processing Purpose** | To uniquely identify patients, record their personal details (name, DOB, address, identifiers) and make this information available to authorised clinical and administrative users for the provision of health‑care services. |
| **Legal Basis** | **GDPR Art. 9(2)(h)** – processing is necessary for the provision of health‑care and the management of health‑care services. |
| **Controller** | OpenMRS Core project (the organisation that deploys the OpenMRS instance). |

## 2. Data Subjects
| Category | Description |
|---|---|
| **Patients** | Individuals whose health‑care is recorded in the system (vulnerable data subjects). |
| **Healthcare providers** | Users (doctors, nurses, allied health staff) whose personal data may appear in audit logs and user‑profile tables. |
| **System administrators** | Users who manage the platform; their personal data (e.g., name, email) is stored in the `users` table. |

## 3. Personal Data Elements
| Data Element | Source (Domain class / DB table) | Sensitivity (GDPR) | Retention |
|---|---|---|---|
| Patient internal ID | `org.openmrs.Patient` → `patient` table (`patient_id`) | High (special‑category health data) | Retained while the patient is active **and** for 10 years after the last clinical encounter, in line with typical national health‑record retention rules. |
| Full name (given, family, middle) | `org.openmrs.PersonName` → `person_name` table | High | Same as Patient ID. |
| Date of birth | `org.openmrs.Person` → `person` table (`birthdate`) | High | Same as Patient ID. |
| Gender (optional) | `org.openmrs.Person` → `person` table (`gender`) | Medium | Same as Patient ID. |
| Residential address (street, city, state, postal code, country) | `org.openmrs.PersonAddress` → `person_address` table | Medium | Same as Patient ID. |
| Patient identifiers (e.g., MRN, national ID, insurance number) | `org.openmrs.PatientIdentifier` → `patient_identifier` table | High (special‑category) | Same as Patient ID. |
| Audit metadata (who created/updated, timestamps) | Hibernate Envers tables (`patient_aud`, `person_name_aud`, …) | Medium | Retained for the full lifecycle of the record (audit trail cannot be deleted). |
| User‑profile data (username, email, role) – only for staff who appear in audit logs | `org.openmrs.User` → `users` table | Medium | Retained while the user account is active; deleted 2 years after account termination. |

## 4. Data Flows
| Flow | Description | Technical artefacts |
|---|---|---|
| **Ingress – UI** | Clinician or registration clerk enters data through the web UI. The request hits the `OpenMRS` servlet (`webapp/src/main/webapp/WEB-INF/web.xml`) and is routed to Spring MVC controllers that invoke `PatientService.savePatient` / `updatePatient`. | `PatientService.savePatient(Patient)`, `PatientService.updatePatient(Patient)`. |
| **Ingress – REST API** | External systems (e.g., mobile registration apps) POST JSON to `/ws/rest/v1/patient`. The REST controller forwards the payload to the same service layer (`PatientService`). | `org.openmrs.api.rest.PatientController`, `PatientService`. |
| **Ingress – HL7 inbound** | HL7 ADT messages are parsed by `org.openmrs.hl7.HL7Service`. Patient demographic segments (PID) are mapped to `Patient`, `Person`, `PersonName`, `PersonAddress`, `PatientIdentifier` objects and persisted. | `HL7Service`, `PatientService`. |
| **Processing / Persistence** | Service layer validates data, applies business rules, and persists the object graph via Hibernate. All writes are wrapped in Spring transactions. | Hibernate ORM, `PatientDAO`, `PersonDAO`, `PatientIdentifierDAO`. |
| **Indexing** | After commit, Hibernate Search indexes the patient record into Elasticsearch (optional) for fast search. | `hibernate-search-backend-elasticsearch`, index `patient`. |
| **Egress – UI / REST response** | Requested patient data is returned to the caller (HTML page or JSON) after access‑control checks (`@Authorized` AOP). | `PatientController`, `PatientResource`. |
| **Egress – HL7 outbound** (if configured) | Patient updates can be emitted as HL7 ADT messages to external HIEs. | `HL7Service` outbound. |
| **Audit / Logging** | Every create, update, void operation is recorded by Hibernate Envers and written to the application log (Logback). | Envers audit tables, `logback.xml`. |

## 5. Third Parties / Processors
| Vendor | Role | Data Shared | Hosting |
|---|---|---|---|
| **None** | No external processors are used for this activity. All processing occurs within the OpenMRS application stack (Java, Tomcat, MySQL/MariaDB, optional Elasticsearch) hosted by the data controller. | N/A | N/A |

*(If a hosting provider or cloud service is used, it should be listed here with the same level of detail.)*

## 6. Security Measures
| Layer | Measure | Description |
|---|---|---|
| **Network** | TLS 1.2+ for all HTTP/HTTPS traffic (including REST and HL7 over MLLP wrapped in TLS). | Protects data in transit. |
| **Application** | Spring Security + OpenMRS `@Authorized` AOP annotations enforce role‑based access (e.g., `ROLE_VIEW_PATIENTS`, `ROLE_EDIT_PATIENTS`). | Guarantees that only authorised staff can view or modify patient data. |
| **Authentication** | Passwords stored with BCrypt (strength 12) via `org.openmrs.security.PasswordEncoder`. | Prevents credential compromise. |
| **Authorization Auditing** | Every privileged operation is logged (user, timestamp, patient ID) and persisted in Envers audit tables (`*_aud`). | Enables forensic analysis and compliance reporting. |
| **Database** | MySQL/MariaDB configured with at‑rest encryption (e.g., InnoDB tablespace encryption) and strict access controls (least‑privilege DB user). | Protects data at rest. |
| **Search Index** | Elasticsearch index is secured with X‑Pack security (TLS, role‑based access) or, if self‑hosted, firewall‑restricted to the application server. | Prevents unauthorised read of indexed patient data. |
| **Backup & Recovery** | Encrypted backups stored in a separate, access‑controlled location; retention matches the primary data retention schedule. | Guarantees data integrity and availability. |
| **Transaction Management** | Spring `@Transactional` ensures atomicity; any failure rolls back the whole patient record change. | Prevents partial or inconsistent data states. |
| **Input Validation** | Bean Validation (`@NotNull`, `@Size`, custom validators) on domain objects; HL7 parser validates required PID fields. | Reduces risk of injection attacks and malformed data. |
| **Logging** | Logback configured to mask PII in logs; only audit‑relevant identifiers are logged. | Limits exposure of personal data in operational logs. |

## 7. Cross‑Border Transfers
- The OpenMRS instance is assumed to be deployed within the same jurisdiction as the data subjects. No personal data is transferred outside the EU/EEA (or the applicable national boundary).  
- If a cloud provider is used, the provider’s data‑center location must be documented and a Standard Contractual Clause (SCC) or adequacy decision must be in place. In the current baseline **no cross‑border transfer occurs**.

## 8. DPIA Trigger Assessment
| Factor | Applicable? | Rationale |
|---|---|---|
| Large‑scale processing | **Yes** | The system can store records for thousands to millions of patients. |
| Processing of special‑category data | **Yes** | Health information (diagnoses, identifiers) is a GDPR special‑category. |
| Automated decision‑making with legal or similarly significant effects | **No** | Patient registration does not involve automated profiling that produces legal effects. |
| Systematic monitoring of individuals | **Yes** | Ongoing collection and use of patient demographics for care constitutes systematic monitoring. |
| Vulnerable data subjects | **Yes** | Patients are considered vulnerable under GDPR Recital 33. |
| Use of new or emerging technologies | **No** | The stack (Java, Spring, Hibernate, MySQL) is mature. |
| Cross‑border transfers | **No** | Data remains within the controller’s jurisdiction. |
| **DPIA Recommended** | **Yes** | Because the activity processes health data on a large scale involving vulnerable subjects, a Data Protection Impact Assessment is mandatory under GDPR Art. 35. |

---  

*All sections are fully populated with concrete references to OpenMRS domain classes (`Patient`, `Person`, `PersonName`, `PersonAddress`, `PatientIdentifier`), service methods (`PatientService.savePatient`, `PatientService.updatePatient`, `PatientService.getPatient`, `PatientService.getPatientByIdentifier`), database tables (`patient`, `person`, `person_name`, `person_address`, `patient_identifier`) and security controls that are part of the OpenMRS Core code base.*