# HACCAM Core Runtime System Design V0.1

**Title:** HACCAM Core Runtime System Design V0.1
**Document filename:** `HACCAM_CORE_RUNTIME_SYSTEM_DESIGN_V0_1_090926.md`
**Date:** 2026-09-09
**Version:** V0.1
**Status:** **APPROVED**
**Implementation:** **NOT YET AUTHORIZED**
**Parent Architecture Authority:** `HACCAM_PROCESS_MODEL_V0_1_090726.md`, dated 2026-09-07, V0.1, PROPOSED FOR HUMAN REVIEW
**Purpose:** Define the complete first HACCAM Core realtime runtime design from which a later Go server and protobuf contract can be implemented without inventing architecture.
**Scope:** One domain-neutral HACCAM Core runtime, the existing `Fin_FeedSat_1` Producer boundary, external Consumer delivery, and the implementation seams needed for later TransSat and ObsSat participation. This document does not implement any runtime, proto, satellite, persistence, or deployment.

## Executive Summary

This document is subordinate to the HACCAM Process Model. The Process Model defines HACCAM architecture, terminology, ownership, and H-01 through H-09. This Runtime System Design selects a conservative first Go organization that realizes those responsibilities in one process without treating each H-process as a microservice.

The first HACCAM Core is one domain-neutral Go server/runtime containing:

- configured, in-memory H-02 Registry KB knowledge for internal/external participants and services;
- an in-memory H-03 contract catalog and compatibility rules;
- in-memory H-04 subscriptions;
- H-05 Routing Table governance and bounded asynchronous `RouterService` publication;
- latest-state-only H-06 Intelligence Mirrors;
- H-07 lifecycle state;
- H-08 isolation at adapter, event, route, and receiver boundaries;
- H-09 structured logs and bounded metrics;
- H-01 adapters that connect outward to Producer-owned publication boundaries.

The first H-01 adapter is specific to the existing `finfeedsat.v1` gRPC boundary. It connects as a client to `Fin_FeedSat_1` and listens to `StreamDecisions`, `StreamPriceEvents`, and `StreamVolumeEvents`. The FeedSat is not changed to push into HACCAM. The adapter validates participant, contract, message identity, and routing-relevant provenance, then preserves the satellite-owned serialized protobuf message without translating its science into a HACCAM schema. Generated satellite protobuf types remain satellite-owned edge contracts.

The first external HACCAM server boundary is intentionally small: an external Consumer identifies itself and expresses H-04 interest, H-05_def/gov derives Routing Table rows from the H-02 Registry KB, H-03 compatibility, and that intent, and the HACCAM-owned `RouterService` executes H-05_deliver publication. The Registry KB, Routing Table, and `RouterService` are distinct. H-06 is updated before `RouterService` delivery eligibility is evaluated. Each destination stream has a bounded queue and independent goroutine. A full queue drops delivery for that receiver, records the failure, and never blocks H-01, H-06, another route, or satellite science.

The strongest runtime invariant is:

> HACCAM governs participant identity, service and contract compatibility, registration, subscription, routing, mirroring, lifecycle, failure isolation, diagnostics, and delivery. HACCAM does not determine how many satellites exist, how many entities a satellite owns, how many information streams it publishes, or what authoritative information a satellite chooses to create.

Accordingly, the design supports `0..N FeedSat`, `0..N TransSat`, and `0..N ObsSat`. It contains no AAPL, MSFT, NVDA, or AMZN enum, constant, route, mirror slot, or assumption. Those are current `Fin_FeedSat_1` configuration facts. The local FeedSat code supports a configurable symbol set and currently defaults to one symbol; its current four-symbol live configuration does not become HACCAM architecture.

V0.1 is realtime and in-memory. Restart loses runtime availability evidence, subscriptions, derived Routing Table rows, lifecycle observations, mirrors, delivery queues, counters, and diagnostic continuity. Configured Registry KB and contract definitions are reconstructed from configuration. Producer streams are reconnected and mirrors rebuild only from each Producer's current catch-up and subsequent live events. No replay, exactly-once, durable history, or restoration is claimed.

## 1. Authority, Precedence, and Document Relationship

```text
HACCAM Process Model V0.1
        |
        | architectural and terminology authority
        v
HACCAM Core Runtime System Design V0.1
        |
        | implementation design authority after human approval
        v
future HACCAM Go + Proto implementation
```

If this design conflicts with the Process Model, the Process Model controls. Current implementation evidence may produce a reconciliation note, but it does not silently modify architecture.

This document uses H-01 through H-09 as logical responsibilities. It does not redefine them as executables, packages, network services, or a sequential pipeline.

## 2. Runtime Definition and Boundary

### 2.1 What HACCAM Core is

HACCAM Core is the first executable realization of `HAC_Core_Matrix` and common satellite services. It is the runtime owner of Matrix relationships, registration state, HACCAM-visible contract compatibility, subscriptions, routes, delivery state, Intelligence Mirror objects, lifecycle view, isolation decisions, and diagnostics.

### 2.2 What HACCAM Core is not

HACCAM Core is not:

- a Satellite and therefore has no `SatelliteRole` or `SubscriberType`;
- a Finance scientific processor;
- an owner of FeedSat Adaptive, Price, Volume, Bar, or other authoritative intelligence;
- a scientific super-schema, scientific validator, or interpreter of satellite mathematics;
- an owner of future TransSat-created authoritative information;
- a controller required for FeedSat science;
- a durable event store, broker, replay system, or persistence layer;
- a peer-satellite shortcut;
- a production security boundary in V0.1.

### 2.3 Cardinality and independence

The runtime model is:

```text
0..N FeedSat
0..N TransSat
0..N ObsSat
```

Each satellite has exactly one explicit `SatelliteRole`, one explicit real `SubscriberType`, one Domain Plane association, and one unique `satellite_id` within the runtime. `SatelliteRole` and `SubscriberType` remain independent classifications.

The runtime never derives entity count or information-type count from role. An entity identifier is opaque to HACCAM except for validation under H-03 and exact matching under H-04. HACCAM neither allocates entity identifiers nor dictates satellite science.

### 2.4 Logical identity is independent of deployment

Physical deployment location is not part of satellite identity. A host, process, container, pod, mobile device, application, or network endpoint may host `0..N` logical HACCAM satellites. Co-location changes none of the following:

- `satellite_id`, `SatelliteRole`, or `SubscriberType`;
- semantic or scientific ownership;
- satellite-owned contract ownership;
- Domain Plane association;
- HCM relationships or governed HACCAM paths.

Future packaging may place `Fin_FeedSat_1` and the Decision Strategy Engine in one Android application or device. Bundled does not mean coupled. They remain separate logical satellites, and their required relationship remains FeedSat -> HACCAM -> Decision Strategy Engine. No private in-process scientific shortcut is permitted.

## 3. Observed `Fin_FeedSat_1` Boundary

Read-only inspection on 2026-09-09 observed local repository HEAD `3d763a1` on `main`, with no changes made during inspection.

| Fact | Current observation | Runtime consequence |
| :--- | :--- | :--- |
| Architectural identity | defaults: `Fin_FeedSat_1`, `FeedSat`, `PRODUCER`, `Finance` | H-02 config validates these as separate fields |
| Proto package | `finfeedsat.v1`; Go package `finfeedsatv1` | first H-01 adapter imports this edge contract |
| Services | `MarketFeedService`, `IngestionService`, `OperationsService`, `ModelService`, `SemanticService` | adapter uses only justified read boundaries |
| Scientific streams | `StreamDecisions`, `StreamPriceEvents`, `StreamVolumeEvents` | first Matrix-visible information candidates |
| Optional observation | `StreamBars` | not a required intelligence path; may be diagnostic later |
| Health/readiness | `GetFeedHealth`, `GetActiveSource`, `GetHealth`, `GetReadiness` | H-07 evidence; unary polling only |
| Semantic contract | `GetTerm`, `ListTerms`, `GetSemanticContract`; contract version `1.0` | H-03 edge evidence, not realtime science input |
| Entity identity | `symbol`; Bar also has `instrument_id` | adapter maps source entity without imposing HACCAM cardinality |
| Lineage | `event_id`, `market_snapshot_id`, `source_timestamp`, interval time, `accepted_sequence` | retained where supplied |
| P-03 detail | decision or typed skip; model/schema versions and state hashes | Finance Domain payload; skip is not fabricated intelligence |
| P-04 detail | status, emission, cockpit, or skip | Finance Domain payload; Price scalars absent from edge remain absent |
| P-04V detail | status, emission, or skip; interval start/end | Finance Domain payload; no independent EffectiveTime is invented |
| Catch-up | latest event per symbol precedes live stream; duplicate `event_id` suppressed at catch-up boundary | HACCAM receives latest-plus-live, not history |
| Cancellation | handlers observe stream context and unsubscribe | H-01 cancellation closes client streams cleanly |
| Slow client | FeedSat subscriber queue is bounded at 16 and publication is nonblocking/drop-on-full | HACCAM must consume promptly and expose possible gaps |
| Symbols | source config accepts a configurable list, maximum 30 on current plan; default `AAPL` | current AAPL/MSFT/NVDA/AMZN use is configuration, never architecture |

The adapter must not call `TriggerGapFill`; that RPC mutates FeedSat ingestion and is outside HACCAM mirroring.

## 4. Fundamental Information and Ownership Model

### 4.1 One owner per fact

| Fact | Runtime owner | Semantic authority |
| :--- | :--- | :--- |
| Satellite scientific intelligence | producing satellite | producing satellite |
| H-01 boundary acceptance and metadata extraction | H-01 | does not interpret or replace source scientific authority |
| Participant, service, endpoint, and capability knowledge | H-02 | HACCAM for its Registry KB fact |
| HACCAM-visible contract compatibility | H-03 | H-03 |
| Subscription intent | H-04 | H-04 |
| Routing Table row / permitted route | H-05_def/gov | H-05_def/gov |
| `RouterService` publication queue/state | H-05_deliver | H-05_deliver |
| Intelligence Mirror object | H-06 | H-06 owns the mirror; source satellite owns mirrored meaning |
| Lifecycle view | H-07 | H-07; source health remains source-owned |
| Isolation/drop decision | H-08 | H-08 |
| Diagnostic fact | H-09 | H-09; never authoritative intelligence |
| Future Decision Strategy | `DS_TransSat` | `DS_TransSat`, including after H-01 re-entry |

`F_i != F~_i` and `D_DS != D~_DS` express ownership and identity, not necessarily numerical inequality.

### 4.2 Internal runtime record

The minimal internal runtime record contains only HACCAM-owned metadata plus an unchanged serialized satellite-owned protobuf message:

- source `satellite_id`;
- Domain Plane identifier;
- opaque source entity identifier;
- source service identity and exact satellite-owned message identity;
- satellite-owned contract identity and version registered by H-03;
- source event identity when supplied;
- source sequence and a boolean indicating whether one was supplied;
- source provenance in its source-owned representation when required for routing or traceability;
- HACCAM receipt and mirror-update times as explicit Unix-millisecond values;
- immutable serialized bytes of the unchanged satellite-owned protobuf message.

This runtime record is not a second scientific schema. HACCAM does not define its scientific fields, decode it to judge scientific correctness, rename concepts, normalize mathematics, or take semantic ownership. The serialized bytes are only a protobuf transport representation governed by exact contract and message identity; they are not an architectural information concept or an arbitrary ungoverned byte container.

### 4.3 Boundary adaptation without scientific translation

```text
FeedSat protobuf message
        |
        v
H-01 participant/contract/message validation
and routing-provenance extraction
        |
        v
HACCAM runtime record:
metadata + unchanged satellite-owned encoded message
        |
        +--> H-06 mirror replacement
        |
        +--> H-05_deliver `RouterService` publication through active Routing Table rows

HACCAM runtime record
        |
        v
HACCAM delivery serialization
        |
        v
HACCAM protobuf Matrix-visible information
```

This separation prevents FeedSat code generation, field layout, defaults, and Finance vocabulary from becoming the HACCAM internal model. An adapter knows only enough about its registered satellite contract to identify the message and extract approved routing/provenance fields. It does not create a HACCAM Adaptive, Price, or Volume representation. Later adapters can preserve other satellite-owned contracts without rewriting H-02 through H-09.

## 5. Overall Runtime Architecture

```text
     N Producer satellites
       |       |       |
       +-------+-------+
               |
         H-01 adapters
               |
               v
      governed runtime record
     metadata + encoded message
               |
               v
      H-06 Intelligence Mirror ---------------------+
                                                    |
H-02 Registry KB --------+                          |
H-03 Contract Governance +--> H-05_def/gov          |
H-04 Subscription Intent +         |                |
                                   v                |
                             ROUTING TABLE <---------+
                                   |
                                   v
                                                                        RouterService / H-05_deliver
                           |       |       |
                           +-------+-------+
                                   |
                           N Consumer satellites

H-07 Health & Lifecycle, H-08 Failure Isolation, and H-09 Diagnostics
cross-cut adapters, state, governance, mirror updates, and delivery.
```

H-02, H-03, and H-04 are control/governance inputs. They are not sequential event-processing stages.

## 6. H-01 Matrix Ingress / Satellite Adapter

### 6.1 Architectural responsibility versus first adapter

H-01 is the canonical responsibility for all satellite-produced HACCAM-visible information. The first implementation is a `Fin_FeedSat_1`-specific outbound gRPC client adapter. The adapter is replaceable and repeatable per configured Producer; H-01 core handling is not Finance-specific.

```text
FeedSat science (independent)
        |
        v
FeedSat-owned existing gRPC publication
        |
        v
Fin_FeedSat_1 H-01 adapter (HACCAM-owned client)
        |
        v
HACCAM validation, mirror, and routing
```

### 6.2 Connection ownership and lifecycle

- HACCAM owns the outbound `grpc.ClientConn`, stream contexts, reconnect loop, and adapter goroutines.
- One adapter supervisor exists per configured Producer endpoint.
- The supervisor dials with a child context and finite dial timeout.
- After connection, it verifies registered contract compatibility once, polls H-07 evidence independently, and starts one stream worker per configured source service/message identity.
- Each stream worker receives a satellite-owned message, extracts only approved routing/provenance metadata, serializes the unchanged message under its registered contract/message identity, and submits the runtime record to one bounded adapter-to-core queue.
- The queue has a configured positive bound. A full queue rejects that event, increments `ingress_rejected_total{reason="queue_full"}`, marks the adapter degraded, and does not block the FeedSat receive loop beyond the immediate nonblocking attempt.
- Connection or all-required-stream failure transitions the participant to `disconnected`; one optional-stream failure may transition it to `degraded`.
- Reconnect uses exponential backoff with jitter, configured initial and maximum durations, and resets after a stable connection interval.
- Cancellation interrupts dial, backoff, receive, and health polling.

### 6.3 Boundary validation and preservation

For each event the adapter validates:

1. configured source satellite exists and is a Producer-capable `SubscriberType`;
2. source Domain Plane and edge contract match H-02/H-03 configuration;
3. source entity identifier is nonempty;
4. source service and message identities are registered with the exact satellite-owned contract/version in H-02/H-03;
5. the declared satellite-owned contract and version are approved for that Producer;
6. routing-relevant identifiers required by the registered adapter rule are present;
7. source provenance is preserved without assigning stronger semantics or normalizing source time;
8. the unchanged satellite-owned message can be serialized for governed transport.

H-01 does not validate scientific enum choices, outcome coherence, mathematics, or scientific correctness. A boundary-invalid event is isolated individually; it does not terminate another stream. Repeated boundary failures may mark that stream degraded, but they never modify or reinterpret FeedSat science.

### 6.4 Sequence and duplicate handling

- `accepted_sequence` is retained as source provenance and compared only within `(satellite, source service, contract/version, message identity, entity, adapter connection generation)`.
- It is not treated as a global HACCAM sequence.
- Exact repeated nonempty `event_id` values within a bounded in-memory recent-ID set for the same mirror key are rejected as duplicates.
- A lower/equal sequence with a different event ID is diagnosed as regression/ambiguity and rejected unless the H-03 adapter rule explicitly permits it.
- Reconnect begins a new connection generation. The FeedSat's latest-event catch-up may repeat the current event; event ID suppression avoids a duplicate replacement where possible.
- No continuity, replay, or exactly-once claim is made when events were dropped or HACCAM was absent.

### 6.5 Producer failure and recovery

On Producer disconnect, existing mirrors remain as last received values but are marked unavailable-by-disconnect in HACCAM metadata; HACCAM does not rewrite the preserved satellite-owned message. Corresponding Routing Table rows may remain represented for diagnostics but are not active for publication while source availability is invalid. On recovery, H-05_def/gov can reactivate valid rows and catch-up/live events update mirrors normally.

## 7. H-02 Registry Knowledgebase

### 7.1 V0.1 model

H-02 is the in-memory HACCAM Registry Knowledgebase (Registry KB) of known participants, services, endpoints, capabilities, and runtime relationship evidence needed by Core. It represents both external HACCAM participants and internal HACCAM-owned service providers. V0.1 knowledge is loaded from validated HACCAM configuration and augmented with H-07 availability/lifecycle evidence; establishing a Consumer stream activates known capability but does not invent an unconfigured participant or service.

The internal model distinguishes these concepts without requiring public protobuf messages:

| Concept | HACCAM knowledge |
| :--- | :--- |
| Participant | stable identity and `internal HACCAM service` or `external satellite` classification; `satellite_id`, `SatelliteRole`, and `SubscriberType` exist only for satellites |
| ServiceProvider | participant-to-service relationship and provider capability |
| ServiceConsumer | participant-to-service/message relationship and consumer capability |
| ServiceEndpoint | service address/transport where a network endpoint applies; an in-process capability need not have one |
| ContractCapability | published or accepted satellite-owned contract identity/version plus service/message identities |
| Runtime evidence | configured, available, connected, degraded, or unavailable evidence supplied through H-07 |

One participant may expose multiple services; one service may publish multiple message identities; one satellite may be both provider and consumer; and an internal HACCAM component may provide a service without becoming a satellite. Participant identity, service identity, endpoint, contract capability, and provider/consumer relationship are separate facts. Physical host/process/device identity is not substituted for any of them.

The Registry KB knows **who exists, what service they provide or consume, where a network service can be reached, and which contracts/messages they publish or accept**. It does not contain scientific schemas, calculations, value-correctness rules, or algorithm meaning. It is a knowledge source for routing governance, not the Routing Table and not a delivery engine.

### 7.2 First configured knowledge example

| Provider / Consumer | Classification | Role / SubscriberType | Service and direction | Contract/message capability | Availability |
| :--- | :--- | :--- | :--- | :--- | :--- |
| `Fin_FeedSat_1` | external satellite | FeedSat / `PRODUCER` | existing `ModelService` gRPC publication; inbound to HACCAM through H-01 | existing `Fin_FeedSat_1`-owned contract; publishes `finfeedsat.v1.DecisionEvent`, `finfeedsat.v1.PriceEvent`, and `finfeedsat.v1.VolumeEvent` | derived from H-07 runtime health/lifecycle evidence |
| `RouterService` | internal HACCAM service | none; not a satellite | HACCAM-owned gRPC publishing service; outbound from HACCAM | publishes permitted HACCAM delivery representations while preserving source contract/message identity | derived from Core/H-07 service evidence |
| Decision Strategy Engine (DSE) | external satellite | TransSat / `PRODUCER_CONSUMER` | Consumer stream endpoint; receives from `RouterService` and may later publish authoritative DSE output through H-01 | consumes selected `Fin_FeedSat_1` service/message identities; later output uses a DSE-owned contract | derived from H-07 runtime health/lifecycle evidence |

These are first configuration examples only. The Registry KB supports `0..N` FeedSats, TransSats, ObsSats, internal HACCAM services, services per participant, and routes without Core scientific code changes.

### 7.3 Rules

- Every participant and service identity is nonempty and unique in its configured scope. A satellite also has a unique nonempty `satellite_id`.
- External satellites require explicit non-`UNSPECIFIED` role and subscriber type. Internal HACCAM services have neither and must not be forced into satellite enums.
- `PRODUCER` and `PRODUCER_CONSUMER` may advertise information.
- `CONSUMER` and `PRODUCER_CONSUMER` may establish subscriptions.
- An endpoint is required only where the configured transport crosses an applicable process/device/network boundary.
- Replacement means lifecycle reconnection of the same configured participant/service identity, not silent registration of a new participant.
- A Domain Plane mismatch prevents route creation; it does not rewrite the participant.
- Removing a configured participant is a configuration restart operation in V0.1.
- H-02 supplies knowledge to H-05_def/gov but never creates, owns, or executes Routing Table rows.

The H-02 Registry KB is an internal runtime capability, not a V0.1 network service. No `RegistryService`, registration RPC, or service-discovery RPC is proposed. The network-capable HACCAM-owned outbound publisher is `RouterService`, described under H-05_deliver.

## 8. H-03 Contract / Dictionary

H-03 is not a proto file. It governs contract identity, compatibility, participant relationships, and the rules under which a satellite-owned message may become HACCAM-visible and routable. The producing satellite owns the scientific contract and the meaning and mathematics of its payload. Registering that contract in H-03 does not transfer ownership.

### 8.1 H-03 catalog content

For each routable satellite publication, the in-memory catalog records:

- Domain Plane;
- permitted publisher identity, `SatelliteRole`, and `SubscriberType`;
- satellite-owned contract identity/version;
- exact satellite-owned service/message identity;
- routing-relevant identifiers and provenance that HACCAM may extract;
- permitted Consumer contracts and subscription relationships;
- compatibility rule: exact in V0.1 unless explicitly approved otherwise.

V0.1 has no separate route or routing identity namespace: exact participant, service, contract/version, message identity, Domain Plane, destination/subscription, and optional entity-filter facts provide the routing dimensions. H-03 decides service/contract/message compatibility; it does not store resolved routes or inspect scientific meaning. H-03 does not define what `V_N`, `Q_G`, strength, Price prediction, Volume phase, or any other scientific value means; it does not calculate or judge those values.

### 8.2 Finance evidence classification

| Current FeedSat concept | Classification | HACCAM use |
| :--- | :--- | :--- |
| `DecisionEvent` and nested Adaptive terms | Finance Domain Plane, FeedSat-owned scientific contract | preserved encoded message identified as `finfeedsat.v1.DecisionEvent` |
| `PriceEvent`, emission, cockpit, skip | Finance Domain Plane, FeedSat-owned scientific contract | preserved encoded message identified as `finfeedsat.v1.PriceEvent` |
| `VolumeEvent`, quantities, indicator, phase | Finance Domain Plane, FeedSat-owned scientific contract | preserved encoded message identified as `finfeedsat.v1.VolumeEvent` |
| `symbol`, `event_id`, `market_snapshot_id`, timestamps | shared-canonical candidates/evidence | identity and provenance where present |
| `accepted_sequence` | FeedSat-local publication cursor | scoped provenance, not global order |
| Feed/Component health enums | edge health vocabulary | adapted into H-07 evidence, not science |
| SemanticService contract `1.0` | FeedSat semantic evidence | startup compatibility input |
| generated `finfeedsat.v1` types | edge contract | never internal HACCAM model |

Finance names remain in the satellite-owned Finance contract. HACCAM records exact contract/message identities without defining a parallel Finance taxonomy. A future Domain Plane registers its satellite-owned contract/message identities and an adapter; it does not add scientific vocabulary to HACCAM Core or redesign H-01 through H-09.

### 8.3 Versioning and enum governance

V0.1 uses explicit satellite-owned contract identity, version, and message identity with exact compatibility. Unknown contracts, versions, or message identities fail closed before HACCAM accepts the encoded message for mirroring/routing and emit H-09 diagnostics. HACCAM does not inspect scientific enum values to decide compatibility. H-03 governance changes require review because they alter permitted relationships, even when protobuf wire compatibility would technically remain.

## 9. H-04 Subscription Management

H-04 answers **WHO wants WHAT**. It never determines what a Producer creates.

A V0.1 subscription contains:

- subscription identity assigned by HACCAM;
- subscriber `satellite_id`;
- Domain Plane;
- zero or more source `satellite_id` filters (empty means any compatible source in that Domain Plane);
- zero or more source service identity filters (empty means any compatible service from matched sources);
- zero or more exact entity filters (empty means any entity from matched sources);
- one or more exact satellite-owned message identities and accepted contract identities/versions;
- lifecycle: active, cancelling, removed;
- creation and removal timestamps.

The filter language is exact-match only. No wildcard expressions, predicates over encoded scientific messages, query language, or scientific filter is designed in V0.1.

For the first server-streaming boundary, the subscription lifetime equals the RPC stream lifetime. Stream cancellation changes the subscription to removed and causes H-05 to remove associated routes. Multiple streams from the same configured Consumer are separate subscriptions.

## 10. H-05 Routing & Distribution

H-05 contains two distinct subordinate responsibilities and one derived operational state. H-05_def/gov constructs and maintains the Routing Table. H-05_deliver executes outbound publication through the HACCAM-owned `RouterService`. `RouterService` is logically internal to HACCAM ownership but may cross process, device, or network boundaries through gRPC; it is not a Satellite or Domain Plane scientific processor.

### 10.1 H-05_def/gov

H-05_def/gov creates a permitted route only when all are true:

1. H-02 source exists and may produce;
2. H-02 receiver exists and may consume;
3. source and receiver Domain Plane association is compatible;
4. H-03 recognizes the satellite-owned contract/version/message identity and confirms Consumer compatibility;
5. H-04 records receiver interest matching source participant/service, message identity, contract, and any entity filter;
6. the route does not point a satellite directly to a peer publication boundary.

H-05_def/gov produces and maintains the Routing Table from H-02 Registry KB facts, H-03 compatibility decisions, and H-04 subscription intent. The subscription is a Consumer's **WHO wants WHAT** intent; a Routing Table row is the resolved, permitted source-to-destination relationship. Subscription creation alone never authorizes delivery.

A minimal conceptual Routing Table row is:

| Field | Ownership / purpose |
| :--- | :--- |
| source provider identity | Registry KB participant/provider reference |
| source `satellite_id` | present only when the source is a satellite |
| source service identity | exact publishing service |
| source contract identity/version | exact satellite-owned contract governed by H-03 |
| message identity | exact satellite-owned message identity |
| destination provider/consumer identity | Registry KB destination participant reference |
| destination `satellite_id` | present only when the destination is a satellite |
| destination service identity | exact `RouterService`-facing Consumer service/stream relationship |
| Domain Plane | compatibility boundary copied from resolved facts |
| subscription identity | H-04 intent from which the row was resolved |
| optional entity filter | exact filter copied from the subscription |
| route state and availability relationship | active, disabled, or removing state resolved from relevant participant/service availability evidence |
| `created_at_unix_ms`, `updated_at_unix_ms` | HACCAM-owned row lifecycle times |

The table is dynamic and cardinality-independent. Each row is identified by its concrete governed source, destination, contract/message, Domain Plane, subscription, entity-filter, and state facts; V0.1 adds no synthetic route identifier. It is resolved delivery topology, not participant knowledge storage, service discovery, contract interpretation, or scientific meaning. Rows are created, updated, disabled, or removed when subscriptions, Registry KB availability/capabilities, or H-03 decisions change. Governance is not a per-event synchronous pipeline.

### 10.2 H-05_deliver and `RouterService`

`RouterService` is the higher-level HACCAM-owned gRPC outbound publishing service and the external/network expression of H-05_deliver; it is not the entirety of H-05. It obtains active Routing Table rows and publishes H-06 current/live Matrix-visible information to authorized destinations. It does not own Registry KB facts, decide contract compatibility, create subscription intent, alter satellite science, or create peer shortcuts. It is not the Registry KB, the Routing Table, a mere queue, a Satellite, or a scientific processor.

For every active external subscription stream:

- one bounded queue is owned by H-05_deliver;
- one sender goroutine is the sole receiver from and closer-user of that queue;
- H-05_deliver obtains matching active Routing Table rows and performs a nonblocking enqueue per destination;
- the immutable runtime record can be referenced by multiple queues;
- sender serialization copies HACCAM metadata and the unchanged encoded satellite-owned message into the outbound delivery message, then calls `Send`;
- stream context cancellation or send failure removes the subscription/routes and terminates the sender;
- queue full drops the delivery for only that subscription, increments diagnostics, and marks the Consumer degraded/slow;
- successful later sends may return the route to connected after a configured observation threshold, without claiming delivery of dropped values.

Default recommendation: queue capacity 64 per active delivery stream, configurable with a positive upper bound. Sixty-four is large enough to absorb short scheduler/network pauses and small enough to make slow Consumers visible. It is not an architectural constant.

No retry is performed for an individual dropped realtime value. Retrying into the same full queue would amplify pressure and imply durability that V0.1 does not provide. The Consumer can reconnect and receive current latest mirror values as initial catch-up, followed by live updates.

H-05 never decodes, changes, or assigns scientific meaning to the encoded message. The Consumer uses the delivered satellite-owned contract version and message identity to decode the exact scientific contract it subscribed to.

For each accepted HACCAM-visible event: H-06 first contains the current mirror; H-05_def/gov has already resolved permitted routes; `RouterService`/H-05_deliver obtains matching active rows; `RouterService` publishes to each authorized destination through isolated bounded delivery state; and the unchanged satellite-owned encoded message is delivered with only HACCAM-owned delivery/provenance metadata. Failure of one destination does not affect another.

### 10.3 Fan-out and route removal

Fan-out iterates a snapshot of matching active Routing Table rows under no route lock while enqueueing. Routing Table mutation uses copy-on-write or a short `RWMutex`; no network send occurs while holding it. Removal marks a row removing, detaches it from future snapshots, then signals its sender context. Only the delivery owner closes its queue, preventing send-on-closed-channel races.

## 11. H-06 Intelligence Mirror

### 11.1 Mirror grain and key

The key is exactly:

```text
(source satellite_id, source service identity, source contract identity/version,
 message identity, source entity identifier)
```

No fixed satellite, service, contract, message, or entity cardinality exists. The mirror key is derived from source identity and contract facts; it is not a Routing Table row or a separate route/routing identity namespace.

### 11.2 Mirror value

Each latest-state mirror contains:

- key;
- immutable serialized bytes of the unchanged satellite-owned protobuf message;
- source event identity, when supplied;
- source sequence and presence, when supplied;
- source provenance in the source contract's representation, when registered as routing/traceability metadata;
- source `market_snapshot_id` or equivalent lineage, when supplied;
- H-03 contract identity/version;
- HACCAM receipt and mirror-update Unix-millisecond timestamps;
- adapter connection generation;
- availability reason: current, source disconnected, incompatible, or rejected-latest-attempt;
- last rejection diagnostic reference without replacing the last valid encoded message.

### 11.3 Update and concurrency semantics

- One mirror store owns a map guarded by an `RWMutex`.
- Boundary validation, metadata extraction, and serialization of the unchanged satellite message complete before the lock is acquired.
- Replacement and retrieval copy immutable value references under a short lock.
- A valid newer event atomically replaces the prior value for its key.
- Duplicate/regressive/invalid events do not replace the last valid value.
- H-05 receives the accepted replacement after the mirror lock is released.
- Read snapshots never expose mutable map or encoded-message ownership.

V0.1 stores no history. Restart loses all mirrors. No stale time threshold is invented; disconnect/unavailable evidence is explicit, and receipt/update timestamps permit later policy.

H-06 owns the mirror record and its HACCAM metadata. The Producer continues to own the encoded message's scientific meaning and contract. H-06 performs no scientific transformation.

## 12. H-07 Health & Lifecycle

Lifecycle states follow the Process Model:

| State | Entered by | Evidence |
| :--- | :--- | :--- |
| configured | startup registry load | valid H-02 config, no connection yet |
| connecting | adapter/consumer session supervisor | dial or stream establishment in progress |
| connected | adapter/consumer session supervisor | required connection/streams established |
| degraded | H-01/H-05/H-07 | optional stream loss, malformed events, queue drops, or unhealthy source evidence while relationship remains usable |
| disconnected | adapter/consumer session supervisor | connection/stream absent after previously established or cancelled |
| failed | supervisor/runtime owner | nonrecoverable config/contract condition or reconnect budget explicitly exhausted |

V0.1 recommends unlimited reconnect attempts bounded in frequency; therefore ordinary network failure remains disconnected, not failed. Invalid static identity/contract fails startup rather than entering a misleading runtime loop.

H-07 is the sole writer of lifecycle state. H-01, H-05, and health pollers submit evidence. Scientific statuses such as Adaptive initializing, Price warm-up, or Volume maturing never become satellite lifecycle states.

## 13. H-08 Failure Isolation

| Failure | Isolation boundary | Must continue |
| :--- | :--- | :--- |
| one Producer disconnects | its adapter supervisor and streams | other adapters, mirrors, routes, Consumers, all satellite science |
| one boundary-invalid Producer event | acceptance of that event | same stream, other events, H-06 last valid value, unrelated work |
| one adapter fails/panics | adapter supervisor recovery boundary | core and other adapters; affected Producer science remains independent |
| one Consumer disconnects | its RPC context, queue, sender | H-01, H-06, other routes/Consumers, Producer science |
| one Consumer is slow | its bounded queue | all upstream and unrelated delivery |
| one route fails | route owner | other routes, including same source to other Consumers |
| H-09 sink fails | diagnostics sink | runtime data/control paths; fallback counter/log where possible |
| HACCAM shuts down/fails | HACCAM process boundary | every satellite's independent science |

Adapter and route goroutine entry points use a narrow panic recovery wrapper that records stack/context, marks only that boundary failed/degraded, and returns control to its supervisor. Core invariant/store corruption is not masked; the process exits rather than continuing with unknown ownership state.

## 14. H-09 Diagnostics

V0.1 uses structured logs and in-process counters/gauges exposed through a conventional metrics endpoint if selected during implementation. Diagnostic emission must be nonblocking with respect to ingress and delivery.

Required events/counters:

- HACCAM startup, readiness, shutdown, and forced shutdown;
- satellite configured, connection attempt, connected, disconnected, degraded, reconnect scheduled;
- ingress received, accepted, duplicate, regressive, malformed, incompatible, queue-full rejected;
- mirror created/replaced and last update timestamp;
- subscription created/removed;
- route established/removed;
- delivery enqueued, succeeded, dropped, send failed;
- Consumer slow/unavailable and recovery observation;
- adapter failure/panic;
- current counts for satellites by lifecycle, mirrors, subscriptions, routes, and delivery queue occupancy.

Logs include participant/service identity, `satellite_id` where applicable, Domain Plane, contract/message identity, entity where safe, subscription identity, route state, and source event identity where supplied. These concrete dimensions identify the affected relationship without another identity namespace. Logs do not include encoded scientific messages by default. Diagnostics are realtime and are lost on restart.

## 15. External Satellite Relationships

### 15.1 Producer path

```text
Fin_FeedSat_1 science
        |
        +--> FeedSat-owned publication (continues without HACCAM)
                    |
                    v
            H-01 outbound adapter
                    |
                    v
             H-06 latest mirror
```

### 15.2 Consumer path

```text
External Producer: Fin_FeedSat_1
                     |
                     v
                    H-01
                     |
                     v
                    H-06 -----------------------------+
                                                                   |
H-02 Registry KB --------+                         |
H-03 Contract Governance +--> H-05_def/gov         |
H-04 Subscription Intent +         |               |
                                             v               |
                                     ROUTING TABLE <--------+
                                             |
                                             v
                                  RouterService / H-05_deliver
                                             |
                                             v
                                 External Consumer: DSE
```

H-07 lifecycle evidence, H-08 isolation, and H-09 diagnostics cross-cut this flow. H-02, H-03, and H-04 inform route governance and are not per-event data stages.

### 15.3 Producer+Consumer path

```text
RouterService / H-05_deliver --> TransSat consumes Matrix-visible information
                         |
                         v
                 TransSat-owned science
                         |
                         v
              new authoritative information
                         |
                         v
                 canonical H-01 ingress
                         |
                         v
               H-06 mirror / H-05 routes
```

There is no peer-satellite shortcut and no feedback into FeedSat scientific state.

### 15.4 Internal HACCAM use

H-06 and `RouterService`/H-05_deliver observe the same accepted immutable runtime record produced by H-01: HACCAM-owned metadata plus the unchanged encoded satellite-owned message. H-06 owns latest-state replacement; H-05_deliver receives notification after replacement and consults the Routing Table. H-05_def/gov consumes H-02/H-03/H-04 facts, not scientific event content. H-07 and H-09 receive evidence through nonblocking internal calls/counters. These responsibilities are not Satellites, do not subscribe through an external satellite RPC, and do not form a second routing architecture or scientific model.

### 15.5 Co-located satellites

Logical paths are unchanged by packaging:

```text
one Android application or host
        +-- Fin_FeedSat_1 (FeedSat / PRODUCER)
        +-- Decision Strategy Engine (TransSat / PRODUCER_CONSUMER)

required logical path:
Fin_FeedSat_1 -> H-01 -> HCM -> Routing Table -> RouterService/H-05_deliver -> Decision Strategy Engine
```

An in-process transport optimization may be considered only if it implements the same H-01/H-03/H-04/H-05/H-06 governance and failure boundaries. It may not become a direct scientific shortcut.

## 16. Proposed First HACCAM Proto Design

No `.proto` file is created by this design. Proposed package: `haccam.v1`.

Every proposed element below passes the ownership test: HACCAM owns the service, subscription/routing metadata, mirror/receipt metadata, and governed delivery representation. HACCAM does not own or redefine the encoded satellite message. No Finance scientific enum, message, status, calculation, or HACCAM Finance scientific contract belongs in this proto.

### 16.1 Service inventory and rationale

#### `RouterService`

The approved V0.1 design-only service boundary is:

```proto
service RouterService {
        rpc StreamMatrixVisibleInformation(SubscriptionRequest)
                        returns (stream MatrixVisibleInformation);
}
```

No protobuf file or generated code is authorized by this declaration.

| Item | Design |
| :--- | :--- |
| H-process | H-04 request boundary; H-05_def/gov validation; H-05_deliver execution |
| Architectural service | HACCAM-owned `RouterService`: higher-level outbound publisher using active Routing Table state |
| Purpose | publish governed realtime Matrix-visible information to authorized external Consumers |
| Caller | satellite with `SubscriberType = CONSUMER` or `PRODUCER_CONSUMER` |
| Server | HACCAM Core |
| RPC | `StreamMatrixVisibleInformation(SubscriptionRequest) returns (stream MatrixVisibleInformation)` |
| Direction | server streaming |
| Failure | invalid identity/filter: `INVALID_ARGUMENT`; unregistered or wrong subscriber type: `FAILED_PRECONDITION`; incompatible contract: `FAILED_PRECONDITION`; shutdown: `UNAVAILABLE`; cancellation ends subscription |
| Why network boundary | the Consumer is an external satellite process; H-05 delivery must cross that boundary |

`RouterService` names the higher-level HACCAM provider responsibility and remains distinct from the Routing Table and broader H-05 route governance. `StreamMatrixVisibleInformation` expresses H-04 subscription intent at stream establishment, H-05_def/gov route validation/resolution, and H-05_deliver outbound realtime publication. No additional RPC is proposed for V0.1.

No H-02 registration RPC is required in V0.1 because the Registry KB is configured and internal. No separate create/delete subscription RPC is required because the stream request and context define subscription lifecycle. No diagnostics RPC is required because H-09 uses logs/metrics. Core health should use standard `grpc.health.v1.Health`, not a new HACCAM message.

A future satellite publication RPC may be added as another H-01 adapter when a Producer lacks an existing publication boundary. It is not required for the first runtime and is not specified speculatively. This does not change H-01 architecture or internal ingress handling.

### 16.2 Messages

#### `SubscriptionRequest`

Owner: H-04. Producer: external Consumer. Consumers: H-04 and H-05_def/gov.

| Field | Type | Semantics |
| :--- | :--- | :--- |
| `subscriber_satellite_id` | `string` | required; must match a configured H-02 Consumer-capable identity |
| `domain_plane` | `string` | required; exact configured association in V0.1 |
| `source_satellite_ids` | `repeated string` | optional filter; empty means any compatible source in the Domain Plane |
| `source_service_ids` | `repeated string` | optional filter; empty means any compatible registered source service |
| `entity_ids` | `repeated string` | optional exact-match filter; empty means any entity |
| `message_identities` | `repeated string` | required, nonempty; exact satellite-owned protobuf message identities requested by the Consumer |
| `accepted_contracts` | `repeated ContractIdentity` | required, nonempty; contracts the Consumer can decode |

Required/optional semantics are validated by the server; proto3 field presence alone is not treated as semantic validity. Repeated filters are deduplicated. The request is immutable for a stream; changing interest requires reconnecting with a new request.

#### `MatrixVisibleInformation`

This name directly uses the Process Model term “Matrix-visible information”; it is not a new architectural envelope concept. Owner: H-05 delivery representation. Producer: HACCAM Core. Consumer: subscribed satellite.

| Field | Type | Semantics |
| :--- | :--- | :--- |
| `source_satellite_id` | `string` | required authoritative Producer identity |
| `source_service_id` | `string` | required exact publishing service identity |
| `domain_plane` | `string` | required semantic context |
| `entity_id` | `string` | required opaque source-owned entity identity |
| `contract` | `ContractIdentity` | required satellite-owned contract identity/version |
| `message_identity` | `string` | required fully qualified satellite-owned protobuf message identity, for example `finfeedsat.v1.DecisionEvent` |
| `source_event_id` | `string` | optional source identity; empty only when source lacks one |
| `source_sequence` | `optional uint64` | optional source-scoped cursor; never global order |
| `source_timestamp_text` | `string` | optional source timestamp preserved as supplied; HACCAM assigns no stronger semantics |
| `interval_start_unix_ms` | `optional int64` | optional source interval start copied when supplied by the registered contract |
| `interval_end_unix_ms` | `optional int64` | optional source interval end copied when supplied by the registered contract |
| `market_snapshot_id` | `string` | optional source-owned lineage copied when supplied; HACCAM does not define its scientific meaning |
| `haccam_received_at_unix_ms` | `int64` | required HACCAM-owned receipt time using project convention |
| `haccam_mirror_updated_at_unix_ms` | `int64` | required HACCAM-owned H-06 update time using project convention |
| `encoded_message` | `bytes` | required serialized unchanged satellite-owned protobuf message; governed by `contract` and `message_identity` |

The message is a HACCAM delivery representation, not a new semantic information object. It combines HACCAM-owned routing, mirror, and receipt metadata with an unchanged satellite-owned encoded message. Semantic and contract ownership remain with `source_satellite_id`. The bytes cannot be accepted or routed without an exact registered contract/version/message identity and permitted relationship. The Consumer uses that identity and its satellite/Domain Plane contract implementation to decode the message. HACCAM does not decode it to interpret or validate science.

#### `ContractIdentity`

HACCAM owns only this registration/identification metadata as used by H-03. The producing satellite owns the scientific contract identified by the metadata. Metadata is supplied through HACCAM configuration or a Consumer request and used by H-03/H-05; registration never transfers scientific-contract ownership to HACCAM.

| Field | Type | Semantics |
| :--- | :--- | :--- |
| `name` | `string` | required registered identifier of the Producer-owned contract |
| `version` | `string` | required exact Producer-owned contract version in V0.1 |
| `proto_package` | `string` | required satellite-owned protobuf package identity |

No generic compatibility range is introduced. Any later range/negotiation requires H-03 design approval.

### 16.3 Enums

#### `SatelliteRole`

Authorized by the Process Model satellite ontology.

```text
SATELLITE_ROLE_UNSPECIFIED = 0   // wire-safe, always invalid for registration
SATELLITE_ROLE_FEED_SAT = 1
SATELLITE_ROLE_TRANS_SAT = 2
SATELLITE_ROLE_OBS_SAT = 3
```

#### `SubscriberType`

Authorized by the Process Model subscriber classification.

```text
SUBSCRIBER_TYPE_UNSPECIFIED = 0          // wire-safe, never architecturally valid
SUBSCRIBER_TYPE_PRODUCER = 1
SUBSCRIBER_TYPE_CONSUMER = 2
SUBSCRIBER_TYPE_PRODUCER_CONSUMER = 3
```

These enums belong in `haccam.v1` even though the first delivery RPC does not carry them: generated/config-loading boundary types for H-02 must use the approved values consistently. Internal Go enums remain distinct validated types. No entity or Finance symbol enum is permitted.

### 16.4 Proto inventory

| Proto Element | Type | Purpose | H-Process Authority | Required for V0.1? | New Term? |
| :--- | :--- | :--- | :--- | :--- | :--- |
| `haccam.v1` | package | versioned HACCAM edge namespace | H-03 | yes | retained proto term |
| `RouterService` | service | HACCAM-owned outbound publication using active Routing Table state | H-05_deliver | yes | approved V0.1 proto term |
| `StreamMatrixVisibleInformation` | RPC | subscription establishment, route resolution, and realtime publication | H-04/H-05_def/gov/H-05_deliver | yes | approved V0.1 proto term |
| `SubscriptionRequest` | message | WHO wants WHAT | H-04 | yes | retained proto term |
| `MatrixVisibleInformation` | message | HACCAM metadata plus unchanged encoded satellite-owned message; not a scientific schema | H-03/H-05/H-06 | yes | retained proto term |
| `ContractIdentity` | message | exact H-03 name/version | H-03 | yes | retained proto term |
| `SatelliteRole` | enum | role ontology | H-02/Process Model | yes | retained Process Model vocabulary |
| `SubscriberType` | enum | relationship classification | H-02/Process Model | yes | retained Process Model vocabulary |
| `grpc.health.v1.Health` | standard service | Core liveness/readiness | H-07 | yes | no |
| future publication RPC | deferred service/RPC | later H-01 adapter for a Producer needing HACCAM-owned ingress edge | H-01 | no | not proposed now |

## 17. Go Runtime Component Inventory

Recommended module organization uses a small number of behavior-oriented packages; package names are implementation terminology, not architecture.

| Component/package | Purpose / H-process | Inputs / outputs | State and concurrency owned | Failure boundary | Explicit non-responsibilities |
| :--- | :--- | :--- | :--- | :--- | :--- |
| `cmd/haccam-core` | composition root | config/signals; starts runtime | root context, startup/shutdown ordering | process | no H-process logic |
| `internal/core` | accepted-information coordinator | governed runtime record; mirror update then route notification | no long-lived queue beyond configured ingress dispatcher | core invariant boundary | no scientific protobuf definitions, decoding, or science |
| `internal/adapter/finfeedsat` | first H-01 adapter | `finfeedsat.v1` streams to HACCAM metadata plus unchanged encoded messages | connection supervisor, stream goroutines, bounded ingress queue | per configured Producer | no FeedSat modification, scientific transformation/validation, or routing |
| `internal/registry` | H-02 | Registry KB of internal/external participants, services, endpoints, provider/consumer relationships, and contract capabilities | knowledge indexes and lock; H-07 evidence view | startup validation / corrupted state isolation | no Routing Table ownership, delivery, or scientific schema |
| `internal/contract` | H-03 | registered satellite contract/message identities and relationship compatibility | immutable catalog after startup | incompatible contract/version/message | no scientific meaning, mathematics, or payload correctness validation |
| `internal/subscription` | H-04 | stream requests; subscription snapshots | subscription map and lock | per subscription | no route permission or delivery |
| `internal/routing` | H-05_def/gov | Registry KB snapshots, H-03 decisions, H-04 intent; active route snapshots | Routing Table and lock/copy-on-write state | per route-definition operation | no Registry KB ownership, network publication, or science |
| `internal/router` | H-05_deliver / `RouterService` | active Routing Table rows and H-06 current/live records; outbound gRPC publications | bounded per-destination queues and sender goroutines | per destination/stream | no route permission, Registry KB ownership, contract decision, subscription ownership, or scientific transformation |
| `internal/mirror` | H-06 | accepted runtime records; latest snapshots | keyed map and `RWMutex` | per update validation before lock | no history, delivery, or scientific transformation |
| `internal/lifecycle` | H-07 | evidence from adapters/routes/health | sole lifecycle writer; small event queue or synchronized calls | per participant | no scientific status interpretation |
| `internal/diagnostics` | H-09 and H-08 evidence | structured events/counters | bounded/nonblocking diagnostics | sink | no control over satellite science |
| `internal/server` | `RouterService` proto boundary | gRPC requests/responses | RPC contexts; delegates publication state to `internal/router` | per RPC | no Registry, Routing Table, contract-governance, or internal protobuf model ownership |

Separate `internal/routing` and `internal/router` packages are recommended because route governance and network publication have different state and failure boundaries. One package with rigorously separate components could implement the same architecture; package names do not redefine H-05. H-08 is implemented through boundaries in adapter, core, routing, router, and diagnostics rather than a package that catches every error.

## 18. Concurrency, Locks, Queues, and Cancellation

### 18.1 Goroutine ownership

The composition root owns an `errgroup` under one root context. Each adapter supervisor owns its connection and stream worker goroutines. `RouterService`/H-05_deliver owns one sender goroutine per active external stream. The gRPC server owns request handler goroutines. No goroutine is started without an owner, cancellation path, and join path.

### 18.2 Channels

| Channel | Owner/closer | Bound | Full behavior |
| :--- | :--- | :--- | :--- |
| adapter-to-core ingress | adapter supervisor | configurable, recommended 256 per Producer | reject event, diagnose, mark degraded; never block FeedSat receive indefinitely |
| delivery queue | route/stream owner | configurable, recommended 64 | drop for that Consumer only |
| diagnostics, if asynchronous | diagnostics owner | configured small bound | drop diagnostic and increment self-drop counter |

No unbounded channel, slice-as-queue, or retry accumulation is allowed.

Queues carry immutable runtime records containing HACCAM metadata and preserved encoded messages. Workers do not decode scientific content. Fan-out shares immutable encoded bytes where safe and never mutates them for an individual Consumer.

### 18.3 Locks

Registry, subscription, route, lifecycle, and mirror state have separate ownership/locks. Lock order is avoided by taking snapshots from one component before calling another. No protobuf send, network dial, logging sink, or channel wait occurs while a state lock is held.

### 18.4 Context propagation

Root cancellation flows to listeners, RPCs, adapters, health polling, reconnect waits, and route senders. Per-adapter and per-RPC child contexts permit isolated cancellation. Values are not used to hide required dependencies; only request-scoped diagnostic identity belongs in context.

## 19. Configuration

HACCAM configuration is a small validated file plus environment overrides for deployment-local addresses/log level.

| Category | V0.1 content |
| :--- | :--- |
| Core server | listen address; health/metrics address if separate |
| Registry KB bootstrap | N internal/external participants, satellite role/type where applicable, services, Domain Plane associations, endpoints, and provider/consumer capabilities |
| Adapter | adapter kind, satellite-owned contract/version/service/message identities, optional entity filters |
| Contract | H-03 registrations for exact satellite-owned contract/version/message identities and permitted relationships |
| Queue bounds | ingress, delivery, diagnostics capacities with positive maxima |
| Reconnect | dial timeout, initial/max backoff, jitter, stable-reset interval |
| Health | poll interval and timeout |
| Diagnostics | structured log level and metrics enabled flag |
| Shutdown | bounded graceful drain timeout |

HACCAM configuration never defines a Finance scientific contract and never contains FeedSat scientific parameters, market credentials, symbols as architecture, model mode, pricing mode, transformation math, or satellite-owned entity limits. An adapter may optionally request an entity subset as HACCAM consumption configuration; it does not configure what the FeedSat owns or produces.

Invalid identity, duplicate satellite, invalid role/type, missing required endpoint, unknown contract, nonpositive queue bound, or inconsistent Producer/Consumer capability fails startup before listeners report ready.

## 20. Startup and Shutdown

### 20.1 Startup sequence

```text
load and validate HACCAM configuration
  -> construct immutable H-03 contract catalog
        -> load H-02 configured participant/service knowledge, including `RouterService`
        -> initialize empty H-06 mirror and H-04 subscriptions
        -> initialize empty H-05 Routing Table and `RouterService` delivery state
  -> initialize H-07 lifecycle and H-09 diagnostics
  -> bind gRPC/health listeners but report NOT_SERVING
        -> start `RouterService`/H-05_deliver and gRPC serving
  -> start H-01 adapter supervisors for configured Producers
  -> adapters transition configured -> connecting -> connected/degraded
  -> report HACCAM Core SERVING when internal components are operational
```

Readiness means the Core can accept a valid Consumer and manage adapters; it does not require every external satellite to be connected. External lifecycle is visible separately.

### 20.2 Shutdown sequence

```text
receive cancellation/signal
  -> report NOT_SERVING and reject new subscription streams
  -> cancel root context
  -> stop adapter reconnect and receive loops; cancel FeedSat streams
  -> stop accepting new mirror updates and route enqueues
  -> cancel route senders; optionally drain only until bounded timeout
  -> GracefulStop gRPC, then force Stop at timeout
  -> join all owned goroutines
  -> close listeners and diagnostic sinks
  -> exit without calling or changing any Producer science
```

Queue closure follows ownership rules. Shutdown may lose queued realtime delivery; this is explicit and consistent with in-memory V0.1.

## 21. Realtime State and Restart Loss

Lost on restart:

- every H-06 mirror and update/availability marker;
- active H-04 stream subscriptions;
- H-05 route and queued delivery state;
- runtime connection generations and lifecycle observations;
- recent duplicate-ID sets;
- counters, gauges, and log continuity;
- any runtime-only registration activation.

Reconstructed on startup:

- configured H-02 identities/endpoints;
- configured H-03 catalog;
- adapter definitions and queue/reconnect policy.

Relearned after connection:

- Producer health and stream state;
- latest information supplied by Producer catch-up and subsequent live publication;
- Consumer subscriptions when Consumers reconnect.

The design intentionally does not solve restoration, replay, checkpointing, or durable delivery.

## 22. DS_TransSat Compatibility

The Process Model is unambiguous: `DS_TransSat` has `SatelliteRole = TransSat` and `SubscriberType = PRODUCER_CONSUMER`. No runtime-design conflict exists.

This runtime can later support DS-01 by the same H-04/H-05 Consumer delivery boundary. The Decision Strategy Engine decodes `Fin_FeedSat_1` information using the FeedSat-owned contract it subscribed to, not a HACCAM scientific schema. DS-02, DS-03, and DS-04 remain wholly outside this design. When DS-04 is implemented, its authoritative information and satellite-owned contract must enter through an H-01 adapter and then use the same boundary validation, H-06, and H-05 path. No neural network, deterministic transform, collective matrix structure, or Decision Strategy schema is designed here.

## 23. Domain Neutrality

HACCAM Core understands participant and service identity, endpoint and capability facts, Domain Plane, satellite-owned contract identity/version and message identity, routing-relevant entity identity, provenance, mirror replacement, subscription intent, resolved routes, and delivery. It does not understand Adaptive, Price, Volume, symbol classification, scientific status, or Finance mathematics.

H-03 registers and governs compatibility around satellite-owned Domain Plane contracts; it does not own their vocabulary or science. The `Fin_FeedSat_1` adapter may reference Finance edge message identities and extract approved routing/provenance fields, but it does not translate their scientific content. Adding a future Domain Plane requires registrations for its satellite-owned contracts/messages and an adapter, not scientific vocabulary in Core or changes to registry, subscription, routing, mirror grain, lifecycle, isolation, or diagnostics.

## 24. HACCAM Core Runtime Invariant Register

| ID | Invariant | Enforced by |
| :--- | :--- | :--- |
| CORE-INV-01 | One owner per fact; crossing HACCAM never silently transfers semantic ownership. | H-03, H-06, boundary adaptation |
| CORE-INV-02 | Runtime supports N satellites and no hard-coded participant count. | config, H-02, H-06 keying |
| CORE-INV-03 | Every satellite has an explicit non-UNSPECIFIED `SubscriberType`. | config/proto validation, H-02 |
| CORE-INV-04 | `SatelliteRole` is independent of `SubscriberType`. | H-02 types/validation |
| CORE-INV-05 | A satellite owns its science and authoritative output decisions. | system boundary, adapters |
| CORE-INV-06 | HACCAM does not determine entity cardinality or identifiers. | H-01, H-04, H-06 dynamic keys |
| CORE-INV-07 | HACCAM does not determine Producer information count or scientific output. | H-03 registrations, H-01 listening only |
| CORE-INV-08 | FeedSats remain peer-independent; `F_i -/-> F_j`. | no peer APIs/routes |
| CORE-INV-09 | Satellite-produced Matrix-visible information enters canonically through H-01. | adapter interface, core coordinator |
| CORE-INV-10 | H-06 owns mirror objects but not mirrored semantic authority. | mirror model/provenance |
| CORE-INV-11 | H-04 records WHO wants WHAT and never commands production. | subscription model |
| CORE-INV-12 | H-05_def/gov alone determines permitted delivery. | routing governance |
| CORE-INV-13 | H-05_deliver alone executes bounded asynchronous delivery. | routing delivery queues |
| CORE-INV-14 | Consumer failure cannot block Producer science, H-01, H-06, or unrelated Consumers. | nonblocking enqueue, per-stream queue |
| CORE-INV-15 | HACCAM failure cannot stop FeedSat science. | outbound optional client relationship |
| CORE-INV-16 | Generated satellite protobuf types remain satellite-owned edge contracts, not the HACCAM internal model. | adapter/server boundary adaptation |
| CORE-INV-17 | HACCAM Core remains domain-neutral. | package boundaries, H-03/adapters |
| CORE-INV-18 | V0.1 is realtime and in-memory; no durability or exactly-once claim. | state stores, documentation, tests |
| CORE-INV-19 | No queue is unbounded. | config validation, channel construction |
| CORE-INV-20 | Source sequence is scoped provenance, not global HACCAM order. | H-01 duplicate/sequence logic |
| CORE-INV-21 | H-02/H-03/H-04 govern; they are not sequential data hops. | core flow and routing API |
| CORE-INV-22 | Scientific lifecycle/status is not satellite lifecycle. | H-07 evidence adaptation |
| CORE-INV-23 | Current four Finance symbols and one FeedSat are configuration facts only. | no symbols/counts in core/proto |
| CORE-INV-24 | HACCAM never defines, recalculates, normalizes, or assumes ownership of satellite scientific mathematics or facts. | H-01, H-03, package boundaries, tests |
| CORE-INV-25 | A satellite-produced information item retains its satellite-owned contract and semantic ownership while mirrored and routed. | contract/message identity, H-05, H-06 |
| CORE-INV-26 | H-03 governs contract compatibility and relationships; it does not validate scientific correctness of encoded messages. | H-03 catalog API and tests |
| CORE-INV-27 | H-01 validates the HACCAM boundary but does not translate satellite science into a HACCAM scientific schema. | adapter interface and tests |
| CORE-INV-28 | H-05 delivery preserves and does not alter satellite-owned scientific information. | immutable encoded message, delivery tests |
| CORE-INV-29 | HACCAM proto definitions are limited to HACCAM-owned responsibilities and never duplicate Domain Plane scientific schemas. | proto review and dependency tests |
| CORE-INV-30 | Physical deployment or packaging is not satellite identity. | H-02 identity model, deployment-neutral tests |
| CORE-INV-31 | Co-located satellites remain independent and use governed HACCAM paths where required. | topology, no shortcut API, integration tests |
| CORE-INV-32 | H-02 is authoritative HACCAM knowledge of known internal/external participants, services, endpoints, and capabilities; it is not the Routing Table. | Registry KB model and ownership tests |
| CORE-INV-33 | The Routing Table is HACCAM-owned resolved operational route state derived by H-05_def/gov from H-02, H-03, and H-04 facts. | routing API and route-construction tests |
| CORE-INV-34 | `RouterService` executes outbound publication only from active Routing Table rows; it does not invent scientific meaning or determine scientific ownership. | `RouterService` boundary and publication tests |
| CORE-INV-35 | H-02 does not absorb H-05 route ownership, and H-05 does not absorb H-02 knowledgebase ownership. | package/state ownership tests |
| CORE-INV-36 | H-04 subscription intent and H-05 Routing Table state are distinct objects with distinct lifecycles. | subscription/route tests |
| CORE-INV-37 | Internal HACCAM services are not Satellites and have no `SatelliteRole` or `SubscriberType`. | Registry KB validation |
| CORE-INV-38 | V0.1 routing is represented only by concrete Registry KB, contract/version, message, subscription, source/destination, Domain Plane, optional entity-filter, and route-state dimensions. No separate route or routing identity namespace is used. | model/proto review and route-resolution tests |

## 25. First Implementation Validation Plan

### 25.1 Unit and contract tests

1. Validate all role/subscriber combinations and reject `UNSPECIFIED`.
2. Load zero, one, and multiple configured satellites without core code changes.
3. Preserve representative serialized Decision, Price, Volume, and typed skip messages without HACCAM scientific field definitions or mutation; reject only boundary-invalid contract/message metadata.
4. Verify mirror keys distinguish source satellite, source service, contract/version, message identity, and entity without a separate routing identity.
5. Verify duplicate event ID and scoped sequence regression do not replace last valid mirror.
6. Verify subscription exact matching for any source participant/service, message identity, contract/version, and entity-filter combinations.
7. Verify H-05_def/gov rejects wrong Domain Plane, subscriber capability, and contract version.
8. Fill one delivery queue and prove mirror updates and another Consumer continue.
9. Cancel every context path and use goroutine-leak/race tests.
10. Run `go test -race ./...` and bounded-queue stress tests.
11. Load `Fin_FeedSat_1`, DSE, and the internal HACCAM `RouterService` into the Registry KB; verify only the first two are Satellites and `RouterService` has no satellite role/type.
12. Register multiple services for one participant and verify participant, service, endpoint, contract capability, and provider/consumer relationships remain distinct.
13. Create H-04 intent without valid H-02/H-03 facts and verify no Routing Table row and no delivery result.
14. Combine valid Registry KB, H-03 compatibility, and subscription facts and verify H-05_def/gov creates the expected concrete source-to-destination row.
15. Make the contract incompatible and verify route creation fails without scientific inspection.
16. Disable/remove a Registry KB provider and verify corresponding rows become inactive/removed before further `RouterService` publication.
17. Verify `RouterService` publishes only through active Routing Table rows and cannot publish from subscription intent alone.
18. Stall one DSE route and verify another Consumer route continues through isolated bounded queues.
19. Add a second FeedSat and second Consumer using configuration/contract facts only; verify no new Core scientific code.
20. Verify all valid routes resolve from concrete participant/service/contract/message/destination/subscription/Domain Plane/entity-filter dimensions with no synthetic route identifier or separate routing identity field.
21. Verify diagnostics identify an affected route using those concrete dimensions and subscription identity without introducing another namespace.
22. Corrupt/reject Registry KB input and verify H-08 isolates the Core relationship failure without modifying or stopping external satellite science.

### 25.2 Real `Fin_FeedSat_1` integration

| Test | Method | Required result |
| :--- | :--- | :--- |
| Producer connectivity | start current FeedSat, then HACCAM | H-01 connects without FeedSat changes |
| Realtime ingress | enable existing scientific streams | accepted events and H-09 counts appear |
| Multiple entities | use the FeedSat's current configured entities | each observed entity creates independent keys; no HACCAM source edit |
| Multiple message identities | receive compatible Decision/Price/Volume events | independent service/contract/message/entity mirrors; Finance message set remains satellite-contract evidence, not Core switch logic |
| H-06 latest state | observe repeated updates | only latest valid value per key remains |
| Provenance | compare edge event with mirror/delivery | source satellite, entity, event/snapshot/sequence/times retained where supplied |
| Noninterference | stop HACCAM while FeedSat remains live | FeedSat ingestion/science/publication continues normally |
| Reconnect | restart HACCAM without FeedSat restart | streams reconnect; catch-up/live rebuild mirrors; no history claim |
| Consumer | run controlled configured Consumer | subscription creates permitted routes and receives current/live values |
| Slow Consumer | stall one Consumer until queue full | drops diagnosed; Producer ingress, mirrors, and second Consumer continue |
| Producer loss | stop/restrict FeedSat endpoint | only its adapter disconnects; last mirrors marked unavailable-by-disconnect |
| Opaque scientific transport | compare serialized FeedSat message before ingress and after authorized delivery | Consumer decodes the same satellite-owned message; HACCAM defines none of its scientific fields |
| Contract/message decoding identity | inspect delivered metadata | Consumer can select the exact satellite contract version and protobuf message type needed to decode |
| Unknown contract | submit an unregistered contract/version/message identity | HACCAM rejects before scientific inspection or mirror update |
| Science-independent routing | route Adaptive, Price, and Volume identities through the common Core path | no Finance-science switch or mathematical interpretation in Core |
| Future Domain Plane | route a non-Finance test contract | same H-01 through H-09 mechanisms; no scientific vocabulary added to Core |
| Co-located logical satellites | host Producer and Consumer test doubles in one process | separate IDs, roles, types, contracts, subscriptions, routes, and lifecycle remain visible |
| No co-located shortcut | inspect/test co-located Producer-to-Consumer flow | every delivery still traverses H-01, H-06, and governed H-05 path |

### 25.3 Multiple-satellite extensibility

Use controlled Producer and Consumer test doubles to configure at least two Producers, two Consumers, multiple entities, and multiple satellite-owned message identities. Prove configuration alone creates separate adapters, mirrors, routes, and failure boundaries. A second real FeedSat is not required. Adding the test participants must not add core `switch` cases keyed by satellite ID, entity, Domain Plane, or scientific message meaning.

### 25.4 Required noninterference sequence

```text
Fin_FeedSat_1 running and producing
        -> HACCAM starts and consumes
        -> HACCAM stops or is killed
        -> Fin_FeedSat_1 remains healthy and continues science/publication
        -> HACCAM restarts and reconnects
```

Evidence includes timestamped FeedSat logs/events before, during, and after HACCAM absence plus HACCAM lifecycle/ingress diagnostics after restart.

## 26. Prototype V0.1 Exclusions

Explicitly excluded, not permanently rejected:

- persistence, Redis, MongoDB, database schemas;
- event sourcing, durable history, durable queues, replay, snapshots, checkpoints, and restart restoration;
- Zero Trust implementation, mTLS, workload identity, authorization, and entitlements;
- Railway, Kubernetes, Azure, or other infrastructure/deployment design;
- DS_TransSat implementation, DS-03 neural network, deterministic transform, or Decision Strategy schema;
- trading execution, broker/OMS, ledger, P&L, or risk engine;
- FeedSat scientific or proto modifications;
- `Fin_FeedSat_1_Viewer` integration;
- peer-to-peer satellite communication;
- distributed exactly-once delivery.

## 27. Failure Isolation Diagram

```text
 Producer A science --> publication --> [Adapter A] --+
       continues if HACCAM fails          X failure    |
                                                     v
 Producer B science --> publication --> [Adapter B] -> H-06
       continues independently                         |
                                                      v
                                                H-05 fan-out
                                                /     |      \
                                      [queue C1] [queue C2] [queue C3]
                                           X slow       |        X fail
                                                       v
                                              Consumer 2 continues

An X is contained at its adapter or delivery queue. It does not propagate
to another Producer, H-06 update, unrelated route, or satellite science.
```

## 28. Internal HACCAM Structure

```text
+------------------------- one HACCAM Core executable ------------------------+
|                                                                            |
| H-01 adapter supervisors --> govern + preserve bytes --> H-06 latest mirror|
|                                                        |                   |
| H-02 Registry KB --------+                             |                   |
| H-03 Contract Governance +--> H-05_def/gov             |                   |
| H-04 Subscription Intent +         |                   |                   |
|                                    v                   |                   |
|                              ROUTING TABLE <------------+                   |
|                                    |                                       |
|                                    v                                       |
|                    RouterService / H-05_deliver queues                     |
|                                                                            |
| H-07 lifecycle evidence <--- adapters/routes/health                         |
| H-08 isolation boundaries ---> adapters/events/routes/receivers             |
| H-09 structured diagnostics <--- every responsibility                       |
|                                                                            |
| external gRPC: RouterService publication + standard health                 |
+----------------------------------------------------------------------------+
```

## 29. Traceability Matrix

| Runtime Design Element | HACCAM Process Model Authority | H-Process | Proto Requirement | Go Runtime Requirement | V0.1 Status |
| :--- | :--- | :--- | :--- | :--- | :--- |
| outbound FeedSat adapter | existing gRPC integration boundary; canonical ingress | H-01 | imports `finfeedsat.v1` client edge | per-Producer supervisor/adapter | required/proposed |
| internal runtime record | generated satellite types remain satellite edge contracts | H-01/H-03 | HACCAM metadata plus governed encoded message | metadata plus immutable bytes; no scientific schema | required/proposed |
| Registry Knowledgebase | Satellite Registry expanded as Core participant/service knowledge | H-02 | role/type enums only for satellites; no Registry RPC | internal participant, service, endpoint, capability, and evidence indexes | required/proposed |
| contract catalog | H-03 governs contracts, not science | H-03 | satellite contract/version/message identity | immutable relationship catalog; no scientific interpretation | required/proposed |
| Consumer interest | WHO wants WHAT | H-04 | `SubscriptionRequest` | stream-scoped in-memory state | required/proposed |
| Routing Table / route governance | H-02 + H-03 + H-04 determine permitted relationship | H-05_def/gov | no separate RPC | dynamic resolved route rows/snapshots | required/proposed |
| `RouterService` outbound publication | H-06 through active Routing Table rows to receiver | H-05_deliver | `StreamMatrixVisibleInformation` | bounded per-destination queue | required/proposed |
| mirror key/value | source satellite/service/contract/message/entity conceptual grain | H-06 | source service/contract/message identity and encoded satellite message | latest-state map/lock; no scientific transform | required/proposed |
| satellite lifecycle view | configured through failed states | H-07 | standard health for Core | sole lifecycle writer | required/proposed |
| per-path isolation | noninterference rules | H-08 | stream errors/cancellation | supervisor/per-route boundaries | required/proposed |
| realtime visibility | minimum diagnostics | H-09 | no custom diagnostics proto | structured logs/metrics | required/proposed |
| Producer+Consumer re-entry | DS-04 returns through canonical ingress | H-01/H-05/H-06 | future adapter edge, not specified | same internal ingress interface | compatible/deferred |
| satellite-owned encoded message | one owner per fact; H-03 contract governance | H-01/H-03/H-05/H-06 | governed `bytes` plus exact contract/version/message identity | immutable preserved bytes; Consumer decodes | required/proposed |
| physical co-location independence | satellite ontology and governed paths | H-02/H-05 | none | logical identity/routing independent of host/process/device | required/proposed |
| restart behavior | latest-state mirrors; sophisticated recovery deferred | H-06/H-08 | none | empty in-memory reconstruction | required/proposed |

## 30. Proto and Internal Naming Status

No Domain Plane or scientific architectural term is proposed. `Registry KB`, `Routing Table`, and `RouterService` clarify conventional responsibilities already assigned to H-02 and H-05; they do not create new H-processes.

| Term | Why an identifier is needed | H-process | Classification / status | Definition |
| :--- | :--- | :--- | :--- | :--- |
| `haccam.v1` | protobuf requires a package namespace | H-03 | proto terminology; retained | first versioned HACCAM edge-contract namespace |
| `RouterService` | protobuf requires a service identifier | H-05_deliver | approved V0.1 proto terminology | HACCAM-owned outbound gRPC publisher using active Routing Table state |
| `StreamMatrixVisibleInformation` | protobuf requires an RPC identifier | H-04/H-05_def/gov/H-05_deliver | approved V0.1 proto terminology | establishes stream-scoped subscription intent, resolves permitted routes, and publishes current/live permitted information |
| `SubscriptionRequest` | protobuf requires a request message identifier | H-04 | proto terminology; retained | boundary expression of WHO wants WHAT |
| `MatrixVisibleInformation` | protobuf requires a delivery message identifier; phrase already exists architecturally | H-03/H-05/H-06 | proto terminology; retained | HACCAM-owned metadata plus unchanged encoded satellite-owned message; not a new semantic or scientific object |
| `ContractIdentity` | protobuf requires a compact name/version type | H-03 | proto terminology; retained | exact H-03 contract name and version |
| `connection generation` | source sequence cannot safely span unknown reconnect semantics | H-01 | runtime/internal terminology | in-memory identifier for one adapter connection lifetime; lost on restart |

Package names in the Go inventory are implementation terminology only and require implementation review, not promotion into HACCAM architecture.

## 31. Process Model Reconciliation Notes

| Existing Process Model statement | Current observed implementation fact | Architectural meaning changes? | Recommended future Process Model update |
| :--- | :--- | :--- | :--- |
| `Fin_FeedSat_1` repository exists but copy not performed | repository is populated and running code is now FeedSat-branded at HEAD `3d763a1` | no; FeedSat remains independent Producer | replace historical copy status with current repository/commit evidence |
| current edge package described as `quantram.v1` | current package is `finfeedsat.v1` and Go package is `finfeedsatv1` | no; still a satellite-owned edge contract | update boundary inventory and paths |
| QuanTRAM environment names and service evidence | current config uses `FIN_FEEDSAT_*` and explicitly carries satellite identity/role/type/Domain Plane | no; reinforces H-02 evidence | update configuration table |
| Prototype V1 configures three entities | current reported live configuration uses AAPL, MSFT, NVDA, AMZN; code remains configurable and defaults to AAPL | no; both counts are configuration, not architecture | remove fixed prototype cardinality from normative statements or mark it historical validation configuration |
| initial source commit `1e760ca` | current FeedSat HEAD is `3d763a1` | no | record current evidence date/commit |
| DS_TransSat required in Prototype V1 | this runtime increment explicitly excludes DS_TransSat implementation | no conflict in ontology; increment scope is narrower | note staged implementation sequence if Process Model is revised |

## 32. Human Review Decisions

### 32.1 Previously resolved architectural decisions

| Decision | Resolution |
| :--- | :--- |
| A: `google.protobuf.Any` | NOT APPROVED. It is not part of the V0.1 design. |
| B: HACCAM-defined Finance scientific payload messages | NOT APPROVED. HACCAM defines no Adaptive, Price, or Volume schema. |
| C: new HACCAM Finance FeedSat Intelligence scientific contract | NOT APPROVED. No parallel HACCAM scientific contract is created. |
| D: satellite-owned scientific contract | RESOLVED. The producing satellite retains ownership; H-03 registers/governs compatibility and relationships around it. |
| E: HACCAM proto responsibility | RESOLVED. HACCAM proto defines only HACCAM-owned H-01 through H-09 services, metadata, governance, mirror, and delivery constructs. |
| F: physical co-location | RESOLVED. Location/packaging changes no logical identity, ownership, role, type, contract, or governed path. |
| G: encoded delivery representation | RESOLVED. `encoded_message` carries unchanged satellite-owned serialized protobuf bytes with exact source contract/message identity. |

### 32.2 Registry, Routing Table, and `RouterService` decisions resolved by this correction

| Decision | Resolution |
| :--- | :--- |
| A: Registry scope | RESOLVED. H-02 is the internal HACCAM Registry KB of internal/external service providers/consumers, services, endpoints, capabilities, and lifecycle evidence. |
| B: Registry network boundary | RESOLVED. H-02 requires no dedicated gRPC/network service in V0.1. |
| C: Routing Table | RESOLVED. It is separate HACCAM-owned operational route state, not Registry knowledge. |
| D: route governance | RESOLVED. H-05_def/gov constructs and maintains the Routing Table from H-02 Registry KB, H-03 compatibility, and H-04 intent. |
| E: `RouterService` responsibility | RESOLVED. `RouterService` is the HACCAM-owned higher-level outbound gRPC publishing service. |
| F: publication execution | RESOLVED. H-05_deliver executes `RouterService` publication through active Routing Table rows. |
| G: internal services | RESOLVED. Internal HACCAM services are not Satellites and have no `SatelliteRole` or `SubscriberType`. |
| H: subscription versus route | RESOLVED. H-04 intent and H-05 Routing Table rows are distinct objects and lifecycle states. |
| I: scientific semantics | RESOLVED. Registry, routing governance, and `RouterService` publication introduce no scientific meaning or ownership. |

### 32.3 Final Registry/Routing/`RouterService` naming and key decisions

| Decision | Resolution |
| :--- | :--- |
| Router gRPC service name | RESOLVED: `RouterService` is approved for V0.1. The former candidate `MatrixDeliveryService` and alternative `RoutingService` are rejected for V0.1. |
| Router RPC name | RESOLVED: `StreamMatrixVisibleInformation` is approved for V0.1. |
| `route_id` | RESOLVED: NOT USED IN V0.1. A Routing Table row is represented and diagnosed by its concrete governed dimensions. Any future operational-key need requires separate design and review. |
| `routing_identity_id` | RESOLVED: REMOVED / NOT USED IN V0.1. No replacement identity abstraction is introduced. |

There are no remaining Registry/Routing/`RouterService` architectural decisions open in V0.1.

No DS_TransSat `SubscriberType` decision is required: the Process Model explicitly sets it to `PRODUCER_CONSUMER`.

### 32.4 Human Design Approval

HACCAM Core Runtime System Design V0.1 is **APPROVED** as of 2026-09-09.

This approval freezes the V0.1 runtime architecture defined by this document as the implementation design authority subordinate to the HACCAM Process Model. It establishes the approved baseline for subsequent protobuf and implementation work; it does not authorize that work. Any later material architectural change must be explicitly identified, reviewed against the HACCAM Process Model, approved by a human, and recorded through the project's normal design/change governance. Approved architectural decisions must not be silently changed during implementation.

**Implementation remains NOT YET AUTHORIZED and requires separate explicit human authorization.**

## 33. Design Completeness / Wholeness Check

| Required determination | Present |
| :--- | :--- |
| what HACCAM Core is/is not and parent authority | yes, Sections 1-2 |
| satellite roles, subscriber types, cardinality, Domain Plane | yes, Sections 2, 7, 23 |
| ownership, scientific noninterpretation, and noninterference | yes, Sections 4, 6, 8, 13, 24 |
| H-01 through H-09 | yes, Sections 6-14 |
| Producer, Consumer, Producer+Consumer, and co-located logical flow | yes, Section 15 |
| Registry KB, contracts, subscription intent, Routing Table, `RouterService`, mirror | yes, Sections 7-11 |
| lifecycle, isolation, diagnostics | yes, Sections 12-14 |
| complete proposed HACCAM-owned proto boundary and governed encoded-message inventory | yes, Section 16 |
| internal Go organization | yes, Section 17 |
| concurrency/backpressure/cancellation | yes, Section 18 |
| configuration/startup/shutdown/restart loss | yes, Sections 19-21 |
| DS_TransSat future compatibility without implementation | yes, Section 22 |
| validation and exclusions | yes, Sections 25-26 |
| required ASCII diagrams | yes, Sections 5, 15, 27, 28 |
| traceability | yes, Section 29 |
| proto/internal naming status and resolved human decisions | yes, Sections 30, 32 |
| Process Model reconciliation without modifying it | yes, Section 31 |

**Wholeness result:** COMPLETE AND APPROVED AS HACCAM CORE RUNTIME SYSTEM DESIGN V0.1. Section 32 contains no remaining Registry/Routing/`RouterService` decision. Implementation remains NOT YET AUTHORIZED and requires separate explicit human authorization.

## Architecture Correction Summary

H-02 is the internal Registry KB for known internal/external participants, services, endpoints, capabilities, relationships, and lifecycle evidence. It is neither a network service nor the Routing Table. H-05_def/gov derives and maintains the separate dynamic Routing Table from H-02 knowledge, H-03 service/contract/message compatibility, and H-04 subscription intent. H-05_deliver executes publication through the HACCAM-owned, network-capable `RouterService` using only active Routing Table rows and bounded isolated delivery state. `RouterService` and `StreamMatrixVisibleInformation` are the final approved V0.1 service/RPC names.

V0.1 does not use `route_id`, and `routing_identity_id` is removed. No separate route or routing identity namespace exists. Each Routing Table row is represented by concrete source/destination participant and service facts, source satellite where applicable, contract/version, message identity, destination/subscription, Domain Plane, optional exact entity filter, route state, and HACCAM-owned timestamps. No replacement abstraction was introduced.

All prior scientific ownership corrections remain intact: the satellite-owned scientific contract is the only scientific contract for satellite output; HACCAM preserves unchanged encoded bytes; H-01 does not scientifically translate; H-03 does not interpret science; H-05 does not alter science; and H-06 owns only the mirror record. `google.protobuf.Any`, normalized protobuf timestamps, and a HACCAM-owned Finance delivery contract remain excluded.

### Materially changed sections

- Executive Summary;
- Section 4, ownership/runtime-record dimensions;
- Section 5, overall architecture diagram;
- Sections 7, 8, 10, and 11, Registry KB, contract governance, Routing Table, `RouterService`, and mirror-key terminology;
- Sections 14 and 15, diagnostics and end-to-end relationships;
- Sections 16 through 18 and Section 20, proto consequences, Go components, concurrency, and startup;
- Sections 24 and 25, invariants and validation;
- Sections 28 through 30, internal structure, traceability, and naming status;
- Section 32, final human decisions;
- Section 33, completeness/wholeness check;
- Architecture Correction Summary;
- Section 34, change log.

### Remaining genuine human decisions

None within the Registry/Routing/`RouterService` scope. Human design approval was recorded on 2026-09-09. Any later implementation authorization remains a separate action.

### Correction consistency check

- Registry KB is not the Routing Table;
- the Routing Table is not `RouterService`;
- H-02 owns the internal knowledgebase and no Registry gRPC service is proposed;
- H-03 owns HACCAM service/contract/message compatibility governance and does not own or interpret satellite science;
- H-04 owns subscription intent: WHO wants WHAT;
- `RouterService` is HACCAM-owned, logically internal, network-capable, and the outbound gRPC publisher;
- `RouterService` is not a Satellite and has no `SatelliteRole` or `SubscriberType`;
- H-05_def/gov owns route construction, maintenance, and permission;
- H-05_deliver owns `RouterService` publication execution and per-destination bounded delivery state;
- H-04 subscription intent remains distinct from resolved route state;
- internal HACCAM services are not Satellites;
- `route_id` is not used in V0.1;
- `routing_identity_id` is removed, and no replacement routing identity namespace exists;
- no HACCAM-defined Finance scientific message or scientific contract is proposed;
- `encoded_message` remains the unchanged serialized satellite-owned protobuf message governed by exact source contract/version/message identity;
- satellite-owned encoded information retains its scientific ownership;
- H-01, H-03, H-05, and H-06 are explicitly prohibited from scientific interpretation or transformation;
- `google.protobuf.Any` is not part of the active V0.1 design;
- HACCAM-owned times use `*_unix_ms`; source time remains source-owned and is not normalized;
- physical host, process, container, pod, application, and device placement do not define satellite identity;
- the design continues to support `0..N` satellites, internal services, routes, entities, services, contracts, and message identities;
- the Process Model remains the parent architectural authority;
- V0.1 remains realtime, in-memory, bounded, nonblocking, and implementation-unauthorized.

## 34. Change Log

| Version | Date | Status | Change |
| :--- | :--- | :--- | :--- |
| V0.1 | 2026-09-09 | PROPOSED FOR HUMAN REVIEW | Initial complete HACCAM Core realtime runtime design. Defines single-process Go organization, current `Fin_FeedSat_1` adapter evidence, in-memory H-02 through H-07 state, H-08 isolation, H-09 diagnostics, bounded concurrency, startup/shutdown, minimal proposed proto, validation, traceability, reconciliation notes, and human decisions. No implementation authorized. |
| V0.1 correction | 2026-09-09 | PROPOSED FOR HUMAN REVIEW | Corrected satellite scientific-contract ownership throughout. Removed `google.protobuf.Any`, normalized protobuf timestamps, and the proposed HACCAM-owned Finance delivery contract. Defined governed transport of unchanged satellite-owned serialized protobuf messages with HACCAM-only metadata; constrained H-01/H-03/H-05/H-06 against scientific interpretation; added physical deployment independence, co-location rules, validation, resolved decisions, and correction consistency summary. Implementation remains unauthorized. |
| V0.1 Registry/Routing correction | 2026-09-09 | PROPOSED FOR HUMAN REVIEW | Defined H-02 as the internal participant/service Registry KB; separated H-04 intent, H-05_def/gov Routing Table ownership, and H-05_deliver Router publication; removed active `routing_identity_id`; added first provider/consumer examples, conceptual route rows, package boundaries, invariants, validation, traceability, and three remaining naming/key decisions. No implementation authorized. |
| V0.1 final correction | 2026-09-09 | PROPOSED FOR HUMAN REVIEW | Approved `RouterService` and `StreamMatrixVisibleInformation`; rejected `route_id` for V0.1; ratified removal of `routing_identity_id`; corrected semantic-compatibility wording; and completed final terminology, invariant, validation, section-reference, and consistency cleanup. Implementation remains NOT YET AUTHORIZED. |
| V0.1 approval | 2026-09-09 | APPROVED | Human approval recorded for HACCAM Core Runtime System Design V0.1. The V0.1 runtime architecture is now the approved design baseline subordinate to the HACCAM Process Model. No architectural content was changed by this approval action. Implementation remains NOT YET AUTHORIZED and requires separate explicit human authorization. |
