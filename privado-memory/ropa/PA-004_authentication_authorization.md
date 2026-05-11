# Processing Activity: Authentication and Authorization (PA‑004)

## 1. Overview

| Field | Value |
|---|---|
| **PA Name** | Authentication and Authorization (PA‑004) |
| **Business Function** | User login, session creation, privilege enforcement for OpenMRS Core |
| **Processing Purpose** | Verify user identity and enforce role‑based access control so that only authorised personnel can view or modify patient and system data. |
| **Legal Basis** | **GDPR Art. 6 (1)(c)** – processing is necessary for compliance with a legal obligation (health‑care data protection regulations) and **Art. 9 (2)(a)** – processing of special category data (health‑care staff credentials) is allowed when necessary for the performance of a task carried out in the public interest. |
| **Controller** | Groq (the organisation that deploys and operates the OpenMRS instance) |

## 2. Data Subjects
- **Healthcare staff / system users** – doctors, nurses, administrators, data clerks and any other person who is granted a user account in OpenMRS.

## 3. Personal Data Elements

| Data Element | Source (code / DB) | Sensitivity (GDPR) | Retention |
|---|---|---|---|
| **username** | Collected from the login form; persisted in the `users` table via `UserDAO.saveUser(User, String)` (see `UserDAO.java:24‑30`). | Medium (personal data) | Retained for the lifetime of the user account; deleted only when the account is purged (`UserService.purgeUser`). |
| **password (hashed)** | Collected from the login form; hashed and stored in the `user_credentials` table by `UserDAO.saveUser(User, String)` (same file). | **High** – authentication secret (special category under Art. 9 because it enables access to health data). | Retained for the lifetime of the user account; replaced on password change (`UserService.changePassword`). |

*Both elements are stored in the relational database (MySQL/MariaDB or PostgreSQL) that backs OpenMRS Core.*

## 4. Data Flows  

```
+----------------+      +---------------------------+      +-------------------+
|  User (browser|      |  OpenMRS Authentication   |      |  Relational DB    |
|  or API client) | --> |  Scheme (UsernamePassword | -->  |  (users, user_    |
|                |      |  AuthenticationScheme)   |      |  credentials)    |
+----------------+      +---------------------------+      +-------------------+

Legend:
→  HTTP POST /openmrs/ws/rest/v1/session (login endpoint)
→  Context.authenticate(Credentials)  (Context.java:221‑227)
→  UserContext.authenticate() → AuthenticationDAO.validateCredentials()
→  UserDAO.saveUser() / UserDAO.changePassword() → INSERT/UPDATE users & user_credentials
```

**Legacy / Commented‑out flows** – None identified for this PA; all authentication paths are active.

## 5. Third Parties / Processors

| Vendor | Role | Data Shared | Hosting |
|---|---|---|---|
| *None* | No external processors are used for authentication or credential storage. All processing occurs inside the OpenMRS container and the internal relational database. | – | – |

*(If an external LDAP or SSO provider were configured, it would be listed here, but the default OpenMRS deployment uses the built‑in DAO‑based scheme.)*

## 6. Security Measures (PA‑specific)

### Positive Controls
| Control | Implementation Detail |
|---|---|
| **Password hashing** | Passwords are never stored in clear text; `UserDAO.saveUser` hashes the password using BCrypt (Spring Security’s `PasswordEncoder`). |
| **Transport security** | Login requests must be made over HTTPS (Tomcat is typically configured with TLS; OpenMRS documentation recommends `https://` for all UI/API traffic). |
| **Role‑Based Access Control (RBAC)** | Privilege checks are enforced by `Context.hasPrivilege(String)` and `Context.requirePrivilege(String)` (see `Context.java:542‑549` and `558‑574`). |
| **Session isolation** | `Context.openSession()` creates a thread‑local `UserContext` that isolates each authenticated user (see `Context.java:417‑424`). |
| **Audit logging** | Successful and failed authentication attempts are logged via SLF4J/Logback (`logback.xml` in the Docker image). |
| **Account lockout / throttling** | Not built‑in but can be added via a custom `AuthenticationScheme` module; flagged as a **concern** below. |

### Concerns / Recommendations
- **Brute‑force protection** – The default authentication scheme does not implement account lockout or exponential back‑off. Deploy a custom `AuthenticationScheme` or enable a reverse‑proxy rate‑limiting rule.
- **Password policy enforcement** – Ensure `UserService.changePassword` validates length, complexity, and reuse (currently only length is enforced). Consider adding a password‑strength validator.
- **Credential exposure in logs** – Verify that no password values are ever logged; the `@Logging` annotation on `SerializationService.deserialize` demonstrates awareness, but audit the entire code base for accidental logging of `Credentials`.

## 7. Cross‑Border Transfers
- No data is transferred outside the host environment. All authentication data stays within the Docker container and the internal relational database, which are provisioned in the same jurisdiction as the controller.

## 8. DPIA Trigger Assessment

| Factor | Applicable? | Comments |
|---|---|---|
| Large‑scale processing | **No** – limited to the number of system users (typically < 10 000). |
| Processing of **special‑category data** (Art. 9) | **Yes** – passwords enable access to health records, which are special‑category data. |
| Automated decision‑making | **No** – authentication is a binary allow/deny check, not a profiling activity. |
| Systematic monitoring of data subjects | **Yes** – authentication logs are kept for security monitoring. |
| Vulnerable data subjects | **Yes** – health‑care staff may be considered vulnerable in the context of professional confidentiality. |
| Use of new or emerging technologies | **No** – standard Java/Spring authentication. |
| Cross‑border transfers | **No** – all processing stays on‑premises. |
| **DPIA Recommended** | **Yes** – because special‑category data (passwords) are processed and systematic monitoring (audit logs) occurs. A DPIA should document the adequacy of hashing, access controls, and any additional safeguards (e.g., rate limiting). |

---

**Prepared by:** Groq Privacy & Compliance Team  
**Date:** 2026‑05‑11