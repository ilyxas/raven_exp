# Raven Architecture Review

## 1) Understanding of the proposed architecture

### Core model
Raven is defined as a **platform-independent behavioral system** whose current ESP32/FreeRTOS implementation is only one execution backend. The central architectural idea is role separation:
- **Activities**: orchestration and mode decisions.
- **Services**: subsystem execution/workers.
- **State (Controllers)**: canonical shared truth.
- **Adapters/Transport**: boundary translation between external representations and internal semantic contracts.

The document emphasizes that current domains (navigation, telemetry, joystick, manual mode) are **validation probes**, not the final product boundary.

### Main layers
The architecture describes five runtime layers/concerns:
1. **Coordinator** (composition root in `app_main`) for construction, wiring, startup, shutdown.
2. **Activities** for orchestration/FSM-like behavior.
3. **Services** for ongoing/reactive subsystem work.
4. **Controllers/Shared State** as thread-safe canonical stores.
5. **Event Bus** (`esp_event_loop`) as decoupled event/command transport glue.

### Ownership rules
The intended ownership model is strict:
- Meaningful logic executes in the **owner component context**.
- Cross-task payloads must be **ownership-safe/self-contained at enqueue**.
- Shared data used by multiple components belongs in **Controller state**, not scattered local copies.
- Activities do not execute low-level subsystem/hardware work.
- Services do not perform system-wide orchestration decisions.
- Event callbacks stay thin; callbacks forward and return.

### Communication model
Communication is mixed:
- **Events**: broadcast facts (“something happened”).
- **Commands**: directed intent (“you should do X”).
- **Routing**: explicit admission policy (known message + contract + decoder/adapter + registered route + intended receiver).
- **Transport origin is semantically irrelevant** after adaptation (TCP/RF/mock/sim/synthetic are equivalent once translated to internal contracts).

### Execution model
Execution is explicitly **hybrid**:
- **Message-driven** behavior via event bus and queues.
- **Tick-driven** behavior via periodic task/tick scheduling.

`BaseTask` is presented as substrate capability (async tasks, queues, tick scheduling, payload lifetime handling, context isolation), while role semantics are meant to remain above RTOS mechanics.

### What must exist for this to work
At minimum, the architecture requires:
- A **composition root** that owns all wiring and lifecycle.
- Distinct **orchestration vs subsystem** roles.
- A **thread-safe canonical state layer** for shared truth.
- A **validated adapter/routing boundary** between external transport and internal contracts.
- A **delivery mechanism** that preserves payload lifetime across async boundaries.
- A policy that enforces **owner-context execution** for business logic.
- Support for both **event** and **time/tick** triggers.

---

## 2) Required architectural elements (essential vs optional)

### Essential (must exist)
1. **Role separation**: Activities orchestrate, Services perform, State stores truth.
2. **Single source of truth** for shared data in thread-safe controllers.
3. **Composition root** (single place for object graph/lifecycle).
4. **Explicit routing/admission policy** for message handling.
5. **Owner-context execution** (no business logic in ingress callbacks/foreign contexts).
6. **Safe inter-task payload ownership/lifetime guarantees**.
7. **Event vs command semantic distinction**.
8. **Ability to run message-driven and tick-driven logic coherently**.
9. **Platform abstraction boundary** preserving architecture outside ESP32.

### Strongly implied, but backend-specific (can vary by implementation)
- FreeRTOS tasks/queues as the concrete execution primitive.
- ESP-IDF event loop as concrete event bus implementation.
- `BaseTask`/`BaseActivity`/`BaseService` as current abstractions.

### Examples / current probes (not mandatory product scope)
- Navigation, telemetry, joystick, manual mode, current state models.
- Any particular sensor or autonomy workflow named as future direction.

### Assumptions (ambiguous in original)
- Whether commands are always event-bus mediated vs sometimes direct queue post.
- Which layer owns schema/version evolution for message contracts.
- Required ordering/consistency guarantees across event + state updates.

---

## 3) Potential future architectural problems

1. **Boundary ambiguity between Event Bus and direct queue commands**
   - Risk: two command paths can diverge in semantics, observability, retries, and validation.
   - Fragility: behavior depends on which path was used, hurting predictability.

2. **Unspecified consistency model for state + events**
   - Risk: consumers may observe event “X happened” before/after state reflects X, with no defined guarantee.
   - Fragility: race-condition bugs and non-reproducible mode transitions.

3. **Message contract governance not explicit**
   - Risk: payload shape drift and ad-hoc decoders increase incompatibility over time.
   - Fragility: integration regressions as subsystems grow.

4. **Ownership safety rule is strong but not operationally specified**
   - Risk: contributors may not know when deep copy vs move vs reference is allowed.
   - Fragility: latent lifetime bugs across task boundaries.

5. **Platform-independence intent vs ESP-specific language coupling**
   - Risk: conceptual layers slowly absorb backend assumptions.
   - Fragility: difficult simulation/desktop parity and backend replacement.

6. **Coordinator can become a god-object**
   - Risk: composition root accumulates orchestration or policy logic over time.
   - Fragility: startup/teardown complexity and poor testability.

7. **Tick + message interaction policy is underspecified**
   - Risk: unclear precedence, conflict resolution, and timing assumptions under load.
   - Fragility: mode flapping, starvation, or delayed safety reactions.

8. **Routing admission rules are strict but lifecycle/dynamics unclear**
   - Risk: unclear handling of late registration, route updates, and unavailable targets.
   - Fragility: startup races and brittle dynamic extension.

9. **Debuggability/traceability gap in async model**
   - Risk: hard to reconstruct causal chains across adapters, bus, queues, and ticks.
   - Fragility: long MTTR and weak confidence in behavior under stress.

10. **Examples may still be treated as architecture constraints**
   - Risk: teams optimize for current probe domains instead of architectural invariants.
   - Fragility: accidental scope lock-in.

---

## 4) Proposed improved architecture document

# Raven — Revised Architectural Contract

## Purpose and scope
Raven is a platform-independent behavioral architecture. This document defines **mandatory architectural contracts** independent of execution backend. ESP32/FreeRTOS is a current backend, not the architecture itself.

Current runtime domains (navigation, telemetry, joystick, manual mode) are **reference probes** used to validate these contracts. They are not the architecture’s product scope.

## Architectural primitives

1. **Activity (Orchestration Role)**
   - Owns mode/policy decisions and transition intent.
   - Consumes state snapshots and relevant events.
   - Emits directed commands.

2. **Service (Subsystem Role)**
   - Owns subsystem execution and external/hardware interaction.
   - Produces state updates and publishes factual events.
   - Executes directed commands addressed to it.

3. **State Controller (Canonical Shared State)**
   - Sole canonical store for data needed by multiple components.
   - Provides thread-safe write and snapshot read interfaces.
   - Contains no orchestration or subsystem policy logic.

4. **Adapter Boundary (Ingress/Egress Translation)**
   - Converts transport/wire/protocol representations into internal semantic contracts and vice versa.
   - Performs validation and rejection of invalid external payloads.
   - Does not define behavioral meaning.

5. **Composition Root (Coordinator)**
   - Sole owner of construction, dependency wiring, startup order, and shutdown order.
   - Contains no runtime business policy.

6. **Execution Substrate**
   - Provides scheduling, queueing, periodic triggers, and isolation contexts.
   - Hosts role execution but does not change role semantics.

## Non-negotiable contracts

1. **Separation of concerns**
   - Activities orchestrate.
   - Services perform subsystem work.
   - State stores shared truth.

2. **Owner-context execution**
   - Business logic must run only in the owner component’s execution context.
   - Ingress callbacks and transport handlers are forwarding boundaries only.

3. **Safe asynchronous delivery**
   - Any cross-context message must have defined ownership/lifetime at enqueue time.
   - Receiver correctness must not depend on sender-local buffer lifetime.

4. **Explicit admission routing**
   - Message acceptance requires known contract, successful validation, and explicit destination legitimacy.
   - Unknown or invalid messages are rejected at boundary.

5. **Event/Command semantics**
   - Events communicate facts.
   - Commands communicate directed intent.
   - The chosen delivery mechanism must preserve this semantic distinction.

6. **Canonical state discipline**
   - Shared data is read from controller state, not from private cached copies distributed across components.

7. **Backend independence**
   - Contracts above Adapter and Substrate layers remain valid in simulation/desktop/backend alternatives.

8. **Dual trigger model**
   - Architecture supports both event-triggered and time-triggered behavior.
   - Components relying on both must declare their conflict-resolution policy.

## Required operational clarifications

To keep the architecture reliable at scale, every backend implementation must define:
- Message contract lifecycle (schema ownership, compatibility, deprecation).
- State/event ordering guarantees (or explicit lack thereof).
- Error handling and dead-letter behavior for rejected/undeliverable messages.
- Startup/shutdown sequencing constraints and readiness signaling.
- Observability requirements for tracing causal paths across async boundaries.

## Extension policy

New subsystems or domains are valid if they honor the contracts above. Examples in this document are illustrative and do not restrict future architecture scope.

---

## 5) Meaningful differences from original

1. **Reframed as an explicit “architectural contract” document**
   - Changed: stronger separation between mandatory contracts and illustrative examples.
   - Why: prevent readers from conflating current subsystems with final scope.
   - Risk reduced: accidental scope lock-in and misinterpretation.
   - Tradeoff: document becomes more normative and less narrative.

2. **Made architectural primitives backend-neutral first, backend-specific second**
   - Changed: ESP/FreeRTOS details moved under implementation/substrate framing.
   - Why: align language with stated platform-independence axiom.
   - Risk reduced: backend leakage into core semantics.
   - Tradeoff: less immediate specificity for ESP-only contributors.

3. **Added mandatory operational clarifications section**
   - Changed: explicitly requires definitions for schema lifecycle, ordering, dead-letter/error handling, startup readiness, and observability.
   - Why: original states principles but leaves scale-critical behavior implicit.
   - Risk reduced: race conditions, integration drift, and debugging blind spots.
   - Tradeoff: increased upfront governance/documentation burden.

4. **Normalized communication semantics independent of transport path**
   - Changed: retained event/command distinction while requiring semantic consistency regardless of whether delivery is via bus or direct queue.
   - Why: original allows multiple mechanisms but does not fully constrain equivalence.
   - Risk reduced: hidden behavior divergence by transport path.
   - Tradeoff: may constrain optimization shortcuts in specific backends.

5. **Strengthened dual-trigger rule with conflict-resolution expectation**
   - Changed: tick-driven + message-driven coexistence now requires explicit conflict policy where both apply.
   - Why: hybrid execution is powerful but underdefined under contention.
   - Risk reduced: mode flapping, non-deterministic decisions, timing bugs.
   - Tradeoff: extra design work for components using both triggers.

6. **Clarified extension policy as contract-based, not example-based**
   - Changed: explicit statement that new domains are valid if contracts are honored.
   - Why: preserve openness while protecting invariants.
   - Risk reduced: architectural rigidity caused by historical examples.
   - Tradeoff: requires stronger review discipline around contract compliance.

