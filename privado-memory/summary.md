# OpenMRS Core – Executive Audit Summary  

**Date:** 2026‑05‑08 **Repository:** <https://github.com/khemrajrathore/openmrs-core.git> **Branch:** master  

---

## 1. Executive Overview  

OpenMRS Core is an open‑source, Java‑based Electronic Medical Record (EMR) platform. It supplies the foundational data model, services, and web UI that enable health‑care programs to capture, store, and report clinical, administrative, and demographic information. The codebase is organized as a Maven multi‑module project and is built on a layered, service‑oriented architecture that separates domain objects, DAO implementations, business services, and UI/REST controllers.

---

## 2. Architecture & Tech Stack  

| Layer | Technology | Version / Reference | Primary Packages / Paths |
|-------|------------|---------------------|--------------------------|
| **Language** | Java | Java 8 (source/target set in parent POM) | `src/main/java/**/*.java` |
| **Build** | Maven | 4.0.0 POM model | `pom.xml`, `api/pom.xml`, `web/pom.xml`, … |
| **DI / MVC** | Spring Framework | `${springVersion}` (typically 5.x) | `org.springframework.*` |
| **ORM** | Hibernate | 5.x (via `org.hibernate:hibernate-core`) | `org.openmrs.api.db.hibernate.*` |
| **Migrations** | Liquibase | 4.x (custom extensions) | `org.openmrs.liquibase.*` |
| **Logging** | SLF4J / Logback (custom `Slf4JLogService`) | – | `org.openmrs.logging.*` |
| **Web UI** | Spring MVC, JSP, Velocity (1.7) | – | `web/src/main/java/org/openmrs/web/*`, `webapp/src/main/webapp/WEB-INF/*.jsp` |
| **Container** | Servlet API 4.0.1 (provided) | – | `javax.servlet.*` |
| **Clinical Standards** | HL7 v2.x, optional FHIR modules | – | `org.openmrs.hl7.*`, `org.openmrs.module.fhir2.*` |
| **Utility Libraries** | Apache Commons IO 2.19.0, Commons Lang3 3.17.0, Commons Collections 3.2.2 | – | `org.apache.commons.*` |
| **Search / Index** | Hibernate Search (Lucene / Elasticsearch) | – | `org.hibernate.search.*` |

### Modular Maven Structure  

- **Parent POM** aggregates six modules: `tools`, `test`, `api`, `web`, `webapp`, `liquibase`.  
- Each module can be built/published independently, allowing fine‑grained versioning.

### Layered Service‑Oriented Design  

1. **Domain Model** – POJOs under `api/src/main/java/org/openmrs/` (e.g., `Patient`, `Encounter`, `Obs`).  
2. **DAO Layer** – Interfaces in `org.openmrs.api.db.*`; Hibernate implementations in `org.openmrs.api.db.hibernate.*`.  
3. **Service Layer** – Interfaces in `org.openmrs.api.*` (e.g., `PatientService`, `EncounterService`) with implementations in `org.openmrs.api.impl.*`.  
4. **Handler / Save‑Handler Pattern** – Centralised business‑logic hooks (e.g., `BaseVoidHandler`, `PersonSaveHandler`) located in `api/src/main/java/org/openmrs/api/handler/`.  

---

## 3. Core Business Features  

| Business Activity | Primary Domain Model(s) | Service Layer (API) | Key Supporting Classes |
|-------------------|--------------------------|---------------------|------------------------|
| Patient registration | `Patient` (extends `Person`) | `PatientService` | `PatientIdentifier`, `PersonName`, `PersonAddress` |
| Encounter management | `Encounter` | `EncounterService` | `EncounterProvider`, `Location` |
| Observation capture | `Obs` | `ObsService` | `Concept`, `ConceptAnswer`, `ConceptDatatype` |
| Orders & prescriptions | `Order`, `DrugOrder`, `TestOrder`, `ServiceOrder` | `OrderService` | `OrderSet`, `OrderFrequency` |
| Allergy management | `Allergy`, `Allergen`, `AllergyReaction` | `AllergyService` | `Allergies` (constants) |
| Visit tracking | `Visit`, `VisitType` | `VisitService` | `VisitAttribute` |
| Provider management | `Provider` | `ProviderService` | `ProviderAttribute` |
| User & role management | `User`, `Role`, `Privilege` | `UserService` | `UserProperty`, `UserRole` |
| Cohort management | `Cohort`, `CohortMembership` | `CohortService` | – |
| Diagnosis handling | `Diagnosis` | `DiagnosisService` | `DiagnosisAttribute` |
| Condition handling | `Condition` | `ConditionService` | – |
| Medication dispensing | `MedicationDispense` | `MedicationDispenseService` | – |
| HL7 messaging | `HL7Message` (internal) | `HL7Service` | `HL7InQueue`, `HL7OutQueue` |
| Form & resource handling | `Form`, `FormResource` | `FormService` | – |
| Program & workflow | `Program`, `ProgramWorkflow`, `ProgramWorkflowState` | `ProgramWorkflowService` | – |
| Location management | `Location`, `LocationTag` | `LocationService` | `LocationAttribute` |
| Global properties | `GlobalProperty` | `AdministrationService` | – |
| Auditing / serialization | All domain classes implement `Auditable` / `Retireable` | – | – |

---

## 4. Personal Identifiable Information (PII) Inventory  

| Data Element | Type | Code Location | DB Table | Typical Usage | Sensitive Flag |
|--------------|------|---------------|----------|---------------|----------------|
| Given / Middle / Family Name, Prefix, Family Name Prefix | `String` | `PersonName.java` | `person_name` | Patient/Person identification | ✔ |
| Person Addresses (address1‑address15) | `String` | `PersonAddress.java` | `person_address` | Demographic/contact info | ✔ |
| Birthdate, Birthdate Estimated | `Date` / `Boolean` | `Person.java` | `person` | Age calculations, eligibility | ✔ |
| Gender | `String` | `Person.java` | `person` | Reporting, clinical logic | ✔ |
| Patient Identifiers (MRN, national IDs) | `String` | `PatientIdentifier.java` | `patient_identifier` | Unique patient lookup | ✔ |
| Contact details (email, phone) | `String` | `PersonAttribute.java` (custom attrs) | `person_attribute` | Notifications, communication | ✔ |
| User credentials (username, password hash, secret question/answer, activation key) | `String` | `User.java`, `LoginCredential.java` | `users`, `user_password` | Authentication | ✔ (high) |
| Observation values (free‑text notes, numeric, coded) | `String`/`Number` | `Obs.java` | `obs` | Clinical documentation | ✔ (clinical) |
| Encounter notes, provider comments | `String` | `Encounter.java` (notes field) | `encounter` | Clinical workflow | ✔ |
| Drug order details (drug name, dosage, frequency) | `String`/`Number` | `DrugOrder.java` | `orders` | Medication management | ✔ |
| Allergy details (substance, reaction) | `String` | `Allergy.java` | `allergy` | Clinical safety | ✔ |
| Diagnosis, Condition descriptions | `String` | `Diagnosis.java`, `Condition.java` | `diagnosis`, `condition` | Clinical decision support | ✔ |
| Location coordinates (if custom attributes) | `String`/`Number` | `LocationAttribute.java` | `location_attribute` | Facility mapping | ✔ (potential) |
| HL7 message payloads (inbound/outbound) | `String` (XML/ER7) | `HL7Message.java` | `hl7_in_queue`, `hl7_out_queue` | Inter‑system exchange | ✔ (clinical) |

*All of the above fields are stored in relational tables via Hibernate and are subject to the same access‑control policies defined by OpenMRS’s role‑based security model.*

---

## 5. Third‑Party Vendors & Services  

| Vendor / Service | Role in OpenMRS | Artifact (Maven) | URL / Source | Data Shared |
|------------------|-----------------|------------------|--------------|-------------|
| Mozilla (MPL 2.0) | License text | – | <http://mozilla.org/MPL/2.0/> | – |
| HL7 / HAPI | HL7 parsing & building | `ca.uhn.hapi:hapi-base`, `hapi-structures-v2` | <https://repo1.maven.org/maven2/ca/uhn/hapi/> | No PII (library only) |
| Hibernate ORM | Persistence layer | `org.hibernate:hibernate-core` | <https://repo1.maven.org/maven2/org/hibernate/> | – |
| Hibernate Search (Lucene/Elasticsearch) | Full‑text indexing | `org.hibernate.search:hibernate-search-orm` (Lucene) / `hibernate-search-backend-elasticsearch` | <https://repo1.maven.org/maven2/org/hibernate/search/> | Indexed clinical data (potentially PII) |
| Liquibase | DB schema migrations | `org.liquibase:liquibase-core` | <https://repo1.maven.org/maven2/org/liquibase/> | – |
| Spring Framework | DI, transaction, MVC | `org.springframework:spring-context`, `spring-webmvc`, … | <https://repo1.maven.org/maven2/org/springframework/> | – |
| Jackson | JSON (de)serialization for REST | `com.fasterxml.jackson.core:jackson-databind` | <https://repo1.maven.org/maven2/com/fasterxml/jackson/core/> | – |
| Apache Velocity | Templating for reports/pages | `org.apache.velocity:velocity-engine-core` | <https://repo1.maven.org/maven2/org/apache/velocity/> | – |
| SLF4J / Logback | Logging | `org.slf4j:slf4j-api`, `ch.qos.logback:logback-classic` | <https://repo1.maven.org/maven2/org/slf4j/> | – |
| Snyk, Codacy, Travis CI, Coveralls | CI/CD security & quality checks | – | Various SaaS URLs | – |
| OpenMRS Implementation‑ID Service | Registers a unique instance identifier (analytics) | – | <https://openmrs.org/implementation-id> | Instance metadata only |

---

## 6. Data Stores  

| Table / Entity | Store Type | Purpose | Contains PII? | Configuration |
|----------------|------------|---------|---------------|---------------|
| `person` / `person_name` / `person_address` | Relational (MySQL/PostgreSQL) via Hibernate | Core demographic data | **Yes** (names, DOB, gender, address, identifiers) | JPA annotations in `org.openmrs.Person` and related classes |
| `patient` | Relational | Patient‑specific extensions (identifiers, status) | **Yes** (inherits all Person fields) | `org.openmrs.Patient` |
| `users`, `user_password` | Relational | Authentication accounts, hashed passwords, secret Q/A | **Yes** (username, email, password hash) | `org.openmrs.User` |
| `encounter` | Relational | Encounter metadata (date, type, provider, location) | **Yes** (links to patient, provider) | `org.openmrs.Encounter` |
| `obs` | Relational | Clinical observations (numeric, coded, free‑text) | **Yes** (free‑text may contain PII) | `org.openmrs.Obs` |
| `orders` (incl. `drug_order`, `test_order`) | Relational | Prescription / test orders | **Yes** (drug names, dosage, patient link) | `org.openmrs.Order` |
| `visit` | Relational | Grouping of encounters | **Yes** (patient link, dates) | `org.openmrs.Visit` |
| `hl7_in_queue`, `hl7_out_queue` | Relational | Staging of inbound/outbound HL7 messages | **Potentially Yes** (message payloads) | `org.openmrs.hl7` entities |
| `concept`, `concept_answer`, `concept_name` | Relational | Terminology metadata (non‑PII) | No | `org.openmrs.Concept` |
| `location`, `location_attribute` | Relational | Facility information (optional custom attrs) | No (unless custom PII added) | `org.openmrs.Location` |
| Liquibase tables (`DATABASECHANGELOG`, `DATABASECHANGELOGLOCK`) | Relational | Schema migration tracking | No | `liquibase` XML files |

---

## 7. Data‑Flow Map (Ingress → Internal → Egress)  

### 7.1 Ingress Paths (PII entry points)

| # | Mechanism | Entry Class / Layer | PII Introduced | Primary Service(s) |
|---|-----------|---------------------|----------------|--------------------|
| 1 | Web UI – Patient Registration Form | `PatientFormServlet` → `PatientService.savePatient()` | Names, DOB, gender, address, identifiers, contact info, attributes | `PatientService`, `PersonService` |
| 2 | Web UI – User Account Creation | `UserFormServlet` → `UserService.saveUser()` | Username, email, password (hashed), secret Q/A, linked Person data | `UserService`, `LoginCredentialService` |
| 3 | REST API (JSON) | Controllers under `web/src/main/java/org/openmrs/web/rest/v1/*` | All domain objects (Patient, Encounter, Obs, Order, etc.) | All `*Service` interfaces |
| 4 | HL7 ADT inbound messages | `HL7Service.processMessage()` → `AdtMessageHandler` | Patient identifiers, names, DOB, gender, address, encounter data | `HL7Service`, `PatientService`, `EncounterService` |
| 5 | Bulk Data Import (CSV/Excel) | Import modules (`org.openmrs.module.importexport.*`) | Large batches of patients, encounters, observations, users | `ImportExportService` → underlying domain services |
| 6 | Scheduler / Background Jobs (e.g., sync, reporting) | `org.openmrs.scheduler.Task` implementations | May read/write PII from DB for batch processing | Service layer (any) |
| 7 | FHIR optional modules (if enabled) | FHIR controllers (`org.openmrs.module.fhir2.web.*`) | Same domain objects exposed via FHIR JSON/XML | `FHIR*Service` wrappers |

### 7.2 Internal Flow  

1. **Controller / Servlet** → **Service Interface** (transactional)  
2. **Service Implementation** → **DAO Interface** (Hibernate Session)  
3. **Hibernate** → **Relational DB** (MySQL/PostgreSQL)  
4. **Interceptors / Handlers** (e.g., `ImmutableObsInterceptor`, `BaseVoidHandler`) enforce business rules and audit.  
5. **Caching** (second‑level Hibernate cache, optional Redis) may hold serialized entities.  
6. **Search Indexing** – Hibernate Search writes selected fields (including some PII) to Lucene/Elasticsearch indexes for free‑text search.  
7. **Event Bus** – `ApplicationEventPublisher` emits domain events that can be consumed by modules or external listeners.

### 7.3 Egress Paths  

| # | Destination | Egress Class / Layer | PII Exposed | Controls |
|---|-------------|----------------------|------------|----------|
| 1 | REST API responses | `*Resource` classes (Jackson serialization) | All requested domain fields (subject to `@JsonIgnore` and role‑based filters) | Spring Security, `@PreAuthorize`, field‑level view filters |
| 2 | HL7 outbound messages | `HL7Service.sendMessage()` → `Hl7MessageBuilder` | Patient demographics, encounter data, observations | Message templates, configurable field masking |
| 3 | PDF / CSV reports (web UI) | `ReportController` → JasperReports / CSV writers | Selected columns (configurable) | Role‑based access to report generation |
| 4 | Search index (Lucene/ES) | `HibernateSearchIntegrator` | Indexed fields (often names, identifiers) | Index‑level security not native; relies on application‑level filtering |
| 5 | Audit logs (logback) | `Slf4JLogService` | Action metadata (who changed what, timestamps) | Logs may contain identifiers; log rotation & access controls required |
| 6 | External modules (e.g., FHIR, custom integrations) | Module‑specific services | Any domain data the module chooses to expose | Module‑level security configuration |

---

## 8. Key Findings (Synthesised from Architecture, Business Features, Data Elements, and Flows)

1. **Extensive PII Coverage** – Nearly every core domain object (Person, Patient, User, Encounter, Obs, Order, etc.) stores personally identifiable or health‑related data.  
2. **Layered Service Design** – Clear separation of concerns makes it straightforward to inject security checks at the service layer, but also means that any missing check can propagate downstream.  
3. **Multiple Ingress/Egress Vectors** – Data can enter via UI forms, REST calls, HL7 messages, and bulk imports; it can leave via REST, HL7 outbound, reports, and search indexes.  
4. **Third‑Party Dependencies** – The stack relies on many mature libraries (Spring, Hibernate, HAPI, Jackson). Most are well‑maintained, but version drift can introduce vulnerabilities.  
5. **Search Index Exposure** – Indexes may contain unencrypted PII (names, identifiers) and are not protected by OpenMRS’s role‑based security out‑of‑the‑box.  
6. **Audit & Logging** – Comprehensive audit events are generated, but logs may inadvertently capture raw PII if not filtered.  
7. **Configuration‑Driven Security** – Access control is driven by OpenMRS privileges/roles defined in the database; misconfiguration can lead to over‑exposure.  

---

## 9. Risk Indicators  

| Risk | Description | Likelihood | Impact |
|------|-------------|------------|--------|
| **Unauthorized access to PII** | Insufficient role‑based restrictions on REST endpoints or UI pages could expose patient data. | Medium | High (HIPAA‑type breach) |
| **PII leakage via search indexes** | Lucene/Elasticsearch indexes are not encrypted and may be readable by OS users or external services. | Medium | High |
| **Unencrypted data at rest** | Default DB configuration stores PII in clear text; no column‑level encryption is applied. | High | High |
| **Password handling weaknesses** | Passwords are hashed with SHA‑512 + per‑user salt; lack of adaptive hashing (e.g., bcrypt) may be weaker against brute‑force. | Medium | Medium |
| **Third‑party library vulnerabilities** | Out‑of‑date versions of Spring, Hibernate, or HAPI could contain known CVEs. | Medium | Medium |
| **Improper data masking in HL7 outbound** | Custom HL7 templates may inadvertently include full identifiers. | Low | High |
| **Insufficient audit log protection** | Logs may be accessible to non‑privileged users or retained longer than required. | Medium | Medium |
| **Configuration drift in CI/CD** | Security‑related Maven/Gradle plugins (Snyk, Codacy) may be disabled or mis‑configured, reducing detection of new issues. | Low | Medium |

---

## 10. Recommendations for Privado Review  

1. **Data‑at‑Rest Protection**  
   - Enable column‑level encryption for high‑risk fields (e.g., identifiers, DOB, gender) using database‑native encryption or application‑level encryption libraries.  
   - Rotate encryption keys periodically and store them in a secure vault (e.g., HashiCorp Vault, AWS KMS).

2. **Strengthen Authentication**  
   - Replace SHA‑512 password hashing with an adaptive algorithm such as BCrypt or Argon2.  
   - Enforce multi‑factor authentication for privileged users.

3. **Search Index Hardening**  
   - Limit indexed fields to non‑PII where possible; if PII must be indexed, encrypt the index or store it on a secured filesystem with strict OS permissions.  
   - Apply role‑based filtering before returning search results.

4. **Fine‑Grained API Security**  
   - Review all REST controllers for proper `@PreAuthorize` annotations.  
   - Implement field‑level view filtering (Jackson Mix‑ins) to hide sensitive attributes from users lacking the required privilege.

5. **Audit Log Sanitization**  
   - Mask or omit PII in log statements; configure Logback to use a custom layout that redacts identifiers.  
   - Secure log storage (restricted file permissions, log rotation, retention policies).

6. **HL7 / FHIR Message Controls**  
   - Audit all outbound HL7/FHIR templates to ensure only required fields are transmitted.  
   - Provide configuration switches to strip or hash identifiers when sending to external systems.

7. **Dependency Management**  
   - Adopt a Dependabot / Renovate workflow to keep third‑party libraries up‑to‑date.  
   - Run Snyk/OWASP Dependency‑Check in CI and fail builds on critical CVEs.

8. **Configuration & CI/CD Hardening**  
   - Ensure security‑related CI steps (Snyk, Codacy, static analysis) are mandatory and fail on warnings.  
   - Store all secret keys (DB passwords, encryption keys) outside of the repository (environment variables or secret managers).

9. **Periodic Privado‑Specific Review**  
   - Conduct quarterly data‑flow and PII inventory reviews to capture new modules or custom extensions.  
   - Perform penetration testing focused on the identified ingress/egress vectors.

---

## 11. Audit Metadata  

| Item | Value |
|------|-------|
| **Audit Date** | 2026‑05‑08 |
| **Repository URL** | <https://github.com/khemrajrathore/openmrs-core.git> |
| **Branch Analyzed** | master |
| **Scope** | Full source‑code static analysis (architecture, tech stack, business features, data elements, third‑party services, data stores, data flows). |
| **Tools Used** | Manual code inspection, Maven dependency tree, domain‑model exploration, Liquibase changelog review. |
| **Prepared By** | OpenAI‑ChatGPT (executive audit synthesis) |

---  

*This executive audit summary consolidates the architectural, functional, and data‑privacy aspects of the OpenMRS Core codebase to aid Privado’s security and compliance review.*