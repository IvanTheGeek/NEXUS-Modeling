# F# Notes — Patterns, Learnings, and Scott Wlaschin's Influence

This file is a living document. Add to it whenever you encounter a meaningful F# error, learn a pattern, or encounter a nuance that isn't obvious. It is read before any F# design or programming work in this repo.

---

## Scott Wlaschin — Domain Modeling Made Functional

Scott Wlaschin ([@ScottWlaschin](https://twitter.com/ScottWlaschin)) is the author of *Domain Modeling Made Functional* (Pragmatic Programmers) and the creator of [fsharpforfunandprofit.com](https://fsharpforfunandprofit.com). His teachings are a foundational influence on how this codebase models the domain.

### Core Ideas

**Make illegal states unrepresentable.**
Use the type system to encode invariants. If a value can never be null, never make it nullable. If a field can only hold two possible values, use a discriminated union — not a string, not a bool, not an int. The compiler enforces what the code can't express in prose.

```fsharp
// Bad — "kind" can hold any string
type Actor = { Name: string; Kind: string }

// Good — only valid kinds compile
type ActorKind =
    | Human          of role: string
    | Automation     of name: string
    | ExternalSystem of name: string
```

**The domain model is the design.**
Types are not DTOs or database rows. They are executable documentation. If a non-developer can read the type definitions and understand the domain, the model is right.

**Composition over inheritance.**
F# has no inheritance for data types. You compose behavior by combining functions and types. Functions are values — pass them in records, return them from functions, store them in maps.

**Total functions over partial ones.**
A function that can fail on certain inputs is a lie. If a function might not return a result, encode that in the return type: `option`, `Result`, or a custom DU. Never throw exceptions for domain failures — `Result<'T, string>` or `Result<'T, DomainError>` is the right tool.

```fsharp
// Total — caller is forced to handle both cases
type CommandHandler<'TState, 'TCommand, 'TEvent> =
    Event<'TState> list -> Command<'TCommand> -> Result<Event<'TEvent> list, string>
```

**Parse, don't validate.**
Validate at the boundary. Once data enters the domain, it is already valid by construction. Don't repeat null checks and range checks inside domain functions — use smart constructors and wrapper types at the edge.

---

## Key fsharpforfunandprofit.com Patterns

### Railway-Oriented Programming (ROP)

Scott's term for chaining `Result`-returning functions. The "happy path" flows straight through; errors are carried along the railway without explicit checking at every step.

```fsharp
// Bind (>>=) threads Result values together
let (>>=) result f =
    match result with
    | Ok v    -> f v
    | Error e -> Error e

// Or use the built-in computation expression
let workflow input =
    result {
        let! validated = validate input
        let! processed = process validated
        return processed
    }
```

Use this pattern in CommandHandlers that have multiple validation steps.

### Single-Case Discriminated Unions for Wrapper Types

Wrap primitives to prevent mixing them up. A `CustomerId` and an `OrderId` are both `int` under the hood but they are not interchangeable.

```fsharp
type CustomerId = CustomerId of int
type OrderId    = OrderId    of int

// Unwrap with pattern matching
let (CustomerId id) = someCustomerId
```

This pattern is not yet used in the core library — it becomes relevant when modeling specific domains on top of it.

### Active Patterns

Use active patterns to make `match` expressions more readable when the logic for a case is non-trivial.

```fsharp
let (|IsPlayer|IsDealer|) (actor: Actor) =
    match actor.Kind with
    | Human _ -> IsPlayer
    | _       -> IsDealer
```

---

## F# Compiler Behavior — Things to Know

### Compile Order Is Explicit and Mandatory

Unlike C#, F# files compile in the order listed in the `.fsproj`. A type used before it is defined is a compile error. This is by design — it prevents circular dependencies.

```xml
<Compile Include="Core.fs" />      <!-- Actor, Event, Command, View -->
<Compile Include="Handlers.fs" />  <!-- uses Event, Command, View from Core.fs -->
<Compile Include="GWT.fs" />       <!-- uses Event, Command, View -->
<Compile Include="Slice.fs" />     <!-- uses CommandHandler, EventHandler, GWT -->
<Compile Include="Path.fs" />      <!-- uses SliceRef -->
<Compile Include="Grouping.fs" />  <!-- uses SliceRef -->
```

If you get `The type 'X' is not defined` and the type clearly exists — check the compile order in the `.fsproj`.

### `module` vs `namespace`

- `namespace` — types only, no `let` bindings at the top level. Use for type definitions.
- `module` — can contain both types and `let` bindings. Required when the file contains functions.

The core library uses `namespace EventModeling` because it is type definitions only. The testing and test files use `module` because they contain functions.

### Generic Type Parameters Must Be Used

F# infers generics. If you write a generic function and the type parameter cannot be resolved, you get an "unexpected type variable" error. Provide explicit type annotations at the binding site when inference is ambiguous.

```fsharp
// Explicit annotation on the binding resolves ambiguity
let shuffleDeckSlice : CommandSlice<Actor, DeckShuffled, ShuffleDeck, DeckShuffled> =
    ...
```

### `rec` Is Explicit

Recursive functions and mutually recursive types require the `rec` keyword. Without it, a function cannot call itself.

```fsharp
let rec groupingToTest (registry: SliceRegistry) (grouping: Grouping) : Test =
    match grouping with
    | Flow(name, sliceRefs) -> ...
    | Workflow(name, children) ->
        testList name (List.map (groupingToTest registry) children)  // recursive
    | Area(name, workflows) -> ...
```

For mutually recursive types, use `and`:

```fsharp
type A = { B: B }
and  B = { A: A }
```

### `open` Scoping

`open` is scoped to the file. Each file that needs `EventModeling` types must `open EventModeling`. There is no global `using` equivalent. This is intentional — it keeps dependencies explicit.

### Pattern Matching Is Exhaustive

The compiler warns (and can error) on incomplete match expressions. This is a feature. When you add a case to a discriminated union, the compiler will identify every match expression that needs updating.

```fsharp
// Adding a new case to Grouping will produce warnings at every match site
type Grouping =
    | Flow     of name: string * slices: SliceRef list
    | Workflow of name: string * children: Grouping list
    | Area     of name: string * workflows: Grouping list
    // | SubArea of ...   <-- adding this would flag groupingToTest immediately
```

### `DateTimeOffset` vs `DateTime`

Always use `DateTimeOffset` for event timestamps. `DateTime` is ambiguous — it doesn't carry timezone information. `DateTimeOffset` always knows its offset from UTC.

```fsharp
OccurredAt = DateTimeOffset.UtcNow   // correct
OccurredAt = DateTime.UtcNow         // wrong type
```

In GWT example data, use `DateTimeOffset.MinValue` as a sentinel — it signals "this timestamp is example data, not a real occurrence time." The test adapters strip `OccurredAt` before comparing.

---

## Expecto Notes

### Test Functions Must Be `unit -> unit`

`testCase` takes a `string` label and a `unit -> unit` function. The function body uses `Expect.*` assertions. Any unhandled exception or `failtest` call fails the test.

```fsharp
testCase "my test" <| fun () ->
    Expect.equal actual expected "message"
```

### `<|` vs Parentheses

`testCase "label" <| fun () -> ...` is idiomatic. It avoids nested parentheses. `<|` is function application, right-associative: `f <| x` is `f x`.

### `testList` Nests Tests

`testList "name" [ test1; test2 ]` groups tests. The name appears in the output hierarchy. `groupingToTest` builds this tree directly from the `Grouping` DU.

### `runTestsWithCLIArgs`

The entry point for an Expecto test executable. Reads standard Expecto CLI arguments (`--filter`, `--summary`, etc.) and runs the provided `Test` value.

---

## Errors Encountered and Solutions

<!-- Add entries here as they occur. Format:
### Error: <compiler message or symptom>
**Context**: what you were doing
**Cause**: why it happened
**Fix**: what resolved it
-->

