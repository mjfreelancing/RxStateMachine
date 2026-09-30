# Example Use Cases

These scenarios span the domains consumers actually build. Each maps to concrete features. They are
the **definition of "most, if not all, business requirements"** — if a new requirement looks like one
of these, the framework covers it. Feature briefs in [features/](features/) cite these by ID.

| #     | Use case                                 | Domain              | Key framework features exercised                                                                                                             |
| ----- | ---------------------------------------- | ------------------- | -------------------------------------------------------------------------------------------------------------------------------------------- |
| UC-1  | **Order / checkout lifecycle**           | E‑commerce          | Basic permits, guards, entry actions, state observable → progress UI, persist-on-change, cancel paths, Mermaid/D2 diagram export (see [diagrams.md](diagrams.md)) |
| UC-2  | **Payment & refund processing**          | Payments / fintech  | Async entry actions (gateway call), guarded transitions, timeout → auto-cancel, parameterized payload (receipt), error stream                |
| UC-3  | **Approval / document workflow**         | Content / HR        | Guard "N approvals collected", escalation via timer trigger, `IsInState` for draft-of-published, history                                     |
| UC-4  | **Device lifecycle & provisioning**      | IoT                 | Connection states, heartbeat timeouts (observable triggers), reconnection with backoff (timer), telemetry throttling via Rx                  |
| UC-5  | **Socket / session connection FSM**      | Networking          | Reentrant transitions (reconnect), `InternalTransition` (heartbeat ping), queued firing under load, thread-safety                            |
| UC-6  | **Background job / pipeline processing** | DevOps / data       | Retry-with-backoff (timer triggers), max-retry guard, async actions (worker calls), permitted-triggers UI                                    |
| UC-7  | **UI wizard / multi-step form**          | Web/desktop         | State observable drives step rendering; guard enables "Next"; `OnEntryFrom` runs per-step logic; parameterized payload                       |
| UC-8  | **Saga orchestration**                   | Distributed systems | External state storage + snapshot persistence, async actions, error policies, timeouts; survives restarts                                    |
| UC-9  | **Game / player state machine**          | Games               | High-frequency observable input, debounce/throttle, immediate firing, testable with virtual time                                             |
| UC-10 | **Media player / playback control**      | Media               | Parameterized transitions (seek position), internal transitions (volume), guarded transitions (buffered?)                                    |

Code on this page is illustrative; API names are settled at planning time.

## Two detailed examples

**UC-2 — Payment processing** shows async + timeout + persistence:

```csharp
var machine = new StateMachine<PaymentState, PaymentTrigger>(PaymentState.Initiated);

machine.Configure(PaymentState.Initiated)
    .Permit(PaymentTrigger.Begin, PaymentState.Authorizing);

machine.Configure(PaymentState.Authorizing)
    .OnEntryAsync(ct => AuthorizeAsync(ct))                        // async side effect on entry
    .Permit(PaymentTrigger.Authorized, PaymentState.Authorized)
    .Permit(PaymentTrigger.Failed, PaymentState.Failed)
    .TimeoutAfter(TimeSpan.FromSeconds(30), PaymentTrigger.Failed); // state-scoped: cancelled on leaving Authorizing

machine.Configure(PaymentState.Authorized)
    .Permit(PaymentTrigger.Capture, PaymentState.Captured)
    .Permit(PaymentTrigger.Refund, PaymentState.Refunded);

// Telemetry + persistence are just observers
machine.Subscribe(state => _metrics.StateChanged(state));
machine.Subscribe(state => _repo.Save(transactionId, state));
```

The example follows the library's own rules: async work uses an async entry action and the timeout is
scoped to the state (see [design-decisions.md](design-decisions.md), DD-01 and DD-17).

**UC-5 — Connection FSM** shows reentry, internal transitions, and thread-safe queued firing:

```csharp
machine.Configure(ConnectionState.Connecting)
    .Permit(ConnectionTrigger.Connected, ConnectionState.Open)
    .Permit(ConnectionTrigger.Failed, ConnectionState.Retrying);

machine.Configure(ConnectionState.Open)
    .InternalTransition(ConnectionTrigger.Heartbeat, _ => _lastSeen = DateTimeOffset.UtcNow) // no exit/entry
    .Permit(ConnectionTrigger.Lost, ConnectionState.Retrying);

machine.Configure(ConnectionState.Retrying)
    .PermitReentry(ConnectionTrigger.Retry)               // re-run entry (backoff reset)
    .Permit(ConnectionTrigger.Connected, ConnectionState.Open);
```
