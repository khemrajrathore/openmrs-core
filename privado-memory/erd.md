# Engineering Doc

## System overview
OpenMRS Core is a Java‑based, patient‑centric electronic medical record (EMR) platform that runs as a web application inside a servlet container (Tomcat).  It stores demographic, clinical, and encounter data for patients and exposes that data through a REST‑style API and a server‑side JSP/Velocity web UI.  The system is used by health‑care providers in low‑resource settings worldwide and is typically deployed via Docker (or Docker‑Compose) on a single host or a small cluster, backed by a relational database (MySQL/MariaDB or PostgreSQL) and optionally an Elasticsearch node for full‑text search. The current live codebase (v2.8‑SNAPSHOT) is built with Maven and runs on Java 8+ (`api/src/main/java/org/openmrs/Address.java:1`).  

## Tech stack & runtime
| Layer | Technology | Source |
|-------|------------|--------|
| Language | Java 8+ (`api/src/main/java/org/openmrs/Address.java:1`) |
| Build | Maven 3.8 (`api/pom.xml:1`) |
| Dependency Injection / AOP | Spring Framework (`api/src/main/java/org/openmrs/api/context/ServiceContext.java:1`) |
| ORM | Hibernate 5 (`api/src/main/java/org/openmrs/api/db/hibernate/HibernateContextDAO.java:1`) |
| DB Migration | Liquibase with custom extensions (`api/src/main/java/liquibase/ext/change/core/InsertWithUuidDataChange.java:1`) |
| Search | Hibernate Search (Lucene & optional Elasticsearch) (`api/pom.xml:152‑161`) |
| Web UI | JSP + Velocity (`webapp/src/main/webapp/WEB-INF/web.xml:1`) |
| REST API | Spring MVC controllers (`web/src/main/java/org/openmrs/web/controller/PatientController.java:1`) |
| Caching | Spring Cache (Ehcache) (`api/src/main/java/org/openmrs/api/cache/OpenmrsCacheConfiguration.java:1`) |
| Logging | SLF4J + Logback (`api/src/main/java/liquibase/ext/logging/slf4j/Slf4JLogger.java:1`) |
| Container runtime | Docker (`Dockerfile:1`) |
| Orchestration (optional) | Docker‑Compose (`docker-compose.yml:1`) |

## Ingress
### HTTP(S) requests
* **Web UI** – All browser traffic hits the Tomcat servlet container (`startup.sh:31‑34`). The `OpenmrsFilter` (`web/src/main/java/org/openmrs/web/filter/OpenmrsFilter.java:1`) is the first filter in `web.xml` and wraps each request with a `UserContext`.
* **REST API** – Controllers under `org.openmrs.web.controller.*` (`web/src/main/java/org/openmrs/web/controller/PatientController.java:1`) expose JSON endpoints under `/openmrs/ws/rest/v1/*`.
* **Health check** – `/health/started` is handled by `OpenmrsFilter` (`web/src/main/java/org/openmrs/web/filter/StartupFilter.java:71‑78`).

### Startup / installation wizard
* **InitializationFilter** – The very first request after container start is intercepted by `InitializationFilter` (`web/src/main/java/org/openmrs/web/filter/initialization/InitializationFilter.java:1`) which presents the installation wizard when no runtime properties exist.

### External services
* **Elasticsearch** – When `OMRS_SEARCH=elasticsearch` (`startup-init.sh:71‑73`) the application contacts the ES container (`docker-compose.es.yml:13‑22`) for indexing and query.

## Egress
* **Relational database** – All persistent entities are stored via Hibernate to MySQL/MariaDB or PostgreSQL (`api/src/main/java/org/openmrs/api/db/hibernate/HibernateContextDAO.java:1`). The JDBC URL is built in `startup-init.sh` (`startup-init.sh:84‑106`).
* **Elasticsearch** – If enabled, search queries are sent to the ES node (`startup-init.sh:71‑73`).
* **File system** – Uploaded files, module JARs, OWA resources, and configuration files are written under `${OMRS_HOME}/data` (`startup-init.sh:44‑48`).
* **Email / notifications** – `MailMessageSender` (`api/src/main/java/org/openmrs/notification/mail/MailMessageSender.java:1`) sends SMTP messages for alerts and reminders.

## Internal topology
OpenMRS is organized around a **Service‑Context** that provides access to domain services (PatientService, UserService, ConceptService, etc.). Each service delegates to a DAO that uses Hibernate to talk to the relational DB. Filters (`OpenmrsFilter`, `StartupFilter`, `InitializationFilter`) manage request‑level concerns (authentication, locale, installation). The search layer (`Hibernate Search`) proxies queries to Lucene or Elasticsearch. The diagram below reflects the actual runtime flow.

```mermaid
flowchart TD
    %% Ingress
    subgraph Ingress
        UI[Web UI (JSP/Velocity)] -->|HTTP| Tomcat
        API[REST API (JSON)] -->|HTTP| Tomcat
        Health[Health check /health/started] -->|HTTP| Tomcat
        InitWizard[Installation Wizard] -->|HTTP| Tomcat
    end

    %% Core filters
    Tomcat --> OpenmrsFilter[OpenmrsFilter] --> StartupFilter[StartupFilter]
    StartupFilter -->|first‑run| InitializationFilter[InitializationFilter]
    StartupFilter -->|normal| ServiceContext[ServiceContext]

    %% Services & DAOs
    ServiceContext --> PatientService[PatientService]
    ServiceContext --> UserService[UserService]
    ServiceContext --> ConceptService[ConceptService]
    PatientService --> PatientDAO[PatientDAO]
    UserService --> UserDAO[UserDAO]
    ConceptService --> ConceptDAO[ConceptDAO]

    %% Persistence
    PatientDAO -->|Hibernate| RelDB[(Relational DB<br/>MySQL/PostgreSQL)]
    UserDAO --> RelDB
    ConceptDAO --> RelDB

    %% Search
    ServiceContext --> Search[Hibernate Search]
    Search -->|Lucene| Lucene[(Lucene Index)]
    Search -->|Elasticsearch| ES[(Elasticsearch)]

    %% Egress
    RelDB --> FileStore[(File System<br/>${OMRS_HOME}/data)]
    ES --> External[External ES Service]

    %% Outbound notifications
    ServiceContext --> MailSender[MailMessageSender]
    MailSender --> SMTP[(SMTP Server)]

    %% Flow direction
    UI --> OpenmrsFilter
    API --> OpenmrsFilter
    Health --> OpenmrsFilter
    InitWizard --> OpenmrsFilter
```

## Data stores
| Store | What is persisted | Source |
|-------|-------------------|--------|
| **Relational DB** (MySQL/MariaDB or PostgreSQL) | Core domain entities: Patient, Person, Encounter, Obs, Concept, Order, etc. (`api/src/main/java/org/openmrs/api/db/hibernate/Patient.hbm.xml:1`, `api/src/main/java/org/openmrs/api/db/hibernate/Concept.hbm.xml:1`) |
| **Elasticsearch** (optional) | Full‑text indexes for concepts, patients, observations (`startup-init.sh:71‑73`) |
| **File system** (`${OMRS_HOME}/data`) | Uploaded binaries, module JARs, OWA resources, configuration files (`startup-init.sh:44‑48`) |
| **Cache** (Ehcache) | Frequently accessed metadata (concepts, global properties) (`api/src/main/java/org/openmrs/api/cache/OpenmrsCacheConfiguration.java:1`) |
| **Log files** | Application logs written via SLF4J/Logback (`api/src/main/java/liquibase/ext/logging/slf4j/Slf4JLogger.java:1`) |

## Deployment & infrastructure
* **Docker image** – Multi‑stage Dockerfile builds the WAR with Maven (`Dockerfile:1‑84`) and produces two runtime images: a development image (Tomcat 8.5, `Dockerfile:86‑138`) and a production image (`Dockerfile:140‑210`).
* **Docker‑Compose** – `docker-compose.yml` defines a MariaDB service (`docker-compose.yml:13‑23`) and the OpenMRS API container (`docker-compose.yml:25‑44`). An optional Elasticsearch compose file (`docker-compose.es.yml`) adds an ES node (`docker-compose.es.yml:13‑22`).
* **CI/CD** – GitHub Actions workflows (`.github/workflows/build.yaml:1‑30`, `.github/workflows/build-2.x.yaml:1‑30`) compile, test, and publish coverage; CodeQL (`.github/workflows/codeql-analysis.yml:1‑30`) runs static analysis; Scorecard (`.github/workflows/scorecard.yml`) checks supply‑chain security.
* **Environment variables** – All runtime configuration is driven by `startup-init.sh` variables (`startup-init.sh:13‑45`), e.g., `OMRS_DB_HOSTNAME`, `OMRS_DB_NAME`, `OMRS_SEARCH`, `OMRS_ADMIN_USER_PASSWORD`. Secrets (DB passwords, admin password) are injected via Docker‑Compose or CI secret stores.
* **Port exposure** – Tomcat listens on 8080 (`Dockerfile:165‑166`), the health endpoint is reachable on the same port.

## Cross‑cutting concerns
| Concern | Implementation | Source |
|---------|----------------|--------|
| **Authentication** | `UserContext` created per HTTP session (`OpenmrsFilter.java:45‑58`); `Context.authenticate(username,password)` (`api/src/main/java/org/openmrs/api/context/Context.java:84‑92`) |
| **Authorization** | `@Authorized` annotations on service methods (`PatientService.java:13‑15`, `UserService.java:13‑15`) and `TransactionAttributeSourceAdvisor` in `applicationContext.xml` (`APP_CONTEXT:19‑27`) |
| **Transaction management** | Spring declarative transactions via `@Transactional` and the `transactionInterceptor` bean (`APP_CONTEXT:31‑44`) |
| **Logging** | SLF4J logger per class (`OpenmrsFilter.java:9‑12`); Logback configuration in `logback.xml` (not listed but part of the WAR) |
| **Internationalization** | `LocaleUtility` and per‑session locale set in `OpenmrsFilter` (`OpenmrsFilter.java:71‑78`) |
| **Feature flags / configuration** | All runtime properties are read from `${OMRS_HOME}/${OMRS_WEBAPP_NAME}-server.properties` (`startup-init.sh:106‑124`). Additional `OMRS_EXTRA_*` env vars are appended (`startup-init.sh:138‑152`). |
| **Observability** | Health endpoint (`/health/started`) (`OpenmrsFilter.java:71‑78`); JVM metrics via standard JMX (exposed by Tomcat). |
| **Security** | CSRF guard filter (comment in `OpenmrsFilter.java:84‑88`), password policies enforced in `UserService.changePassword` (`UserService.java:31‑38`). |

## Open questions
* **Scalability** – The current architecture runs a single Tomcat instance with a single relational DB. Horizontal scaling (multiple app nodes, load balancer, distributed cache) is not evident in the source.
* **Module lifecycle** – While the module directory is mounted (`startup-init.sh:44‑48`), the exact runtime loading/unloading mechanism (OSGi‑style vs. Spring) is not fully visible in the provided files.
* **Search backend selection** – The code supports both Lucene and Elasticsearch, but the decision logic and fallback behavior are not completely clear from the snippets.
* **Backup & disaster recovery** – No explicit backup scripts or snapshot mechanisms are present in the repository.
* **Metrics & tracing** – Apart from the health endpoint, there is no built‑in Prometheus or OpenTelemetry exporter; integration would need to be added.