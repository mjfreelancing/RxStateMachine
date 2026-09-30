# API and Package Sketch (illustrative, non-normative)

Everything on this page is a sketch to convey shape. Names, signatures, and layout are decided at
planning time and may change. Where this page and a feature brief or
[design-decisions.md](design-decisions.md) disagree, those win.

## Package & namespaces

```
RxStateMachine/                            (library)
├── StateMachine{TState,TTrigger}.cs       (implements IObservable<TState> + observer of triggers)
├── StateMachineConfiguration.cs           (fluent builder; Configure(...).Permit(...))
├── Transition.cs                          (record: Source, Destination, Trigger, Payload, Kind, Timestamp)
├── StateChange.cs                         (record: Previous, Current, Timestamp, Kind)
├── GuardResult.cs
├── PermittedTriggers.cs
├── StateMachineError.cs
├── TriggerWithParameters.cs
├── StateMachineOptions.cs                 (Scheduler, FiringMode, UnhandledTriggerPolicy, ExceptionPolicy)
├── Reactive/
│   ├── ObservableStateMachineExtensions.cs (DistinctUntilChanged-style helpers, timer helpers)
│   └── StateMachineObserver.cs            (optional convenience base for lifecycle observers)
├── Persistence/        IStateMachinePersistence, SnapshotPersistence
├── Diagnostics/        MermaidFormatter, D2Formatter, TextReport
└── Validation/         StateMachineValidator (fail-fast config checks)
```

## Core type sketch

The trigger-input surface follows [design-decisions.md](design-decisions.md) DD-02: plain triggers
and payload-carrying triggers are both accepted and behave identically to `Fire`.

```csharp
public sealed class StateMachine<TState, TTrigger> : IObservable<TState>, IObserver<TriggerWithParameters<TState, TTrigger>>
    where TState : notnull
    where TTrigger : notnull
{
    public StateMachine(TState initialState, StateMachineOptions? options = null);
    public StateMachine(Func<TState> accessor, Action<TState> mutator, StateMachineOptions? options = null); // external state storage

    public StateMachineConfiguration<TState, TTrigger> Configure(TState state); // fluent

    public TState State { get; }
    public IObservable<TState> States { get; }                       // == this
    public IObservable<StateChange<TState>> StateChanges { get; }
    public IObservable<Transition<TState, TTrigger>> Transitions { get; }
    public IObservable<Transition<TState, TTrigger>> TransitionsCompleted { get; }
    public IObservable<GuardResult<TState, TTrigger>> GuardResults { get; }
    public IObservable<PermittedTriggers<TState, TTrigger>> PermittedTriggers { get; }
    public IObservable<StateMachineError> Errors { get; }

    public Transition<TState, TTrigger> Fire(TTrigger trigger);
    public Transition<TState, TTrigger> Fire<TArg>(TriggerWithParameters<TTrigger, TArg> trigger, TArg arg);
    public Task<Transition<TState, TTrigger>> FireAsync(TTrigger trigger, CancellationToken ct = default);
    public Task<Transition<TState, TTrigger>> FireAsync<TArg>(TriggerWithParameters<TTrigger, TArg> trigger, TArg arg, CancellationToken ct = default);

    public IEnumerable<TTrigger> GetPermittedTriggers();
    public Task<IEnumerable<TTrigger>> GetPermittedTriggersAsync(CancellationToken ct = default);
    public StateMachineInfo GetInfo();
    public bool IsInState(TState state);                            // honors hierarchy

    // Observer input (hybrid) — lets any observable drive the machine
    void IObserver<TriggerWithParameters<TState, TTrigger>>.OnNext(TriggerWithParameters<TState, TTrigger> value);
}
```

Known inconsistencies in this sketch, to be resolved at planning time: the observer type for plain
triggers versus payload wrappers (DD-02), and the two different generic shapes used for
`TriggerWithParameters`.

## Configuration surface

```csharp
machine.Configure(OrderState.Draft)
    .Permit(OrderTrigger.Submit, OrderState.Submitted)
    .InternalTransition(OrderTrigger.Touch, _ => OnTouch())
    .OnExit(_ => NotifyLeaving());

machine.Configure(OrderState.Submitted)
    .PermitIf(OrderTrigger.Pay, OrderState.Paid, () => _payment.Succeeded, "payment confirmed")
    .PermitIfAsync(OrderTrigger.Ship, OrderState.Shipping, p => _stock.ReserveAsync(p), "stock reserved")
    .OnEntryFrom(OrderTrigger.Pay, _ => StartShippingTimer())
    .PermitReentry(OrderTrigger.RefreshStatus);
```
