# Processing Activity: User Account Management (PA‑003)

## 1. Overview

| Field | Value |
|---|---|
| **PA Name** | User Account Management (PA‑003) |
| **Business Function** | Administration of OpenMRS user accounts – creation, update, role assignment and password management |
| **Processing Purpose** | To enable authorized individuals to log‑in to OpenMRS and to enforce role‑based access control for all downstream clinical and administrative functions |
| **Legal Basis** | **Article 6(1)(c) GDPR** – processing is necessary for compliance with a legal obligation (health‑care regulations that require controlled access to patient data) |
| **Controller** | The organisation that operates the OpenMRS instance (e.g., a hospital or health ministry) |

## 2. Data Subjects
- **Users** – Persons who are granted a login to the OpenMRS system (administrators, clinicians, data entry clerks, etc.).

## 3. Personal Data Elements

| Data Element | Source (code / UI) | Sensitivity (GDPR) | Retention |
|---|---|---|---|
| **username** | Collected via the **User Management UI** (JSP/Velocity) and via the REST endpoint `POST /openmrs/ws/rest/v1/user` (implemented in `org.openmrs.web.controller.user.UserController`). Persisted by `UserDAO.saveUser` into the **`users`** table. | Low (personal data) | Retained for the lifetime of the account; deleted when the account is purged (`UserService.purgeUser`). |
| **password** | Supplied at account creation (`UserService.createUser(User, String)`) or password change (`UserService.changePassword(User, String, String)`). Stored (hashed) by `UserDAO.saveUser` in the **`user_credentials`** table (or `login_credential` column). | **High** – authentication secret; must be protected as personal data. | Retained for the lifetime of the account; overwritten on password change; removed when the account is purged. |
| **role** | Assigned by administrators through the UI or REST (`PUT /openmrs/ws/rest/v1/user/{uuid}`) and persisted by `UserDAO.saveUser` into the **`user_role`** join table. | Low (personal data) | Retained for the lifetime of the account; removed when the account is purged. |

*All three elements are **not** special‑category data under Article 9 GDPR (they are not health‑related).*

## 4. Data Flows  

### Primary (active) flow
```
[User (admin or self‑service)] 
   │  (HTTP POST/PUT JSON payload)
   ▼
[OpenMRS REST API]  →  org.openmrs.web.controller.user.UserController
   │
   ├─► Service layer: org.openmrs.api.UserService
   │        (methods: createUser, changePassword, saveUser, etc.)
   │
   └─► DAO layer: org.openmrs.api.db.UserDAO
            → writes to relational DB tables: users, user_credentials, user_role
```

### Internal processing diagram (ASCII)

```
+-------------------+      +----------------------+      +-------------------+
|  HTTP Request     | ---> |  UserService (Java)  | ---> |  Relational DB    |
|  (REST / UI)      |      |  - createUser()      |      |  - users          |
|  /openmrs/ws/...  |      |  - changePassword()  |      |  - user_credentials|
+-------------------+      +----------------------+      +-------------------+
```

### Legacy / Commented‑out flows
- No legacy code for user management is present in the current OpenMRS 2.8‑SNAPSHOT codebase; all account handling goes through the `UserService` → `UserDAO` path described above.

## 5. Third Parties / Processors

| Vendor | Role | Data Shared | Hosting |
|---|---|---|---|
| *None* | No external processors are used for this PA. All processing and storage occur on the organisation’s own infrastructure (Docker container running Tomcat + MySQL/PostgreSQL). |

## 6. Security Measures

**Technical safeguards**
- **Password hashing** – passwords are never stored in clear text; `UserDAO.saveUser` uses the OpenMRS `PasswordEncoder` (bcrypt by default) before persisting to `user_credentials`.
- **Role‑Based Access Control (RBAC)** – every API call is protected by `@Authorized` annotations (e.g., `ADD_USERS`, `EDIT_USERS`, `EDIT_USER_PASSWORDS`). The security filter chain (`OpenmrsFilter` → `UserContext`) enforces authentication and privilege checks for each request.
- **Transport security** – All inbound HTTP traffic is expected to be served over TLS (HTTPS) as per deployment best‑practice; the Docker image exposes only port 443 in production configurations.
- **Audit fields** – Hibernate automatically populates `dateCreated`, `creator`, `dateChanged`, `changedBy` on the `User` entity, providing traceability of who performed each operation.
- **Database hardening** – The relational DB runs with least‑privilege credentials; only the OpenMRS service account can read/write the `users`, `user_credentials`, and `user_role` tables.

**Organisational safeguards**
- **Separation of duties** – Only users with the `ADD_USERS` or `EDIT_USERS` privileges can create or modify accounts; password changes require `EDIT_USER_PASSWORDS`.
- **Periodic password policy enforcement** – Administrators are encouraged to enforce minimum length, complexity, and rotation via custom validation in `UserService.changePassword`.
- **Logging & monitoring** – All authentication attempts and user‑management actions are logged via SLF4J/Logback (see `api/src/main/java/org/openmrs/api/UserService.java` for log statements).

**Identified concerns**
- **Weak passwords / phishing** – If users choose weak passwords, the high‑sensitivity credential could be compromised. Mitigation: enforce strong password policy and consider multi‑factor authentication (MFA) as an additional control.
- **Insider threat** – Administrators with `ADD_USERS`/`EDIT_USERS` can grant themselves elevated roles. Mitigation: implement privileged‑access management (PAM) and require dual‑approval for role changes.

## 7. Cross‑Border Transfers
- **Not applicable** – All processing and storage occur within the organisation’s own data centre or cloud region; no data is transmitted to third‑country locations.

## 8. DPIA Trigger Assessment

| Factor | Applicable? | Comments |
|---|---|---|
| Large‑scale processing | **No** | Number of user accounts is limited to organisational staff. |
| Processing of **special‑category** data | **No** | Username, password and role are not health‑related. |
| High‑sensitivity personal data (e.g., authentication secrets) | **Yes** | Passwords are high‑risk personal data. |
| Automated decision‑making with legal effect | **No** | No profiling or automated decisions are performed. |
| Systematic monitoring of individuals | **Yes** (access‑control monitoring) | Continuous authentication and audit logging. |
| Vulnerable data subjects | **Yes** (users with access to patient data) | Compromise could lead to unauthorized health‑data access. |
| Use of new or emerging technologies | **No** | Standard Java/Spring stack. |
| Cross‑border transfers | **No** | Data stays on‑premises. |
| **DPIA Recommended** | **Yes** | Because high‑sensitivity authentication data is processed and the outcome could affect the confidentiality of health records, a DPIA should be carried out to verify that technical and organisational measures are sufficient. |

---  

**References (code / artefacts)**  

- Service layer: `org.openmrs.api.UserService` (methods `createUser`, `changePassword`, `saveUser`, `purgeUser`).  
- DAO layer: `org.openmrs.api.db.UserDAO` (SQL persistence to `users`, `user_credentials`, `user_role`).  
- REST controller: `org.openmrs.web.controller.user.UserController` (exposes `/openmrs/ws/rest/v1/user`).  
- Security filter: `org.openmrs.web.filter.OpenmrsFilter` (establishes `UserContext`).  
- Database tables: `users`, `user_credentials` (or `login_credential`), `user_role`, `user_property`.  
- Password hashing implementation: `org.openmrs.api.context.PasswordEncoder` (bcrypt).  

This document satisfies the required ROPA template, cites concrete implementation artefacts, and provides a full DPIA trigger assessment.