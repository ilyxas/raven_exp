1. EventBus vs direct queue
    → architectural rule already exists
    → callbacks forward into owner queue
    → remaining risk is enforcement

2. State/event consistency
    → update-before-publish defined
    → FSM gating closes invalid concurrent flow
    → snapshots are current-valid, not historical snapshots

3. Message contract governance
    → real open issue
    → local-per-component organization exists
    → global contract governance not solved yet

4. Payload ownership/lifetime
    → deep-copy semantics explicitly defined
    → BaseTask frees after handling
    → remaining open area = failure paths/contracts

5. Platform independence leakage
    → mostly generic concern
    → accidental real hit in implementation
    → requirement already exists architecturally

6. Coordinator god-object
    → already strongly constrained in document
    → only future discipline risk

7. Tick + message interaction
    → not a bug
    → intentional owner-context scheduled sampling model
    → should become explicit architecture pattern