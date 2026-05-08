## Summary
The OpenMRS Core repository contains numerous classes that store personal identifiable information (PII). This inventory lists every personal data element found in the domain models, identifies its type, where it is defined in the source code, the corresponding database table, how it is used within the system, and notes on protection. The data spans basic identifiers (names, addresses, birthdate, gender), contact and authentication details (email, username, passwords, secret questions/answers, activation keys), location coordinates, and extensive clinical information (allergies, conditions, diagnoses, observations, drug orders, visits, provider details, person attributes, and encounter data). Sensitive data flags highlight health/medical data, biometric‑potential fields, credential information, and death‑related records.

## Data Elements

| Data Element | Type | Location in Code | Storage (DB Table) | Usage | Protection |
|---|---|---|---|---|---|
| Given Name | String | `PersonName.java` | `person_name` | Patient/Person identification | |
| Middle Name | String | `PersonName.java` | `person_name` | Patient/Person identification | |
| Family Name | String | `PersonName.java` | `person_name` | Patient/Person identification | |
| Prefix | String | `PersonName.java` | `person_name` | Patient/Person identification | |
| Family Name Prefix | String | `PersonName.java` | `person_name` | Patient/Person identification | |
| Address1 | String | `PersonAddress.java` | `person_address` | Patient/Person identification | |
| Address2 | String | `PersonAddress.java` | `person_address` | Patient/Person identification | |
| Address3 | String | `PersonAddress.java` | `person_address` | Patient/Person identification | |
| Address4 | String | `PersonAddress.java` | `person_address` | Patient/Person identification | |
| Address5 | String | `PersonAddress.java` | `person_address` | Patient/Person identification | |
| Address6 | String | `PersonAddress.java` | `person_address` | Patient/Person identification | |
| Address7 | String | `PersonAddress.java` | `person_address` | Patient/Person identification | |
| Address8 | String | `PersonAddress.java` | `person_address` | Patient/Person identification | |
| Address9 | String | `PersonAddress.java` | `person_address` | Patient/Person identification | |
| Address10 | String | `PersonAddress.java` | `person_address` | Patient/Person identification | |
| Address11 | String | `PersonAddress.java` | `person_address` | Patient/Person identification | |
| Address12 | String | `PersonAddress.java` | `person_address` | Patient/Person identification | |
| Address13 | String | `PersonAddress.java` | `person_address` | Patient/Person identification | |
| Address14 | String | `PersonAddress.java` | `person_address` | Patient/Person identification | |
| Address15 | String | `PersonAddress.java` | `person_address` | Patient/Person identification | |
| City/Village | String | `PersonAddress.java` | `person_address` | Patient/Person identification | |
| County/District | String | `PersonAddress.java` | `person_address` | Patient/Person identification | |
| State/Province | String | `PersonAddress.java` | `person_address` | Patient/Person identification | |
| Country | String | `PersonAddress.java` | `person_address` | Patient/Person identification | |
| Postal Code | String | `PersonAddress.java` | `person_address` | Patient/Person identification | |
| Latitude | String | `PersonAddress.java` | `person_address` | Geolocation of address | |
| Longitude | String | `PersonAddress.java` | `person_address` | Geolocation of address | |
| Birthdate | Date | `Person.java` | `person` | Patient/Person identification & age calculations | |
| Gender | String | `Person.java` | `person` | Demographic data | |
| Death Date | Date | `Person.java` | `person` | Record of death | |
| Cause of Death (coded) | Concept | `Person.java` | `person` | Clinical cause of death | |
| Cause of Death (non‑coded) | String | `Person.java` | `person` | Clinical cause of death | |
| Patient Identifier | String | `PatientIdentifier.java` | `patient_identifier` | Unique patient IDs (MRN, SSN, etc.) | |
| Email | String | `User.java` | `users` | Contact information for user accounts | |
| Username | String | `User.java` | `users` | Login credential | |
| Password (hashed) | String | `LoginCredential.java` | `login_credential` | Authentication (hashed) | |
| Salt | String | `LoginCredential.java` | `login_credential` | Authentication (hashing) | |
| Secret Question | String | `LoginCredential.java` | `login_credential` | Password recovery | |
| Secret Answer | String | `LoginCredential.java` | `login_credential` | Password recovery | |
| Activation Key | String | `LoginCredential.java` | `login_credential` | Account activation | |
| Location (lat/long) | String | `PersonAddress.java` | `person_address` | Geolocation of address | |
| Allergies | String (embedded Allergen) | `Allergy.java` | `allergy` | Clinical allergy information | |
| Conditions | String (CodedOrFreeText) | `Condition.java` | `conditions` | Ongoing health issues | |
| Diagnoses | String (linked to Concept) | `Diagnosis.java` | `encounter_diagnosis` | Clinical diagnosis per encounter | |
| Observations (clinical data) | Various (numeric, coded, text, complex) | `Obs.java` | `obs` | Clinical observations recorded during encounters | |
| Drug Orders | Various (dose, drug, frequency, etc.) | `DrugOrder.java` | `order` (and related tables) | Prescribed medication details | |
| Visit Data (start/stop datetime, type, location) | Date, String, etc. | `Visit.java` | `visit` | Grouping of encounters into a visit | |
| Provider Info (person link, identifier, role, speciality) | String, Concept | `Provider.java` | `provider` | Healthcare worker details | |
| Person Attributes (custom key‑value pairs) | String | `PersonAttribute.java` | `person_attribute` | Extensible personal data (e.g., national ID) | |
| Encounter Data (datetime, patient, location, type) | Date, references | `Encounter.java` | `encounter` | Core interaction record between patient and provider | |

## Sensitive Data Flags
| Flag | Affected Data Elements |
|---|---|
| **Health / Medical Data** | Allergies, Conditions, Diagnoses, Observations, Drug Orders, Visit Data, Provider Info, Encounter Data |
| **Biometric Potential** | Birthdate, Gender |
| **Credentials** | Password (hashed), Salt, Secret Question, Secret Answer, Activation Key |
| **Death Records** | Death Date, Cause of Death (coded & non‑coded) |

These flags indicate data that is especially sensitive under privacy regulations (e.g., HIPAA, GDPR) and should be protected with strong technical and administrative safeguards.