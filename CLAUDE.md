# NEXUS-EventModeling

## Sync Contract

The condensed reference sections below (`## Event Modeling` and `## F#`) mirror the full content in `EVENT_MODELING.md` and `FSHARP.md`.
**When either of those files is updated, update the corresponding section here.**

---

## Purpose

This is the NEXUS ecosystem-wide base library for Event Modeling in F#. It defines the core types, testing utilities, and conventions used across all NEXUS projects that practice Event Modeling.

This repo is the source of truth for:
- The `EventModeling` core library (Actor, Command, Event, View, Slice, Path, Grouping types)
- The `EventModeling.Testing` utility library (GWT test adapters, grouping test runner)
- The canonical example domain (card game) that exercises the full model

## Stack

- **Language**: F# on .NET 10.0
- **Test framework**: Expecto
- **Solution**: `EventModeling.slnx`

## Before Any Design or Programming Discussion

Read `EVENT_MODELING.md` — it defines the methodology, vocabulary, and mental model. Every design decision in this codebase is grounded in that document.

Read `FSHARP.md` — it captures F# patterns, Scott Wlaschin's teachings, and accumulated learnings specific to this codebase.

## Project Layout

```
EventModeling/           Core library — domain types only
EventModeling.Testing/   Test utility library — GWT adapters + grouping runner
EventModeling.Tests/     Executable test suite — card game example domain
```

## Key Principles

- All time values are `DateTimeOffset` in UTC. `OccurredAt` on events is always UTC. Timezone conversion belongs in the view/presentation layer only.
- Slices are independent and immutable once finalized. No shared logic between slices.
- GWT example data IS the test suite. Model and tests are the same artifact.
- Handlers are pure functions. No side effects, no I/O, no state.
- `SliceRef` decouples the Grouping/Path hierarchy from the internal generic types of each slice.

## F# Conventions

- Compile order in `.fsproj` matters — types must be defined before they are used.
- Prefer discriminated unions over enums for domain concepts (see `ActorKind`, `Grouping`, `SliceRef`).
- Use F# records for all data types — immutable by default.
- `Result<'T, string>` for CommandHandler failures — the string is a business rule violation message.
- `Map` (F# immutable map) is the correct type for `SliceRegistry`.

---

## Event Modeling

> Condensed from `EVENT_MODELING.md`. Update that file AND this section together.

- **Actor**: source of decisions and actions — `Human of role`, `Automation of name`, `ExternalSystem of name`
- **Command**: intent expressed by an Actor; handled by a hidden pure CommandHandler; blue box
- **Event**: immutable fact, past tense, never changes; the backbone and the only coupling point; orange box
- **View**: shaped data produced by a hidden EventHandler for an Actor to consume; green box
- **CommandSlice**: `Actor → Command → Event(s)` — data flows top-down; state change
- **ViewSlice**: `Event(s) → View → Actor` — data flows bottom-up; state view
- **Cadence**: CommandSlice → ViewSlice → CommandSlice → ViewSlice (sine wave)
- **GWT**: Given/When/Then — simultaneously the business spec and the test suite; example data IS the tests
  - CommandSlice GWT: Given = prior events (state), When = command, Then = resulting events
  - ViewSlice GWT: Given = events, When = filter/sort/transform criteria, Then = shaped view
- **Flow**: smallest grouping — short sequence of slices for one focused interaction
- **Workflow**: series of Flows or nested Workflows for a larger process; nestable, no depth limit
- **Area**: top-level grouping of related Workflows
- Groupings are header rows above the grid — purely organizational, no logic
- **Slice immutability**: finalized slices are replaced, never modified; no code shared between slices
- **Conway's Law**: event rows can mirror team/system/bounded-context boundaries
- Handlers are hidden; GWT is visible — GWT rows are the specification handlers implement

---

## F#

> Condensed from `FSHARP.md`. Update that file AND this section together.

**Scott Wlaschin / Domain Modeling Made Functional**
- Make illegal states unrepresentable — encode invariants in types, not prose
- Use discriminated unions instead of strings/bools/ints for domain concepts
- Total functions: if a function can fail, encode it in the return type (`Result`, `option`) — never throw for domain failures
- Parse, don't validate: validate at the boundary; inside the domain everything is already valid by construction
- Compose functions; F# has no data inheritance

**Compiler rules**
- Compile order in `.fsproj` is mandatory — a type used before it is defined is a compile error
- `namespace` for type-only files; `module` when the file contains `let` bindings/functions
- Recursive functions require `rec`; mutually recursive types require `and`
- Pattern matching is exhaustive — adding a DU case will flag every incomplete match
- `open` is file-scoped — every file that needs a namespace must open it explicitly

**Types and time**
- Always `DateTimeOffset` UTC — never `DateTime`; `OccurredAt` is always UTC; timezone conversion is a view concern
- Use `DateTimeOffset.MinValue` as sentinel in GWT example data (runtime timestamp is not part of the spec)
- Test adapters strip `OccurredAt` before comparing events — only `Name` and `Data` are asserted
- `Result<'T, string>` for CommandHandler failures; the string is a business rule violation message
- Single-case DUs wrap primitives to prevent mixing incompatible IDs (e.g. `CustomerId of int`)

**Expecto**
- `testCase "label" <| fun () -> ...` — `<|` avoids nested parens, idiomatic
- `testList "name" [...]` — groups tests; mirrors the Grouping hierarchy
- `runTestsWithCLIArgs [] args allTests` — standard entry point in `Program.fs`
