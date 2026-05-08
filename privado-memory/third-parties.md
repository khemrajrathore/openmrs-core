**privado‑memory/third‑parties.md**

---

## Vendors & Services  

| Vendor / Service | Purpose / Role in OpenMRS | SDK / Library (artifact) | Resolved Domain / URL | Data Shared |
|------------------|---------------------------|--------------------------|-----------------------|-------------|
| **Mozilla** | License text (MPL 2.0) used in source file headers | N/A (text reference) | <http://mozilla.org/MPL/2.0/> | – |
| **OpenMRS** | Project‑level licensing & disclaimer | N/A (text reference) | <http://openmrs.org/license> | – |
| **HL7 / HAPI** | Parsing, building and handling HL7 messages (e.g., ADT, ORM) | `ca.uhn.hapi:hapi-base`, `hapi-structures-v2` | <https://repo1.maven.org/maven2/ca/uhn/hapi/> | – |
| **Hibernate ORM** | Object‑relational mapping, persistence layer | `org.hibernate:hibernate-core` | <https://repo1.maven.org/maven2/org/hibernate/> | – |
| **Hibernate Search (Lucene / Elasticsearch)** | Full‑text indexing & search of clinical data | `org.hibernate.search:hibernate-search-orm` (Lucene) / `hibernate-search-backend-elasticsearch` | <https://repo1.maven.org/maven2/org/hibernate/search/> | – |
| **Liquibase** | Database change‑management (schema migrations) | `org.liquibase:liquibase-core` | <https://repo1.maven.org/maven2/org/liquibase/> | – |
| **Spring Framework** | Core application framework (DI, transaction management, MVC, etc.) | `org.springframework:spring-context`, `spring-webmvc`, … | <https://repo1.maven.org/maven2/org/springframework/> | – |
| **Jackson** | JSON serialization / deserialization (used by REST API) | `com.fasterxml.jackson.core:jackson-databind` | <https://repo1.maven.org/maven2/com/fasterxml/jackson/core/> | – |
| **Apache Velocity** | Templating engine for generating dynamic content (e.g., reports) | `org.apache.velocity:velocity-engine-core` | <https://repo1.maven.org/maven2/org/apache/velocity/> | – |
| **JSTL** | JSP Standard Tag Library – UI rendering helpers | `javax.servlet:jstl` | <https://repo1.maven.org/maven2/javax/servlet/jstl/> | – |
| **SLF4J / Logback** | Logging façade & implementation | `org.slf4j:slf4j-api`, `ch.qos.logback:logback-classic` | <https://repo1.maven.org/maven2/org/slf4j/> | – |
| **Snyk** | Automated security‑vulnerability scanning (CI step) | N/A (Snyk CLI / GitHub Action) | <https://snyk.io/> | – |
| **Codacy** | Static code analysis / quality reporting (CI step) | N/A (Codacy CLI / GitHub Action) | <https://www.codacy.com/> | – |
| **Travis CI** | Continuous‑integration service (build & test) | N/A (Travis service) | <https://travis-ci.org/> | – |
| **Coveralls** | Test‑coverage reporting (CI step) | N/A (Coveralls service) | <https://coveralls.io/> | – |
| **Maven Central** | Primary artifact repository for all third‑party libraries | N/A (repository) | <https://repo1.maven.org/maven2/> | – |
| **OpenMRS Implementation‑ID Service** | Registers a unique implementation identifier for a running instance (used for analytics / support) | N/A (REST call) | <https://openmrs.org> | Implementation UUID, version, host name (no patient‑level data) |

*All of the above libraries are bundled with the OpenMRS server and run locally; they do **not** receive patient‑level or other clinical data from the OpenMRS instance.*

---

## External APIs Consumed  

| API | Provider | Purpose | Data Sent |
|-----|----------|---------|-----------|
| **Implementation‑ID registration** | OpenMRS (openmrs.org) | When a server starts it may POST a small payload containing a generated UUID, OpenMRS version, and host information to the central OpenMRS service. This helps the OpenMRS community track installations for support and analytics. | Implementation UUID, version, host name (no PHI) |
| **(None other)** | – | The core platform does not call external SaaS APIs (e.g., payment gateways, messaging services). All clinical processing stays within the host. |

---

## Webhooks  

*OpenMRS Core does not expose or consume any external webhook endpoints.*  

---

## Resolved Domains & URLs  

- <http://mozilla.org/MPL/2.0/> – MPL 2.0 license reference.  
- <http://openmrs.org/license> – OpenMRS Healthcare Disclaimer.  
- <https://repo1.maven.org/maven2/> – Maven Central repository (hosts all third‑party JARs).  
- <https://openmrs.org> – Implementation‑ID registration endpoint and general project site.  
- <https://snyk.io/> – Security‑scan service (used in CI).  
- <https://www.codacy.com/> – Code‑quality analysis service (used in CI).  
- <https://travis-ci.org/> – Continuous‑integration service (used in CI).  
- <https://coveralls.io/> – Test‑coverage reporting service (used in CI).  

---

### How the list was derived  

1. **Grep of source files** – Looked for strings that reference external licenses, URLs, or third‑party classes (e.g., `http://mozilla.org/MPL/2.0/`, `http://openmrs.org/license`).  
2. **`pom.xml` inspection** – Parsed the Maven descriptor (`api/pom.xml`) to enumerate all declared dependencies, then mapped each artifact to its vendor and typical purpose (ORM, search, DB migrations, JSON, templating, etc.).  
3. **Search for known integration points** – Scanned for HL7/HAPI imports, Hibernate Search usage, and any REST calls that target domains outside the local host. Only the Implementation‑ID registration was found.  
4. **CI/CD configuration files** – Checked `.travis.yml`, GitHub Actions, and other CI scripts for references to external services (Snyk, Codacy, Coveralls).  
5. **Cross‑checked with known OpenMRS documentation** – Confirmed that OpenMRS is primarily self‑hosted; therefore most “third‑party” items are libraries rather than SaaS providers.  

All identified third‑party components are listed above, together with the minimal data they may exchange (none of them transmit patient‑level data). This satisfies the request to produce a `privado‑memory/third‑parties.md` file summarising OpenMRS Core’s third‑party integrations.