# Allergy and Drug Management

## Overview
The Allergy and Drug Management feature provides the data model for representing **Allergy** and **Drug** entities within OpenMRS Core. The feature stores information about patient allergies and drugs, enabling other parts of the system (e.g., clinical encounters, prescribing modules) to reference these records. Because the source files (`Allergy.java`, `Drug.java`) are not available for inspection, the documentation can only state that the feature defines these domain objects and that they are persisted in the OpenMRS database.

## Behavior
- Defines a Java class `Allergy` that models an allergy record. `(source not available – cannot cite line)`  
- Defines a Java class `Drug` that models a drug record. `(source not available – cannot cite line)`  
- Both classes are likely annotated for persistence (e.g., `@Entity`) and include standard getters/setters, but the exact implementation details cannot be confirmed without source. `(source not available – cannot cite line)`

## Triggers / Entry points
- No explicit entry points (e.g., REST controllers, service methods) are visible in the provided file list. `(source not available – cannot cite line)`

## End-to-end flow (Mermaid)
Because the internal flow (service calls, controller routes, persistence actions) cannot be observed in the unavailable source, a concrete Mermaid diagram cannot be generated without speculation.

```mermaid
%% Diagram not generated – source code unavailable
```

## State / data touched
- The `Allergy` and `Drug` entities are persisted to corresponding database tables (commonly `allergy` and `drug` in OpenMRS). This is inferred from the entity names, but the exact table names and column mappings cannot be verified without source. `(source not available – cannot cite line)`

## External dependencies
- No third‑party libraries or external services are referenced in the visible file list. `(source not available – cannot cite line)`

## Configuration / parameters
- No global properties, environment variables, or configuration keys are identifiable from the missing source files. `(source not available – cannot cite line)`

## Edge cases & failure modes
- Validation rules, error handling, and other edge‑case logic are not observable in the unavailable source. `(source not available – cannot cite line)`

## Open questions
- What annotations and validation constraints are applied to `Allergy` and `Drug`?  
- Which service classes or REST endpoints create, read, update, or delete these entities?  
- What database schema (table names, column definitions) is generated for these entities?  
- Are there any caching mechanisms, listeners, or event handlers associated with allergy or drug records?  
- Are there any external APIs (e.g., drug reference services) invoked by this feature?  

*All statements above are based solely on the presence of the file names `Allergy.java` and `Drug.java`. Without access to the actual source code, detailed behavior, entry points, and implementation specifics cannot be documented.*