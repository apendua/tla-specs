# tla-specs

Working material for one question: how do you keep a Go implementation from drifting away from its TLA+ (or PlusCal) specification, and enforce that in CI?

## Goal

Find a practical, repeatable workflow that combines two checks covering opposite directions:

- **Conformance tests** (spec to code): replay behaviors generated from the spec against the Go code and compare states.
- **Trace validation** (code to spec): log events from Go test runs and let the TLC model checker decide whether each log is a behavior the spec allows.

Neither is a proof. The aim is cheap, automated evidence that fails the build when the code and the spec disagree.

## What is here

- [`.claude/skills/tla-go-conformance/SKILL.md`](.claude/skills/tla-go-conformance/SKILL.md): an agent skill that distils the approach. It covers how Go handlers map to spec actions, instrumentation that stays out of production binaries, a trace validation recipe for TLC, when gated fakes are needed, negative guard tests by spec mutation, and a CI layout. Claude Code picks it up automatically when working in this repository.

## Status

Early. The approach is written down but not yet exercised end to end. In particular, the trace spec in the skill is a sketch that has not been run.

## Planned next

- A toy Go two-phase commit with a known bug, validated against its spec with TLC.
- A Go replay driver for spec-generated behaviors.
- A script for the spec-mutation loop that turns counterexamples into negative tests.
- Reading exercises for getting back into TLA+.

## Background

- Cirstea, Kuppe, Loillier, Merz. [Validating Traces of Distributed Programs Against TLA+ Specifications](https://arxiv.org/abs/2404.16075) (SEFM 2024).
- Howard, Kuppe, Ashton, Chamayou, Crooks. [Smart Casual Verification of the Confidential Consortium Framework](https://www.usenix.org/conference/nsdi25/presentation/howard) (NSDI 2025).
- Davis, Hirschhorn, Schvimer. [eXtreme Modelling in Practice](https://arxiv.org/abs/2006.00915) (VLDB 2020).
- [etcd-io/raft issue #111](https://github.com/etcd-io/raft/issues/111): TLA+ trace validation in a Go codebase.

## License

MIT, see [LICENSE](LICENSE).
