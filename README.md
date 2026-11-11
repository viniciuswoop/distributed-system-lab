# Distributed Systems Lab

> A hands-on lab for turning distributed systems theory into running, measurable systems.

This repository is where I implement the use cases from **System Design Interview (Vol. 1 & 2)** using the mechanisms from **Designing Data-Intensive Applications** and the architectural reasoning from **Software Architecture in Practice**, then test whether the designs hold up under load and failure.

It isn't a collection of interview answers. Every case study starts from measurable quality goals and ends with an implementation that follows the design and experiments that try to prove it.

---

## Vision

> **Be able to design a distributed system, justify every trade-off, and prove the design works by measuring it.**

Reading about quorums, partitioning or fan-out is not the same as watching a service that goes down at 500 writes/s. This lab intends to close the gap between reading a concept and having seen it work, and fail, in running code.

### Goals

- **Practice architectural reasoning**: state requirements as testable quality attribute scenarios, pick tactics deliberately and record the trade-offs.
- **Build the mechanisms**, not just name them: replication, partitioning, consensus, idempotency, stream processing.
- **Measure everything**: every important design claim is backed by a load test, a fault injection or both.
- **Make every system observable**: each case ships with metrics, structured logs, distributed traces, a Grafana dashboard and alerts tied to its quality attribute scenarios. It should be possible to diagnose a failure from the telemetry alone, without reading the code.
- **Document the journey**: decisions, failed hypotheses and lessons learned are part of the output.

### Guiding principles

- **Quality attributes drive the design.** No component exists without a scenario that justifies it.
- **Every claim needs evidence.** "It scales" means nothing without a number and an experiment behind it.
- **Break it on purpose.** A design isn't understood until I've seen how it fails.
- **Simple first.** Start with the simplest design that meets the scenarios, then evolve it and record why in an ADR.
- **Self-contained cases.** Each case runs on its own, with no dependencies on other folders in the repo.
- **Write it down.** If it isn't documented, it wasn't learned.

---

## How the three books fit together

| Book | Role in the lab | Shows up as |
|---|---|---|
| *System Design Interview* (Alex Xu, Vol. 1 & 2) | **The problems** | Each folder in `cases/` |
| *Designing Data-Intensive Applications* (Martin Kleppmann) | **The mechanisms** | Data model, replication, partitioning and consistency choices |
| *Software Architecture in Practice* (Bass, Clements, Kazman) | **The reasoning** | Quality attribute scenarios, tactics, ADRs, architecture views |

Every case follows the same loop:

```
Problem ──► Measurable quality goals ──► Design decisions (ADRs) ──► Code ──► Experiments ──► Retro
              (QA scenarios)               (tactics + DDIA mechanisms)          (load + chaos)
```

---

## Repository structure

```
distributed-systems-lab/
├── README.md
└── cases/              # One self-contained folder per system design use case
```

Cases are added to this tree as they are implemented.

Every case has the same layout:

```
cases/NN-case-name/
├── README.md                          # Summary, how to run, key results
├── design/
│   ├── 01-functional-requirements.md
│   ├── 02-non-functional-requirements.md
│   ├── 03-architecture.md
│   ├── diagrams/                      # draw.io diagrams (.drawio.svg)
│   └── adr/                           # One file per decision: NNNN-title.md
├── src/
├── test/                              # Automated tests: unit, integration
├── deploy/                            # Everything needed to run the case, including its observability stack
├── experiments/                       # One folder per experiment: hypothesis, load/chaos scripts, results
└── RETRO.md
```

---

## Case study method

Before any code is written, each case produces four artifacts:

1. **Functional requirements**: what the system must do, and what is explicitly out of scope.
2. **Non-functional requirements**: written as quality attribute scenarios in the six-part SAiP form (source, stimulus, artifact, environment, response, response measure) and prioritized. Back-of-envelope estimates (QPS, storage, bandwidth) live here and become the load-test targets.
3. **Architecture**: the design itself. Diagrams, components and their responsibilities, the API, data model, the replication, partitioning and consistency strategy, failure handling, and the tactic each decision applies.
4. **Architecture decision records**: one per significant trade-off, linked to the non-functional requirements that drove it.

**Definition of done:** every high-priority quality attribute scenario has an experiment with recorded results, and `RETRO.md` is written. The lab as a whole is done when every case in `cases/` meets this definition.

---

## Getting started

> 🚧 No cases are implemented yet.

**Prerequisites:** each case's README lists the technologies and dependencies it needs.

Each case's README also explains how to run it, how to run its experiments, and which ports and dashboards it exposes.

---

## Conventions

- **Decisions** are recorded as ADRs in each case's `design/adr/NNNN-title.md`.
- **Diagrams** are made in draw.io and saved as `.drawio.svg` in `design/diagrams/`, so they render on GitHub and stay editable. They are embedded in `03-architecture.md`.
- **Experiments** are reproducible: hypothesis, scripts and raw results all live in the experiment folder.

---

## References

- Alex Xu. *System Design Interview: An Insider's Guide*, Vol. 1.
- Alex Xu & Sahn Lam. *System Design Interview: An Insider's Guide*, Vol. 2.
- Martin Kleppmann. *Designing Data-Intensive Applications*. O'Reilly.
- Len Bass, Paul Clements & Rick Kazman. *Software Architecture in Practice*. Addison-Wesley.

---

## License

MIT. See [LICENSE](LICENSE).
