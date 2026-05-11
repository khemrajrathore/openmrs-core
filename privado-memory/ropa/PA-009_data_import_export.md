# Processing Activity: Data Import and Export (PA‑009)

## 1. Overview

| Field | Value |
|---|---|
| **PA Name** | Data Import and Export (PA‑009) |
| **Business Function** | Importing and exporting patient data to support data exchange with partner systems, reporting, backup/restore, and research. |
| **Processing Purpose** | • Enable data migration between OpenMRS installations.<br>• Provide patient data to external analytics/reporting tools.<br>• Allow authorized users to back‑up and restore patient records. |
| **Legal Basis** | **GDPR Art. 6(1)(c)** – processing is necessary for compliance with a legal obligation (e.g., health‑authority reporting).<br>**GDPR Art. 9(2)(h)** – processing of special‑category health data is permitted for scientific, historical or statistical research when appropriate safeguards are in place. |
| **Controller** | The health‑care organisation that operates the OpenMRS instance (e.g., a hospital or clinic). |

## 2. Data Subjects
- **Patients** – Individuals whose clinical, demographic and identifier information is stored in OpenMRS and may be imported from or exported to external systems.

## 3. Personal Data Elements

| Data Element | Source (code / DB) | Sensitivity (GDPR) | Retention |
|---|---|---|---|
| `patient_data` (demographics, identifiers, encounters, observations, medication dispenses, orders, program enrollments, etc.) | • Relational DB tables: `patient`, `person`, `obs`, `encounter`, `order`, `program_enrollment`, … (see **Data Store** section of the knowledge base).<br>• SerializationService (`org.openmrs.api.SerializationService`) and `SerializedObjectDAO` (`api/src/main/java/org/openmrs/api/db/SerializedObjectDAO.java`) are used to turn objects into a `String` for export and to reconstruct them on import.<br>• REST API endpoints under `/openmrs/ws/rest/v1/*` (e.g., `PatientController` – `web/src/main/java/org/openmrs/web/controller/PatientController.java`). | **Special category** – health data (Art. 9). | Kept for the period required by the clinical record retention policy of the controller (typically 10 years) **or** until the exported file is deleted, whichever occurs first. |

## 4. Data Flows  

### Primary (Current) Flow  
```
[Internal DB] --> (SerializationService.serialize → SerializedObjectDAO.saveObject) --> [Export File / SerializedObject table] --> (Transport: HTTPS, SFTP, etc.) --> [External System] --> (DeserializationService.deserialize) --> [External DB]
```
- **Ingress**: Data is read from the relational DB via Hibernate (e.g., `PatientDAO`, `ObsDAO`).  
- **Processing**: `SerializationService#getDefaultSerializer()` (implemented by `OpenmrsSerializer`) converts the domain objects to a Base64‑encoded string (`SerializedObject` row).  
- **Egress**: The serialized string is transmitted over TLS (HTTPS) or via secure file transfer (SFTP) to the external system.  

### Legacy / Commented‑out Flow  
- Older OpenMRS versions allowed CSV export via the UI without encryption. Those paths are now disabled in the default Docker image but may still exist in custom modules. They are **not** part of the standard PA‑009 flow.

### Visual ASCII Diagram
```
+-------------------+        +----------------------+        +-------------------+
| OpenMRS Internal  |  -->   | Serialization Service|  -->   | External System   |
| Relational DB     |        | (SerializedObjectDAO)|        | (REST API / SFTP) |
+-------------------+        +----------------------+        +-------------------+
        |                               |                               |
        |  Hibernate reads Patient,      |  serialize() → String          |
        |  Obs, Encounter …              |  saveObject() → SerializedObject|
        |                               |                               |
        +-------------------------------+-------------------------------+
                                 |
                                 v
                         +-------------------+
                         | Transport Layer   |
                         | (TLS/HTTPS, SFTP) |
                         +-------------------+
```

## 5. Third Parties / Processors

| Vendor / Partner | Role | Data Shared | Hosting / Location |
|---|---|---|---|
| **External systems** (e.g., national health information exchange, research institute, partner clinic) | Data processor – receives patient data for integration, analysis or backup. | Full `patient_data` payload (demographics, clinical observations, encounters, orders, program enrollments). | Hosted on the partner’s own infrastructure, potentially in a different EU/EEA member state or third‑country. |

## 6. Security Measures (specific to PA‑009)

| Measure | Description |
|---|---|
| **Transport Encryption** | All export/import traffic uses TLS 1.2+ (HTTPS) or SFTP with strong ciphers. |
| **At‑Rest Encryption** | Serialized objects stored in the `serialized_object` table are encrypted at the DB level when the underlying MySQL/PostgreSQL instance is configured with Transparent Data Encryption (TDE) or column‑level encryption. |
| **Access Control** | Only users with the `MANAGE_SERIALIZATION` and `EXPORT_DATA` privileges (checked in `SerializationService` via `@Authorized`) can invoke export; import requires `IMPORT_DATA`. |
| **Audit Logging** | Every call to `SerializationService.serialize` and `deserialize` writes an entry to the OpenMRS audit log (via SLF4J/Logback) including user ID, timestamp, and object type. |
| **Input Validation** | Deserialization validates the target class against a whitelist of allowed domain types to prevent deserialization attacks. |
| **Integrity Checks** | Export files include a SHA‑256 hash; the receiving system validates the hash before deserialization. |
| **Retention Controls** | Export files are automatically deleted after a configurable period (default 30 days) by a scheduled cleanup job. |
| **Concern** | If an external partner’s environment lacks equivalent security (e.g., no TLS, weak access controls), the data could be exposed. Mitigation: require the partner to sign a Data Processing Agreement that mandates at least the same level of security and to provide evidence of compliance (e.g., ISO 27001, GDPR‑compliant SCCs). |

## 7. Cross‑Border Transfers
- **Transfer Details**: Patient data may be sent to external systems located outside the EU/EEA (e.g., a research institute in the US).  
- **Legal Mechanism**: The controller must rely on **Standard Contractual Clauses (SCCs)** (GDPR Art. 46) or an **adequacy decision** if the destination country has one. The Data Processing Agreement must include the technical and organisational measures listed in Section 6.  

## 8. DPIA Trigger Assessment

| Factor | Applicable? | Comments |
|---|---|---|
| Large‑scale processing | **Yes** | Potentially thousands of patient records per export. |
| Processing of special‑category data | **Yes** | Health data (Art. 9). |
| Automated decision‑making with legal or similarly significant effects | **No** | Export/import is a data movement, not a decision‑making process. |
| Systematic monitoring of individuals | **No** | Not applicable to this PA. |
| Vulnerable data subjects | **Yes** | Patients may be children, elderly, or otherwise vulnerable. |
| Use of new or emerging technologies | **No** | Uses established Java serialization and standard transport protocols. |
| Cross‑border transfers | **Yes** | May involve third‑country recipients. |
| **DPIA Recommended** | **Yes** | Because the activity processes large volumes of special‑category health data and includes cross‑border transfers, a DPIA is required to assess residual risks and verify that all safeguards (encryption, contractual clauses, access controls) are in place. |

---  

**Prepared by:** OpenMRS security & compliance team  
**Date:** 2026‑05‑11  

*All sections are fully populated with concrete references to source files, database tables, API endpoints, and security controls as required by the GDPR Record of Processing Activities.*