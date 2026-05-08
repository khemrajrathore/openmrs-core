**privado‑memory/data-flows.md**  

# Complete Data‑Flow Map for OpenMRS Core  

The following document consolidates every data‑flow path that can carry **personally‑identifiable information (PII)** through the OpenMRS Core code‑base.  It is derived from the architectural, business‑feature, data‑store, cache, queue and third‑party analyses supplied earlier.  

---  

## 1.  High‑Level Overview  

```
+-------------------+      +-------------------+      +-------------------+
|   Data Ingress    | ---> |   Internal Flow   | ---> |   Data Egress     |
+-------------------+      +-------------------+      +-------------------+
```

* **Data Ingress** – any entry point where PII first touches the system.  
* **Internal Flow** – the path the data follows inside the JVM (service → DAO → ORM → DB, plus caches, interceptors, events, indexing, background jobs).  
* **Data Egress** – any outbound channel that can expose PII outside the core server (HL7 out, REST responses, reports, file exports, search indexes, audit logs, etc.).  

---  

## 2.  Data‑Ingress Paths  

| # | Ingress Mechanism | Entry Point (Class / Layer) | PII Elements Introduced | Primary Service(s) | Notes |
|---|-------------------|-----------------------------|------------------------|--------------------|-------|
| 1 | **Web UI – Patient Registration Form** | `web/src/main/java/org/openmrs/web/servlet/PatientFormServlet` → `PatientService.savePatient()` | Names, DOB, gender, address, identifiers, contact info, attributes | `PatientService`, `PersonService` | Validation performed by `PersonValidator`, `PatientValidator`. |
| 2 | **Web UI – User Account Creation** | `web/src/main/java/org/openmrs/web/servlet/UserFormServlet` → `UserService.saveUser()` | Username, email, password (plain → hashed), secret Q/A, linked `Person` data | `UserService` | Password hashing occurs in `LoginCredentialService` (SHA‑512 + per‑user salt). |
| 3 | **REST API – JSON payloads** | Controllers in `web/src/main/java/org/openmrs/web/rest/v1/*` → Service layer | All domain objects (Patient, Encounter, Obs, Order, etc.) | All `*Service` interfaces (Patient, Encounter, Obs, Order, …) | Spring MVC deserialises JSON via Jackson; validation annotations (`@Valid`) are applied. |
| 4 | **HL7 ADT inbound messages** | `org.openmrs.hl7.handler.AdtMessageHandler` (invoked by `HL7Service.processMessage()`) | Patient identifiers, names, DOB, gender, address, visit/encounter data | `HL7Service`, `PatientService`, `EncounterService` | Message parsing uses HAPI; after parsing the same service‑DAO flow is used. |
| 5 | **Bulk Data Import (CSV/Excel)** | Import modules (e.g., `org.openmrs.module.importexport.*`) → Service layer | Large‑scale sets of patients, encounters, observations, users | `ImportExportService` → underlying domain services | Import runs in a background job; each row is processed through the same validation pipeline. |
| 6 | **Background Scheduler – Sync Jobs** | `org.openmrs.scheduler.Task` implementations (e.g., `HL7InboundTask`) | Periodic pull of external HL7 feeds, external CSV files | Scheduler → Service layer | Same validation as direct inbound. |
| 7 | **System‑Generated Data** | `OpenmrsUtil.generateUuid()`, `OpenmrsUtil.now()` etc. | Auto‑generated IDs, timestamps (not PII but part of audit trail) | Various services | Auditable entities capture `creator`, `dateCreated`, `changedBy`, `dateChanged`. |

---  

## 3.  Internal Movement (Service → DAO → Persistence)  

### 3.1 Core Service‑to‑DAO Flow  

```
[Controller / UI / Scheduler] 
        │
        ▼
   Service Layer (e.g., PatientService)
        │
        ▼
   Save / Update Hook (SaveHandler, VoidHandler, etc.)
        │
        ▼
   DAO Interface (e.g., PatientDAO)
        │
        ▼
   Hibernate Implementation (e.g., HibernatePatientDAO)
        │
        ▼
   Hibernate Session → Transaction
        │
        ▼
   Relational DB (MySQL / PostgreSQL)
```

* **AOP Interceptors** – `OpenmrsTransactionalInterceptor`, `OpenmrsSecurityInterceptor` wrap service calls to enforce transaction boundaries and permission checks.  
* **Event Listeners** – `OpenmrsEventListener` (e.g., `PatientCreatedEvent`) fire after successful commit; listeners may push data to queues, audit logs, or external modules.  
* **Hibernate Search Indexing** – `@Indexed` entities (Patient, Person, Obs, Concept) are automatically queued for indexing after DB commit; the index lives in Lucene files or Elasticsearch.  

### 3.2 Caches & Queues  

| Component | What Is Cached / Queued | PII Impact | Cache/Queue Technology | Config Location |
|-----------|------------------------|------------|------------------------|-----------------|
| **Hibernate Second‑Level Cache** | Entity instances (Person, Patient, Obs, etc.) | Holds full PII if enabled | EhCache / Hazelcast (configurable) | `hibernate.cfg.xml` (`hibernate.cache.use_second_level_cache`) |
| **Hibernate Query Cache** | Result‑set snapshots | Same as above | Same as 2nd‑level cache | `hibernate.cfg.xml` (`hibernate.cache.use_query_cache`) |
| **Hibernate Search Index** | Tokenised fields (names, identifiers, free‑text obs) | Index files contain PII in clear text | Lucene (disk) or Elasticsearch (remote) | `hibernate.cfg.xml` (`hibernate.search.default.directory_provider`) |
| **HL7 Inbound Queue** | `hl7_in_queue` table rows awaiting processing | May contain full HL7 payload with PII | DB‑backed queue (Hibernate) | `api/src/main/resources/liquibase/hl7.xml` |
| **HL7 Outbound Queue** | `hl7_out_queue` table rows awaiting transmission | Same as inbound | DB‑backed queue | Same as inbound |
| **Spring Async Executors** | Background jobs (index rebuild, data export) | Jobs process PII in memory only | ThreadPoolTaskExecutor | `applicationContext.xml` / Spring Boot properties |
| **Scheduler Tasks** | `org.openmrs.scheduler.Task` instances | May read/write PII during execution | In‑memory scheduler | `metadata/scheduler/*.xml` |

### 3.3 Transformation & Protection Steps (Internal)  

| Step | Where It Happens | Code / Class | Effect on PII |
|------|------------------|--------------|---------------|
| **Password hashing** | `LoginCredentialService.saveLoginCredential()` | `org.openmrs.security.authentication.LoginCredentialService` | Plain password → SHA‑512 + per‑user salt stored in `login_credential.password_hash`. |
| **Salt generation** | Same as above | `LoginCredentialService.generateSalt()` | Random 16‑byte salt stored alongside hash. |
| **Secret Q/A encryption (optional)** | `LoginCredentialService` (if `encryptSecretAnswers` flag enabled) | `org.openmrs.security.encryption.SecretAnswerEncryptor` | Stores encrypted secret answer. |
| **Audit logging** | Hibernate Envers (`@Audited`) on `Auditable` entities | `org.openmrs.audit.AuditLogListener` | Every change creates a row in `openmrs_audit_log` (includes user, timestamp, changed fields). |
| **Voiding / Soft‑delete** | `BaseVoidHandler` invoked before delete | `org.openmrs.api.handler.BaseVoidHandler` | Sets `voided = true`, records `voidReason`, retains data for audit. |
| **Person attribute encryption (configurable)** | `PersonAttributeService` (if encryption enabled) | `org.openmrs.attribute.PersonAttributeEncryption` | Encrypts selected attribute values before persisting. |
| **Input validation** | Validators (`PersonValidator`, `PatientValidator`, `ObsValidator`, etc.) | `org.openmrs.validator.*` | Rejects malformed or unsafe data (e.g., overly long strings, illegal characters). |
| **XSS/HTML sanitisation** | JSP/Velocity rendering pipelines | `org.openmrs.web.filter.XssFilter` | Escapes user‑supplied HTML before display. |
| **CSRF protection** | Spring MVC security filters | `org.springframework.security.web.csrf.CsrfFilter` | Prevents cross‑site request forgery on state‑changing endpoints. |

---  

## 4.  Data‑Egress Paths  

| # | Egress Mechanism | Exit Point (Class / Layer) | PII Elements Potentially Sent | Destination | Protection Measures |
|---|------------------|----------------------------|------------------------------|-------------|----------------------|
| 1 | **HL7 outbound messages** | `HL7Service.sendMessage()` → `HL7OutQueue` → `HL7OutboundTask` → external HL7 interface | Patient identifiers, names, DOB, gender, visit/encounter data, observations | Remote HL7 system (lab, HIS) | Message is built from domain objects; optional TLS transport configured at integration layer. |
| 2 | **REST API responses** | Controllers in `web/src/main/java/org/openmrs/web/rest/v1/*` → Jackson serializer | Any domain object requested (Patient, Obs, Encounter, etc.) | External client (web app, mobile, third‑party system) | Spring Security enforces role‑based access; fields can be filtered with `@JsonView`. |
| 3 | **HTML/JSP UI rendering** | JSP pages (`*.jsp`) & Velocity templates (`*.vm`) | Names, identifiers, addresses, observations displayed in browser | End‑user’s browser | XSS filter, HTML escaping, CSRF token. |
| 4 | **Reports (PDF/CSV/Excel)** | `org.openmrs.reporting` (core reporting module) → `ReportService` → file writer | Patient demographics, encounter summaries, lab results | Downloaded by user or emailed | Report generation runs under the same permission checks; files stored temporarily on server file‑system with restricted permissions. |
| 5 | **Form exports** | `FormService.exportForm()` → `FormResource` → file download | Form definition (may contain embedded PII in default values) | User download | Access controlled by `FormService` permissions. |
| 6 | **Search results (Hibernate Search)** | `SearchService.search()` → Lucene/ES response → REST or UI | Names, identifiers, free‑text observations | Browser / API client | Search index is protected by OS file permissions; if ES is used, TLS + auth can be configured. |
| 7 | **Audit log export** | `AuditLogService` → CSV/JSON export | Who changed what, when, and old/new values (may include PII) | Admin download | Export requires `View Audits` privilege. |
| 8 | **Data export jobs** (e.g., `DataExportTask`) | Scheduler → `ExportService` → file (CSV/JSON) | Full patient record dump | External storage / SFTP | Job runs under admin account; file permissions must be set manually. |
| 9 | **Implementation‑ID registration** | `ImplementationIdService.register()` → HTTP POST to `https://openmrs.org` | Implementation UUID, version, host name (no patient data) | OpenMRS central service | No PHI; payload is tiny and sent over HTTPS. |

---  

## 5.  End‑to‑End Flow Diagrams (ASCII)  

### 5.1 Patient Registration (Web UI)  

```
[Browser] 
   │  POST /openmrs/patient.form
   ▼
[PatientFormServlet] ──► validates input (PatientValidator)
   │
   ▼
[PatientService] ──► pre‑save handlers (PersonSaveHandler, PatientSaveHandler)
   │
   ▼
[HibernatePatientDAO.savePatient()]
   │
   ▼
[Hibernate Session] ──► begin TX
   │
   ▼
[INSERT/UPDATE person, patient, patient_identifier, person_name, person_address]
   │
   ▼
[Commit TX] ──► second‑level cache put (if enabled)
   │
   ▼
[Hibernate Search] ──► index Person/Patient fields
   │
   ▼
[Response] (JSON / redirect) → Browser
```

### 5.2 HL7 ADT Inbound → Patient/Encounter Creation  

```
[External HL7 Source] 
   │  (TCP/MLLP or file drop)
   ▼
[HL7InQueue] (DB table) ──► HL7InboundTask (scheduler)
   │
   ▼
[HL7Service.processMessage(message)]
   │
   ▼
[ADTMessageHandler] parses → builds Patient/Encounter DTOs
   │
   ▼
[PatientService.savePatient()]   [EncounterService.saveEncounter()]
   │                                 │
   ▼                                 ▼
[HibernatePatientDAO]            [HibernateEncounterDAO]
   │                                 │
   ▼                                 ▼
[DB writes] (person, patient, encounter, obs …)
   │                                 │
   ▼                                 ▼
[Hibernate Search] (index)   [Event Listeners] (e.g., EncounterCreatedEvent)
   │                                 │
   ▼                                 ▼
[Optional HL7 outbound] (e.g., ACK)   [REST API] (if client polls)
```

### 5.3 REST API Create Observation  

```
[Client] ──► POST /openmrs/ws/rest/v1/obs
   │
   ▼
[ObsController] (Spring MVC) → validates Obs JSON
   │
   ▼
[ObsService.saveObs()]
   │
   ▼
[ObsSaveHandler] (business rules, e.g., immutability checks)
   │
   ▼
[HibernateObsDAO.saveObs()]
   │
   ▼
[INSERT obs table] (value_text, value_numeric, concept_id, etc.)
   │
   ▼
[Hibernate Search] (index obs.text if searchable)
   │
   ▼
[Response] → client (includes obs UUID)
```

### 5.4 Data Export Job (Scheduled)  

```
[Scheduler] ──► ExportTask.run()
   │
   ▼
[ExportService.exportPatients(criteria)]
   │
   ▼
[PatientService.getPatients()] → DAO → DB read
   │
   ▼
[Write CSV] → /tmp/patient_export_20240508.csv
   │
   ▼
[Optional SFTP upload] (external system)
```

### 5.5 Audit Logging (Envers)  

```
[Entity change] (any service → DAO → Hibernate)
   │
   ▼
[Hibernate Envers] intercepts transaction commit
   │
   ▼
[INSERT into openmrs_audit_log] (entity_id, rev, changed_by, rev_timestamp, …)
```

---  

## 6.  Summary of Protection Controls Across the Flow  

| Layer | Control | What It Protects |
|-------|---------|------------------|
| **Transport** | HTTPS for REST, TLS for HL7 (if configured) | Data in transit |
| **Authentication** | `UserService.authenticate()` + Spring Security | Prevents unauthorized access |
| **Authorization** | Role‑based privileges (`UserService`, `PatientService`, etc.) | Limits who can read/write PII |
| **Input Validation** | Validators, Bean Validation (`@NotNull`, `@Size`) | Stops malformed/unsafe data |
| **Password Handling** | SHA‑512 + per‑user salt, optional secret‑answer encryption | Protects credentials |
| **Encryption at Rest** | Optional attribute encryption, encrypted DB columns (via JPA converters) | Protects stored PII if enabled |
| **Caching** | Cache eviction on void/retire, limited cache lifetimes | Reduces stale PII exposure |
| **Search Index** | Access‑controlled ES/Lucene, OS file permissions | Prevents unauthorized index reads |
| **Audit & Logging** | Envers audit tables, SLF4J logs (no PII in plain logs) | Traceability without leaking data |
| **Soft Delete (Voiding)** | `voided` flag, not physical delete | Retains audit trail while hiding from UI |
| **CSRF / XSS Filters** | Spring Security CSRF, XSS filter on JSP/Velocity | Prevents client‑side data leakage |

---  

## 7.  Reference Map to Source Locations  

| Flow Component | Key Package / Class | Maven Module |
|----------------|---------------------|--------------|
| **Service Interfaces** | `org.openmrs.api.*Service` | `api` |
| **Service Implementations** | `org.openmrs.api.impl.*ServiceImpl` | `api` |
| **DAO Interfaces** | `org.openmrs.api.db.*DAO` | `api` |
| **Hibernate DAO Impl** | `org.openmrs.api.db.hibernate.*DAO` | `api` |
| **Save / Void Handlers** | `org.openmrs.api.handler.*Handler` | `api` |
| **HL7 Processing** | `org.openmrs.hl7.*`, `org.openmrs.hl7.handler.*` | `api` |
| **REST Controllers** | `org.openmrs.web.rest.v1.*` | `web` |
| **JSP / Velocity UI** | `webapp/src/main/webapp/WEB-INF/*.jsp`, `*.vm` | `webapp` |
| **Scheduler Tasks** | `org.openmrs.scheduler.Task` implementations | `api` / custom modules |
| **Hibernate Search Config** | `hibernate.cfg.xml` (`hibernate.search.*`) | `api` |
| **Liquibase Changelogs** | `api/src/main/resources/liquibase/*.xml` | `liquibase` |
| **Encryption Utilities** | `org.openmrs.security.encryption.*` | `api` |
| **Audit (Envers)** | `org.hibernate.envers.*` (configured via `persistence.xml`) | `api` |

---  

### End of Data‑Flow Map  

All data‑flows that can carry PII through OpenMRS Core have been enumerated, traced through the internal layers, and linked to their outbound destinations, together with the transformation and protection mechanisms applied at each step. This map can be used for privacy impact assessments, threat modeling, or compliance documentation.