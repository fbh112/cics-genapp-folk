# To-Be Architecture — Customer Domain Migrated to Cloud

Domain: **Customer Management** (migrated)
Remaining on mainframe: **Policy**, **Web Services (LGWEBST5)**, **Test Harness**, **Infrastructure**

Based on boundary analysis ([`04`](04-domain-customer-boundary.md)), migration plan ([`08`](08-domain-customer-migration-plan.md)).

---

## 1. Architecture Overview

In the to-be state, the Customer Management domain runs as a **cloud-hosted microservice** exposing a REST API. The mainframe retains the Policy domain, Test Harness, Web Services entry point, and shared infrastructure (`LGSTSQ`). The two systems are integrated via a **CICS adapter layer** on the mainframe that bridges CICS LINK calls to HTTP REST calls against the Customer service.

```mermaid
graph TB
    subgraph Cloud["☁️ Cloud Platform — Customer Service"]
        direction TB

        subgraph CustomerAPI["Customer REST API"]
            AC["POST /customers\n(Add Customer)"]
            IC["GET /customers/{id}\n(Inquire Customer)"]
            UC["PUT /customers/{id}\n(Update Customer)"]
            RND["GET /customers/random\n(Random Customer)"]
        end

        subgraph CustomerLogic["Customer Service Logic"]
            SVC["Customer Service\n(business logic)"]
            LOG["Cloud Logger\n(replaces LGSTSQ)"]
            SEQ["DB Sequence\n(replaces CICS NCS)"]
        end

        subgraph CustomerData["Customer Data Stores"]
            CUST_DB[("Customer DB\nCUSTOMER table\n(migrated from DB2)")]
            CUSTSEC_DB[("Customer Secure DB\nCUSTOMER_SECURE table\n(migrated/retired)")]
        end

        AC --> SVC
        IC --> SVC
        UC --> SVC
        RND --> SVC
        SVC --> CUST_DB
        SVC --> CUSTSEC_DB
        SVC --> LOG
        SVC --> SEQ
    end

    subgraph Mainframe["🖥️ IBM Z Mainframe — Remaining Domains"]
        direction TB

        subgraph Adapter["CICS Adapter Layer\n(new — bridges LINK to REST)"]
            ADAPT_AC["LGACUS01-STUB\nCICS LINK → REST POST /customers"]
            ADAPT_IC["LGICUS01-STUB\nCICS LINK → REST GET /customers/{id}"]
            ADAPT_UC["LGUCUS01-STUB\nCICS LINK → REST PUT /customers/{id}"]
        end

        subgraph TestHarness["Test Harness Domain"]
            TC1["LGTESTC1\n(SSC1)"]
        end

        subgraph WebSvc["Web Services Domain"]
            WS5["LGWEBST5\n(SSST)\nHTTP/SOAP entry point"]
        end

        subgraph Policy["Policy Domain"]
            PPROG["LGAPOL01 · LGIPOL01\nLGUPOL01 · LGDPOL01\n(+ DB2 + VSAM backends)"]
        end

        subgraph Infra["Infrastructure"]
            LGSTSQ["LGSTSQ\n(TSQ logging\nPolicy domain only)"]
            GENACNTL["GENACNTL TSQ\n(retired from Customer)"]
        end

        subgraph MF_Data["Mainframe Data Stores"]
            POL_DB[("POLICY · MOTOR\nHOUSE · ENDOWMENT\nCOMMERCIAL\n(DB2 — Policy domain)")]
            KSDSPOLY_V[("KSDSPOLY VSAM\n(Policy domain)")]
            KSDSCUST_V[("KSDSCUST VSAM\n⚠️ retired")]
        end

        subgraph MF_DB2["Mainframe DB2"]
            CUST_SHADOW[("CUSTOMER shadow table\n(optional — for POLICY FK)\nread-only replica")]
        end
    end

    %% Test Harness calls adapter stubs (replaces direct CICS LINK to Customer programs)
    TC1 -->|"CICS LINK\n(unchanged call)"| ADAPT_AC
    TC1 -->|"CICS LINK\n(unchanged call)"| ADAPT_IC
    TC1 -->|"CICS LINK\n(unchanged call)"| ADAPT_UC

    %% Web Services routes to adapter stubs
    WS5 -->|"CICS LINK\n(rerouted)"| ADAPT_AC
    WS5 -->|"CICS LINK\n(rerouted)"| ADAPT_IC
    WS5 -->|"CICS LINK\n(rerouted)"| ADAPT_UC

    %% Adapter stubs call Customer cloud API over HTTP
    ADAPT_AC -->|"HTTP POST\nJSON payload\n(from LGCMAREA)"| AC
    ADAPT_IC -->|"HTTP GET\ncustomerId"| IC
    ADAPT_UC -->|"HTTP PUT\nJSON payload"| UC

    %% Policy domain checks customer existence (no FK in DB2 after migration)
    PPROG -->|"HTTP GET /customers/{id}\n(customer existence check)"| IC

    %% Policy domain still uses LGSTSQ
    PPROG -->|"CICS LINK"| LGSTSQ

    %% Policy data stays on mainframe
    PPROG --> POL_DB
    PPROG --> KSDSPOLY_V

    %% Shadow CUSTOMER table for POLICY FK (optional)
    CUST_DB -.->|"async sync\n(event / feed)"| CUST_SHADOW
    CUST_SHADOW -.->|"FK reference\n(optional)"| POL_DB

    %% Retired components
    KSDSCUST_V -. "retired" .-x ADAPT_AC
```

---

## 2. Component Inventory — To-Be State

### 2.1 New components (cloud)

| Component | Type | Description |
|---|---|---|
| **Customer REST API** | Cloud microservice | Exposes Add, Inquire, Update Customer as REST endpoints; replaces LGACUS01, LGICUS01, LGUCUS01 and their backends |
| **Customer DB** | Cloud relational DB | Holds `CUSTOMER` and `CUSTOMER_SECURE` tables; migrated from mainframe DB2 |
| **DB Sequence / Identity** | Cloud DB feature | Replaces CICS Named Counter Service (`GENACUSTNUM`); generates customer numbers |
| **Cloud Logger** | Cloud logging service | Replaces `LGSTSQ` calls in Customer programs |

### 2.2 New components (mainframe adapter layer)

| Component | Type | Description |
|---|---|---|
| **LGACUS01-STUB** | CICS COBOL stub | Replaces `LGACUS01` on mainframe; receives CICS LINK with `LGCMAREA` COMMAREA, marshals to JSON, calls `POST /customers`, unmarshals response back to COMMAREA |
| **LGICUS01-STUB** | CICS COBOL stub | Replaces `LGICUS01`; calls `GET /customers/{id}`, maps response to COMMAREA |
| **LGUCUS01-STUB** | CICS COBOL stub | Replaces `LGUCUS01`; calls `PUT /customers/{id}`, maps response to COMMAREA |

> The stubs preserve the exact CICS LINK interface (`DFHCOMMAREA` with `LGCMAREA` layout and `LENGTH(32500)`). Callers (`LGTESTC1`, `LGWEBST5`) require **no code changes** — they still issue the same CICS LINK calls against the same program names.

### 2.3 Components retired

| Component | Reason |
|---|---|
| `LGACDB01`, `LGACDB02`, `LGACVS01` | Replaced by Customer REST API Add endpoint |
| `LGICDB01`, `LGICVS01` | Replaced by Customer REST API Inquire endpoint |
| `LGUCDB01`, `LGUCVS01` | Replaced by Customer REST API Update endpoint |
| `KSDSCUST` VSAM file | Retired — DB is the single store; random lookup via DB query |
| CICS NCS `GENACUSTNUM` | Retired — replaced by DB sequence |
| `GENACNTL` TSQ | Retired — random customer lookup via DB query |

### 2.4 Components unchanged on mainframe

| Component | Status |
|---|---|
| `LGTESTC1` | Unchanged — still issues CICS LINK to stub programs |
| `LGTESTP1–4` | Unchanged — Policy domain; unaffected |
| `LGWEBST5` | Unchanged if stubs match program names; or rerouted to call Customer API directly |
| All Policy programs | Unchanged — Policy domain stays on mainframe |
| `LGSTSQ` | Unchanged — used only by Policy domain after Customer migration |
| `LGCMAREA` copybook | Unchanged — remains the COMMAREA contract between stubs and callers |
| `POLICY`, `MOTOR`, `HOUSE`, `ENDOWMENT`, `COMMERCIAL` DB2 tables | Unchanged |
| `KSDSPOLY` VSAM | Unchanged |

---

## 3. Integration Patterns

### 3.1 CICS Adapter Stub Pattern

The adapter stubs are the key integration mechanism. Each stub:

1. Receives `EXEC CICS LINK` from an existing caller with a standard `LGCMAREA` COMMAREA
2. Extracts the relevant fields from the COMMAREA
3. Marshals them to a JSON request body
4. Calls the Customer REST API using `EXEC CICS WEB SEND` or CICS Web Services outbound call
5. Receives the JSON response
6. Unmarshals the response fields back into the COMMAREA
7. Returns to the caller via `EXEC CICS RETURN`

```
LGTESTC1                LGACUS01-STUB (mainframe)           Customer API (cloud)
    │                           │                                    │
    │  EXEC CICS LINK            │                                    │
    │  PROGRAM('LGACUS01')       │                                    │
    │  COMMAREA(LGCMAREA)  ────► │                                    │
    │                            │  marshal LGCMAREA to JSON          │
    │                            │  EXEC CICS WEB SEND ─────────────► │
    │                            │                          POST /customers
    │                            │                          { firstName, lastName, ... }
    │                            │ ◄───────────────────────────────── │
    │                            │  { customerId: 1000042, rc: "00" } │
    │                            │  unmarshal → LGCMAREA              │
    │ ◄────────────────────────  │                                    │
    │  CA-RETURN-CODE='00'        │                                    │
    │  CA-CUSTOMER-NUM=1000042    │                                    │
```

### 3.2 COMMAREA → REST Mapping

| Operation | COMMAREA fields (in) | REST call | COMMAREA fields (out) |
|---|---|---|---|
| Add Customer | `CA-FIRST-NAME`, `CA-LAST-NAME`, `CA-DOB`, `CA-HOUSE-NAME`, `CA-HOUSE-NUM`, `CA-POSTCODE`, `CA-PHONE-MOBILE`, `CA-PHONE-HOME`, `CA-EMAIL-ADDRESS` | `POST /customers` | `CA-CUSTOMER-NUM` (assigned), `CA-RETURN-CODE` |
| Inquire Customer | `CA-CUSTOMER-NUM` | `GET /customers/{id}` | All 9 customer fields, `CA-RETURN-CODE` |
| Update Customer | `CA-CUSTOMER-NUM` + all 9 fields | `PUT /customers/{id}` | `CA-RETURN-CODE` |

### 3.3 Policy FK Management

After migration, the mainframe DB2 `POLICY` table's FK `REFERENCES customer(customerNumber)` points to a table that no longer exists in mainframe DB2. Two options:

**Option A (recommended for demo)** — Application-level enforcement:

```
LGAPOL01 (Policy Orchestrator)
    │
    ├── Call Customer API: GET /customers/{CA-CUSTOMER-NUM}
    │   ├── RC=00 (found) → proceed with policy insert
    │   └── RC=01 (not found) → return error to caller
    │
    └── EXEC CICS LINK LGAPDB01 (Policy DB2 insert)
```

**Option B (production)** — Shadow CUSTOMER table:
- Customer API publishes `CustomerCreated` / `CustomerUpdated` events to a message bus (e.g., IBM MQ, Kafka)
- A mainframe consumer writes to a read-only shadow `CUSTOMER` table in DB2
- `POLICY` FK constraint references the shadow table — RI preserved without application changes

```mermaid
sequenceDiagram
    participant CS as Customer Service (cloud)
    participant MQ as Message Bus (MQ/Kafka)
    participant MF as Mainframe Consumer
    participant DB2 as Mainframe DB2 shadow CUSTOMER

    CS->>MQ: publish CustomerCreated {id, ...}
    MQ->>MF: deliver event
    MF->>DB2: INSERT INTO CUSTOMER (id only)
    Note over DB2: POLICY FK now satisfiable
```

---

## 4. To-Be Architecture — Layered View

```mermaid
graph LR
    subgraph ExternalClients["External Clients"]
        HTTP_CLIENT["HTTP/SOAP Client\n(existing consumers)"]
        TN3270["3270 Terminal\n(test operator)"]
    end

    subgraph MainframeCICS["IBM Z Mainframe — CICS Region"]
        WS5B["LGWEBST5\nSOAP/HTTP → CICS"]

        subgraph Stubs["Adapter Stubs (new)"]
            S1["LGACUS01-STUB"]
            S2["LGICUS01-STUB"]
            S3["LGUCUS01-STUB"]
        end

        subgraph TestHarnessB["Test Harness"]
            TC1B["LGTESTC1 (SSC1)"]
        end

        subgraph PolicyB["Policy Domain (unchanged)"]
            POL_PROGS["LGAPOL01/DB01/VS01\nLGIPOL01/DB01/VS01\nLGUPOL01/DB01/VS01\nLGDPOL01/DB01/VS01"]
        end

        LGSTSQB["LGSTSQ\n(Policy logging)"]
    end

    subgraph MainframeDB2["IBM Z — DB2"]
        POL_TBLS[("POLICY · MOTOR\nHOUSE · ENDOWMENT\nCOMMERCIAL")]
        CUST_SHADOW_B[("CUSTOMER shadow\n(optional FK replica)")]
        KSDSPOLY_B[("KSDSPOLY VSAM")]
    end

    subgraph CloudPlatform["☁️ Cloud Platform"]
        subgraph CustomerSvc["Customer Microservice"]
            REST_API["REST API\nPOST · GET · PUT\n/customers"]
            BIZ["Business Logic\n(Add/Inquire/Update)"]
            CLOG["Cloud Logger"]
        end

        subgraph CloudDB["Cloud Database"]
            CUST_TBL[("CUSTOMER")]
            CUSTSEC_TBL[("CUSTOMER_SECURE")]
        end

        MQ_BUS["Message Bus\n(optional: CustomerCreated events)"]
    end

    %% External → Mainframe
    HTTP_CLIENT -->|SOAP/HTTP| WS5B
    TN3270 -->|3270| TC1B

    %% Mainframe → Stubs
    WS5B -->|CICS LINK| S1 & S2 & S3
    TC1B -->|CICS LINK| S1 & S2 & S3

    %% Stubs → Cloud API (outbound HTTP from CICS)
    S1 -->|"HTTP POST /customers"| REST_API
    S2 -->|"HTTP GET /customers/{id}"| REST_API
    S3 -->|"HTTP PUT /customers/{id}"| REST_API

    %% Cloud API internals
    REST_API --> BIZ
    BIZ --> CUST_TBL & CUSTSEC_TBL
    BIZ --> CLOG
    BIZ -->|CustomerCreated event| MQ_BUS

    %% Policy on mainframe
    PolicyB -->|SQL| POL_TBLS
    PolicyB -->|CICS File| KSDSPOLY_B
    PolicyB -->|CICS LINK| LGSTSQB
    PolicyB -->|"HTTP GET /customers/{id}\n(customer validation)"| REST_API

    %% Shadow table sync (optional)
    MQ_BUS -.->|"async feed"| CUST_SHADOW_B
    CUST_SHADOW_B -.->|"FK reference"| POL_TBLS
```

---

## 5. Key Architecture Decisions

| Decision | Choice | Rationale |
|---|---|---|
| Integration pattern | **CICS adapter stubs on mainframe** | Callers (`LGTESTC1`, `LGWEBST5`) need zero code changes; stub preserves the existing CICS LINK contract |
| API protocol | **REST / JSON over HTTP** | Industry standard; CICS supports outbound HTTP via `EXEC CICS WEB SEND`; simple to expose from cloud |
| Customer number generation | **DB sequence (identity column)** | Already a fallback in `LGACDB01`; no schema change needed |
| VSAM retirement | **Retire KSDSCUST** | DB is the authoritative store; VSAM was a mirror only |
| POLICY FK management | **Application-level enforcement (demo)** or **shadow table (production)** | Demo: simpler, no schema change; Production: preserves DB RI |
| Security record (`CUSTOMER_SECURE`) | **Carry forward** | Minimal effort; preserves intent of future auth flow |
| Logging | **Cloud-native logging replaces LGSTSQ** | LGSTSQ stays on mainframe for Policy domain; Customer service uses cloud observability |
| Policy customer validation | **HTTP GET /customers/{id} before policy insert** | Replaces DB2 FK constraint; ensures cross-platform referential integrity |
