# OpenMRS Core — As-Is Knowledge Base

Descriptive documentation of the system as it works today — generated from source.

## Engineering Doc
- [erd.md](erd.md) — system overview, ingress, egress, internal topology, deployment

## Features
- [Patient Management](features/patient-management.md) — Managing patient information, including demographics and identifiers.
- [Encounter Management](features/encounter-management.md) — Managing patient encounters, including encounter types and providers.
- [Observation Management](features/observation-management.md) — Managing observations, including obs types and reference ranges.
- [Order Management](features/order-management.md) — Managing orders, including order types and drug orders.
- [Concept Management](features/concept-management.md) — Managing concepts, including concept types and concept answers.
- [Location Management](features/location-management.md) — Managing locations, including location types and tags.
- [User Management](features/user-management.md) — Managing users, including user roles and privileges.
- [Visit Management](features/visit-management.md) — Managing patient visits, including visit types and attributes.
- [Cohort Management](features/cohort-management.md) — Managing cohorts, including cohort memberships.
- [Condition Management](features/condition-management.md) — Managing patient conditions, including clinical status and verification status.
- [Diagnosis Management](features/diagnosis-management.md) — Managing patient diagnoses, including diagnosis attributes.
- [Medication Dispense Management](features/medication-dispense-management.md) — Managing medication dispenses, including dispense services.
- [Program Workflow Management](features/program-workflow-management.md) — Managing program workflows, including workflow states.
- [Provider Management](features/provider-management.md) — Managing providers, including provider attributes.
- [Form Management](features/form-management.md) — Managing forms, including form fields and resources.
- [Alert Management](features/alert-management.md) — Managing alerts, including alert services.
- [Scheduler Management](features/scheduler-management.md) — Managing schedulers, including scheduler services.
- [Administration Management](features/administration-management.md) — Managing administration settings, including global properties.
- [HL7 Message Processing](features/hl7-message-processing.md) — Receives and processes HL7 v2 messages (ADT, ORU) to create/update patients, encounters, and observations.
- [Module System](features/module-system.md) — Dynamically loads, starts, stops, and manages OpenMRS modules (plugins) at runtime.