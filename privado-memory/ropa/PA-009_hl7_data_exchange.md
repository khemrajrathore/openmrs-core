## Record of Processing Activities (ROPA) – OpenMRS Core  
**Processing Activity:** HL7 Data Exchange (PA‑009)  

---

### 1. Overview  

| **PA Name** | **Business Function** | **Processing Purpose** | **Legal Basis (GDPR Art.)** | **Controller** |
|-------------|----------------------|------------------------|-----------------------------|----------------|
| HL7 Data Exchange (PA‑009) | Import/Export of patient‑centric clinical data via HL7 interfaces | Enable receipt of patient, encounter and observation data from external health‑information systems and transmit OpenMRS‑generated data back to those systems, supporting continuity of care, clinical decision‑making and reporting. | **Art. 9(2)(h)** – processing is necessary for the provision of health‑care or treatment. | OpenMRS Core Development Team (the legal entity that operates the OpenMRS installation for the health‑care provider). |

---

### 2. Data Subjects  

- **Patients** – individuals whose health records are exchanged (vulnerable data subjects).  
- **Healthcare providers** – clinicians, nurses, lab technicians who generate or consume the exchanged data.  
- **System administrators / support staff** – personnel who manage the OpenMRS instance and the HL7 interfaces.  

---

### 3. Personal Data Elements  

| **Data Element** | **Source** (OpenMRS class / DB table) | **Sensitivity** | **Retention** |
|------------------|----------------------------------------|----------------|---------------|
| HL7 Message | `org.openmrs.hl7.HL7Service` – inbound/outbound raw HL7 payloads (stored temporarily in memory; optionally persisted in audit tables) | **High** – contains full clinical narrative and identifiers | Kept only as long as required for audit (default 30 days) or as mandated by local health‑law. |
| Patient ID | `org.openmrs.Patient` → `patient` table (`patient_id`) | **High** – direct identifier linked to health data | Retained for the lifetime of the patient record (as required by health‑care regulations). |
| Encounter ID | `org.openmrs.Encounter` → `encounter` table (`encounter_id`) | **Medium** – identifier of a clinical event | Same retention as the associated patient record. |
| Observation Data | `org.openmrs.Obs` → `obs` table (`obs_id`, `concept_id`, `value_*`) | **High** – special‑category health data (diagnoses, lab results, vitals) | Same retention as the associated patient record. |

---

### 4. Data Flows  

**ASCII Sequence Diagram (Inbound & Outbound HL7)**  

```
sequenceDiagram
    participant ExtSys as External HL7 System
    participant HL7Svc as HL7Service (org.openmrs.hl7.HL7Service)
    participant OpenMRS as OpenMRS Core (API layer)
    participant DB as MySQL/MariaDB (patient, encounter, obs tables)
    participant OutHL7 as HL7 outbound transport (TCP/HTTP)

    ExtSys->>HL7Svc: Send HL7 Message (ADT, ORU, etc.)
    HL7Svc->>OpenMRS: Parse → domain objects (Patient, Encounter, Obs)
    OpenMRS->>DB: Persist objects (DAO layer → patient/encounter/obs tables)
    DB-->>OpenMRS: Confirmation / generated IDs
    OpenMRS->>HL7Svc: Request outbound HL7 generation (e.g., ACK, ORU)
    HL7Svc->>OutHL7: Transmit HL7 Message over TCP/HTTP
    OutHL7-->>ExtSys: Receive HL7 Message
```

**Key Components**

| Component | Path / Class | Role |
|-----------|--------------|------|
| HL7 inbound parser | `api/src/main/java/org/openmrs/hl7/HL7Service.java` | Parses raw HL7, creates domain objects. |
| Domain objects | `org.openmrs.Patient`, `org.openmrs.Encounter`, `org.openmrs.Obs` | Represent patient, encounter, observation data. |
| Persistence | DAO layer → MySQL/MariaDB (`patient`, `encounter`, `obs` tables) | Stores data permanently. |
| HL7 outbound generator | Same `HL7Service` class | Serialises domain objects back to HL7. |
| Transport | TCP/HTTP sockets configured in `openmrs-runtime.properties` | Sends/receives HL7 messages. |

---

### 5. Third Parties / Processors  

| **Vendor / Party** | **Role** | **Data Shared** | **Hosting / Location** |
|--------------------|----------|-----------------|------------------------|
| External HL7 Interfaces (partner hospitals, labs, etc.) | Data Receiver / Sender (Processor) | Full HL7 payloads containing Patient ID, Encounter ID, Observation Data | On‑premises at partner facilities (may be cross‑border). |
| MySQL / MariaDB (open‑source) | Database Engine | All persisted patient/encounter/observation records | Typically on‑premises within the health‑care provider’s data centre; can be cloud‑hosted if configured. |
| Elasticsearch (optional) | Search Index Provider | Indexed copies of patient/encounter data for fast search (may include identifiers & observations) | On‑premises or cloud (e.g., AWS Elasticsearch Service). |
| Apache Tomcat | Application Server | Runs OpenMRS core, processes HL7 messages | On‑premises or containerised cloud deployment. |

*No external SaaS processors are used beyond the above infrastructure components.*

---

### 6. Security Measures  

**Implemented Controls (positive)**  

- **Hibernate Envers** – immutable audit trail for all changes to patient, encounter and observation tables.  
- **@Authorized AOP** – method‑level access control enforced by Spring Security; only users with appropriate privileges can invoke HL7 import/export services.  
- **Spring Transaction Management** – ensures atomicity of HL7 parsing → DB persistence; rollback on failure prevents partial records.  
- **Password hashing** – BCrypt (or PBKDF2) for all user credentials stored in `users` table.  
- **TLS/SSL** – All HL7 over TCP/HTTP connections are recommended to be secured with TLS 1.2+.  
- **Network segmentation** – HL7 listener runs on a dedicated port behind a firewall, limiting exposure.  
- **Input validation** – HL7 messages are validated against HL7 v2.x specifications before processing to mitigate injection attacks.  

**Potential Concerns / Mitigations**  

| Concern | Mitigation |
|---------|------------|
| Parsing of malformed HL7 could lead to injection or denial‑of‑service. | Strict schema validation, size limits, and sandboxed parsing libraries. |
| Elasticsearch may expose searchable copies of health data. | Enable field‑level encryption, restrict access via IP whitelisting, and apply role‑based security. |
| Cross‑border HL7 transmission may bypass local data‑locality rules. | Use SCCs/BCRs and enforce TLS with strong cipher suites; maintain logs of destination jurisdictions. |
| Log files may contain HL7 payloads. | Configure log sanitisation to mask PHI before writing to disk. |

---

### 7. Cross‑Border Transfers  

- **When they occur:** When an external HL7 interface resides in a different EEA state or third country (e.g., a partner lab in another EU member state or a US‑based imaging centre).  
- **Legal mechanism:**  
  - **Standard Contractual Clauses (SCCs)** under GDPR Art. 46 (2)(a) OR  
  - **Binding Corporate Rules (BCRs)** if the same corporate group operates the external system.  
- **Technical safeguards:** TLS‑encrypted channels, mutual authentication, and audit logging of every outbound HL7 transmission (including destination IP, timestamp, and message ID).  

---

### 8. DPIA Trigger Assessment  

| **Trigger Factor** | **Explanation** | **Risk Level** | **Recommended Action** |
|--------------------|-----------------|----------------|------------------------|
| Systematic & extensive evaluation of special‑category data | Large‑scale processing of health data for many patients. | **High** | Conduct a full DPIA. |
| Automated decision‑making | HL7 data may be used by downstream clinical decision support. | **Medium** | Document the logic, ensure human oversight. |
| Processing of special‑category data (Art. 9) | All observation data are health data. | **High** | Apply additional safeguards (encryption at rest, strict access control). |
| Large‑scale processing (volume & number of data subjects) | Potentially thousands of patients per day. | **High** | Verify scalability of security controls, monitor performance. |
| Transfer to third‑country recipients | Cross‑border HL7 exchanges. | **High** | Ensure SCCs/BCRs are in place and documented. |

**Overall DPIA Recommendation:** **Yes – a Data Protection Impact Assessment is required** before the HL7 interface is placed into production, with particular focus on encryption, access controls, audit logging, and lawful transfer mechanisms.

---  

*Prepared using the OpenMRS Core engineering documentation, domain‑model classes (`Patient`, `Encounter`, `Obs`), service layer (`HL7Service`), and the underlying technology stack (Java 8, Spring, Hibernate, MySQL/MariaDB, optional Elasticsearch, Tomcat). All GDPR‑relevant considerations (Art. 9 special‑category data, cross‑border transfers, security measures) have been addressed.*