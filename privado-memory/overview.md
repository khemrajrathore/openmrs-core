# privado‑memory/overview.md

## Repo Type
OpenMRS Core is an **open‑source, Java‑based Electronic Medical Record (EMR) system**.  
It is a Maven multi‑module project that provides the core data model, services, and web UI for managing clinical, administrative, and reporting data in health‑care settings.

---

## Tech Stack  

| Layer | Technology | Version / Reference | Key Packages / Paths |
|-------|------------|---------------------|----------------------|
| Language | **Java** | Java 8 (source/target set in the parent POM) | `src/main/java/**/*.java` |
| Build | **Maven** | 4.0.0 POM model (see `pom.xml`) | `pom.xml`, `api/pom.xml`, `web/pom.xml`, … |
| Dependency Management | **Spring Framework** | `${springVersion}` (defined in the parent POM; typically 5.x for the 2.8‑SNAPSHOT line) | `org.springframework:spring-core`, `org.springframework:spring-test` (see `api/pom.xml`) |
| ORM / DB Access | **Hibernate** | 5.x (bundled via `org.hibernate:hibernate-core` in the parent) | `org.openmrs.api.db.hibernate.*` (e.g., `HibernatePatientDAO.java`) |
| Database Migration | **Liquibase** | 4.x (custom extensions under `liquibase/`) | `org.openmrs.liquibase.*` |
| Logging | **SLF4J / Logback** (via custom `Slf4JLogService`) | – | `org.openmrs.logging.*` |
| Utility Libraries | **Apache Commons IO** `2.19.0`, **Commons Lang3** `3.17.0`, **Commons Collections** `3.2.2` | – | `org.apache.commons.*` |
| Web UI | **Spring MVC**, **JSP**, **Velocity** (`velocity-1.7`, `velocity-tools-2.0.1-atlassian-2`) | – | `web/src/main/java/org/openmrs/web/*`, `webapp/src/main/webapp/WEB-INF/*.jsp` |
| Container | **Servlet API** `4.0.1` (provided) | – | `javax.servlet.*` |
| Clinical Standards | **HL7 v2.x** (message parsing, handling) | – | `org.openmrs.hl7.*`, `org.openmrs.hl7.handler.*` |
| Clinical Standards | **FHIR** (supported via optional modules; core contains FHIR‑related DTOs) | – | `org.openmrs.module.fhir2.*` (in separate module, referenced from core) |

---

## Architecture  

1. **Modular Maven Structure** – The top‑level `pom.xml` aggregates six modules: `tools`, `test`, `api`, `web`, `webapp`, and `liquibase`. Each module is independently versioned and can be built or packaged alone.  

2. **Layered Service‑Oriented Design**  
   - **Domain Model** – POJOs under `api/src/main/java/org/openmrs/` (e.g., `Patient.java`, `Encounter.java`, `Obs.java`).  
   - **DAO Layer** – Interfaces in `org.openmrs.api.db.*` and Hibernate implementations in `org.openmrs.api.db.hibernate.*`.  
   - **Service Layer** – Interfaces in `org.openmrs.api.*` (e.g., `PatientService`, `EncounterService`) with implementations in `org.openmrs.api.impl.*`.  
   - **Handler / Save‑Handler Pattern** – Centralised business‑logic hooks (e.g., `BaseVoidHandler`, `PersonSaveHandler`) located in `api/src/main/java/org/openmrs/api/handler/`.  

3. **Persistence** – Hibernate maps the domain model to relational tables; custom interceptors (`ImmutableObsInterceptor`, `ImmutableOrderInterceptor`) enforce immutability rules.  

4. **Clinical Messaging** – HL7 message parsing is performed by classes in `api/src/main/java/org/openmrs/hl7/` and `api/src/main/java/org/openmrs/hl7/handler/`.  

5. **Extensibility** – New modules can plug into the core via the OpenMRS module framework (not shown in the tree but referenced in the parent POM).  

---

## Code Organization  

```
openmrs-core/
├─ api/                     # Core API, data model, services, DAOs
│   ├─ src/main/java/
│   │   └─ org/openmrs/
│   │       ├─ Address.java
│   │       ├─ Patient.java
│   │       ├─ api/                # Service interfaces
│   │       ├─ api/impl/           # Service implementations
│   │       ├─ api/db/             # DAO interfaces
│   │       ├─ api/db/hibernate/   # Hibernate DAO implementations
│   │       ├─ hl7/                # HL7 message handling
│   │       ├─ logging/            # Logging utilities
│   │       └─ … (≈ 300+ domain classes)
│   └─ src/main/resources/        # Spring config, messages, etc.
├─ web/                     # Spring MVC controllers, UI helpers
│   └─ src/main/java/org/openmrs/web/
├─ webapp/                  # WAR packaging, JSPs, static assets
│   └─ src/main/webapp/WEB-INF/
├─ liquibase/               # Database change‑sets and custom Liquibase extensions
├─ test/                    # Unit & integration tests
└─ tools/                   # Build helpers, code generators, scripts
```

- **Domain Packages** – `org.openmrs.*` (e.g., `org.openmrs.Patient`, `org.openmrs.Obs`).  
- **Service Packages** – `org.openmrs.api.*` (interfaces) and `org.openmrs.api.impl.*` (implementations).  
- **DAO Packages** – `org.openmrs.api.db.*` and `org.openmrs.api.db.hibernate.*`.  
- **HL7 Packages** – `org.openmrs.hl7.*`, `org.openmrs.hl7.handler.*`.  
- **FHIR (optional)** – Not in the core tree but referenced via `org.openmrs.module.fhir2.*` when the FHIR module is added.  

---

## Build & Deployment  

| Step | Tool / Command | Details |
|------|----------------|---------|
| **Compile & Package** | `mvn clean install` (run at repository root) | Compiles all modules, runs unit tests, produces a `openmrs-webapp-2.8.0‑SNAPSHOT.war` and a `openmrs-api-2.8.0‑SNAPSHOT.jar`. |
| **Continuous Integration** | **GitHub Actions** (`.github/workflows/*.yaml`) – `build.yaml`, `build-2.x.yaml` | Executes Maven build, runs tests, performs code‑quality scans (CodeQL, Scorecard). |
| **Docker Development** | `docker-compose -f docker-compose.yml up` | Uses `Dockerfile` (Java 8 base) and `docker-compose.yml` to spin up a MySQL/PostgreSQL container plus the OpenMRS webapp container. |
| **Deployment Artifact** | WAR file (`webapp/target/openmrs-webapp-*.war`) | Deployable to any servlet container (Tomcat, Jetty, etc.). |
| **Database Migration** | Liquibase runs on startup (via `LiquibaseProvider`) | Applies change‑sets located under `liquibase/src/main/resources/liquibase/`. |
| **Testing** | `mvn test` (module `test`) | Executes JUnit tests, integration tests, and uses the `openmrs-test` test‑jar for mock services. |
| **Versioning** | Maven coordinates `org.openmrs:openmrs:2.8.0‑SNAPSHOT` | All sub‑modules inherit this version. |

**Key Build Files**  
- Root `pom.xml` – defines modules, dependencyManagement, and plugin versions.  
- `api/pom.xml` – pulls in Spring Core, Hibernate, and the OpenMRS API dependencies.  
- `web/pom.xml` – adds Spring MVC, Velocity, and servlet/JSP dependencies.  
- `Dockerfile` – builds a minimal OpenJDK‑8 image with the compiled WAR.  

---  

*This overview captures the concrete packages, versions, and folder paths observed in the current snapshot of the OpenMRS Core repository, highlighting its role as a standards‑compliant (HL7, optional FHIR) healthcare EMR.*