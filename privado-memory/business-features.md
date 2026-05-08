# `privado-memory/business-features.md`

## Core Features  

| Business Activity | Primary Domain Model(s) | Service Layer (API) | Key Supporting Classes |
|-------------------|--------------------------|---------------------|------------------------|
| **Patient registration** | `org.openmrs.Patient` (extends `Person`) | `org.openmrs.api.PatientService` | `PatientIdentifier`, `PatientIdentifierType`, `PersonName`, `PersonAddress` |
| **Encounter management** | `org.openmrs.Encounter` | `org.openmrs.api.EncounterService` | `EncounterProvider`, `EncounterRole`, `Location` |
| **Observation / Clinical data capture** | `org.openmrs.Obs` | `org.openmrs.api.ObsService` | `Concept`, `ConceptAnswer`, `ConceptDatatype`, `ConceptName`, `ConceptReferenceTerm` |
| **Orders & prescriptions** | `org.openmrs.Order` (base), `org.openmrs.DrugOrder`, `org.openmrs.TestOrder`, `org.openmrs.ServiceOrder` | `org.openmrs.api.OrderService` | `OrderSet`, `OrderSetMember`, `OrderFrequency`, `OrderGroup` |
| **Allergy management** | `org.openmrs.Allergy` (and related `Allergen`, `AllergyReaction`, `AllergySeverity`) | `org.openmrs.api.AllergyService` | `Allergies` (constants), `AllergyProperties` |
| **Visit tracking** | `org.openmrs.Visit` (and `VisitType`) | `org.openmrs.api.VisitService` | `VisitAttribute`, `VisitAttributeType` |
| **Provider management** | `org.openmrs.Provider` | `org.openmrs.api.ProviderService` | `ProviderAttribute`, `ProviderAttributeType` |
| **User & role management** | `org.openmrs.User`, `org.openmrs.Role`, `org.openmrs.Privilege` | `org.openmrs.api.UserService` | `UserProperty`, `UserRole`, `UserPrivilege` |
| **Cohort management** | `org.openmrs.Cohort`, `org.openmrs.CohortMembership` | `org.openmrs.api.CohortService` | – |
| **Diagnosis handling** | `org.openmrs.Diagnosis` | `org.openmrs.api.DiagnosisService` | `DiagnosisAttribute`, `DiagnosisAttributeType` |
| **Condition handling** | `org.openmrs.Condition` (with `ConditionClinicalStatus`, `ConditionVerificationStatus`) | `org.openmrs.api.ConditionService` | – |
| **Medication dispensing** | `org.openmrs.MedicationDispense` | `org.openmrs.api.MedicationDispenseService` | – |
| **HL7 messaging** | `org.openmrs.hl7.HL7Message` (internal representation) | `org.openmrs.api.HL7Service` | `HL7InQueue`, `HL7OutQueue`, `HL7Exception` |
| **Form & form‑resource handling** | `org.openmrs.Form`, `org.openmrs.FormResource` | `org.openmrs.api.FormService` | – |
| **Program & workflow management** | `org.openmrs.Program`, `org.openmrs.ProgramWorkflow`, `org.openmrs.ProgramWorkflowState` | `org.openmrs.api.ProgramWorkflowService` | – |
| **Location & location‑tag management** | `org.openmrs.Location`, `org.openmrs.LocationTag` | `org.openmrs.api.LocationService` | `LocationAttribute`, `LocationAttributeType` |
| **Global properties & configuration** | `org.openmrs.GlobalProperty` | `org.openmrs.api.AdministrationService` | – |
| **Serialization / audit** | All `*` classes implement `Auditable` / `Retireable` | `org.openmrs.api.SerializationService` | `OpenmrsUtil`, `OpenmrsRevisionEntity` (Envers) |

---

## User‑Facing Functionality  

| Feature | UI / API Entry Point | Domain Model(s) | Service |
|---------|----------------------|-----------------|---------|
| **Login / authentication** | `UserService.authenticate()`, UI login page | `User` | `UserService` |
| **User profile & preferences** | `User.getUserProperties()` | `User` | `UserService` |
| **Role & privilege assignment** | `UserService.saveUser()`, `RoleService` | `User`, `Role`, `Privilege` | `UserService`, `RoleService` |
| **Patient search & view** | `PatientService.getPatientByUuid()`, `PatientService.getPatients()` | `Patient`, `PersonName`, `PersonAddress` | `PatientService` |
| **Register new patient** | `PatientService.savePatient()` | `Patient`, `PatientIdentifier`, `PersonName`, `PersonAddress` | `PatientService` |
| **Create / edit encounter** | `EncounterService.saveEncounter()` | `Encounter`, `EncounterProvider` | `EncounterService` |
| **Record observations** | `ObsService.saveObs()` | `Obs`, `Concept` | `ObsService` |
| **Prescribe medication / order tests** | `OrderService.saveOrder()` | `DrugOrder`, `TestOrder`, `ServiceOrder` | `OrderService` |
| **View / edit allergies** | `AllergyService.saveAllergy()` | `Allergy`, `Allergen`, `AllergyReaction` | `AllergyService` |
| **Manage visits** | `VisitService.saveVisit()` | `Visit`, `VisitAttribute` | `VisitService` |
| **Provider directory** | `ProviderService.getAllProviders()` | `Provider`, `ProviderAttribute` | `ProviderService` |
| **Cohort creation & management** | `CohortService.saveCohort()` | `Cohort`, `CohortMembership` | `CohortService` |
| **Diagnosis entry** | `DiagnosisService.saveDiagnosis()` | `Diagnosis`, `DiagnosisAttribute` | `DiagnosisService` |
| **Condition entry** | `ConditionService.saveCondition()` | `Condition` | `ConditionService` |
| **Medication dispensing** | `MedicationDispenseService.saveMedicationDispense()` | `MedicationDispense` | `MedicationDispenseService` |
| **HL7 inbound/outbound** | `HL7Service.processMessage()`, `HL7Service.sendMessage()` | `HL7Message` | `HL7Service` |
| **Form design & usage** | `FormService.saveForm()` | `Form`, `FormResource` | `FormService` |
| **Program enrollment** | `ProgramWorkflowService.enrollPatient()` | `Program`, `ProgramWorkflow`, `ProgramWorkflowState` | `ProgramWorkflowService` |
| **Location administration** | `LocationService.saveLocation()` | `Location`, `LocationTag` | `LocationService` |
| **Global property editing** | `AdministrationService.saveGlobalProperty()` | `GlobalProperty` | `AdministrationService` |

---

## Background Processes  

| Process | Trigger | Core Classes Involved | Description |
|---------|---------|-----------------------|-------------|
| **HL7 inbound queue processing** | Scheduler / message listener | `HL7Service`, `HL7InQueue`, `HL7Message`, `HL7Exception` | Parses incoming HL7 messages, creates/updates patients, encounters, observations, etc. |
| **HL7 outbound queue** | `HL7Service.sendMessage()` or scheduled batch | `HL7OutQueue`, `HL7Message` | Queues outbound HL7 messages for external systems (e.g., lab, HIS). |
| **Audit & versioning** | Every `save*`, `void*`, `purge*` operation | Envers entities (`OpenmrsRevisionEntity`), `Auditable` interface | Tracks changes, creates revision entries for compliance. |
| **Cache eviction / refresh** | Cache configuration changes, data updates | `OpenmrsCacheManagerFactoryBean`, `CacheConfig` | Maintains second‑level Hibernate cache and Spring cache for domain objects. |
| **Scheduled purge of voided data** | Nightly job (configured in `OpenmrsScheduler`) | DAO layer (`*DAO`), `Context` | Permanently removes voided records after retention period. |
| **Automatic patient identifier generation** | `PatientIdentifier` creation | `IdentifierGenerator` (implementation of `IdentifierSource`), `PatientIdentifierType` | Generates sequential or custom identifiers per site/location. |
| **Order auto‑expire / discontinue** | Scheduler | `OrderService`, `Order` | Checks for orders past their stop date and marks them discontinued. |
| **Cohort evaluation** | On‑demand or scheduled | `CohortService`, `CohortDefinition` (if using reporting module) | Evaluates dynamic cohort criteria against patient data. |
| **Medication stock reconciliation** | Inventory module (optional) | `MedicationDispenseService`, `Drug`, `DrugIngredient` | Updates drug stock levels when a dispense event is recorded. |

---

## Integration Points  

| Integration Target | Service / API | Domain Model(s) | Typical Use‑Case |
|--------------------|---------------|-----------------|------------------|
| **External HL7 systems** | `HL7Service` (inbound/outbound) | `Patient`, `Encounter`, `Obs`, `Order`, `Visit` | Exchange of patient demographics, lab results, orders. |
| **FHIR / REST API** | `org.openmrs.module.webservices.rest.web.resource.impl.*` (exposes REST resources) | All core domain objects | Mobile apps, third‑party EMR integration. |
| **Reporting / BI tools** | `ReportingService` (module) | `Cohort`, `Obs`, `Encounter`, `Visit` | Generate statistical reports, dashboards. |
| **OpenMRS Modules** (e.g., `appointments`, `htmlformentry`) | Module‑specific services (e.g., `AppointmentService`) | `Appointment`, `AppointmentType` (module) | Extend core functionality without changing core code. |
| **Message Queues (JMS, Kafka)** | Custom listeners using `Context.getService()` | Any core entity | Asynchronous processing of large data loads or external notifications. |
| **Authentication providers (LDAP, OAuth2)** | `AuthenticationScheme` implementations (`DaoAuthenticationScheme`, `LdapAuthenticationScheme`) | `User` | Centralized login management. |
| **External drug databases** | `DrugService` (core) + module adapters | `Drug`, `DrugIngredient` | Synchronize drug catalog with national formulary. |
| **Laboratory Information Systems (LIS)** | HL7 inbound/outbound, custom adapters | `Obs` (complex data), `Order` (lab orders) | Send lab orders, receive results. |
| **Billing / ERP systems** | Service layer calls (`OrderService`, `VisitService`) | `Order`, `Visit`, `Patient` | Export chargeable events for invoicing. |

---

## Domain Model  

Below is a concise map of the most important domain classes, grouped by functional area. All classes are located in the `org.openmrs` package (core module).

| Area | Core Classes |
|------|--------------|
| **Person & Demographics** | `Person`, `PersonName`, `PersonAddress`, `PersonAttribute`, `PersonAttributeType` |
| **Patient** | `Patient`, `PatientIdentifier`, `PatientIdentifierType`, `PatientProgram`, `PatientState` |
| **Encounter & Visit** | `Encounter`, `EncounterProvider`, `EncounterRole`, `Visit`, `VisitType`, `VisitAttribute`, `VisitAttributeType` |
| **Observations** | `Obs`, `Concept`, `ConceptName`, `ConceptDescription`, `ConceptAnswer`, `ConceptSet`, `ConceptReferenceTerm`, `ConceptReferenceTermMap` |
| **Orders** | `Order` (base), `DrugOrder`, `TestOrder`, `ServiceOrder`, `OrderSet`, `OrderSetMember`, `OrderFrequency`, `OrderGroup` |
| **Allergies** | `Allergy`, `Allergen`, `AllergenType`, `AllergyReaction`, `AllergySeverity`, `Allergies` (constants) |
| **Diagnoses & Conditions** | `Diagnosis`, `DiagnosisAttribute`, `DiagnosisAttributeType`, `Condition`, `ConditionClinicalStatus`, `ConditionVerificationStatus` |
| **Medication Dispensing** | `MedicationDispense` |
| **Providers** | `Provider`, `ProviderAttribute`, `ProviderAttributeType` |
| **Users & Security** | `User`, `UserProperty`, `Role`, `Privilege`, `UserRole`, `UserPrivilege` |
| **Cohorts** | `Cohort`, `CohortMembership` |
| **Programs & Workflows** | `Program`, `ProgramWorkflow`, `ProgramWorkflowState`, `ProgramAttributeType` |
| **Locations** | `Location`, `LocationTag`, `LocationAttribute`, `LocationAttributeType` |
| **Global Configuration** | `GlobalProperty` |
| **Forms** | `Form`, `FormResource`, `FormField`, `Field`, `FieldAnswer`, `FieldType` |
| **HL7 Messaging** | `HL7Message` (internal), `HL7InQueue`, `HL7OutQueue` |
| **Audit / Base Classes** | `BaseOpenmrsObject`, `BaseOpenmrsData`, `BaseOpenmrsMetadata`, `BaseChangeableOpenmrsData`, `Auditable`, `Retireable`, `Voidable` |

### Example Relationships  

* **Patient ↔ Person** – `Patient` extends `Person`; `personId` is the primary key for both.  
* **Patient ↔ PatientIdentifier** – One‑to‑many; identifiers are linked to a `PatientIdentifierType`.  
* **Encounter ↔ Patient** – Many‑to‑one; each encounter belongs to a patient.  
* **Obs ↔ Encounter** – Many‑to‑one; observations are attached to an encounter (or can be stand‑alone).  
* **Order ↔ Encounter** – Many‑to‑one; orders are usually created within an encounter.  
* **Provider ↔ Person** – Many‑to‑one; a provider is a person with additional professional attributes.  
* **User ↔ Person** – Many‑to‑one; a system user is linked to a person record.  
* **Visit ↔ Patient** – Many‑to‑one; a visit groups multiple encounters for a patient.  

These relationships are persisted via JPA/Hibernate annotations (e.g., `@ManyToOne`, `@OneToMany`, `@JoinColumn`) and are searchable through Hibernate Search annotations (`@Indexed`, `@FullTextField`, etc.) for fast UI lookup.

---  

*Prepared from the OpenMRS Core source tree (v2.x) – all class names and service interfaces are taken directly from the repository.*