# Raven — Architecture

This document defines the conceptual model of the Raven runtime, the role boundaries between its layers, and the architectural invariants that every contributor must respect. It should be read as the interpretation guide for the codebase, especially where temporary implementation shortcuts or early experimental subsystems might otherwise obscure the intended design.

---

## Core Axioms

The current concrete domains in Raven — navigation, telemetry, joystick input, manual mode, and the present state models — should be understood as early architectural probes, not as the final shape of the system. They were introduced quickly to test responsiveness, stress behaviour, ownership boundaries, routing discipline, and the interaction between message-driven and tick-driven execution. They are useful because they let the runtime be exercised under load and in motion, but they do not define the limit of the architecture. More serious capabilities will be added later, including richer sensor-driven navigation and more advanced autonomy.

### 1. Raven is a platform-independent behavioural system

Raven must not be treated as “ESP32 firmware with logic inside it”. The intended model is higher-level: Raven is a platform-independent behavioural system, and ESP32/FreeRTOS is currently one execution backend for that system.

The architecture must therefore preserve its meaning outside ESP hardware:
- the same roles and contracts should remain valid in simulation,
- desktop-hosted execution is legitimate,
- hardware presence is optional at the architectural level,
- platform-specific details belong below the behavioural core.

### 2. Activities, Services, and State belong to the behavioural core

Activities, Services, and State are not ESP-specific constructs. They exist above the platform layer because they express the system’s behaviour:
- Activities own orchestration and mode logic,
- Services own subsystem work,
- State owns canonical truth.

If platform-specific dependencies appear inside these layers, that should be treated as temporary leakage, not as the intended design.

### 3. Semantic meaning is more important than message origin

For the behavioural core, the origin of a message is not architecturally important. A message may arrive:
- from TCP,
- from RF control,
- from mock input,
- from simulation,
- from synthetic sensor sources,
- from local generation inside the system.

What matters is:
- the meaning of the message,
- the payload contract,
- the legitimacy of the route,
- the ownership of the handling.

A component should react to a valid semantic contract, not to the transport path by which it arrived.

### 4. External transport is an adapter boundary, not the semantic center

Wire formats, TCP framing, `msg_id`, decoders, encoders, and gateways are boundary mechanisms. Their role is to:
- accept an external representation,
- validate it,
- translate it into an internal semantic contract,
- deliver it to the correct owner.

Transport origin must not become part of the behavioural meaning of the system.

### 5. Routing is strict admission policy

Routing in Raven is not generic packet forwarding. It is a strict admission policy.

A component may receive a message only if:
- the message is known,
- its payload matches the expected contract,
- the system has an explicit decoder or adapter for it,
- a route is explicitly registered,
- the target is architecturally intended to receive it.

No component should ever receive data it is not prepared to handle.

### 6. Roles must remain separate from execution machinery

Activities and Services are role-level concepts. They should be understood in terms of responsibility, not in terms of RTOS primitives.

- An Activity is an orchestration role.
- A Service is a subsystem execution role.

Task, queue, tick, and scheduling machinery belong to the execution substrate below those roles. The domain model should not be forced to think in FreeRTOS terms.

### 7. The execution substrate exists to host roles, not define them

`BaseTask` is an execution mechanism. It currently provides:
- asynchronous task execution,
- queue-driven delivery,
- periodic tick scheduling,
- payload lifetime handling,
- isolation between component contexts.

This machinery is powerful and intentional, but it is not the semantic definition of the upper layers. `BaseActivity` and `BaseService` exist so that orchestration and subsystem roles can be expressed above raw task mechanics.

### 8. Crossing a task boundary requires ownership-safe delivery

When a message crosses a task boundary, it must become self-contained at enqueue time. Queue transport does not deep-copy payload memory automatically, so Raven must do that explicitly.

This is not an implementation accident. It is an architectural guarantee:
- the receiver must not depend on sender buffer lifetime,
- payload ownership must remain valid until handling completes,
- inter-task delivery must be memory-safe by construction.

### 9. Business logic executes only in owner context

Meaningful work must execute in the context of the component that owns it. It must not execute:
- in an event callback,
- inside a transport ingress boundary,
- in a foreign task context.

Messages may be delivered from many sources, but the decision logic and subsystem work must run only in the owner’s own execution context.

### 10. State is the single source of truth

If data matters to more than one component, it must not remain a scattered local detail. It belongs in State.

Activities and Services may read from State and contribute updates through controlled ownership paths, but State remains the canonical source of truth for shared system knowledge.

### 11. Activities orchestrate; Services perform

This boundary is fundamental.

- Activities decide what the system should do.
- Services perform subsystem work.
- State stores system truth.
- Adapters connect the core to the outside world.

The architecture loses clarity if an Activity starts doing low-level subsystem execution directly, or if a Service starts making system-wide orchestration decisions.

### 12. Tick-driven behaviour is a first-class part of the model

Tick-based logic is not a workaround. It is part of the execution model.

A system in which an Activity can, for example, terminate manual mode from its own periodic tick while the rest of the runtime continues to operate coherently demonstrates that Raven supports both:
- message-driven behaviour,
- time-driven behaviour.

That matters more than any one navigation example. It shows that the substrate can support future policy loops, supervision, autonomy logic, and sensor-driven behaviour.

### 13. Hardware is only one possible provider

Real hardware matters, but it is not the only legitimate source of system inputs. The lower layers may provide:
- physical sensors,
- simulated sensors,
- mock devices,
- synthetic control streams,
- replayed data.

The behavioural core should remain valid regardless of which provider is active. Only the lowest layer needs to know whether a source is real or synthetic.

### 14. Current subsystems validate the architecture; they do not limit it

The current navigation, telemetry, joystick, and manual-control flows are useful because they let the architecture be exercised quickly. They validate:
- responsiveness,
- stress handling,
- message routing,
- ownership boundaries,
- the interaction between orchestration and execution.

They should not be mistaken for the final conceptual boundary of Raven. They are present examples of the model, not its final scope.

The axioms above explain how the repository should be interpreted. The layer definitions below describe the stable role model used to realise those axioms in the current implementation.

---

## Layers

### Activities

Activities are the high-level orchestration / FSM-like behaviour layer.  
They represent the **mode** the vehicle is currently operating in (boot, idle, manual, autonomous, recovery, fault, …) and own the decision flow for transitioning between those modes.

**Responsibilities**

- Subscribe to system events and react to state changes.
- Decide which services should be active for a given mode.
- Post directed commands to services when a mode transition requires action.
- Read shared state snapshots to make orchestration decisions.
- Own a FreeRTOS task and an inbound queue; all decision logic runs inside that task.

**Must not**

- Directly access hardware drivers or low-level peripheral APIs.
- Mutate shared state directly (that is the controller's job).
- Perform blocking I/O.
- Execute business logic inside an ESP-IDF event callback.

---

### Services

Services are **asynchronous subsystem workers** responsible for ongoing or reactive operational work.

They encapsulate interaction with a single hardware subsystem or software stack (motor driver, IMU, rangefinder, Wi-Fi, telemetry, lighting, audio, …).

**Responsibilities**

- Run continuously inside their own FreeRTOS task.
- Read from hardware / external stacks and write results into shared state.
- Publish broadcast events when something noteworthy happens (sensor threshold crossed, connection status changed, …).
- Receive and execute directed commands posted to their inbound queue.

**Must not**

- Make system-wide mode decisions (that is the activity's job).
- Access another service's internals directly.
- Scatter mutable state across their own fields when that state is consumed by other layers — it belongs in a controller.

---

### Controllers / Shared State

Controllers are **thread-safe canonical state stores** — the single source of truth for any piece of system-wide data.

There is one definitive answer to "what is the current battery level?", "is Wi-Fi connected?", "what is the vehicle's last known pose?" and it lives in a controller, not scattered across service member variables.

**Responsibilities**

- Expose a thread-safe write API for services to push updates.
- Expose a thread-safe snapshot / read API for activities and services to query the current state.
- Protect internal data with mutexes or atomics as appropriate.

**Must not**

- Contain business logic.
- Post events or send commands.
- Own a FreeRTOS task.

---

### Coordinator

The Coordinator is the **root composition and bootstrap layer** — the only place in the system where all other objects are constructed and wired together.

**Responsibilities**

- Initialise the ESP-IDF event loop.
- Construct and own controller instances.
- Construct and own service instances, injecting controller references.
- Construct and own activity instances, injecting controller and service references.
- Register all event subscriptions.
- Start the FreeRTOS tasks for services and activities in the correct order.
- Provide a graceful shutdown path.

**Must not**

- Contain operational logic (it only wires and starts, it does not decide).
- Be called from multiple places; there is exactly one Coordinator, created in `app_main`.

---

### Event Bus

The Event Bus is the **event-driven glue** between layers.  
It is implemented on top of the ESP-IDF system event loop (`esp_event_loop`).

**Responsibilities**

- Deliver typed, broadcast events to all registered subscribers.
- Carry directed commands from one component to another (wrapped as typed command events or posted directly to a target queue — see `docs/event-model.md`).
- Decouple producers from consumers so that new subsystems can be added without modifying existing code.

**Must not**

- Be used to share large mutable data structures — use shared state for that.
- Execute business logic inside a callback — callbacks must forward to the owner task's queue and return immediately.

---

## Architectural Invariants

The following rules are non-negotiable. They protect the coherence of the system as it grows.

1. **Shared state is the single source of truth.**  
   No service or activity maintains its own private copy of data that other layers need. If the data matters to more than one component, it lives in a controller.

2. **Events communicate facts; commands communicate directed intent.**  
   An event says "this happened" (obstacle detected, battery low, Wi-Fi connected). A command says "you specifically should do this" (stop motors, start telemetry stream). Do not conflate the two.

3. **Activities orchestrate; services perform subsystem work.**  
   An activity never drives hardware directly. A service never decides system-wide mode transitions.

4. **Event callbacks must remain thin.**  
   The ESP-IDF event loop delivers callbacks on its own internal task. Any non-trivial work must be forwarded to the receiving component's own queue and processed in that component's task context.

5. **Business logic executes in owner task context, not inside event handlers.**  
   This prevents priority inversion, stack overflows on the event-loop task, and subtle re-entrancy bugs.

6. **Not every class or subsystem needs its own FreeRTOS task.**  
   Tasks add memory overhead and scheduling complexity. A FreeRTOS task is justified when a component genuinely needs concurrent, independent execution (blocking I/O, periodic sampling, queue-driven event processing). Controllers, for example, never need their own task.

7. **Prefer a clean foundation over premature framework complexity.**  
   Avoid generic DI containers, runtime service registries, or heavy template abstractions at this stage. Clarity and explicit wiring are more valuable than flexibility that is not yet needed.

---

## Layer Interaction Summary

```
┌──────────────────────────────────────────────────────────┐
│                        Activity                          │
│  (FSM / orchestration — runs in its own FreeRTOS task)   │
│  reads state snapshots · posts commands · reacts to events│
└───────────────┬─────────────────────────┬────────────────┘
                │ directed commands        │ event subscriptions
                ▼                          ▼
┌──────────────────────────┐   ┌──────────────────────────┐
│         Service          │   │        Event Bus          │
│  (subsystem worker task) │──▶│  (esp_event_loop glue)    │
│  writes state · publishes│   │  broadcast facts          │
│  events · handles cmds   │   │  thin callbacks only      │
└───────────┬──────────────┘   └──────────────────────────┘
            │ writes / reads
            ▼
┌──────────────────────────┐
│  Controller / Shared State│
│  (thread-safe, no task)   │
│  single source of truth   │
└──────────────────────────┘

All objects constructed and wired by: Coordinator (app_main)
```

---

