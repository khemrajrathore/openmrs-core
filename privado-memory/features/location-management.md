# Location Management

## Overview
The Location Management feature lets OpenMRS store, retrieve, and organise physical places (hospitals, clinics, rooms, districts, etc.). A **Location** can have a type (a `Concept`), an address, a parent‑child hierarchy, and a set of **LocationTag** objects that categorise it. Custom data can be attached through **LocationAttribute** / **LocationAttributeType**.  
Typical callers are UI components, modules, or external integrations that invoke the `LocationService` API (e.g., to create a new clinic, look up a location by name, or retire an obsolete site). The service returns fully‑populated domain objects that are persisted in the database.

## Behavior
- **Create / update a location** – `LocationService.saveLocation(Location)` validates that the location has a name, resolves any transient tags to existing tags, saves custom attributes, then delegates to `LocationDAO.saveLocation(Location)` which uses `saveOrUpdate` and cascades child locations if needed.  
  `api/src/main/java/org/openmrs/api/LocationService.java:45` → `api/src/main/java/org/openmrs/api/impl/LocationServiceImpl.java:61‑82` → `api/src/main/java/org/openmrs/api/db/hibernate/HibernateLocationDAO.java:70`

- **Read a location** – `LocationService.getLocation(Integer)` and `LocationService.getLocation(String)` forward to the DAO which builds a simple criteria query on the `location` table.  
  `api/src/main/java/org/openmrs/api/LocationService.java:57‑71` → `api/src/main/java/org/openmrs/api/impl/LocationServiceImpl.java:92‑106` → `api/src/main/java/org/openmrs/api/db/hibernate/HibernateLocationDAO.java:84‑102`

- **Default location lookup** – `LocationService.getDefaultLocation()` reads the global property `OpenmrsConstants.GLOBAL_PROPERTY_DEFAULT_LOCATION_NAME`, falls back to “Unknown Location” / “Unknown”, and finally to the first location (id = 1) if nothing matches.  
  `api/src/main/java/org/openmrs/api/LocationService.java:81‑95` → `api/src/main/java/org/openmrs/api/impl/LocationServiceImpl.java:136‑166`

- **Search by name fragment** – `LocationService.getLocations(String)` delegates to the overloaded `getLocations(nameFragment, …)` which serialises attribute values, builds a criteria query with a `LIKE` on the lower‑cased name, and optionally paginates.  
  `api/src/main/java/org/openmrs/api/LocationService.java:103‑115` → `api/src/main/java/org/openmrs/api/impl/LocationServiceImpl.java:184‑207` → `api/src/main/java/org/openmrs/api/db/hibernate/HibernateLocationDAO.java:250‑277`

- **Tag‑based queries** –  
  * `getLocationsByTag(LocationTag)` iterates over all non‑retired locations and collects those whose `tags` set contains the supplied tag.  
    `api/src/main/java/org/openmrs/api/LocationService.java:119‑131` → `api/src/main/java/org/openmrs/api/impl/LocationServiceImpl.java:115‑129`  
  * `getLocationsHavingAllTags(List<LocationTag>)` returns all locations if the tag list is empty, otherwise calls `LocationDAO.getLocationsHavingAllTags`.  
    `api/src/main/java/org/openmrs/api/LocationService.java:133‑144` → `api/src/main/java/org/openmrs/api/impl/LocationServiceImpl.java:131‑138` → `api/src/main/java/org/openmrs/api/db/hibernate/HibernateLocationDAO.java:306‑340`  
  * `getLocationsHavingAnyTag(List<LocationTag>)` builds a result list by scanning all non‑retired locations and adding any that contain at least one of the supplied tags.  
    `api/src/main/java/org/openmrs/api/LocationService.java:146‑158` → `api/src/main/java/org/openmrs/api/impl/LocationServiceImpl.java:140‑155`

- **Retire / unretire a location** – `retireLocation(Location, String)` marks the location as retired, stores the reason, and saves it again; `unretireLocation(Location)` clears the retired flag and saves.  
  `api/src/main/java/org/openmrs/api/LocationService.java:165‑176` → `api/src/main/java/org/openmrs/api/impl/LocationServiceImpl.java:168‑181`

- **Purge a location** – `purgeLocation(Location)` calls `LocationDAO.deleteLocation(Location)`, which issues a direct `delete` on the Hibernate session.  
  `api/src/main/java/org/openmrs/api/LocationService.java:178‑186` → `api/src/main/java/org/openmrs/api/impl/LocationServiceImpl.java:183‑188` → `api/src/main/java/org/openmrs/api/db/hibernate/HibernateLocationDAO.java:124‑128`

- **Location tags** – `saveLocationTag`, `getLocationTag`, `getLocationTagByName`, `retireLocationTag`, `unretireLocationTag`, and `purgeLocationTag` are thin wrappers around DAO methods that perform simple `saveOrUpdate` or `delete`.  
  `api/src/main/java/org/openmrs/api/LocationService.java:138‑170` → `api/src/main/java/org/openmrs/api/impl/LocationServiceImpl.java:190‑226` → `api/src/main/java/org/openmrs/api/db/hibernate/HibernateLocationDAO.java:130‑152`

- **Location attribute types** – CRUD operations (`getAllLocationAttributeTypes`, `getLocationAttributeType`, `saveLocationAttributeType`, etc.) delegate directly to DAO methods that use Hibernate `saveOrUpdate` / `get`.  
  `api/src/main/java/org/openmrs/api/LocationService.java:201‑260` → `api/src/main/java/org/openmrs/api/impl/LocationServiceImpl.java:236‑280` → `api/src/main/java/org/openmrs/api/db/hibernate/HibernateLocationDAO.java:340‑380`

- **Address template** – `getAddressTemplate()` reads the global property `OpenmrsConstants.GLOBAL_PROPERTY_ADDRESS_TEMPLATE`; if empty it returns `OpenmrsConstants.DEFAULT_ADDRESS_TEMPLATE`. `saveAddressTemplate(String)` writes the property.  
  `api/src/main/java/org/openmrs/api/LocationService.java:272‑284` → `api/src/main/java/org/openmrs/api/impl/LocationServiceImpl.java:306‑322`

## Triggers / Entry points
- **Service methods** defined in `LocationService` are the public entry points used by UI code, modules, and REST controllers.  
  `api/src/main/java/org/openmrs/api/LocationService.java:37‑285`
- **Implementation** resides in `LocationServiceImpl`, which is obtained via `Context.getLocationService()`.  
  `api/src/main/java/org/openmrs/api/impl/LocationServiceImpl.java:44‑46`
- **DAO layer** (`LocationDAO` and `HibernateLocationDAO`) is invoked by the service implementation for all persistence actions.  
  `api/src/main/java/org/openmrs/api/db/LocationDAO.java` (interface) and `api/src/main/java/org/openmrs/api/db/hibernate/HibernateLocationDAO.java`

## End‑to‑end flow (Mermaid)

```mermaid
sequenceDiagram
    participant Caller as "UI / Module"
    participant Service as "LocationService"
    participant Impl as "LocationServiceImpl"
    participant DAO as "LocationDAO"
    participant Hibernate as "HibernateLocationDAO"
    participant DB as "Database"

    %% Create / update
    Caller->>Service: saveLocation(Location)
    Service->>Impl: saveLocation(Location)
    Impl->>Impl: validate name (throw APIException if null)   // api/src/main/java/org/openmrs/api/impl/LocationServiceImpl.java:61
    Impl->>Impl: resolve transient tags (lookup by name, replace or throw) // impl:64‑78
    Impl->>Impl: CustomDatatypeUtil.saveAttributesIfNecessary // impl:80
    Impl->>DAO: saveLocation(Location)
    DAO->>Hibernate: saveOrUpdate(Location)   // hibernate:70
    Hibernate->>DB: INSERT/UPDATE location rows (and child rows if any) // hibernate:70‑73
    DB-->>Hibernate: persisted entity
    Hibernate-->>DAO: Location
    DAO-->>Impl: Location
    Impl-->>Service: Location
    Service-->>Caller: Location

    %% Read by id
    Caller->>Service: getLocation(id)
    Service->>Impl: getLocation(id)
    Impl->>DAO: getLocation(id)
    DAO->>Hibernate: session.get(Location.class, id) // hibernate:84
    Hibernate->>DB: SELECT * FROM location WHERE location_id = ?
    DB-->>Hibernate: row
    Hibernate-->>DAO: Location
    DAO-->>Impl: Location
    Impl-->>Service: Location
    Service-->>Caller: Location

    %% Retire
    Caller->>Service: retireLocation(loc, reason)
    Service->>Impl: retireLocation(loc, reason)
    Impl->>Impl: setRetired(true), setRetireReason(reason) // impl:168‑170
    Impl->>Service: saveLocation(loc)   // recursive call
    Service->>Impl: saveLocation(loc)   // same path as create/update
    ... (as above) ...

    %% Purge
    Caller->>Service: purgeLocation(loc)
    Service->>Impl: purgeLocation(loc)
    Impl->>DAO: deleteLocation(loc)
    DAO->>Hibernate: session.delete(loc) // hibernate:124
    Hibernate->>DB: DELETE FROM location WHERE location_id = ?
    DB-->>Hibernate: OK
    Hibernate-->>DAO: void
    DAO-->>Impl: void
    Impl-->>Service: void
    Service-->>Caller: void
```

## State / data touched
- **Tables**  
  * `location` – core fields, hierarchy (`parent_location`), retired flag, address columns. (`HibernateLocationDAO.saveLocation`, `getLocation`, `getAllLocations`, etc.)  
  * `location_tag` – tag definitions. (`saveLocationTag`, `getLocationTagByName`, etc.)  
  * `location_tag_map` – many‑to‑many link between locations and tags. (`Location.tags` mapping, tag queries).  
  * `location_attribute_type` – definitions of custom attributes. (`saveLocationAttributeType`, `getAllLocationAttributeTypes`).  
  * `location_attribute` – values of custom attributes attached to locations. (`LocationAttribute`, DAO methods).  
  * Global properties table (via `AdministrationService`) for address template and default location name. (`LocationServiceImpl.getDefaultLocation`, `getAddressTemplate`, `saveAddressTemplate`).  

- **Collections / caches** – `Location.tags` (`Set<LocationTag>`), `Location.childLocations` (`Set<Location>`), and Hibernate second‑level cache for `Location` and `LocationTag` (enabled by `@Cache(READ_WRITE)` on `Location`). (`Location` class annotations).

## External dependencies
- **OpenMRS core services** – `Context.getAdministrationService()` for global properties, `Context.getAuthenticatedUser()` for audit fields when retiring tags. (`LocationServiceImpl.getDefaultLocation`, `LocationServiceImpl.retireLocationTag`).  
- **Custom datatype utilities** – `CustomDatatypeUtil.saveAttributesIfNecessary` to persist attribute values. (`LocationServiceImpl.saveLocation`).  
- **Hibernate/JPA** – Criteria API (`CriteriaBuilder`, `CriteriaQuery`) used throughout `HibernateLocationDAO` for all queries. (`HibernateLocationDAO` methods).  
- **Spring transaction management** – `@Transactional` on `LocationServiceImpl` ensures atomic DB operations. (`LocationServiceImpl` class annotation).

## Configuration / parameters
- **Global properties**  
  * `OpenmrsConstants.GLOBAL_PROPERTY_DEFAULT_LOCATION_NAME` – name of the default location used by `LocationService.getDefaultLocation()`. (`LocationServiceImpl.getDefaultLocation`).  
  * `OpenmrsConstants.GLOBAL_PROPERTY_ADDRESS_TEMPLATE` – XML template for address formatting; falls back to `OpenmrsConstants.DEFAULT_ADDRESS_TEMPLATE`. (`LocationServiceImpl.getAddressTemplate`, `saveAddressTemplate`).  

- **Method parameters** – most service methods accept explicit parameters (e.g., `nameFragment`, `includeRetired`, pagination `start`/`length`, attribute map). These are passed directly to DAO queries.

## Edge cases & failure modes
- **Missing name on save** – throws `APIException("Location.name.required")`. (`LocationServiceImpl.saveLocation`, line 61).  
- **Transient tag without name** – throws `APIException("Location.tag.name.required")`. (`LocationServiceImpl.saveLocation`, line 66).  
- **Transient tag not found in DB** – throws `APIException("Location.cannot.add.transient.tags")`. (`LocationServiceImpl.saveLocation`, line 73‑78).  
- **Retire tag without reason** – throws `APIException("Location.retired.reason.required")`. (`LocationServiceImpl.retireLocationTag`, line 197‑199).  
- **Null or empty search strings** – `LocationServiceImpl.getLocationTags` returns all tags when the search string is empty. (`LocationServiceImpl.getLocationTags`, line 221‑225).  
- **Pagination parameters** – if `start` or `length` are null/zero, the DAO query does not apply limits, returning all matches. (`HibernateLocationDAO.getLocations`, lines 267‑274).  
- **Default location fallback** – if the configured default name does not exist, the service tries “Unknown Location”, then “Unknown”, then finally the location with id = 1. (`LocationServiceImpl.getDefaultLocation`, lines 136‑166).  

## Open questions
- **Hierarchy persistence details** – while `Location` has a `parentLocation` field and a `childLocations` collection, the DAO does not contain explicit queries for moving nodes; how are deep hierarchy updates (e.g., re‑parenting) handled in practice?  
- **Attribute value validation** – `CustomDatatypeUtil.saveAttributesIfNecessary` is invoked, but the concrete validation rules for each `LocationAttributeType` are defined elsewhere; what constraints are enforced at the database level?  
- **Cache invalidation** – the `@Cache(READ_WRITE)` annotation enables second‑level caching, but the code does not show explicit cache eviction on retire/purge; does Hibernate manage this automatically?  
- **Concurrency handling** – the service methods are transactional, but there is no explicit optimistic‑locking version field on `Location`; how does the system prevent lost updates in concurrent environments?