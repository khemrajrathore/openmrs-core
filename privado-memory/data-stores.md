# privado‑memory/data‑stores.md  

## Databases  

| Table (Entity) | Store | Type | Purpose | PII Held? | Config Location |
|----------------|-------|------|---------|-----------|-----------------|
| **Person** | MySQL / PostgreSQL | Relational (Hibernate ORM) | Core demographic data (names, addresses, attributes) | Yes – name, DOB, gender, address, identifiers | `hibernate.cfg.xml` + JPA annotations in `org.openmrs.Person` |
| **Patient** | MySQL / PostgreSQL | Relational (Hibernate ORM) | Patient‑specific extensions (patientId, allergy status, identifiers) | Yes – all Person fields + patient‑only identifiers | `hibernate.cfg.xml` + JPA annotations in `org.openmrs.Patient` |
| **User** | MySQL / PostgreSQL | Relational (Hibernate ORM) | System login accounts, roles, user properties | Yes – username, email, linked Person data | `hibernate.cfg.xml` + JPA annotations in `org.openmrs.User` |
| **Encounter** | MySQL / PostgreSQL | Relational (Hibernate ORM) | Clinical encounter metadata (date, type, provider, location) | Yes – links to Patient, provider, location | `hibernate.cfg.xml` + JPA annotations in `org.openmrs.Encounter` |
| **Obs** | MySQL / PostgreSQL | Relational (Hibernate ORM) | Clinical observations (numeric, coded, text, complex) | Yes – observation values may contain PII (e.g., free‑text notes) | `hibernate.cfg.xml` + JPA annotations in `org.openmrs.Obs` |
| **Order** (incl. DrugOrder, TestOrder, etc.) | MySQL / PostgreSQL | Relational (Hibernate ORM) | Prescription / test orders and their lifecycle | Yes – drug names, dosages, patient linkage | `hibernate.cfg.xml` + JPA annotations in `org.openmrs.Order` |
| **Concept** | MySQL / PostgreSQL | Relational (Hibernate ORM) | Terminology / metadata for coded data | No (concept definitions are non‑PII) | `hibernate.cfg.xml` + JPA annotations in `org.openmrs.Concept` |
| **Location** | MySQL / PostgreSQL | Relational (Hibernate ORM) | Facility / site information | No (unless custom attributes added) | `hibernate.cfg.xml` + JPA annotations in `org.openmrs.Location` |
| **Visit** | MySQL / PostgreSQL | Relational (Hibernate ORM) | Higher‑level grouping of encounters | Yes – patient linkage, dates | `hibernate.cfg.xml` + JPA annotations in `org.openmrs.Visit` |
| **Program / PatientProgram** | MySQL / PostgreSQL | Relational (Hibernate ORM) | Enrollment in care programs | Yes – patient linkage, dates, state | `hibernate.cfg.xml` + JPA annotations in `org.openmrs.Program` / `PatientProgram` |
| **HL7 Queue tables** (`hl7_in_queue`, `hl7_out_queue`) | MySQL / PostgreSQL | Relational (Hibernate ORM) | Staging of inbound/outbound HL7 messages | Potentially Yes (message payloads may contain PII) | `hibernate.cfg.xml` + JPA annotations in `org.openmrs.hl7`‑related entities |
| **Liquibase changelog tables** (`DATABASECHANGELOG`, `DATABASECHANGELOGLOCK`) | MySQL / PostgreSQL | Relational | Tracks schema migrations | No | `liquibase` XML files under `api/src/main/resources/liquibase` |

*All of the above tables are created/maintained by the Liquibase migration system, which is version‑controlled in the repository.*

## Caches & Queues  

| Component | Type | What it Caches / Queues | PII Impact | Config Location |
|-----------|------|--------------------------|------------|-----------------|
| **Hibernate Second‑Level Cache** (enabled via `@Cache` annotations) | In‑memory (EhCache / Hazelcast, configurable) | Entity and collection data (e.g., Person, Patient, Concept) | May hold PII if cached entities contain it | `hibernate.cfg.xml` (`hibernate.cache.use_second_level_cache`, `hibernate.cache.region.factory_class`) |
| **Hibernate Query Cache** | In‑memory | Result sets of frequently run HQL/Criteria queries | Same as above | `hibernate.cfg.xml` (`hibernate.cache.use_query_cache`) |
| **Hibernate Search Indexes** (Lucene by default, optional Elasticsearch) | Disk‑based Lucene index files or remote ES cluster | Full‑text searchable fields (names, concepts, observations) | Indexes may contain PII (e.g., patient names) – must be protected at OS/ES level | `hibernate.cfg.xml` (`hibernate.search.default.directory_provider`, `hibernate.search.elasticsearch.host`) |
| **HL7 Message Queues** (DB‑backed tables) | Relational queue tables | Incoming/outgoing HL7 messages awaiting processing | May contain PII in message payloads | `hibernate.cfg.xml` + service‑level config (`hl7.inbound.queue.enabled`, etc.) |
| **Spring / Async Task Executors** | In‑memory thread pools | Background jobs (e.g., index rebuild, data export) | No direct PII storage, but tasks may process PII | `applicationContext.xml` / Spring Boot properties |
| **Docker Compose Service Dependencies** | Container orchestration | Spins up MySQL/PostgreSQL containers for dev/test | No PII (development data only) | `docker-compose.yml` |

## File/Object Storage  

| Storage Area | What it Holds | PII? | Access / Retention | Config Location |
|--------------|---------------|------|--------------------|-----------------|
| **Complex Obs Files** (`obs.value_complex` points to a file on the filesystem or a BLOB) | Images, PDFs, audio, video, or any large binary observation | Yes – clinical images, scanned documents may contain PII | Stored under `${openmrs.data_directory}/obs` (default `~/.OpenMRS/obs`) – governed by OS file permissions; can be cleaned via `obs.cleanup` global property | `OpenmrsConstants` (`OPENMRS_DATA_DIRECTORY`) and `applicationContext.xml` |
| **Document/Attachment Entities** (`Document`, `Attachment`) | Uploaded documents linked to patients/encounters | Yes – may contain PHI | Same as above; can be archived or purged via custom scripts | `hibernate.cfg.xml` (if mapped as BLOB) |
| **Log Files** (`openmrs.log`, Docker container logs) | Application logs, audit trails | Potentially Yes (stack traces may include identifiers) | Rotated via Logback configuration (`logback.xml`) | `logback.xml` |
| **Backup Dumps** (MySQL/PG dump files) | Full database snapshots | Yes – entire DB content | Retention defined by backup policy (e.g., keep 30 days) | Backup scripts / Docker volume mounts |

## Data Retention Patterns  

| Data Category | Typical Retention (default) | Configurable? | Reasoning |
|---------------|-----------------------------|---------------|-----------|
| **Person / Patient core demographics** | Indefinite (legal requirement) | Yes – via custom `global_property` (e.g., `person.retention.period`) | Patient records must be kept for the lifetime of care and often for statutory periods. |
| **Encounter / Visit** | 7 years (common health‑system guideline) | Yes – `encounter.retention.years` | Clinical encounters are often required for audit and research. |
| **Obs (including complex files)** | 7 years (aligned with encounter) | Yes – `obs.retention.years` & `obs.complex.retention.days` | Observations are tied to encounters; complex files may be archived separately. |
| **Orders** | 5 years after completion/void | Yes – `order.retention.years` | Prescription and test orders have shorter statutory hold periods. |
| **HL7 Message Queues** | 30 days (processing window) | Yes – `hl7.queue.retention.days` | Messages are transient; after processing they can be purged. |
| **Audit / Log tables** (`audit_log`, `openmrs_log`) | 1 year (configurable) | Yes – `log.retention.days` | For security monitoring; older logs may be archived. |
| **Liquibase changelog tables** | Indefinite (migration history) | No | Needed for schema version tracking. |
| **Cache entries** | In‑memory TTL (default 60 min) | Yes – cache provider settings | Cached data is volatile; eviction policies control lifespan. |
| **Search indexes** | Re‑indexed on data change; old index files removed on rebuild | Yes – index retention controlled by search engine config | Indexes are regenerated; stale files are cleaned automatically. |

### How Retention Is Enforced  

* **Global Properties** – OpenMRS exposes many retention‑related settings via the `global_property` table; administrators can adjust periods without code changes.  
* **Scheduled Jobs** – The `SchedulerService` runs jobs such as `ObsPurgeJob`, `EncounterPurgeJob`, and `CacheEvictionJob` that read the global properties and delete/archieve data accordingly.  
* **Database Constraints** – Foreign‑key cascades (`ON DELETE CASCADE`) ensure that when a patient is voided, related child rows are also voided, preserving referential integrity while keeping audit trails.  
* **Docker Compose** – Development containers are often started with fresh databases; data persistence is achieved by mounting a volume (`./data/mysql:/var/lib/mysql`). Retention policies are irrelevant for ephemeral dev environments but are documented for production deployments.  

---

**Key Configuration Files Referenced**  

* `api/src/main/resources/hibernate.cfg.xml` – Hibernate dialect, datasource, second‑level cache, search index settings.  
* `api/src/main/resources/liquibase/changelog.xml` – Defines all schema objects (tables, indexes, constraints).  
* `docker-compose.yml` – Declares MySQL/PostgreSQL services, volumes, and environment variables (e.g., `MYSQL_DATABASE=openmrs`).  
* `src/main/resources/logback.xml` – Log rotation and retention.  
* `src/main/resources/applicationContext.xml` (or Spring Boot `application.properties`) – Enables async executors, scheduler, and HL7 queue settings.  

---  

*This document summarizes the data‑store landscape of the OpenMRS Core repository, covering relational databases, caching/search layers, file storage, and the retention mechanisms that govern them.*