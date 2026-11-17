---
name: "tla-go-conformance"
description: "Use when keeping Go code in sync with a TLA+ or PlusCal spec: designing conformance tests, trace validation with TLC, guard tests, or instrumentation."
---

# Keeping Go code faithful to TLA+ specs

The goal is a CI-enforced check that a Go implementation has not drifted from its TLA+ (or PlusCal) spec. The approach combines two checks that cover opposite directions. Neither is a proof: evidence is bounded by state and schedule coverage, so say so when reporting results.

| Check | Direction | Catches | Misses |
|---|---|---|---|
| Conformance tests: replay spec behaviors against Go, compare states through an abstraction function | spec -> code | Code that cannot do what the spec requires; guards too strong; corner-case interleavings TLC finds | Code doing what the spec forbids (the spec never generates disabled actions) |
| Trace validation: log events from Go runs, TLC checks the log is a spec behavior | code -> spec | Code doing what the spec forbids; guards too weak; wrong values | Anything tests never execute; rare interleavings need schedule exploration |

Adopt in this order: (1) trace validation over existing tests, (2) sequential replay of spec behaviors, (3) gated fakes for the few schedules TLC shows to matter.

## 1. Map the Go code to the spec

- A handler is several actions, not one. Each in-flight request has a program counter and locals (`pc[r]`, saved reads). Cut at every interaction with shared state: DB call, lock, message send, RPC.
- Fold pure local logic into the adjacent action. Everything inside one transaction or one mutex is a single action.
- Branching is guarded alternatives: one sub-action per outcome with disjoint guards (one Go return path each), or `IF`/`CASE` inside an action. Outcome types are tagged records. Decisions the spec does not model become nondeterminism.
- A crash or failure between two steps is its own action, enabled at that `pc` value.
- Use PlusCal for handlers and worker loops (one `process` per handler, a label at every shared-state call; a missing label hides a real interleaving). Use plain TLA+ for reactive state machines and for abstract specs meant to be refined.
- Naming convention: one spec action or PlusCal label = one trace event name = one Go call site.

## 2. Keep the spec small

- Specify only the concurrency and distribution core. Leave out validation, serialization, plumbing.
- Abstract data to model values. Prefer several small specs (one per protocol or invariant) and small models.
- Conformance pulls the spec toward the code. If the spec is too abstract to map events onto, either keep two levels linked by refinement (trace-validate only the detailed one) or log coarse events and let hidden steps stutter.

## 3. Instrument without leaking into production

Prerequisite: dependencies are injected as interfaces. A handler that uses a global DB handle cannot be traced at the boundary, gated, or faked; fix that first.

- At dependency boundaries: tracing decorators in `_test.go` files or a test-only package. Production code contains nothing.
- Inside the code: a tiny package with two files selected by a build tag, exposing a constant. Guard every call site with the constant so the block and its argument evaluation compile out.

```go
// tlatrace/on.go   //go:build tlatrace    const Enabled = true
// tlatrace/off.go  //go:build !tlatrace   const Enabled = false

if tlatrace.Enabled {
    tlatrace.Event("Debit", "req", id, "amt", amt)
}
```

- CI conformance job: `go test -tags tlatrace ./...`. Release builds never pass the tag. Add a `go tool nm` check that trace-writer symbols are absent from the release binary.
- Do not use runtime flags or environment variables to switch tracing.
- Log each event atomically with the state change it describes (same lock or transaction).
- Several processes: one log file per process, each entry stamped with a logical clock, merged before validation.

### What to log

- Log event name plus arguments; add updates to spec variables where arguments do not determine them. This is usually the sweet spot.
- Less information means more search for TLC. Variables-only or names-only logs can blow up the state space.
- Sparse logs can falsely accept a trace. They still catch real bugs, so start sparse and tighten.
- Do not log steps invisible to the spec (for example resends); they are stuttering.

## 4. Trace validation recipe

Log one JSON object per line. Write a trace spec that extends the real spec and constrains each step to the next logged event. The module below is a sketch that has not been run; expect to fix details on first use.

```
---- MODULE TransferTrace ----
EXTENDS Transfer, Json, IOUtils, Sequences, TLC
Trace == ndJsonDeserialize(IOEnv.TRACE)
VARIABLE l
ev == Trace[l]
TraceInit == Init /\ l = 1
TraceNext == /\ l <= Len(Trace) /\ l' = l + 1
             /\ \/ ev.action = "Read"  /\ Read(ev.req) /\ seen'[ev.req] = ev.balance
                \/ ev.action = "Debit" /\ Debit(ev.req)
TraceSpec == TraceInit /\ [][TraceNext]_<<vars, l>>
TraceAccepted == LET d == TLCGet("stats").diameter
                 IN IF d - 1 = Len(Trace) THEN TRUE
                    ELSE Print(<<"Trace failed at event", d>>, FALSE)
====
```

Config: `SPECIFICATION TraceSpec`, `POSTCONDITION TraceAccepted`, the spec's `INVARIANTS`, `CHECK_DEADLOCK FALSE`.

Run: `TRACE=run1.ndjson java -cp tla2tools.jar:CommunityModules-deps.jar tlc2.TLC -config TransferTrace.cfg TransferTrace.tla` (`Json` and `IOUtils` come from TLA+ CommunityModules).

- Acceptance is weaker than refinement: a trace passes if at least one spec behavior is compatible with it; TLC fills in unlogged values.
- Initial state and constants must match the test setup. Generate the `.cfg` per test or derive it from the trace.
- Coarse logs: add a disjunct letting a hidden spec action occur without advancing `l`; keep it bounded.
- One Go step covering two spec actions: use TLC's action composition, or log two events.
- Breadth-first search is most informative. Depth-first is often faster (`-Dtlc2.tool.queue.IStateQueue=StateDeque`).
- A rejection has no counterexample. Use the reached depth (first unmatched event), the longest matched prefix, and the TLA+ debugger with breakpoints.
- A rejected trace means one of three things: a Go bug, a logging bug, or a spec that is wrong about the system. Determine which before changing anything.

## 5. Conformance tests (replay spec behaviors)

- Generate behaviors with TLC (simulation or state-graph dump) or Apalache (ITF JSON traces).
- A Go driver applies each action to the real code and compares state through an abstraction function (Go state -> spec state).
- Sequential behaviors need no scheduling control: run each handler to completion and treat it as several consecutive spec steps.

### When gated fakes are needed

A gated fake wraps a dependency so each call blocks until the test driver releases it. A handler blocked on a gate is exactly `pc[r]` at that label.

| Test | Gates? |
|---|---|
| Trace validation over ordinary tests | No |
| Replay where handlers run one after another | No |
| Replay where handlers' steps interleave | Yes |
| Negative guard tests via an interleaving | Yes |

- Without gates: run handlers concurrently with random delays or faults, many times, and validate every trace. Coverage is probabilistic.
- Gate granularity must match spec atomicity: gate a transaction as a whole, not each call inside it.
- Gates control only dependency boundaries. In-memory races between gates need `-race`, and the spec must model that shared memory if it matters. `testing/synctest` helps detect that all goroutines are blocked.
- Refactor a handler into explicit step functions only when it truly is a persisted state machine (saga, workflow, outbox).

## 6. Testing guards

Replaying spec behaviors checks assignments and half of each guard: a Go guard that is too strong fails the replay; one that is too weak is never exercised.

- Entry-point guards (actions an outside party initiates): at each spec state, enumerate candidate actions, subtract the enabled ones, attempt the rest in Go, assert rejection with state unchanged. Where the guard is input validation, model rejection as its own spec action.
- Mid-handler guards cannot be attempted from outside. Use the spec-mutation procedure below.
- Internal enabling conditions (for example commit only after quorum) need trace validation plus fault injection, randomized delays or deterministic simulation.

### Negative tests by spec mutation

1. Remove one guard from the spec and run TLC. A counterexample to an invariant shows the guard matters and gives the schedule that falsifies it. (Equivalent without mutation: check the invariant `~(pc[r] = label /\ ~Guard)`.)
2. Cut the counterexample at the first step where the original guard is false. The prefix is a valid behavior of the original spec.
3. Replay the prefix against Go (gated fakes if handlers interleave), then release the handler and let it run freely.
4. Validate the resulting trace against the original spec. Correct code diverges onto the failure path and is accepted. Code with a weak or stale check (check-then-act) follows the counterexample and TLC rejects the trace at that step.

If TLC finds no counterexample, report which of four causes applies: the guard is redundant; the invariants are too weak (a missing invariant); the guard protects a liveness or refinement property; the model is too small.

Also:

- Extract several counterexamples per guard; a guard can be false in distinct ways.
- Dependency fakes can return arbitrary state to reach states the spec calls unreachable. Label these defensive tests, not conformance tests.
- Mutate the Go guards too (weaken or delete each) and confirm the suite fails.
- Automate: a script disables one guard at a time, runs TLC, and saves each counterexample prefix as a replay script.

## 7. CI layout

1. TLC on the spec itself (invariants, properties).
2. Conformance job with the build tag: replay tests.
3. Trace validation over the logs of all tests, one TLC run per trace file.
4. Periodic spec-mutation and Go-guard-mutation runs.
5. Binary check that tracing is absent from release builds.

Track TLC coverage statistics and the guard mutation score alongside pass/fail.

## Known precedents

Check current details before citing; several of these were not re-verified.

- Cirstea, Kuppe, Loillier, Merz, "Validating Traces of Distributed Programs Against TLA+ Specifications" (SEFM 2024, arXiv:2404.16075): the trace validation method; found discrepancies in every system tried; Java instrumentation only.
- etcd-io/raft issue #111: Go precedent; `etcdraft.tla`, `Traceetcdraft.tla`, tracing behind the `with_tla` build tag, random-fault test producing traces.
- Howard et al., "Smart Casual Verification of the Confidential Consortium Framework" (NSDI 2025): spec reverse-engineered from C++ code, corrected by trace validation, counterexamples turned into tests, six bugs found, runs in CI.
- Davis, Hirschhorn, Schvimer, "eXtreme Modelling in Practice" (VLDB 2020) and Davis's 2025 MongoDB blog retrospective: generated tests succeeded for a self-contained algorithm; trace-checking the server failed because of multithreaded state snapshots and a spec more abstract than the code.
- Go tooling: `pgregory.net/rapid`, `github.com/anishathalye/porcupine`, `testing/synctest`, TLA+ CommunityModules, PGo.
- A first exercise: a toy Go two-phase commit containing the counter bug from the Cirstea et al. paper (counting Prepared messages instead of tracking the set of senders, so a resend commits early).