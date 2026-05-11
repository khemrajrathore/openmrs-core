# Record of Processing Activities (ROPA) – OpenMRS Core  
**Processing Activity:** System Notifications (PA‑010)  

---  

## 1. Overview  

| **PA Name** | **Business Function** | **Processing Purpose** | **Legal Basis (GDPR Article)** | **Controller** |
|-------------|----------------------|------------------------|--------------------------------|----------------|
| System Notifications (PA‑010) | Notify users (clinicians, admins, patients) of system events, clinical alerts, or patient‑related updates via email or in‑system messages. | To ensure timely communication of clinical or administrative information that may affect patient care, workflow coordination, or system administration. | **Art. 9(2)(h)** – processing is necessary for the provision of health or social care services; **Art. 6(1)(f)** – legitimate interests of the controller (effective care delivery). | OpenMRS Core Development Team (the legal entity that operates the OpenMRS installation). |

---  

## 2. Data Subjects  

- **Patients** – individuals whose health information may be referenced in a notification (e.g., “Lab result ready for patient #123”).  
- **Healthcare Providers** – clinicians, nurses, pharmacists, or other staff who receive notifications about patient care.  
- **System Administrators / Support Staff** – users who receive system‑level alerts (e.g., “Scheduled backup completed”).  

---  

## 3. Personal Data Elements  

| **Data Element** | **Source (code / DB)** | **Sensitivity** | **Retention** |
|------------------|------------------------|----------------|----------------|
| **Notification ID** | `Message` entity – `api/src/main/java/org/openmrs/notification/Message.java`; DB table `message` (PK `message_id`). | Low (technical identifier). | Kept as long as the message record exists (default 30 days for in‑system messages; configurable). |
| **Recipient User ID** | `User` entity – `api/src/main/java/org/openmrs/api/User.java`; DB table `users` (PK `user_id`). | Low (internal identifier). | Same retention as the associated message (30 days) or until user deletion. |
| **Patient ID (optional)** | `Patient` entity – `api/src/main/java/org/openmrs/Patient.java`; DB table `patient` (PK `patient_id`). | **High – special‑category health data** (identifies a patient whose health information may be referenced). | Deleted together with the message (30 days) or when the patient record is voided/merged per OpenMRS retention policy. |
| **Message Content** | `Message` entity field `text`; DB column `message.text`. May contain free‑text clinical information. | **High – special‑category health data** (can disclose diagnoses, lab results, medication). | Same as Notification ID (default 30 days) or longer if a regulatory retention rule applies (e.g., 2 years for audit logs). |
| **Timestamp / Log entry** | `MessageLog` entity – `api/src/main/java/org/openmrs/notification/MessageLog.java`; DB table `message_log`. | Low (audit). | Retained for audit purposes – **2 years** (configurable via `openmrs.properties`). |

*All data elements are stored in MySQL/MariaDB (primary datastore). Optional search indexes are stored in Elasticsearch but contain only hashed identifiers, not raw message content.*  

---  

## 4. Data Flows  

### 4.1 High‑level ASCII diagram  

```
+-------------------+        +-------------------+        +-------------------+
|   Application UI |  -->   | MessageService    |  -->   | MailMessageSender |
| (REST / Web UI)  |        | (api/src/main/java|        | (api/src/main/java|
|                   |        | /org/openmrs/     |        | /org/openmrs/     |
|                   |        | notification/     |        | notification/     |
+-------------------+        | MessageService.java) |   | MailMessageSender.java) |
                               | creates Message   |        | forwards to SMTP |
                               +-------------------+        +-------------------+
                                        |                         |
                                        v                         v
                               +-------------------+        +-------------------+
                               |   MySQL / MariaDB |        |   External SMTP   |
                               |   (tables:       |        |   server (e.g.    |
                               |   message,       |        |   Postfix, SendGrid)|
                               |   message_log)   |        +-------------------+
                               +-------------------+
```

### 4.2 Detailed flow (in‑system notification)  

```
[Caller] --> MessageService.createMessage(notificationId, recipientUserId, patientId?, content)
    |
    v
MessageService.persist(Message)  --> INSERT into `message` table
    |
    v
MessageService.sendMessage()
    |
    +---> If email flag set --> MailMessageSender.send(message)
    |                               |
    |                               v
    |                       SMTP connection (TLS) --> External mail server
    |
    +---> If in‑system flag set --> Message is stored; UI polls `MessageService.getMessagesForUser(userId)`
```

### 4.3 Logging flow  

```
MessageService.sendMessage()
    |
    v
MessageLog entry created (MessageLog.java) --> INSERT into `message_log`
    |
    v
Log retained for audit (2 years) → accessible via OpenMRS audit UI.
```

---  

## 5. Third Parties / Processors  

| **Vendor** | **Role** | **Data Shared** | **Hosting / Location** |
|------------|----------|-----------------|------------------------|
| Email service / SMTP provider (e.g., Postfix, SendGrid, Amazon SES) | Email transmission service | Recipient email address (derived from User profile), Message Content (may contain health data) | Typically hosted in the same data‑center or cloud region as the OpenMRS instance; if external (e.g., SendGrid), data may cross borders – see §7. |
| (Optional) Monitoring / Log aggregation service (e.g., ELK stack) | Log collection & analysis | MessageLog entries (metadata only, no PHI) | Hosted on the same premises or a managed cloud service. |

*No other third‑party processors are involved in PA‑010.*  

---  

## 6. Security Measures  

**Technical & Organizational Controls**  

| Measure | Description | Effectiveness |
|---------|-------------|---------------|
| **Hibernate Envers Auditing** | Every change to `Message` and `MessageLog` is versioned; audit trail stored in `message_audit` tables. | Guarantees traceability and tamper‑evidence. |
| **`@Authorized` AOP (OpenMRS security)** | Service methods (`MessageService.sendMessage`, `MessageService.getMessagesForUser`) are annotated with `@Authorized` to enforce role‑based access (e.g., `VIEW_NOTIFICATIONS`, `MANAGE_NOTIFICATIONS`). | Prevents unauthorized read/write. |
| **Spring Transaction Management** | All DB writes are wrapped in transactions; rollback on failure. | Guarantees data integrity. |
| **Password hashing (BCrypt)** | User passwords stored as BCrypt hashes; no clear‑text credentials. | Protects credential leakage. |
| **TLS for SMTP** | `MailMessageSender` forces TLS (`STARTTLS`) when connecting to the SMTP server. | Secures email in transit. |
| **Database encryption at rest** (optional, via MySQL/MariaDB Transparent Data Encryption) | Encrypts the underlying tables (`message`, `message_log`). | Mitigates data exposure on disk theft. |
| **Network segmentation** | Application server, database, and mail server are placed on separate VLANs with firewall rules limiting traffic to required ports (e.g., 443, 3306, 587). | Reduces attack surface. |
| **Regular patching** | OpenMRS core, Tomcat, Java, MySQL, and mail server are kept up‑to‑date with security patches. | Limits known vulnerabilities. |
| **Backup & Disaster Recovery** | Encrypted backups of the MySQL database (including `message` tables) stored off‑site for 30 days. | Ensures availability and integrity. |

**Potential Concerns & Mitigations**  

| Concern | Mitigation |
|---------|------------|
| Email interception or leakage of PHI in transit. | Enforce TLS, use DKIM/SPF, and consider end‑to‑end encryption for highly sensitive notifications. |
| Unauthorized UI access to in‑system messages. | Strict role‑based UI controls; UI hides message content unless the user has `VIEW_PATIENT_DATA` privilege. |
| Retention of messages longer than necessary. | Automated job (`MessagePurgeJob`) runs nightly to delete messages older than the configured retention period (default 30 days). |
| Third‑party SMTP provider located outside the EU. | If used, rely on Standard Contractual Clauses (SCCs) or ensure the provider offers EU‑adequate safeguards. |

---  

## 7. Cross‑Border Transfers  

- **Default deployment**: All components (OpenMRS application, MySQL/MariaDB, optional Elasticsearch) are installed on servers located within the EU/EEA, so no cross‑border transfer occurs.  
- **If an external SMTP provider is used** (e.g., SendGrid, Amazon SES) and the provider’s data‑processing facilities are outside the EU, the transfer is covered by **Standard Contractual Clauses (SCCs)** or the provider’s **EU‑Adequacy Decision** (e.g., Amazon Web Services).  
- **Data minimisation**: Only the minimal necessary data (recipient email address and message content) is transmitted. No patient identifiers beyond what is required for the notification are sent.  

---  

## 8. DPIA Trigger Assessment  

| **Factor** | **Description** | **Risk Level** | **Mitigation / Recommendation** |
|------------|-----------------|----------------|---------------------------------|
| **Data subject vulnerability** | Patients are a vulnerable group; health data is highly sensitive. | **High** | Apply strict access controls, encrypt data at rest, enforce TLS for email, limit message retention. |
| **Special‑category health data** | Notification content may contain diagnoses, lab results, medication. | **High** | Legal basis Art. 9(2)(h); ensure explicit legitimate interest assessment and documented safeguards. |
| **Systematic monitoring** | Notifications can be used to monitor patient status (e.g., “critical lab result”). | **Medium** | Log all notification sends, retain logs for 2 years, conduct regular audits. |
| **Third‑party processing** | External SMTP provider may process PHI. | **Medium** | Use SCCs or adequacy decisions; limit data to necessary fields; enforce TLS. |
| **Potential for data breach** | Email can be forwarded or intercepted. | **Medium** | Enforce TLS, consider S/MIME or PGP for highly sensitive alerts. |
| **Retention period** | Default 30 days may be insufficient for clinical audit. | **Low** | Align retention with clinical governance policies (e.g., 2 years for audit). |

**Overall DPIA Outcome:** **High risk** – a formal Data Protection Impact Assessment is required before production use. The DPIA should document the legal basis, technical and organisational safeguards, and a clear data‑retention schedule.  

---  

*Prepared on 2026‑05‑11 using the OpenMRS Core source code (Java 8, Spring, Hibernate) and the engineering documentation provided.*