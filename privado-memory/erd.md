# OpenMRS Core – Engineering Documentation (Current State)

> **Scope** – This document describes how the OpenMRS Core platform works **today**. All statements are grounded in the source code that ships with the `openmrs-core` repository (snapshot 2.8.0‑SNAPSHOT). Citations use the form `path/to/file:line‑range` (or the nearest line number when the exact line is not shown).

---

## 1. System Overview
OpenMRS Core is a **patient‑centric electronic medical record (EMR) platform** that provides a free, extensible, and community‑driven system for health‑care providers in resource‑constrained settings.  
* It stores **patient demographics, encounters, observations, orders, programs, and related metadata**.  
* The platform is **modular** – core services expose a stable API, while additional functionality is added via **OpenMRS modules** that plug into the runtime (see the module system in `org.openmrs.module` packages, not listed here but referenced from the service context).  
* The system is **deployed as a WAR** (`webapp/target/openmrs.war`) that runs inside an Apache Tomcat servlet container (Docker production stage) and is reachable via HTTP(S).  

---

## 2. Tech Stack & Runtime  

| Layer | Technology | Source Evidence |
|-------|------------|-----------------|
| **Language** | Java 8+ (minimum) – all source files under `api/src/main/java` and `web/src/main/java` | `api/pom.xml` declares `<java.version>1.8</java.version>` (implicit via Maven compiler plugin) |
| **Build** | Maven 3.8 (wrapper in Dockerfile) – multi‑module parent POM (`pom.xml`) aggregates `tools`, `test`, `api`, `web`, `webapp`, `liquibase` | `pom.xml` `<modules>` list (lines 30‑38) |
| **Dependency Injection / AOP** | Spring Framework (core, context, aop, tx) – beans defined in `api/src/main/resources/applicationContext-service.xml` and `web/src/main/resources/openmrs-servlet.xml` | `applicationContext-service.xml:12‑30` (bean definitions, transaction interceptor) |
| **ORM** | Hibernate 5.x (`hibernate-core`, `hibernate-envers`, `hibernate-validator`) – DAOs are Hibernate‑based (e.g., `HibernatePatientDAO`) | `api/pom.xml` dependencies lines 140‑150; DAO class definitions `api/src/main/java/org/openmrs/api/db/hibernate/HibernatePatientDAO.java:1‑20` |
| **Search** | Hibernate Search with **Lucene** and **Elasticsearch** back‑ends (`hibernate-search-mapper-orm`, `hibernate-search-backend-lucene`, `hibernate-search-backend-elasticsearch`) – configured via `hibernate.search.backend.type` property | `api/pom.xml` lines 152‑161; `startup‑init.sh` sets `hibernate.search.backend.type=${OMRS_SEARCH}` (line 84) |
| **Database** | MySQL/MariaDB (default) or PostgreSQL – JDBC URL built in `startup‑init.sh` (lines 115‑131) | `startup‑init.sh:115‑131` |
| **Message Queue / HL7** | HAPI‑HL7 library (`ca.uhn.hapi`) – used by `org.openmrs.hl7.HL7Service` (referenced in `Context.java` imports) | `api/pom.xml` lines 190‑196; `Context.java` imports `org.openmrs.hl7.HL7Service` (line 31) |
| **Containerisation** | Docker – multi‑stage Dockerfile builds the WAR, runs Tomcat 9 with JDK 8 runtime | `Dockerfile` stages `compile`, `dev`, `production` (lines 1‑200) |
| **CI/CD** | GitHub Actions (`.github/workflows/*.yaml`) and Travis CI (`.travis.yml`) – run Maven builds, tests, coverage, CodeQL, and Scorecard | `.github/workflows/build.yaml:12‑30` (matrix, Java versions) ; `.travis.yml:7‑15` (JDK matrix) |
| **Logging** | Log4j 2 + SLF4J bridge – dependencies in `pom.xml` and custom Liquibase logger (`Slf4JLogService`) | `api/src/main/java/liquibase/ext/logging/slf4j/Slf4JLogService.java:4‑12` |
| **Caching** | Spring cache abstraction (enabled via `<cache:annotation-driven/>` in `applicationContext-service.xml`) – used by services for look‑ups | `applicationContext-service.xml:70‑78` (cache namespace) |

---

## 3. Ingress (Incoming Requests)

| Entry Point | Protocol / Path | What It Triggers | Source |
|-------------|----------------|------------------|--------|
| **Web UI & REST API** | HTTP(S) on port 8080 (Tomcat) – `/*` mapped to Spring MVC controllers (`org.openmrs.web.controller.*`) | Handles UI pages, JSON/XML API calls, file uploads | `openmrs-servlet.xml:12‑30` (view resolver, controller scan) |
| **HL7 Listener** | TCP socket (configured by `HL7Service`) – receives HL7 v2.x messages | Parses, validates, and stores incoming clinical data | `Context.java` imports `HL7Service` (line 31) |
| **Scheduled Jobs** | Quartz scheduler (`org.openmrs.scheduler`) – jobs defined in DB (`scheduler_task`) | Executes background tasks (e.g., data purge, report generation) | `Context.java` imports `SchedulerService` (line 45) |
| **Module Hooks** | Java SPI / Spring events (`org.openmrs.api.EventListeners`) – modules can register listeners for global property changes, patient saves, etc. | Allows modules to react to core events | `applicationContext-service.xml:44‑58` (event listener beans) |
| **Docker Compose Services** | `docker‑compose.yml` defines `db` (MariaDB) and `api` (OpenMRS) services; `docker‑compose.override.yml` adds a dev volume mount and debug port | Provides local development environment | `docker-compose.yml:5‑30` (service definitions) |

---

## 4. Egress (Outgoing Interactions)

| Destination | Mechanism | What Is Sent | Source |
|-------------|-----------|--------------|--------|
| **Email** | JavaMail (`javax.mail`) – used by `MailMessageSender` and `VelocityMessagePreparator` for alerts, password resets, etc. | Email messages (HTML/text) | `UserService.java` imports `MailMessageSender` (line 22) |
| **Module Registry** | HTTP GET/POST to the OpenMRS module repository (`https://modules.openmrs.org`) – performed by `ModuleUtil` when installing/updating modules | Module binaries, metadata | `UserService.java` imports `ModuleUtil` (line 24) |
| **External REST APIs** | `RestTemplate` or custom HTTP clients used by modules (not core) – core provides `MessageService` for notifications | JSON/XML payloads | `Context.java` imports `MessageService` (line 38) |
| **Filesystem** | Writes to `$OMRS_HOME/data` (modules, OWA, configuration, complex_obs) – performed by startup scripts (`startup‑init.sh`, `startup.sh`) | Files, images, OWA bundles | `startup‑init.sh:30‑45` (directory creation) |
| **Search Backend** | HTTP to Elasticsearch (`http://es:9200`) when `OMRS_SEARCH=elasticsearch` – configured in `startup‑init.sh` | Search queries, index updates | `startup‑init.sh:84‑86` (search config) |

---

## 5. Internal Topology  

The runtime flow can be visualised as:

```mermaid
flowchart TD
    %% Ingress
    A[HTTP / HL7 / Scheduler / Module Hook] --> B[Spring MVC Controllers / Service Layer]

    %% Service Layer
    B --> C[Business Services (e.g., PatientService, UserService, OrderService)]

    %% Cross‑cutting concerns
    C --> D[Transaction Interceptor (Spring AOP)]
    C --> E[Authorization Advice (Annotation @Authorized)]
    C --> F[Logging Advice (Annotation @Logging)]

    %% DAO Layer
    D --> G[Hibernate DAOs (e.g., HibernatePatientDAO, HibernateUserDAO)]

    %% Persistence
    G --> H[Relational DB (MySQL/MariaDB or PostgreSQL)]
    G --> I[Elasticsearch (optional search backend)]

    %% Egress
    H --> J[Email (JavaMail)]
    I --> K[Search UI (REST calls)]
    C --> L[Module Registry (HTTP)]
    C --> M[Filesystem (data dir)]

    %% External
    A -.-> N[External HL7 Sender]
    A -.-> O[External Scheduler]
```

* **Transaction Interceptor** – defined in `applicationContext-service.xml` (bean `transactionInterceptor`) and applied via `DefaultAdvisorAutoProxyCreator` (lines 12‑18).  
* **Authorization** – enforced by `@Authorized` annotations on service methods (e.g., `PatientService.savePatient` line 23).  
* **Logging** – `@Logging` annotation on methods such as `UserService.changePassword` (line 23) triggers `LoggingAdvice`.  

---

## 6. Data Stores  

| Store | Type | Primary Content | Relevant Source |
|-------|------|----------------|-----------------|
| **MariaDB / MySQL** | Relational DB | Core tables: `patient`, `person`, `obs`, `encounter`, `order`, `concept`, `location`, `user`, `role`, `privilege`, `program`, `cohort`, etc. | `startup‑init.sh` builds JDBC URL for MySQL (line 115‑131) |
| **PostgreSQL** | Relational DB (alternative) | Same schema as MySQL | `startup‑init.sh` switches driver when `OMRS_DB=postgresql` (line 119‑124) |
| **Elasticsearch** | Document store / search index | Indexed patient, encounter, observation data for fast free‑text search | `startup‑init.sh` sets `hibernate.search.backend.type` (line 84) |
| **Filesystem** | Local disk (`$OMRS_HOME/data`) | Module JARs, OWA bundles, configuration files, complex observation binaries | `startup‑init.sh` creates directories (line 30‑45) |
| **Liquibase changelog** | Version‑controlled DB migration scripts (XML/SQL) | Schema evolution, reference data (e.g., concept dictionaries) | `liquibase` module contains changelog files (not listed but part of the repo) |

---

## 7. Deployment & Infrastructure  

| Aspect | Detail | Source |
|--------|--------|--------|
| **Docker Image** | Multi‑stage Dockerfile builds the WAR with Maven, then copies it into a Tomcat 9 base image (`tomcat:9-jdk8-corretto`). Development stage adds Tomcat 8.5 for hot‑reload. | `Dockerfile:1‑200` |
| **Runtime Container** | Exposes port 8080, runs as non‑root user (`USER 1001`). Startup scripts (`startup‑init.sh`, `startup.sh`) configure DB connection, search backend, and copy the WAR into Tomcat’s `webapps`. | `Dockerfile:140‑170`; `startup.sh:30‑70` |
| **CI/CD** | GitHub Actions builds on Ubuntu and Windows matrices (Java 8‑24) and pushes coverage to Coveralls; CodeQL runs static analysis; Scorecard evaluates supply‑chain security; Travis CI runs legacy builds on multiple JDKs. | `.github/workflows/build.yaml:12‑30`; `.travis.yml:7‑15` |
| **Local Development** | `docker‑compose.yml` spins up a MariaDB container and the OpenMRS API container; `docker‑compose.override.yml` mounts the source tree for live recompilation and exposes debug port 8000. | `docker-compose.yml:5‑30`; `docker-compose.override.yml:12‑28` |
| **Production** | `docker‑compose.yml` (no source mount) builds the WAR once, stores data in named volumes (`db-data`, `openmrs-data`). | Same file lines 5‑30 |

---

## 8. Cross‑Cutting Concerns  

| Concern | Implementation | Source |
|---------|----------------|--------|
| **Authentication & Authorization** | `Context` holds a `UserContext` (ThreadLocal) and a `ServiceContext`. Authentication scheme is pluggable (`AuthenticationScheme` bean). Methods are protected by `@Authorized` (e.g., `PatientService.savePatient`). | `Context.java:84‑106` (authentication scheme init) ; `PatientService.java:23‑30` (annotation) |
| **AOP (Transaction, Logging, Security)** | Spring AOP proxies created by `DefaultAdvisorAutoProxyCreator` and `TransactionAttributeSourceAdvisor`. Logging advice intercepts `@Logging`. | `applicationContext-service.xml:12‑22` (AOP beans) |
| **Caching** | Spring cache abstraction enabled (`<cache:annotation-driven/>`). Services may use `@Cacheable` (e.g., concept look‑ups). | `applicationContext-service.xml:70‑78` |
| **Module System** | Modules are JARs placed under `$OMRS_HOME/data/modules`. The core loads them at startup (`startup‑init.sh` copies distribution modules). Modules can register Spring beans, event listeners, and REST controllers. | `startup‑init.sh:30‑45` (module copy) |
| **Logging** | Log4j 2 configuration (via `log4j2.xml` not shown) and SLF4J bridge; custom Liquibase logger (`Slf4JLogService`). | `liquibase/ext/logging/slf4j/Slf4JLogService.java:4‑12` |
| **Internationalisation** | `MessageSourceService` provides locale‑aware messages; `LocaleUtility` loads supported locales. | `Context.java` imports `LocaleUtility` (line 46) |
| **Error Handling** | `SimpleMappingExceptionResolver` maps uncaught exceptions to `uncaughtException.jsp`. | `openmrs-servlet.xml:44‑52` |

---

## 9. Open Questions & Areas for Improvement  

| Topic | Current State | Open Questions / Potential Work |
|-------|---------------|---------------------------------|
| **Scalability** | Single‑node Tomcat + MySQL; optional Elasticsearch for search. | How to horizontally scale the web tier and DB (e.g., clustering, read replicas)? |
| **Performance** | Hibernate lazy loading; caching enabled but limited to simple look‑ups. | Could second‑level cache (e.g., Redis) improve read‑heavy workloads? |
| **User Experience** | JSP‑based UI with JSTL; limited SPA features. | Migration to a modern front‑end framework (React/Angular) while preserving API stability. |
| **Integration** | HL7 v2.x via HAPI; REST API for external systems. | Support for FHIR (Fast Healthcare Interoperability Resources) and other modern standards. |
| **Security** | Basic username/password auth; optional LDAP via custom `AuthenticationScheme`. | Implement OAuth2/OpenID Connect, improve password policies, and add audit logging for privileged actions. |
| **Testing** | Unit & integration tests run via Maven (`-Pskip-default-test -Pintegration-test`). | Increase test coverage for edge cases, add contract tests for external APIs. |
| **Module Lifecycle** | Modules loaded at startup; no hot‑reload in production. | Enable dynamic module (un)loading without container restart. |

--- 

*All citations refer to files present in the repository snapshot used for this documentation.*