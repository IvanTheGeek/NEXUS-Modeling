# NEXUS-EventModeling

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
