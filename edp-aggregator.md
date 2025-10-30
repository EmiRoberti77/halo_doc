### EDP Aggregator: Architecture and Data Flow

This document explains how the EDP Aggregator works: its components, storage, control flow, and how it produces measurement results. It references code and protos under `src/main/kotlin/org/wfanet/measurement/edpaggregator/...` and `src/main/proto/wfa/measurement/edpaggregator/...`.

Sections
- Components overview
- Storage and metadata
- Control plane interactions
- Data processing pipeline
- Results fulfillment modes
- Crypto and key handling
- End to end sequence

### Components overview

```mermaid
flowchart LR
  Kingdom[Kingdom Orchestrator]
  EDPs[Event Data Providers]
  RF[Requisition Fetcher]
  EG[Event Group Sync]
  DA[Data Availability Sync]
  RP[Results Fulfiller]
  Store[Storage and Metadata]

  Kingdom --> RF
  EDPs --> RF
  RF --> EG
  EG --> DA
  DA --> RP
  RP --> Kingdom
  RP --> Store
```

- Requisition Fetcher: fetches and groups requisitions, validates prerequisites.
- Event Group Sync: syncs event group metadata and mapping.
- Data Availability Sync: tracks blob presence and readiness in storage.
- Results Fulfiller: reads eligible inputs and produces results for the Kingdom.
- Storage and Metadata: encrypted object storage plus metadata in Spanner or equivalent.

### Storage and metadata

Relevant files:
- Schema SQL: `src/main/resources/edpaggregator/spanner/...`
- Protos: `src/main/proto/wfa/measurement/edpaggregator/v1alpha/...`

Key entities:
- Requisition metadata: state of each requisition and references to blobs.
- Impression metadata: event level artifact references and validation flags.
- Grouped requisitions: batches that can be processed together.

```mermaid
flowchart TB
  RM[Requisition metadata]
  IM[Impression metadata]
  GR[Grouped requisitions]
  BLOBS[Encrypted blobs in object store]

  RM --> GR
  IM --> GR
  GR --> BLOBS
```

### Control plane interactions

The Aggregator reacts to control plane signals and schedules:
- Receives requisitions from Kingdom
- Reports statuses and final results back to Kingdom
- Enforces policy and privacy budget configured by Kingdom

```mermaid
flowchart LR
  K1[Kingdom API]
  RF[Requisition Fetcher]
  RP[Results Fulfiller]

  K1 --> RF
  RF --> K1
  RP --> K1
```

### Data processing pipeline

Internally, the Results Fulfiller runs a batched pipeline. Based on the test and implementation classes, a simplified view:

```mermaid
flowchart LR
  SRC[Event and impression sources]
  ER[Event Reader]
  FP[Filter Processor]
  IV[Impression vector builder]
  FV[Frequency vector sink]
  OUT[Measurement outputs]

  SRC --> ER --> FP --> IV --> FV --> OUT
```

Notes
- Event Reader abstracts storage reads and format decoding.
- Filter Processor applies report filters such as time ranges and audience predicates.
- Impression vector builder prepares per user or per key frequencies suitable for reach and kplus reach computation.
- Frequency vector sink writes intermediate vectors for downstream aggregation or direct computation depending on the mode.

### Results fulfillment modes

Two primary modes are supported by the fulfillment layer:
- Direct mode: local computation of aggregates from available inputs
- HMShuffle mode: generation or consumption of shuffled contributions for a shuffle model pipeline

```mermaid
flowchart TB
  IN[Eligible requisition batch]
  DEC{Fulfillment mode}
  DIR[Direct computation]
  HMS[HMShuffle contribution]
  RES[Result to Kingdom]

  IN --> DEC
  DEC --> DIR
  DEC --> HMS
  DIR --> RES
  HMS --> RES
```

Related code:
- ResultsFulfillerApp and ResultsFulfiller
- DirectMeasurementResultFactory and builders for reach frequency impression
- HMShuffleMeasurementFulfiller for shuffle model

### Crypto and key handling

The aggregator reads encrypted blobs and unwraps data keys securely as configured:
- Envelope encryption with encrypted DEKs
- Service specific KEK integration
- Transport layer security parameters under `src/main/proto/wfa/measurement/config/edpaggregator/...`

```mermaid
flowchart LR
  OBJ[Encrypted blob]
  EDEK[Encrypted DEK]
  KEK[Key encryption key service]
  DKEY[Decrypted DEK]
  PLAINTXT[Decrypted content]

  OBJ --> EDEK
  EDEK --> KEK --> DKEY --> PLAINTXT
```

### End to end sequence

```mermaid
sequenceDiagram
  autonumber
  participant KG as Kingdom
  participant RF as Requisition Fetcher
  participant DA as Data Availability
  participant RP as Results Fulfiller
  participant ST as Storage

  KG->>RF: Create requisitions and assignments
  RF->>ST: Ensure blob references and metadata
  RF->>KG: Ack and state updates
  RF->>DA: Signal batch ready for availability checks
  DA->>ST: Verify presence and integrity
  DA->>RP: Notify ready batch
  RP->>ST: Read inputs
  RP->>RP: Build vectors and compute aggregates
  RP->>KG: Send results
```

### References in repo

- Kotlin implementation: `src/main/kotlin/org/wfanet/measurement/edpaggregator/...`
- Configuration protos: `src/main/proto/wfa/measurement/config/edpaggregator/...`
- API and metadata protos: `src/main/proto/wfa/measurement/edpaggregator/v1alpha/...`
- Spanner DDL: `src/main/resources/edpaggregator/spanner/...`


