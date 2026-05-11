# Record of Processing Activities (ROPA) – OpenMRS Core  
**Processing Activity:** **PA‑006 – User/Staff Account Management**  

---

## 1. Overview  

| **PA Name** | **Business Function** | **Processing Purpose** | **Legal Basis (GDPR)** | **Controller** |
|-------------|----------------------|------------------------|------------------------|----------------|
| PA‑006: User/Staff Account Management | Management of system users, roles and privileges | • Enable authenticated access to the EMR <br>• Enforce role‑based authorisation <br>• Support auditability of who performed which clinical actions | **Art. 6(1)(c)** – processing necessary for compliance with a legal obligation (health‑care provider must control access to patient data) <br>**Art. 9(2)(h)** – processing of special‑category data (health‑care staff health information may be stored in user profiles) | OpenMRS Core Development Team (acting on behalf of the health‑care organisation that deploys the instance) |

---

## 2. Data Subjects  

- **Patients** – vulnerable individuals whose clinical records are protected by the system.  
- **Healthcare providers** – doctors, nurses, allied health staff who need an account to record care.  
- **System administrators / IT staff** – users who manage the OpenMRS installation and its security configuration.  

---

## 3. Personal Data Elements  

| **Data Element** | **Source (Domain class / Table)** | **Sensitivity** | **Retention** |
|------------------|-----------------------------------|-----------------|----------------|
| User ID | `org.openmrs.User` → `users.id` (PK) | High (linkable identifier) | While the user account is active; archived audit records retained **7 years** after de‑provisioning (per organisational policy). |
| Username | `org.openmrs.User.username` → `users.username` | Medium (identifying) | While the user account is active; retained with audit for **7 years** after deletion. |
| Hashed Password | `org.openmrs.User.password` → `users.password` (BCrypt hash) | High (credential) | While the user account is active; old password hashes are removed immediately on password change. |
| Role IDs | `org.openmrs.Role` → `roles.role_id` and linking table `user_role` | Medium (authorisation) | While the role is assigned to a user; role definitions retained as long as the system is in use. |
| Privilege IDs | `org.openmrs.Privilege` → `privileges.privilege_id` and linking table `role_privilege` | Medium (authorisation) | While the privilege is assigned to a role; privilege definitions retained as long as the system is in use. |
| Account status (enabled/disabled, voided flag) | `users.voided`, `users.disabled` | Medium | While the account exists; retained with audit for **7 years** after de‑provisioning. |
| Audit metadata (who created/modified the user, timestamps) | Hibernate Envers tables `users_AUD` | High (audit) | **7 years** (or longer if required by national health‑care regulations). |

---

## 4. Data Flows  

### 4.1. Core CRUD flow (ASCII)

```
+----------------+        +----------------+        +-------------------+
|   Admin UI /   |        |   UserService  |        |   MySQL / MariaDB |
|   REST client  | ---->  | (api/src/main/ | ---->  |   tables:         |
|                |        | java/org/openmrs/api/UserService.java) |
+----------------+        +----------------+        |   - users          |
                                                   |   - user_role      |
                                                   |   - roles          |
                                                   |   - role_privilege |
                                                   +-------------------+
        ^                         |                     |
        |                         |                     |
        |   <--- success/failure <---                 |
        +---------------------------------------------+
```

### 4.2. Authentication read‑only flow  

```
+----------------+        +----------------+        +-------------------+
|   Login page   | ---->  |  Authentication| ---->  |   MySQL / MariaDB |
| (Spring MVC)   |        |  Manager (org.|        |   tables:         |
|                |        |  openmrs.api   |        |   - users          |
+----------------+        |  .context.Context) |   |   - user_role      |
                          +----------------+        |   - roles          |
                                                    +-------------------+
        ^                         |                     |
        |   <--- auth token / session -----------------+
        +---------------------------------------------+
```

### 4.3. Auditing (Hibernate Envers)

```
UserService --> Hibernate Session --> INSERT/UPDATE on users
               |
               v
          Envers interceptor
               |
               v
   Writes to users_AUD (audit) table
```

---

## 5. Third Parties / Processors  

| **Vendor** | **Role** | **Data Shared** | **Hosting** |
|------------|----------|-----------------|-------------|
| *None* (all processing is performed by the deploying health‑care organisation) | – | – | – |

*If a cloud provider is used for MySQL/MariaDB or Elasticsearch, that provider becomes a **processor** and must be listed here with the same columns.*

---

## 6. Security Measures  

**Technical & organisational safeguards**

- **Password protection** – passwords are stored only as BCrypt hashes (`User.password` column).  
- **Role‑based access control** – enforced by `@Authorized` AOP annotations on service methods (e.g., `UserService.saveUser`, `UserService.deleteUser`).  
- **Transaction safety** – all write operations are wrapped in Spring `@Transactional` boundaries to guarantee atomicity.  
- **Audit trail** – Hibernate Envers automatically records every change to `users`, `user_role`, `roles`, and `role_privilege` in `*_AUD` tables.  
- **Database hardening** – MySQL/MariaDB runs with `sql_mode=STRICT_TRANS_TABLES`, encrypted connections (TLS), and least‑privilege DB accounts.  
- **Network protection** – Tomcat is configured to accept only HTTPS traffic; HTTP is redirected to HTTPS.  
- **File‑system security** – configuration files and log directories are owned by the `openmrs` OS user with `chmod 750`.  
- **Regular patching** – the underlying OS, Java 8 runtime, Tomcat 9, and all Maven dependencies are kept up‑to‑date via CI pipelines.  

**Potential concerns / mitigations**

| Concern | Mitigation |
|---------|------------|
| Vulnerabilities in third‑party libraries (e.g., Spring, Hibernate) | Continuous dependency‑check scanning; rapid CVE patching. |
| Insider threat – privileged admin accounts | Enforce strong password policy, MFA where possible, and periodic review of role assignments. |
| Insufficient logging of authentication attempts | Enable `org.openmrs.api.context.Context` audit logging and forward logs to a SIEM. |
| Stale user accounts (e.g., former staff) | Automated de‑provisioning workflow tied to HR system; accounts disabled after 90 days of inactivity. |

---

## 7. Cross‑Border Transfers  

- **Current deployment model:** All data (MySQL/MariaDB, optional Elasticsearch, audit logs) are stored on servers located within the EU/EEA (or the same jurisdiction as the health‑care provider).  
- **Transfers outside the EU/EEA:** *None* in the default OpenMRS Core distribution.  
- **If a cloud provider outside the EU is used:** transfers would be governed by **Standard Contractual Clauses (SCCs)** or **Binding Corporate Rules (BCRs)**, and a Data Transfer Impact Assessment would be performed.  

---

## 8. DPIA Trigger Assessment  

| **Factor** | **Assessment** |
|------------|----------------|
| Systematic and extensive evaluation of personal aspects | **No** – only account management, not large‑scale profiling. |
| Processing of special categories of data (Art. 9) | **Yes** – staff health information (e.g., disability status) may be stored in user profiles; also indirect impact on patient data confidentiality. |
| Large‑scale processing (many data subjects) | **No** – number of user accounts is limited to staff size (typically < 1 000). |
| Use of innovative technology or new organisational measures | **No** – standard RBAC and CRUD operations. |
| Likely high risk to the rights and freedoms of data subjects | **Yes** – unauthorised access could lead to breach of patient health data (special category). |
| **Overall DPIA Recommendation** | **A DPIA is required** because the activity processes special‑category data and presents a high risk to patient confidentiality. The DPIA should cover: <br>• Risk of credential compromise <br>• Impact of role‑misconfiguration <br>• Effectiveness of audit and monitoring controls. |

---  

*Prepared on 2026‑05‑11 based on the OpenMRS Core codebase, database schema, and documented service interfaces.*