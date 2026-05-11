# Processing Activity: Order Management (PA‑006)

## 1. Overview

| Field | Value |
|---|---|
| **PA Name** | Order Management (PA‑006) |
| **Business Function** | Creation, update, retrieval, discontinuation and fulfillment‑status handling of clinical orders (e.g., lab tests, drug prescriptions) |
| **Processing Purpose** | To enable clinicians to request, modify and track healthcare services for patients, which is essential for the delivery of medical care |
| **Legal Basis** | **GDPR Art. 6(1)(e)** – processing is necessary for the performance of a task carried out in the public interest (healthcare provision) |
| **Controller** | OpenMRS Core project (the organisation that deploys and operates the OpenMRS instance) |

## 2. Data Subjects
- **Patients** – individuals for whom orders are placed (identified by `patient_id`).  
- **Healthcare Providers** – clinicians or staff who create or modify orders (their identity is captured indirectly via audit fields such as `creator`/`changedBy` on the `Order` entity).

## 3. Personal Data Elements  

| Data Element | Source (code / DB) | Sensitivity (GDPR) | Retention |
|---|---|---|---|
| `order_type` | `Order` class (`org.openmrs.Order`) – populated in `OrderService.saveOrder` (./api/src/main/java/org/openmrs/api/OrderService.java:86) and persisted by `OrderDAO.saveOrder` (./api/src/main/java/org/openmrs/api/db/OrderDAO.java). Stored in relational table **`orders`** (column `order_type_id`). | Low (operational) | Kept for the duration of the clinical episode plus any statutory retention period for medical records (typically 10 years in many jurisdictions). |
| `patient_id` | `Order` holds a reference to a `Patient` (`patient_id` column in **`orders`**). Populated in the same service/DAO flow as above. | **High – Special Category** (health data about a identified person) – GDPR Art. 9(1)(a). | Same as `order_type`. |
| `order_date` | Set in `Order` when the order is created (field `orderDate`). Populated by `OrderService.saveOrder` and persisted in **`orders.order_date`**. | Low | Same as `order_type`. |

*All audit fields (`dateCreated`, `dateChanged`, `creator`, `changedBy`) are automatically managed by the OpenMRS `OpenmrsObject` base class and are stored in the same `orders` table.*

## 4. Data Flows  

### Primary (active) flow
```
[Healthcare Provider UI / REST client] 
   --> HTTP(S) POST /openmrs/ws/rest/v1/order   (REST controller in org.openmrs.web.controller.OrderController)
   --> OrderService.saveOrder(Order, OrderContext)   (./api/src/main/java/org/openmrs/api/OrderService.java:86)
   --> OrderDAO.saveOrder(Order)                     (./api/src/main/java/org/openmrs/api/db/OrderDAO.java)
   --> Relational DB (MySQL/MariaDB or PostgreSQL) – table `orders`
```

### Retrieval flow
```
[Healthcare Provider UI / REST client] 
   --> HTTP(S) GET /openmrs/ws/rest/v1/order/{uuid}
   --> OrderService.getOrderByUuid(uuid)   (./api/src/main/java/org/openmrs/api/OrderService.java:190‑202)
   --> OrderDAO.getOrderByUuid(uuid)       (./api/src/main/java/org/openmrs/api/db/OrderDAO.java)
   --> Relational DB → return Order JSON
```

### Legacy / Commented‑out flows
- No legacy or commented‑out order‑related code is present in the current OpenMRS 2.8‑SNAPSHOT snapshot.

## 5. Third Parties / Processors  

| Vendor | Role | Data Shared | Hosting |
|---|---|---|---|
| **None** (as per the supplied context) | – | – | – |

*The processing is entirely internal: the order data never leaves the OpenMRS container or its backing relational database.*

## 6. Security Measures  

### Positive (implemented) controls
- **Authentication** – `Context.authenticate(...)` validates user credentials before any service call (see **Security** section, `Context.authenticate` implementation).  
- **Authorization** – All Order‑service methods are protected by `@Authorized` annotations (e.g., `@Authorized({ADD_ORDERS, EDIT_ORDERS})` on `saveOrder`).  
- **Transport security** – All ingress/egress traffic is expected to be over HTTPS (TLS termination at the Tomcat front‑end).  
- **Database security** – The relational DB is accessed via a JDBC URL built in `startup-init.sh` (./startup‑init.sh:84‑106) and should be protected by network‑level firewalls and DB‑level authentication.  
- **Audit logging** – Every create, update, void or delete operation records `creator`, `dateCreated`, `changedBy`, `dateChanged` automatically via Hibernate’s entity listeners.  

### Specific concerns for PA‑006
- **Risk of unauthorized access to `patient_id`** (special‑category data).  
- **Potential exposure of order details via mis‑configured REST endpoints** (e.g., missing role checks).  
- **Insufficient encryption at rest if the underlying DB is not encrypted**.

*Mitigation recommendations*: enforce least‑privilege roles, enable DB‑level encryption (transparent data encryption), and conduct regular penetration testing of the REST API.

## 7. Cross‑Border Transfers
- No cross‑border data transfers are performed. All processing, storage and backup occur within the same jurisdiction where the OpenMRS instance is deployed.

## 8. DPIA Trigger Assessment  

| Factor | Applicable? | Comments |
|---|---|---|
| Large‑scale processing | **No** | Order volume is limited to the patient population of the facility. |
| Processing of special‑category data | **Yes** | `patient_id` links to health‑care data (order type, dates). |
| Automated decision‑making with legal or similarly significant effect | **No** | Orders are created/modified by humans; no automated profiling. |
| Systematic monitoring of individuals | **No** | Orders are transactional, not monitoring. |
| Vulnerable data subjects | **Yes** | Patients are a vulnerable group under GDPR Art. 4(15). |
| Use of new or emerging technologies | **No** | Standard Java/Spring/Hibernate stack. |
| Cross‑border transfers | **No** | All data stays on‑premises. |
| **DPIA Recommended** | **Yes** | Because special‑category data are processed and vulnerable subjects are involved, a DPIA should be carried out to verify that all safeguards are adequate. |

---  

**Prepared on:** 2026‑05‑11  
**Prepared by:** OpenMRS ROPA generation assistant (based on the provided system documentation).