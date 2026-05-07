# Raven — Implementation-Ready Plan

Derived from [`architecture.md`](../architecture.md).  
This document translates the architectural vision into a concrete, actionable implementation plan.

---

## 1. Requirements

### Functional Requirements

| ID | Requirement |
|----|-------------|
| FR-01 | The system shall support multiple operating modes: boot, idle, manual, autonomous, recovery, and fault. |
| FR-02 | The system shall accept input from TCP, RF control, mock/simulated sources, and internal generators. |
| FR-03 | Navigation data, telemetry, joystick input, and manual-control flows shall be fully operational. |
| FR-04 | All shared state (battery level, Wi-Fi status, vehicle pose, etc.) shall be accessible through a single canonical store. |
| FR-05 | The system shall support both message-driven and tick-driven (periodic) execution. |
| FR-06 | Hardware sensors may be substituted with simulated, mock, or replayed data without changing behavioural-core logic. |
| FR-07 | Events shall broadcast facts; directed commands shall target a specific component. |
| FR-08 | The system shall provide a graceful shutdown path. |

### Non-Functional Requirements

| ID | Requirement |
|----|-------------|
| NFR-01 | The behavioural core (Activities, Services, State) must compile and run on non-ESP32 hosts (desktop, simulator). |
| NFR-02 | All inter-task message delivery must be memory-safe: payloads must be self-contained at enqueue time. |
| NFR-03 | Business logic must execute only in the owning component's task context (no logic in event callbacks). |
| NFR-04 | Event callbacks must be thin — forward to queue and return immediately. |
| NFR-05 | FreeRTOS tasks are only created when genuinely needed (concurrent/blocking work, periodic sampling). |
| NFR-06 | No generic DI containers or heavy template abstractions; prefer explicit wiring for clarity. |

---

## 2. Core System Responsibilities

| Layer | Core Responsibility |
|-------|---------------------|
| **Activity** | Owns orchestration and mode-transition FSM logic; decides what the system should do. |
| **Service** | Performs subsystem work (hardware interaction, sensor reading, actuator commands). |
| **Controller / State** | Thread-safe single source of truth for all shared system data; no task, no business logic. |
| **Coordinator** | Root composition point: constructs, injects, wires, and starts all other components. |
| **Event Bus** | Decoupled broadcast and directed-command delivery via `esp_event_loop`; callbacks stay thin. |
| **Transport Adapters** | Translate external wire formats into internal semantic contracts; do not bleed into business logic. |

---

## 3. Main Components

### 3.1 Activities

- `BootActivity` — runs on startup, performs hardware checks, transitions to Idle.
- `IdleActivity` — waiting state; monitors for activation signals.
- `ManualActivity` — processes joystick/RF commands; owns manual-mode lifecycle including tick-based timeout.
- `AutonomousActivity` — future: policy loops, path following, sensor-driven decision making.
- `RecoveryActivity` — future: handles fault recovery procedures.
- `FaultActivity` — terminal or degraded state; stops unsafe subsystems.

Each activity:
- Extends `BaseActivity` (which extends `BaseTask`).
- Owns one FreeRTOS task and one inbound queue.
- Reacts to event-bus events and controller state snapshots.
- Posts directed commands to services.

### 3.2 Services

- `NavigationService` — processes navigation commands and publishes pose/route updates.
- `TelemetryService` — collects and streams system telemetry.
- `JoystickService` — reads joystick/RF input and publishes normalised input events.
- `MotorService` — future: drives motor hardware from directed commands.
- `IMUService` — future: reads inertial sensor, writes pose data to state.
- `WifiService` — future: manages Wi-Fi connection lifecycle.

Each service:
- Extends `BaseService` (which extends `BaseTask`).
- Runs in its own FreeRTOS task.
- Writes results to controllers; publishes broadcast events; handles directed commands.

### 3.3 Controllers (Shared State)

- `NavigationController` — last known pose, active route, navigation status.
- `TelemetryController` — system health metrics, sensor readings.
- `InputController` — latest normalised joystick/RF input snapshot.
- `SystemController` — Wi-Fi status, battery level, connection flags.

Each controller:
- Holds data behind a mutex or atomics.
- Exposes a write API (called by services) and a snapshot/read API (called by activities and services).
- Has no FreeRTOS task.

### 3.4 Coordinator

- Single instance, created in `app_main`.
- Constructs all controllers, services, and activities in dependency order.
- Injects controller references into services; injects controller and service references into activities.
- Registers all event-bus subscriptions.
- Starts FreeRTOS tasks in the correct order.
- Provides `shutdown()`.

### 3.5 Event Bus

- Wraps `esp_event_loop`.
- Supports typed broadcast events and typed directed-command delivery.
- All callbacks forward to owner queue and return immediately.

### 3.6 Transport Adapters

- `TcpAdapter` — accepts TCP frames, validates, decodes into internal events/commands.
- `RfAdapter` — accepts RF frames, decodes into joystick/control events.
- `MockAdapter` / `SimAdapter` — injects synthetic inputs for simulation and testing.

---

## 4. Proposed File/Module Structure

```
raven/
├── main/
│   └── app_main.cpp              # Entry point; creates and starts Coordinator
│
├── core/
│   ├── base/
│   │   ├── base_task.hpp/.cpp    # FreeRTOS task + queue + tick machinery
│   │   ├── base_activity.hpp/.cpp
│   │   └── base_service.hpp/.cpp
│   │
│   ├── event_bus/
│   │   ├── event_bus.hpp/.cpp    # esp_event_loop wrapper
│   │   └── event_ids.hpp         # Typed event/command IDs and payload contracts
│   │
│   └── coordinator/
│       └── coordinator.hpp/.cpp  # Root composition and bootstrap
│
├── activities/
│   ├── boot_activity.hpp/.cpp
│   ├── idle_activity.hpp/.cpp
│   ├── manual_activity.hpp/.cpp
│   ├── autonomous_activity.hpp/.cpp   # stub / future
│   ├── recovery_activity.hpp/.cpp     # stub / future
│   └── fault_activity.hpp/.cpp
│
├── services/
│   ├── navigation_service.hpp/.cpp
│   ├── telemetry_service.hpp/.cpp
│   ├── joystick_service.hpp/.cpp
│   ├── motor_service.hpp/.cpp         # stub / future
│   ├── imu_service.hpp/.cpp           # stub / future
│   └── wifi_service.hpp/.cpp          # stub / future
│
├── controllers/
│   ├── navigation_controller.hpp/.cpp
│   ├── telemetry_controller.hpp/.cpp
│   ├── input_controller.hpp/.cpp
│   └── system_controller.hpp/.cpp
│
├── adapters/
│   ├── tcp_adapter.hpp/.cpp
│   ├── rf_adapter.hpp/.cpp
│   ├── mock_adapter.hpp/.cpp
│   └── sim_adapter.hpp/.cpp
│
├── platform/
│   ├── freertos/                  # FreeRTOS-specific shims (task, queue, mutex wrappers)
│   └── host/                      # Desktop/sim shims for non-ESP builds
│
└── docs/
    ├── architecture.md
    └── implementation-ready-plan.md
```

---

## 5. Execution Flow

### 5.1 System Boot

```
app_main
  └─► Coordinator::init()
        ├── esp_event_loop_create_default()
        ├── construct controllers (NavigationController, TelemetryController, …)
        ├── construct services (NavigationService, TelemetryService, …)
        │     └── inject controller references
        ├── construct activities (BootActivity, IdleActivity, ManualActivity, …)
        │     └── inject controller + service references
        ├── construct and start transport adapters (TcpAdapter, RfAdapter, …)
        ├── register all event-bus subscriptions
        └── start FreeRTOS tasks (services first, then activities)
              └─► BootActivity::onStart() → transitions to IdleActivity
```

### 5.2 Manual Mode — Message-Driven Path

```
External Input (TCP / RF)
  └─► TransportAdapter::onFrame()
        └── validate + decode → JoystickInputEvent
              └─► EventBus::post(JoystickInputEvent)
                    └─► ManualActivity::onEvent()  [thin callback]
                          └── enqueue to ManualActivity task queue
                                └─► ManualActivity::processEvent()  [owner context]
                                      ├── read InputController snapshot
                                      ├── post directed command → NavigationService queue
                                      └── NavigationService::handleCommand()
                                            ├── execute navigation logic
                                            └── write result → NavigationController
```

### 5.3 Manual Mode — Tick-Driven Path

```
ManualActivity FreeRTOS task tick (periodic)
  └─► ManualActivity::onTick()  [owner context]
        ├── check elapsed time since last joystick input
        ├── if timeout exceeded → post ModeTransitionEvent(Idle)
        └── EventBus delivers to IdleActivity
```

### 5.4 State Read by Any Component

```
AnyActivity / AnyService
  └─► NavigationController::getSnapshot()
        └── mutex lock → copy data → mutex unlock → return snapshot
```

---

## 6. Task Breakdown

### Phase 1 — Foundation (Core infrastructure)

| Task | Description |
|------|-------------|
| T-01 | Implement `BaseTask`: FreeRTOS task lifecycle, inbound queue, periodic tick. |
| T-02 | Implement `BaseActivity` and `BaseService` extending `BaseTask` with role-specific hooks. |
| T-03 | Implement `EventBus` wrapping `esp_event_loop`; define event/command ID registry. |
| T-04 | Define payload ownership and deep-copy protocol for inter-task delivery. |
| T-05 | Implement `Coordinator` with construction order, injection, and startup sequence. |

### Phase 2 — Controllers (Shared State)

| Task | Description |
|------|-------------|
| T-06 | Implement `NavigationController` with mutex-protected pose and route state. |
| T-07 | Implement `TelemetryController` with health-metric state. |
| T-08 | Implement `InputController` with latest joystick snapshot. |
| T-09 | Implement `SystemController` (Wi-Fi, battery, connection flags). |

### Phase 3 — Services (Subsystem Workers)

| Task | Description |
|------|-------------|
| T-10 | Implement `JoystickService`: decode raw input, write to `InputController`, publish events. |
| T-11 | Implement `NavigationService`: handle directed commands, update `NavigationController`. |
| T-12 | Implement `TelemetryService`: collect metrics, write to `TelemetryController`, stream output. |

### Phase 4 — Activities (Orchestration)

| Task | Description |
|------|-------------|
| T-13 | Implement `BootActivity`: hardware checks, transition to Idle. |
| T-14 | Implement `IdleActivity`: wait for activation, monitor events. |
| T-15 | Implement `ManualActivity`: joystick command forwarding, tick-based timeout. |
| T-16 | Implement `FaultActivity`: halt unsafe subsystems, surface diagnostics. |

### Phase 5 — Transport Adapters

| Task | Description |
|------|-------------|
| T-17 | Implement `TcpAdapter`: frame validation, decode to internal events/commands. |
| T-18 | Implement `RfAdapter`: RF frame decode to joystick events. |
| T-19 | Implement `MockAdapter`/`SimAdapter` for simulation and testing. |

### Phase 6 — Platform Portability

| Task | Description |
|------|-------------|
| T-20 | Extract FreeRTOS-specific primitives into `platform/freertos/` shims. |
| T-21 | Implement `platform/host/` shims for desktop/simulation builds. |
| T-22 | Validate that the behavioural core compiles and runs on a host target. |

### Phase 7 — Future Capabilities (Stubs)

| Task | Description |
|------|-------------|
| T-23 | Stub `AutonomousActivity` and `RecoveryActivity` with placeholder state transitions. |
| T-24 | Stub `MotorService`, `IMUService`, `WifiService` with interface contracts. |

---

## 7. Risks and Unclear Assumptions

| ID | Risk / Assumption | Severity | Notes |
|----|-------------------|----------|-------|
| R-01 | **Payload deep-copy discipline** — if any producer fails to copy heap data before enqueue, the receiver will access freed memory. | High | Requires a consistent copy helper or ownership-transferring smart pointer pattern enforced across all inter-task sends. |
| R-02 | **Callback thinness** — developers may accidentally put logic in event callbacks, violating NFR-03/NFR-04. | High | Needs linting rules or code-review checklist items; not enforceable by the compiler alone. |
| R-03 | **`esp_event_loop` task priority** — the event-loop task runs at a fixed priority; heavy callback forwarding could cause starvation. | Medium | Queue sizes and task priorities need profiling under realistic load. |
| R-04 | **Host-platform build** — `esp_event_loop` is ESP-IDF-specific; platform shims for desktop builds are not yet defined. | Medium | Must be resolved before any CI simulation target is added. |
| R-05 | **Coordinator ordering** — incorrect construction order (e.g. an activity started before its dependent service) will cause null-reference crashes at runtime. | Medium | Dependency order must be documented and enforced in `Coordinator`. |
| R-06 | **Future autonomy complexity** — richer sensor-driven navigation is mentioned but not scoped; adding it later may require controller schema changes that break existing consumers. | Medium | Define controller interfaces with versioning in mind. |
| R-07 | **RF transport specifics** — the RF adapter protocol, frame format, and error-handling behaviour are not yet defined. | Low–Medium | Needs protocol specification before T-18. |
| R-08 | **Mock/Sim injection point** — the exact seam at which synthetic inputs replace real hardware inputs is unspecified (adapter level vs. service level vs. platform level). | Low–Medium | Should be decided before T-19. |
| R-09 | **FreeRTOS task stack sizes** — stack requirements per component are unknown; defaults may cause stack overflows under load. | Low | Needs measurement or configurable stack size parameters per component. |

---

## 8. Questions for Human Approval

The following questions must be answered before or during Phase 1 to avoid costly rework:

1. **Target hardware and SDK version** — Which ESP32 variant (ESP32, ESP32-S3, ESP32-C3, …) and which ESP-IDF version should be treated as the primary build target?

2. **Host-build requirement timeline** — Is desktop/simulator portability (NFR-01) required for the first milestone, or is it a future concern? This determines whether platform shims must be built in Phase 1 or deferred.

3. **Payload ownership strategy** — Should inter-task payloads use explicit deep-copy helpers, a custom arena allocator, or `std::shared_ptr` with a custom deleter? The answer affects `BaseTask` design.

4. **Event-bus directed commands vs. direct queue post** — The architecture mentions both "typed command events via the event bus" and "posting directly to a target queue". Which mechanism should be the canonical approach for directed commands?

5. **Mode-transition ownership** — Which component is authoritative for triggering mode transitions: the Activity itself (on tick/timeout), the EventBus (on a broadcast fact), or a dedicated mode-manager controller?

6. **Controller concurrency model** — Should controllers use `std::mutex` + `std::lock_guard`, FreeRTOS mutexes, or C++ atomics? Consistency across all controllers is important for correctness.

7. **RF protocol specification** — What is the frame format, baud rate, error-correction scheme, and expected message rate for the RF transport? This is a blocker for T-18.

8. **Simulation injection seam** — At which layer should simulated/mock inputs be injected? (Adapter boundary, service boundary, or platform HAL?) This defines the interface contract for T-19–T-22.

9. **Testing strategy** — Should unit tests run on host (native GCC build) or on-device (Unity on ESP32)? This drives the platform-portability priority and mock-adapter design.

10. **Scope of first deliverable** — Which activities and services must be functional for the first integration milestone? (E.g., Manual mode end-to-end with TCP input, NavigationService, and NavigationController?)
