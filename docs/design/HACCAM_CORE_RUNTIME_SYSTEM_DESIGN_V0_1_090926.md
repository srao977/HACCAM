# HACCAM Core Runtime System Design V0.1

**Title:** HACCAM Core Runtime System Design V0.1  
**Document filename:** `HACCAM_CORE_RUNTIME_SYSTEM_DESIGN_V0_1_090926.md`  
**Date:** 2026-09-09  
**Version:** V0.1  
**Status:** **PROPOSED FOR HUMAN REVIEW**  
**Implementation:** **NOT YET AUTHORIZED**  
**Parent Architecture Authority:** `HACCAM_PROCESS_MODEL_V0_1_090726.md`, dated 2026-09-07, V0.1, PROPOSED FOR HUMAN REVIEW  
**Purpose:** Define the complete first HACCAM Core realtime runtime design from which a later Go server and protobuf contract can be implemented without inventing architecture.  
**Scope:** One domain-neutral HACCAM Core runtime, the existing `Fin_FeedSat_1` Producer boundary, external Consumer delivery, and the implementation seams needed for later TransSat and ObsSat participation. This document does not implement any runtime, proto, satellite, persistence, or deployment.

## Executive Summary

This document is subordinate to the HACCAM Process Model. The Process Model defines HACCAM architecture, terminology, ownership, and H-01 through H-09. This Runtime System Design selects a conservative first Go organization that realizes those responsibilities in one process without treating each H-process as a microservice.

The first HACCAM Core is one domain-neutral Go server/runtime containing:

- configured, in-memory H-02 satellite records;
- an in-memory H-03 contract catalog and compatibility rules;
- in-memory H-04 subscriptions;
- H-05 route governance and bounded asynchronous delivery;
- latest-state-only H-06 Intelligence Mirrors;
- H-07 lifecycle state;
- H-08 isolation at adapter, event, route, and receiver boundaries;
- H-09 structured logs and bounded metrics;
- H-01 adapters that connect outward to Producer-owned publication boundaries.

The first H-01 adapter is specific to the existing `finfeedsat.v1` gRPC boundary. It connects as a client to `Fin_FeedSat_1` and listens to `StreamDecisions`, `StreamPriceEvents`, and `StreamVolumeEvents`. The FeedSat is not changed to push into HACCAM. The adapter validates participant, contract, message identity, and routing-relevant provenance, then preserves the satellite-owned serialized protobuf message without translating its science into a HACCAM schema. Generated satellite protobuf types remain satellite-owned edge contracts.

The first external HACCAM server boundary is intentionally small: an external Consumer identifies itself and expresses H-04 interest, and HACCAM streams permitted Matrix-visible information through H-05_deliver. H-02, H-03, and H-04 govern route creation; they are not data stages. H-06 is updated before delivery eligibility is evaluated. Each delivery stream has a bounded queue and independent goroutine. A full queue drops delivery for that receiver, records the failure, and never blocks H-01, H-06, another route, or satellite science.

The strongest runtime invariant is:

> HACCAM governs participant identity, semantic and contract compatibility, registration, subscription, routing, mirroring, lifecycle, failure isolation, diagnostics, and delivery. HACCAM does not determine how many satellites exist, how many entities a satellite owns, how many information streams it publishes, or what authoritative information a satellite chooses to create.

Accordingly, the design supports `0..N FeedSat`, `0..N TransSat`, and `0..N ObsSat`. It contains no AAPL, MSFT, NVDA, or AMZN enum, constant, route, mirror slot, or assumption. Those are current `Fin_FeedSat_1` configuration facts. The local FeedSat code supports a configurable symbol set and currently defaults to one symbol; its current four-symbol live configuration does not become HACCAM architecture.

V0.1 is realtime and in-memory. Restart loses registrations derived at runtime, subscriptions, routes, lifecycle observations, mirrors, delivery queues, counters, and diagnostic continuity. Configured satellite and contract definitions are reconstructed from configuration. Producer streams are reconnected and mirrors rebuild only from each Producer's current catch-up and subsequent live events. No replay, exactly-once, durable history, or restoration is claimed.

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
| Satellite record | H-02 | HACCAM for its registry fact |
| HACCAM-visible contract compatibility | H-03 | H-03 |
| Subscription intent | H-04 | H-04 |
| Permitted route | H-05_def/gov | H-05_def/gov |
| Delivery queue/state | H-05_deliver | H-05_deliver |
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
- routing identity derived from the satellite-owned contract's published message/service identity;
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
        +--> H-05_deliver route matching and bounded fan-out

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
         H-01 adapters                 H-02 Registry
               |                       H-03 Contract / Dictionary
               v                       H-04 Subscriptions
                                governed runtime record               |
                         metadata + encoded message             v
               |                       H-05_def/gov
               v                              |
       H-06 Intelligence Mirror --------------+
               |
               v
        H-05_deliver fan-out
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
- After connection, it verifies registered contract compatibility once, polls H-07 evidence independently, and starts one stream worker per configured routing/message identity.
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
4. routing identity is registered to the exact satellite-owned service/message identity in H-03;
5. the declared satellite-owned contract and version are approved for that Producer;
6. routing-relevant identifiers required by the registered adapter rule are present;
7. source provenance is preserved without assigning stronger semantics or normalizing source time;
8. the unchanged satellite-owned message can be serialized for governed transport.

H-01 does not validate scientific enum choices, outcome coherence, mathematics, or scientific correctness. A boundary-invalid event is isolated individually; it does not terminate another stream. Repeated boundary failures may mark that stream degraded, but they never modify or reinterpret FeedSat science.

### 6.4 Sequence and duplicate handling

- `accepted_sequence` is retained as source provenance and compared only within `(satellite, entity, routing identity, adapter connection generation)`.
- It is not treated as a global HACCAM sequence.
- Exact repeated nonempty `event_id` values within a bounded in-memory recent-ID set for the same mirror key are rejected as duplicates.
- A lower/equal sequence with a different event ID is diagnosed as regression/ambiguity and rejected unless the H-03 adapter rule explicitly permits it.
- Reconnect begins a new connection generation. The FeedSat's latest-event catch-up may repeat the current event; event ID suppression avoids a duplicate replacement where possible.
- No continuity, replay, or exactly-once claim is made when events were dropped or HACCAM was absent.

### 6.5 Producer failure and recovery

On Producer disconnect, existing mirrors remain as last received values but are marked unavailable-by-disconnect in HACCAM metadata; HACCAM does not rewrite the preserved satellite-owned message. Routes remain defined but no new source information is delivered. On recovery, catch-up and live events update mirrors normally.

## 7. H-02 Satellite Registry

### 7.1 V0.1 model

The registry is in-memory and supports N records. V0.1 records are loaded from HACCAM configuration; Consumer stream establishment activates a configured record but does not invent an unconfigured identity.

| Field | HACCAM reason |
| :--- | :--- |
| `satellite_id` | unique participant identity and mirror/route provenance |
| `SatelliteRole` | Process Model ontology validation |
| `SubscriberType` | explicit Producer/Consumer relationship capability |
| Domain Plane | semantic compatibility boundary |
| endpoint, when HACCAM connects outward | H-01 Producer connection |
| edge contract identity/version | H-03 adapter compatibility |
| advertised routing/message identities | identities derived from satellite-owned contracts for route governance; never a parallel scientific taxonomy or command to produce |
| lifecycle state | H-07 view |
| configured/connected evidence | distinguish declaration from runtime observation |

No generic metadata bag, lease protocol, tags, ownership team, or scientific parameters are included.

### 7.2 Rules

- `satellite_id` is nonempty and unique. Duplicate configured identities fail HACCAM startup.
- Role and subscriber type must be explicit; protobuf `UNSPECIFIED` is rejected.
- `PRODUCER` and `PRODUCER_CONSUMER` may advertise information.
- `CONSUMER` and `PRODUCER_CONSUMER` may establish subscriptions.
- Endpoint is required only for an HACCAM-owned outbound adapter.
- Replacement means lifecycle reconnection of the same configured identity, not silent registration of a new participant.
- A Domain Plane mismatch prevents route creation; it does not rewrite the participant.
- Removing a configured participant is a configuration restart operation in V0.1.

## 8. H-03 Contract / Dictionary

H-03 is not a proto file. It governs contract identity, compatibility, participant relationships, and the rules under which a satellite-owned message may become HACCAM-visible and routable. The producing satellite owns the scientific contract and the meaning and mathematics of its payload. Registering that contract in H-03 does not transfer ownership.

### 8.1 H-03 catalog content

For each routable satellite publication, the in-memory catalog records:

- a routing identity derived from or explicitly associated with the satellite-owned service/message identity;
- Domain Plane;
- permitted publisher identity, `SatelliteRole`, and `SubscriberType`;
- satellite-owned contract identity/version;
- exact satellite-owned service/message identity;
- routing-relevant identifiers and provenance that HACCAM may extract;
- permitted Consumer contracts and subscription relationships;
- compatibility rule: exact in V0.1 unless explicitly approved otherwise.

Routing identity is not scientific semantic ownership. It distinguishes publications so H-04 and H-05 can match a Consumer that already understands the satellite-owned contract. H-03 does not define what `V_N`, `Q_G`, strength, Price prediction, Volume phase, or any other scientific value means; it does not calculate or judge those values.

### 8.2 Finance evidence classification

| Current FeedSat concept | Classification | HACCAM use |
| :--- | :--- | :--- |
| `DecisionEvent` and nested Adaptive terms | Finance Domain Plane, FeedSat-owned scientific contract | preserved encoded message; routing identity references `finfeedsat.v1.DecisionEvent` |
| `PriceEvent`, emission, cockpit, skip | Finance Domain Plane, FeedSat-owned scientific contract | preserved encoded message; routing identity references `finfeedsat.v1.PriceEvent` |
| `VolumeEvent`, quantities, indicator, phase | Finance Domain Plane, FeedSat-owned scientific contract | preserved encoded message; routing identity references `finfeedsat.v1.VolumeEvent` |
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
- zero or more exact entity filters (empty means any entity from matched sources);
- one or more routing identities associated with satellite-owned contract/message identities;
- lifecycle: active, cancelling, removed;
- creation and removal timestamps.

The filter language is exact-match only. No wildcard expressions, predicates over encoded scientific messages, query language, or scientific filter is designed in V0.1.

For the first server-streaming boundary, the subscription lifetime equals the RPC stream lifetime. Stream cancellation changes the subscription to removed and causes H-05 to remove associated routes. Multiple streams from the same configured Consumer are separate subscriptions.

## 10. H-05 Routing & Distribution

### 10.1 H-05_def/gov

H-05_def/gov creates a permitted route only when all are true:

1. H-02 source exists and may produce;
2. H-02 receiver exists and may consume;
3. source and receiver Domain Plane association is compatible;
4. H-03 recognizes the satellite-owned contract/version/message identity and confirms Consumer compatibility;
5. H-04 records receiver interest matching source, entity, and routing identity;
6. the route does not point a satellite directly to a peer publication boundary.

Route definitions are recomputed when a subscription starts/ends, participant lifecycle changes, or applicable configured contract state changes. Governance is not a per-event synchronous pipeline.

### 10.2 H-05_deliver

For every active external subscription stream:

- one bounded queue is owned by H-05_deliver;
- one sender goroutine is the sole receiver from and closer-user of that queue;
- mirror update processing performs a nonblocking enqueue per matching route;
- the immutable runtime record can be referenced by multiple queues;
- sender serialization copies HACCAM metadata and the unchanged encoded satellite-owned message into the outbound delivery message, then calls `Send`;
- stream context cancellation or send failure removes the subscription/routes and terminates the sender;
- queue full drops the delivery for only that subscription, increments diagnostics, and marks the Consumer degraded/slow;
- successful later sends may return the route to connected after a configured observation threshold, without claiming delivery of dropped values.

Default recommendation: queue capacity 64 per active delivery stream, configurable with a positive upper bound. Sixty-four is large enough to absorb short scheduler/network pauses and small enough to make slow Consumers visible. It is not an architectural constant.

No retry is performed for an individual dropped realtime value. Retrying into the same full queue would amplify pressure and imply durability that V0.1 does not provide. The Consumer can reconnect and receive current latest mirror values as initial catch-up, followed by live updates.

H-05 never decodes, changes, or assigns scientific meaning to the encoded message. The Consumer uses the delivered satellite-owned contract version and message identity to decode the exact scientific contract it subscribed to.

### 10.3 Fan-out and route removal

Fan-out iterates a snapshot of matching routes under no route lock while enqueueing. Route table mutation uses copy-on-write or a short `RWMutex`; no network send occurs while holding it. Removal marks a route closed, detaches it from future snapshots, then signals its sender context. Only the route owner closes its queue, preventing send-on-closed-channel races.

## 11. H-06 Intelligence Mirror

### 11.1 Mirror grain and key

The key is exactly:

```text
(source satellite_id, source entity identifier, routing identity)
```

No fixed satellite, entity, or routing/message-identity cardinality exists.

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

Logs include `satellite_id`, Domain Plane, routing/message identity, entity where safe, subscription/route identity, and source event identity where supplied. They do not log encoded scientific messages by default. Diagnostics are realtime and are lost on restart.

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
configured Consumer identity (H-02)
        + H-04 WHO wants WHAT
        + H-03 compatibility
                    |
                    v
              H-05_def/gov
                    |
H-06 latest/live --> H-05_deliver --> Consumer satellite
                      bounded async
```

### 15.3 Producer+Consumer path

```text
H-05_deliver --> TransSat consumes Matrix-visible information
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

H-06 and H-05 observe the same accepted immutable runtime record produced by H-01: HACCAM-owned metadata plus the unchanged encoded satellite-owned message. H-06 owns latest-state replacement; H-05_deliver receives notification after replacement. H-07 and H-09 receive evidence through nonblocking internal calls/counters. These responsibilities are not Satellites, do not subscribe through an external satellite RPC, and do not form a second routing architecture or scientific model.

### 15.5 Co-located satellites

Logical paths are unchanged by packaging:

```text
one Android application or host
        +-- Fin_FeedSat_1 (FeedSat / PRODUCER)
        +-- Decision Strategy Engine (TransSat / PRODUCER_CONSUMER)

required logical path:
Fin_FeedSat_1 -> H-01 -> HCM -> H-05_deliver -> Decision Strategy Engine
```

An in-process transport optimization may be considered only if it implements the same H-01/H-03/H-04/H-05/H-06 governance and failure boundaries. It may not become a direct scientific shortcut.

## 16. Proposed First HACCAM Proto Design

No `.proto` file is created by this design. Proposed package: `haccam.v1`.

Every proposed element below passes the ownership test: HACCAM owns the service, subscription/routing metadata, mirror/receipt metadata, and governed delivery representation. HACCAM does not own or redefine the encoded satellite message. No Finance scientific enum, message, status, calculation, or HACCAM Finance scientific contract belongs in this proto.

### 16.1 Service inventory and rationale

#### `MatrixDeliveryService` (new proto terminology; human approval required)

| Item | Design |
| :--- | :--- |
| H-process | H-04 request boundary; H-05_def/gov validation; H-05_deliver execution |
| Purpose | deliver governed realtime Matrix-visible information to an external configured Consumer |
| Caller | satellite with `SubscriberType = CONSUMER` or `PRODUCER_CONSUMER` |
| Server | HACCAM Core |
| RPC | `StreamMatrixVisibleInformation(SubscriptionRequest) returns (stream MatrixVisibleInformation)` |
| Direction | server streaming |
| Failure | invalid identity/filter: `INVALID_ARGUMENT`; unregistered or wrong subscriber type: `FAILED_PRECONDITION`; incompatible contract: `FAILED_PRECONDITION`; shutdown: `UNAVAILABLE`; cancellation ends subscription |
| Why network boundary | the Consumer is an external satellite process; H-05 delivery must cross that boundary |

No H-02 registration RPC is required in V0.1 because the registry is configured. No separate create/delete subscription RPC is required because the stream request and context define subscription lifecycle. No diagnostics RPC is required because H-09 uses logs/metrics. Core health should use standard `grpc.health.v1.Health`, not a new HACCAM message.

A future satellite publication RPC may be added as another H-01 adapter when a Producer lacks an existing publication boundary. It is not required for the first runtime and is not specified speculatively. This does not change H-01 architecture or internal ingress handling.

### 16.2 Messages

#### `SubscriptionRequest`

Owner: H-04. Producer: external Consumer. Consumers: H-04 and H-05_def/gov.

| Field | Type | Semantics |
| :--- | :--- | :--- |
| `subscriber_satellite_id` | `string` | required; must match a configured H-02 Consumer-capable identity |
| `domain_plane` | `string` | required; exact configured association in V0.1 |
| `source_satellite_ids` | `repeated string` | optional filter; empty means any compatible source in the Domain Plane |
| `entity_ids` | `repeated string` | optional exact-match filter; empty means any entity |
| `routing_identity_ids` | `repeated string` | required, nonempty; each identifies a registered satellite-owned message/service identity, not a HACCAM scientific taxonomy |
| `accepted_contracts` | `repeated ContractIdentity` | required, nonempty; contracts the Consumer can decode |

Required/optional semantics are validated by the server; proto3 field presence alone is not treated as semantic validity. Repeated filters are deduplicated. The request is immutable for a stream; changing interest requires reconnecting with a new request.

#### `MatrixVisibleInformation`

This name directly uses the Process Model term “Matrix-visible information”; it is not a new architectural envelope concept. Owner: H-05 delivery representation. Producer: HACCAM Core. Consumer: subscribed satellite.

| Field | Type | Semantics |
| :--- | :--- | :--- |
| `source_satellite_id` | `string` | required authoritative Producer identity |
| `domain_plane` | `string` | required semantic context |
| `entity_id` | `string` | required opaque source-owned entity identity |
| `routing_identity_id` | `string` | required H-03 routing identity associated with the satellite-owned message identity |
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

Owner of this registration metadata: H-03. Owner of the identified scientific contract: the producing satellite. Producer of the metadata: HACCAM configuration or Consumer request. Consumers: H-03/H-05.

| Field | Type | Semantics |
| :--- | :--- | :--- |
| `name` | `string` | required canonical H-03 contract name |
| `version` | `string` | required exact version in V0.1 |
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
| `haccam.v1` | package | versioned HACCAM edge namespace | H-03 | yes | proto term, approval required |
| `MatrixDeliveryService` | service | external Consumer delivery | H-04/H-05 | yes | proto term, approval required |
| `StreamMatrixVisibleInformation` | RPC | subscription lifetime and realtime delivery | H-04/H-05_deliver | yes | proto term, approval required |
| `SubscriptionRequest` | message | WHO wants WHAT | H-04 | yes | proto term, approval required |
| `MatrixVisibleInformation` | message | HACCAM metadata plus unchanged encoded satellite-owned message; not a scientific schema | H-03/H-05/H-06 | yes | proto capitalization/name approval required |
| `ContractIdentity` | message | exact H-03 name/version | H-03 | yes | proto term, approval required |
| `SatelliteRole` | enum | role ontology | H-02/Process Model | yes | no architectural term; proto spelling approval |
| `SubscriberType` | enum | relationship classification | H-02/Process Model | yes | no architectural term; proto spelling approval |
| `grpc.health.v1.Health` | standard service | Core liveness/readiness | H-07 | yes | no |
| future publication RPC | deferred service/RPC | later H-01 adapter for a Producer needing HACCAM-owned ingress edge | H-01 | no | not proposed now |

## 17. Go Runtime Component Inventory

Recommended module organization uses a small number of behavior-oriented packages; package names are implementation terminology, not architecture.

| Component/package | Purpose / H-process | Inputs / outputs | State and concurrency owned | Failure boundary | Explicit non-responsibilities |
| :--- | :--- | :--- | :--- | :--- | :--- |
| `cmd/haccam-core` | composition root | config/signals; starts runtime | root context, startup/shutdown ordering | process | no H-process logic |
| `internal/core` | accepted-information coordinator | governed runtime record; mirror update then route notification | no long-lived queue beyond configured ingress dispatcher | core invariant boundary | no scientific protobuf definitions, decoding, or science |
| `internal/adapter/finfeedsat` | first H-01 adapter | `finfeedsat.v1` streams to HACCAM metadata plus unchanged encoded messages | connection supervisor, stream goroutines, bounded ingress queue | per configured Producer | no FeedSat modification, scientific transformation/validation, or routing |
| `internal/registry` | H-02 | config/lifecycle view; lookups | registry map and lock | startup validation | no discovery, contracts, delivery |
| `internal/contract` | H-03 | registered satellite contract/message identities and relationship compatibility | immutable catalog after startup | incompatible contract/version/message | no scientific meaning, mathematics, or payload correctness validation |
| `internal/subscription` | H-04 | stream requests; subscription snapshots | subscription map and lock | per subscription | no route permission or delivery |
| `internal/routing` | H-05_def/gov + H-05_deliver | registry/contracts/subscriptions/mirror updates; outbound values | route table, per-stream bounded queue and sender goroutine | per route/receiver | no scientific transformation or semantic ownership |
| `internal/mirror` | H-06 | accepted runtime records; latest snapshots | keyed map and `RWMutex` | per update validation before lock | no history, delivery, or scientific transformation |
| `internal/lifecycle` | H-07 | evidence from adapters/routes/health | sole lifecycle writer; small event queue or synchronized calls | per participant | no scientific status interpretation |
| `internal/diagnostics` | H-09 and H-08 evidence | structured events/counters | bounded/nonblocking diagnostics | sink | no control over satellite science |
| `internal/server` | proto boundary | gRPC requests/responses | RPC contexts; delegates queue ownership to routing | per RPC | no internal protobuf model adoption |

H-08 is implemented through boundaries in adapter, core, routing, and diagnostics rather than a package that catches every error.

## 18. Concurrency, Locks, Queues, and Cancellation

### 18.1 Goroutine ownership

The composition root owns an `errgroup` under one root context. Each adapter supervisor owns its connection and stream worker goroutines. Routing owns one sender goroutine per active external stream. The gRPC server owns request handler goroutines. No goroutine is started without an owner, cancellation path, and join path.

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
| Satellite bootstrap | N satellite IDs, role, subscriber type, Domain Plane, optional endpoint |
| Adapter | adapter kind, satellite-owned contract/version/message identities, routing identities, optional entity filters |
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
  -> load H-02 configured satellite records as configured
  -> initialize empty H-06 mirror, H-04 subscriptions, and H-05 routes
  -> initialize H-07 lifecycle and H-09 diagnostics
  -> bind gRPC/health listeners but report NOT_SERVING
  -> start H-05 delivery manager and gRPC serving
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

HACCAM Core understands participant identity, Domain Plane, satellite-owned contract identity/version, routing/message identity, routing-relevant entity identity, provenance, mirror replacement, subscription, and delivery. It does not understand Adaptive, Price, Volume, symbol classification, scientific status, or Finance mathematics.

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

## 25. First Implementation Validation Plan

### 25.1 Unit and contract tests

1. Validate all role/subscriber combinations and reject `UNSPECIFIED`.
2. Load zero, one, and multiple configured satellites without core code changes.
3. Preserve representative serialized Decision, Price, Volume, and typed skip messages without HACCAM scientific field definitions or mutation; reject only boundary-invalid contract/message metadata.
4. Verify mirror keys distinguish source satellite, entity, and contract-derived routing identity.
5. Verify duplicate event ID and scoped sequence regression do not replace last valid mirror.
6. Verify subscription exact matching for any/source/entity/routing-identity combinations.
7. Verify H-05_def/gov rejects wrong Domain Plane, subscriber capability, and contract version.
8. Fill one delivery queue and prove mirror updates and another Consumer continue.
9. Cancel every context path and use goroutine-leak/race tests.
10. Run `go test -race ./...` and bounded-queue stress tests.

### 25.2 Real `Fin_FeedSat_1` integration

| Test | Method | Required result |
| :--- | :--- | :--- |
| Producer connectivity | start current FeedSat, then HACCAM | H-01 connects without FeedSat changes |
| Realtime ingress | enable existing scientific streams | accepted events and H-09 counts appear |
| Multiple entities | use the FeedSat's current configured entities | each observed entity creates independent keys; no HACCAM source edit |
| Multiple message identities | receive compatible Decision/Price/Volume events | independent routing-identity mirrors; Finance message set remains satellite-contract evidence, not Core switch logic |
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
|         |                                      |                |           |
|         |                                      +--> H-05_deliver queues     |
|         |                                              ^                    |
| H-02 Registry ----+                                    |                    |
| H-03 Contract ----+--> H-05_def/gov <-- H-04 Subscriptions                 |
|                                                                            |
| H-07 lifecycle evidence <--- adapters/routes/health                         |
| H-08 isolation boundaries ---> adapters/events/routes/receivers             |
| H-09 structured diagnostics <--- every responsibility                       |
|                                                                            |
| external gRPC: Matrix delivery + standard health                           |
+----------------------------------------------------------------------------+
```

## 29. Traceability Matrix

| Runtime Design Element | HACCAM Process Model Authority | H-Process | Proto Requirement | Go Runtime Requirement | V0.1 Status |
| :--- | :--- | :--- | :--- | :--- | :--- |
| outbound FeedSat adapter | existing gRPC integration boundary; canonical ingress | H-01 | imports `finfeedsat.v1` client edge | per-Producer supervisor/adapter | required/proposed |
| internal runtime record | generated satellite types remain satellite edge contracts | H-01/H-03 | HACCAM metadata plus governed encoded message | metadata plus immutable bytes; no scientific schema | required/proposed |
| configured satellite records | Satellite Registry | H-02 | role/type enums approved; no registration RPC | in-memory registry | required/proposed |
| contract catalog | H-03 governs contracts, not science | H-03 | satellite contract/version/message identity | immutable relationship catalog; no scientific interpretation | required/proposed |
| Consumer interest | WHO wants WHAT | H-04 | `SubscriptionRequest` | stream-scoped in-memory state | required/proposed |
| route permission | H-02 + H-03 + H-04 determine relationship | H-05_def/gov | no separate RPC | route snapshots | required/proposed |
| async Consumer delivery | H-06 to H-05_deliver to receiver | H-05_deliver | server stream | bounded per-stream queue | required/proposed |
| mirror key/value | `(satellite, entity, information type)` conceptual grain | H-06 | routing identity and encoded satellite message | latest-state map/lock; no scientific transform | required/proposed |
| satellite lifecycle view | configured through failed states | H-07 | standard health for Core | sole lifecycle writer | required/proposed |
| per-path isolation | noninterference rules | H-08 | stream errors/cancellation | supervisor/per-route boundaries | required/proposed |
| realtime visibility | minimum diagnostics | H-09 | no custom diagnostics proto | structured logs/metrics | required/proposed |
| Producer+Consumer re-entry | DS-04 returns through canonical ingress | H-01/H-05/H-06 | future adapter edge, not specified | same internal ingress interface | compatible/deferred |
| satellite-owned encoded message | one owner per fact; H-03 contract governance | H-01/H-03/H-05/H-06 | governed `bytes` plus exact contract/version/message identity | immutable preserved bytes; Consumer decodes | required/proposed |
| physical co-location independence | satellite ontology and governed paths | H-02/H-05 | none | logical identity/routing independent of host/process/device | required/proposed |
| restart behavior | latest-state mirrors; sophisticated recovery deferred | H-06/H-08 | none | empty in-memory reconstruction | required/proposed |

## 30. New Terms Requiring Human Approval

No new HACCAM architectural term is proposed.

| Proposed term | Why existing term is insufficient as an identifier | H-process | Classification | Definition |
| :--- | :--- | :--- | :--- | :--- |
| `haccam.v1` | protobuf requires a package namespace | H-03 | proto terminology | first versioned HACCAM edge-contract namespace |
| `MatrixDeliveryService` | protobuf requires a service identifier for existing H-05 delivery | H-05 | proto terminology | server boundary that executes governed delivery to external Consumers |
| `StreamMatrixVisibleInformation` | protobuf requires an RPC identifier | H-04/H-05 | proto terminology | creates a stream-scoped subscription and delivers current/live permitted information |
| `SubscriptionRequest` | protobuf requires a request message identifier | H-04 | proto terminology | boundary expression of WHO wants WHAT |
| `MatrixVisibleInformation` | protobuf requires a delivery message identifier; phrase already exists architecturally | H-03/H-05/H-06 | proto terminology | HACCAM-owned metadata plus unchanged encoded satellite-owned message; not a new semantic or scientific object |
| `ContractIdentity` | protobuf requires a compact name/version type | H-03 | proto terminology | exact H-03 contract name and version |
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

## 32. Human Review Decisions and Remaining Decisions

### 32.1 Resolved architectural decisions

| Decision | Resolution |
| :--- | :--- |
| A: `google.protobuf.Any` | NOT APPROVED. It is not part of the V0.1 design. |
| B: HACCAM-defined Finance scientific payload messages | NOT APPROVED. HACCAM defines no Adaptive, Price, or Volume schema. |
| C: new HACCAM Finance FeedSat Intelligence scientific contract | NOT APPROVED. No parallel HACCAM scientific contract is created. |
| D: satellite-owned scientific contract | RESOLVED. The producing satellite retains ownership; H-03 registers/governs compatibility and relationships around it. |
| E: HACCAM proto responsibility | RESOLVED. HACCAM proto defines only HACCAM-owned H-01 through H-09 services, metadata, governance, mirror, and delivery constructs. |
| F: physical co-location | RESOLVED. Location/packaging changes no logical identity, ownership, role, type, contract, or governed path. |
| G: protobuf vocabulary names | PARTIALLY OPEN. Existing names are retained only where they accurately identify HACCAM-owned responsibilities; approval remains required below. |

### 32.2 Remaining decision: approve the proposed HACCAM proto vocabulary

- **Issue:** protobuf requires concrete package/service/RPC/message identifiers not fixed by the Process Model.
- **Process Model position:** H-03 governs semantics; future proto is an implementation expression and edge contract.
- **Proposed Runtime Design position:** approve `haccam.v1`, `MatrixDeliveryService`, `StreamMatrixVisibleInformation`, `SubscriptionRequest`, `MatrixVisibleInformation`, and `ContractIdentity`.
- **Alternatives:** different identifiers with identical responsibilities; split subscription management into extra RPCs.
- **Consequence:** identifiers become a public edge vocabulary; extra RPCs increase lifecycle and consistency surface.
- **Recommendation:** approve the minimal single-stream boundary as proposed.

### 32.3 Remaining decision: approve the governed encoded-message field names

- **Issue:** protobuf still requires names for the transport field and routing identity even though neither is a new architectural/scientific concept.
- **Process Model position:** HACCAM governs routing/delivery while satellite contracts retain scientific ownership.
- **Proposed Runtime Design position:** use `encoded_message` for governed serialized protobuf bytes and `routing_identity_id` for the identifier associated with the satellite-owned contract/message identity.
- **Alternatives:** `serialized_message` and `message_route_id`; no scientific naming alternatives are acceptable.
- **Consequence:** names become public edge vocabulary and must not imply HACCAM owns message science.
- **Recommendation:** approve `encoded_message` and `routing_identity_id` as proto-only terminology with the definitions in Section 16.

No DS_TransSat `SubscriberType` decision is required: the Process Model explicitly sets it to `PRODUCER_CONSUMER`.

## 33. Design Completeness / Wholeness Check

| Required determination | Present |
| :--- | :--- |
| what HACCAM Core is/is not and parent authority | yes, Sections 1-2 |
| satellite roles, subscriber types, cardinality, Domain Plane | yes, Sections 2, 7, 23 |
| ownership, scientific noninterpretation, and noninterference | yes, Sections 4, 6, 8, 13, 24 |
| H-01 through H-09 | yes, Sections 6-14 |
| Producer, Consumer, Producer+Consumer, and co-located logical flow | yes, Section 15 |
| registry, contracts, subscriptions, routing, mirror | yes, Sections 7-11 |
| lifecycle, isolation, diagnostics | yes, Sections 12-14 |
| complete proposed HACCAM-owned proto boundary and governed encoded-message inventory | yes, Section 16 |
| internal Go organization | yes, Section 17 |
| concurrency/backpressure/cancellation | yes, Section 18 |
| configuration/startup/shutdown/restart loss | yes, Sections 19-21 |
| DS_TransSat future compatibility without implementation | yes, Section 22 |
| validation and exclusions | yes, Sections 25-26 |
| required ASCII diagrams | yes, Sections 5, 15, 27, 28 |
| traceability | yes, Section 29 |
| new terms, resolved decisions, and remaining decisions | yes, Sections 30, 32 |
| Process Model reconciliation without modifying it | yes, Section 31 |

**Wholeness result:** COMPLETE FOR HUMAN DESIGN REVIEW. Implementation remains unauthorized until the human decisions in Section 32 are resolved and the design is approved.

## Architecture Correction Summary

The V0.1 design now makes the satellite-owned scientific contract the only scientific contract for satellite output. HACCAM accepts, mirrors, and delivers an unchanged serialized satellite-owned protobuf message together with HACCAM-owned governance, routing, provenance, receipt, mirror, and lifecycle metadata. H-01 does not scientifically translate it; H-03 does not interpret or validate its science; H-05 does not alter it; H-06 owns only the mirror record. `google.protobuf.Any`, normalized protobuf timestamps, and the proposed HACCAM-owned Finance delivery contract have been removed. Logical satellite identity and governed paths are explicitly independent of physical co-location.

### Materially changed sections

- Executive Summary;
- Sections 2 through 6;
- Sections 7 through 11;
- Sections 13 through 19;
- Sections 22 through 25;
- Sections 28 through 30;
- Sections 32 through 34.

### Remaining genuine human decisions

1. Approve or rename the proto vocabulary listed in Section 32.2.
2. Approve or rename the proto-only fields `encoded_message` and `routing_identity_id` in Section 32.3.

### Correction consistency check

- no HACCAM-defined Finance scientific message or scientific contract is proposed;
- satellite-owned encoded information retains its exact contract/version/message identity and scientific ownership;
- H-01, H-03, H-05, and H-06 are explicitly prohibited from scientific interpretation or transformation;
- HACCAM-owned times use `*_unix_ms`; source time remains source-owned and is not normalized;
- physical host, process, container, pod, application, and device placement do not define satellite identity;
- the design continues to support `0..N` satellites, entities, and routing/message identities;
- the Process Model remains the parent architectural authority;
- V0.1 remains realtime, in-memory, bounded, nonblocking, and implementation-unauthorized.

## 34. Change Log

| Version | Date | Status | Change |
| :--- | :--- | :--- | :--- |
| V0.1 | 2026-09-09 | PROPOSED FOR HUMAN REVIEW | Initial complete HACCAM Core realtime runtime design. Defines single-process Go organization, current `Fin_FeedSat_1` adapter evidence, in-memory H-02 through H-07 state, H-08 isolation, H-09 diagnostics, bounded concurrency, startup/shutdown, minimal proposed proto, validation, traceability, reconciliation notes, and human decisions. No implementation authorized. |
| V0.1 correction | 2026-09-09 | PROPOSED FOR HUMAN REVIEW | Corrected satellite scientific-contract ownership throughout. Removed `google.protobuf.Any`, normalized protobuf timestamps, and the proposed HACCAM-owned Finance delivery contract. Defined governed transport of unchanged satellite-owned serialized protobuf messages with HACCAM-only metadata; constrained H-01/H-03/H-05/H-06 against scientific interpretation; added physical deployment independence, co-location rules, validation, resolved decisions, and correction consistency summary. Implementation remains unauthorized. |
