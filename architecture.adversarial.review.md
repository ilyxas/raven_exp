# Raven — Adversarial Architecture Review

This review is written after reading the original architecture document, the prior review, and the author's responses to that review. It does not repeat findings the author has fully closed. Its goal is to identify realistic runtime failure scenarios that the architecture, as written, still leaves open or underspecified.

---

## Scenario 1: Queue-full drops a safety-critical command silently

**Setup.** A Service is under sustained load — its inbound queue is at capacity because a high-frequency tick loop is generating state-update messages faster than the task drains them.

**Trigger.** An Activity posts a "stop motors" directed command to that Service's queue at the moment the queue is full.

**What happens.** The queue post either fails immediately (non-blocking post) or blocks the Activity's task (blocking post). Neither outcome is specified in the architecture. If non-blocking, the command is silently dropped. If blocking, the Activity's own task is stalled — which stops it from processing its own inbound queue, including other events that may be time-critical.

**Why it survives the prior responses.** The author's responses acknowledge that failure paths for payload ownership remain open (finding 4). Dead-letter and rejection policy for commands are listed in the reviewer's "required operational clarifications" section but the responses do not address them. The architecture states routing is a "strict admission policy" but says nothing about what happens when the transport boundary itself is temporarily unavailable.

**Residual risk.** A motor-stop command during an emergency can be silently discarded with no retry, no error event, and no observable symptom until the vehicle fails to stop.

---

## Scenario 2: Event published before paired state write completes under concurrent writers

**Setup.** Two Services independently observe the same external condition within a short window — for example, both an IMU service and a proximity sensor service detect a potential collision event.

**Trigger.** Service A writes its collision flag to the Controller and publishes a collision event. Concurrently, Service B is mid-write to a related field in the same Controller.

**What happens.** The Activity receives Service A's collision event and reads a state snapshot. That snapshot may contain Service B's field in a partially-written or pre-update state, because Service B's mutex-protected write has not completed. The Activity makes a mode decision on a snapshot that is internally inconsistent across fields — some reflecting the collision, some not.

**Why it survives the prior responses.** The author's response to finding 2 states "update-before-publish defined" and "FSM gating closes invalid concurrent flow." This addresses single-writer ordering. It does not address multi-writer field-level interleaving within a single snapshot. A mutex on the Controller protects individual writes but does not guarantee that a snapshot read across multiple fields sees all writers' updates atomically.

**Residual risk.** Mode transitions triggered by multi-source events can be made on incoherent state snapshots, producing incorrect or irreversible mode decisions.

---

## Scenario 3: Heap exhaustion during deep-copy breaks inter-task delivery silently

**Setup.** The system is running under sustained telemetry and joystick load. Heap is moderately fragmented from earlier allocation and free cycles.

**Trigger.** A cross-task message is dispatched. `BaseTask` attempts to deep-copy the payload before enqueuing it, as required by architectural axiom 8. The heap allocation for the copy fails.

**What happens.** The architecture does not specify the failure contract for deep-copy allocation failures. The sender does not know the message was not delivered. The receiver never sees it. There is no event, no dead-letter path, and no observability hook.

**Why it survives the prior responses.** Finding 4's response confirms deep-copy semantics are defined and BaseTask frees after handling. The explicitly acknowledged remaining open area is "failure paths/contracts." This is exactly that failure path — allocation failure — and it is unaddressed.

**Residual risk.** At memory pressure, the system silently loses inter-task messages — including directed commands — with no diagnostic signal. The failure mode is invisible until behavior diverges.

---

## Scenario 4: Startup event fired before subscriber is registered

**Setup.** The Coordinator constructs objects and registers subscriptions in sequence. Event loop is initialized early. Some Service publishes a "ready" or "connected" fact event as part of its own task startup, which races with the Coordinator's subscription registration loop.

**Trigger.** The Service's task begins and publishes its event before the Activity that subscribes to it has had its subscription registered by the Coordinator.

**What happens.** The event is delivered to zero subscribers and is discarded by the event bus. The Activity never learns the Service is ready. The Activity may wait indefinitely for a fact that already occurred, or may start in an incorrect mode assumption.

**Why it survives the prior responses.** Finding 8 addresses late registration and startup races as "fragility" and the author's responses do not mention finding 8 at all — it is absent from the response document. The Coordinator is required to start tasks "in the correct order" but the architecture does not define what correct order means with respect to subscription readiness, nor does it require Services to gate their initial publications until subscriptions are confirmed.

**Residual risk.** Startup-phase events are structurally unreliable. Any Service that announces its ready state at task start may be announcing to no one, causing Activities to boot into stale or incorrect mode assumptions.

---

## Scenario 5: Controller mutex held across an event publish creates a deadlock

**Setup.** A Service holds a Controller's write mutex while performing a multi-field update. As part of a post-write notification pattern, it publishes an event while still holding the mutex (or the event publish is made by an intermediate helper that has not released the lock).

**Trigger.** The event loop delivers the published event synchronously (or semi-synchronously) to a callback. That callback, following the thin-callback rule, forwards the message to another component's queue. That other component, when it processes the message, reads a Controller snapshot — which requires acquiring the same Controller mutex.

**What happens.** If the event dispatch path is synchronous from the perspective of the publishing task, the mutex is held on the stack above the enqueue call. The consuming component's read then deadlocks against the still-held write lock.

**Why it survives the prior responses.** The architecture defines that callbacks must be thin and forward immediately, and that business logic runs in owner context. However, it does not specify whether the event bus dispatch is synchronous or asynchronous relative to the publishing caller's stack, nor does it require that Controller locks be released before any event is published. The "update-before-publish" ordering guarantee (finding 2 response) assumes this ordering is safe but does not address lock-scope discipline during publication.

**Residual risk.** A Service that publishes an event while holding a Controller write lock can deadlock any downstream component that reads the same Controller in response to that event — a pattern the architecture explicitly encourages.

---

## Scenario 6: Tick starvation under queue backlog causes time-sensitive policy to miss its window

**Setup.** A component uses both message-driven and tick-driven execution. Its task loop processes inbound queue messages and also runs a periodic tick handler. The queue fills with a burst of inbound messages.

**Trigger.** The task drains its queue for an extended period. During this time, the tick callback is due but is not serviced because the task is consuming queue messages.

**What happens.** The tick-driven policy — for example, a manual mode timeout that should terminate manual control after N seconds of inactivity — fires late or not at all during the burst. The system remains in manual mode longer than the policy intends. If the burst is sustained (joystick stream at maximum rate), the tick may never fire.

**Why it survives the prior responses.** Finding 7 is addressed with "intentional owner-context scheduled sampling model — should become explicit architecture pattern." This acknowledges the interaction exists but defers resolution. The author's response confirms this is not yet an explicit pattern. Under sustained queue load, the sampling model breaks: the tick does not fire at its intended interval, and the architecture provides no specification for how task loops must interleave queue drain and tick dispatch to bound tick latency.

**Residual risk.** Time-sensitive policies that rely on tick-driven logic — including safety timeouts, mode expiry, and watchdog-style checks — can be silently delayed or starved by queue backlog, with no architectural mechanism to detect or bound the delay.

---

## Scenario 7: Message contract drift causes silent misinterpretation at runtime

**Setup.** A Service evolves its state update payload schema — adding a field, changing a field's type, or reordering packed fields — without a coordinated contract update to the consuming Activity.

**Trigger.** The updated Service begins publishing messages with the new payload shape. The consuming Activity's decoder, which was written against the old contract, continues to deserialize the payload using the old field layout.

**What happens.** The Activity silently reads incorrect values from the fields it believes it understands. No error is raised. The architecture's "strict admission policy" validates that a route and decoder exist, but the decoder itself is now operating against a mismatched contract. Mode decisions made from these fields are wrong.

**Why it survives the prior responses.** Finding 3 is explicitly acknowledged as a "real open issue" in the author's responses. The response states "global contract governance not solved yet." The adversarial addition here is that this is not only a governance gap — it is a silent runtime failure. The architecture's validation boundary (known message + contract + decoder) assumes contracts are correct, but has no mechanism to detect contract drift between a decoder's expectation and the actual payload at runtime.

**Residual risk.** Any schema evolution in any component can silently corrupt the values read by all consuming components, with the system continuing to operate and make decisions on corrupt data. The failure produces no error signal.

---

## Scenario 8: Coordinator teardown destroys a Controller while a Service task is mid-read

**Setup.** Shutdown is initiated. The Coordinator begins destroying objects. It destroys a Controller instance.

**Trigger.** A Service task — which has not yet received or processed its shutdown signal — is concurrently performing a Controller read for its next periodic state update.

**What happens.** The Controller object is destroyed (mutex, internal data, and all) while the Service task is inside its read path. The Service dereferences a dangling pointer. On ESP32/FreeRTOS, this is a silent memory corruption or hard fault depending on heap behavior.

**Why it survives the prior responses.** The architecture requires the Coordinator to "provide a graceful shutdown path" but does not specify the shutdown sequencing contract. The document does not require Service tasks to be fully halted before their dependency Controllers are destroyed. Finding 8 in the review (routing lifecycle/dynamics) was not addressed in the response document at all. Shutdown ordering across task contexts is a structurally different problem from startup ordering — it requires confirmed task termination, not just sequenced construction.

**Residual risk.** Any shutdown sequence that destroys shared state before confirming all consumer tasks have exited produces use-after-free. The architecture names the Coordinator as the lifecycle owner but does not define the safety contract for teardown.

---

## Scenario 9: Two Activities simultaneously attempt a conflicting mode transition

**Setup.** The system supports multiple Activities. Two Activities independently observe events that each legitimately trigger a mode transition — for example, one observes a low-battery event and wants to transition to recovery mode, while another observes a navigation goal completion and wants to transition to idle mode.

**Trigger.** Both Activities evaluate their transition logic concurrently in their respective task contexts. Both conclude a transition is valid. Both post commands or update mode-related state.

**What happens.** The architecture defines Activities as owning mode decisions, but does not define arbitration between concurrent Activity decisions. State writes are individually mutex-protected but there is no compare-and-swap or optimistic locking semantic for mode transitions. One Activity's transition may overwrite the other's, or both may partially apply.

**Why it survives the prior responses.** The author's response to finding 2 states "FSM gating closes invalid concurrent flow." This implies a single Activity gating transitions through its own FSM. It does not address multi-Activity arbitration — what happens when two valid Activities issue conflicting transitions simultaneously. The architecture describes Activities as plural ("the mode the vehicle is currently in") without specifying how multiple concurrent orchestrators are reconciled.

**Residual risk.** Multi-Activity systems have no specified arbitration protocol. Under concurrent legitimate triggers, the resulting mode can be whichever transition won the last write — a non-deterministic outcome with no error signal.

---

## Scenario 10: Event bus callback blocks on a full owner queue, stalling the event loop task

**Setup.** A component's inbound queue is full. An event arrives on the event bus.

**Trigger.** The event callback — which is executing on the event loop's internal task — attempts to forward the event to the owner component's queue using a blocking enqueue call.

**What happens.** The event loop task blocks. While it is blocked, no other event callbacks in the system are delivered. Every other component that relies on the event bus for commands, facts, or state notifications is frozen. If the owner queue does not drain quickly (e.g., because the owner task is also waiting on a downstream resource), the entire event bus is stalled for an unbounded period.

**Why it survives the prior responses.** The architecture explicitly requires callbacks to be thin and to forward immediately. However, it does not specify whether the forward enqueue must be non-blocking. If contributors use a blocking queue post inside a callback — which is a natural implementation choice — the architectural rule ("return immediately") is violated but there is no enforcement mechanism. The responses do not address this specific failure mode. Finding 1's response says "callbacks forward into owner queue" and "remaining risk is enforcement," which confirms this scenario is in the known-but-unmitigated category.

**Residual risk.** A single saturated component queue can freeze the entire system's event bus, including delivery of safety-critical events to all other components, with no timeout, circuit-breaker, or fallback path specified.
