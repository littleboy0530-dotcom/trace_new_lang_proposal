# Trace Language Design Proposal v0.1

## Introduction

Trace is a debugging-first systems programming language built around a single guiding principle:

> "Writing correct code is difficult. Understanding why code failed should be easy."

Rather than attempting to guarantee correctness at compile time alone, Trace embraces the reality that bugs happen — and invests heavily in making failure analysis fast, precise, and automatic.

Trace is heavily inspired by Rust. Developers familiar with Rust will find most of the language immediately familiar. This document focuses on the areas where Trace diverges from Rust.

---

## Core Principles

1. **Readability First** — Code should clearly communicate intent.
2. **Intent Declaration** — Constraints are part of the type system, not scattered assertions.
3. **Continuous Verification** — Group constraints are enforced at every assignment.
4. **Fail Fast** — Invalid state is never silently tolerated.
5. **Evidence-Based Debugging** — Every crash produces a full diagnostic report in debug builds.
6. **Actor-Controlled Mutation** — All state mutation is explicit and isolated.
7. **Architectural Pressure** — The language nudges developers toward clean, understandable structure.
8. **Developer-Responsible Performance** — Readability and debuggability take priority; performance is the developer's concern.

---

## Type System

### Primitive Types

```
int, float, bool, char, string
```

### Data

Analogous to Rust structs. Data types may contain other Data types. Cyclic and self-referential structures are **forbidden**.

```trace
Data Player {
    name: string,
    health: is Hp,
}
```

### Enum

Analogous to Rust enums. Variants may carry data.

```trace
enum Status {
    Alive(int),
    Dead,
}
```

### Collections

Identical to Rust:

```trace
// Fixed-size array (stack allocated)
let arr: [int; 5] = [1, 2, 3, 4, 5];

// Dynamic vector (heap allocated)
let vec: Vec<int> = Vec::new();
```

### Generics

Identical to Rust:

```trace
Data Stack<T> {
    items: Vec<T>,
}

spec Container<T> {
    fn push(item: T);
    fn pop() -> Option<T>;
}
```

---

## Ownership, Lifetimes, and References

Trace follows Rust's ownership, lifetime, and reference model with one key difference: **there is no `&mut`**. Mutable references do not exist in Trace. All mutation is handled exclusively through Actors (see below).

```trace
let r = &player;      // ✅ Immutable reference
let r = &mut player;  // ❌ Does not exist in Trace
```

Lifetime annotations use identical syntax to Rust:

```trace
fn longest<'a>(x: &'a string, y: &'a string) -> &'a string {
    if x.len() > y.len() { x } else { y }
}
```

---

## Groups

Groups are the primary mechanism for expressing domain invariants in Trace. A Group is a **tag attached to a variable (the container)**, not to the value itself. This distinction is fundamental.

### Declaration

```trace
const MAX_HP: int = 150;
Group Hp is int and self >= 0 and self <= MAX_HP;

Group Even is int and self % 2 == 0;
Group Unsigned is int and self >= 0;
```

### Binding and Unbinding

Groups can be bound to or unbound from a variable at any point. Binding is checked immediately at the point of binding.

```trace
let health = 50;

// Bind a group
health is Hp;       // ✅ Checked immediately: 50 satisfies Hp
health is Unsigned; // ✅ Checked immediately: 50 satisfies Unsigned

// Unbind a specific group
health is not Hp;

// Unbind all groups
health is free;
```

### Permanent Binding

A Group can be bound at declaration time, making it permanent for the lifetime of the variable:

```trace
let health is Hp = 50;  // Hp is permanently bound
```

Groups can also be bound at the field level inside Data:

```trace
Data Player {
    health: is Hp,  // health is always verified against Hp
}
```

### Continuous Verification

Every assignment to a Group-bound variable is checked against all currently bound Groups. A violation causes **immediate program termination**.

```trace
health is Hp;
health = 200;  // ❌ Runtime termination: Hp constraint violated (200 > 150)
```

### Groups Are Attached to the Container, Not the Value

When a value is moved, only the value transfers — the Groups stay with the original variable. When a reference is taken, the Groups are visible through the reference because the container itself is being referenced.

```trace
let x = 50;
x is Hp;
x is Even;

// Value move: Groups do NOT transfer
let y = x;  // y = 50, no groups bound
            // x is invalidated (ownership transferred)

// Reference: Groups ARE visible (referencing the container)
let r = &x; // r sees Hp and Even constraints on x
```

The same applies to function arguments:

```trace
fn process(p: Player) { ... }   // Value move: Groups lost
fn inspect(p: &Player) { ... }  // Reference: Groups visible
```

### Group Inheritance

Groups can inherit from other Groups, adding additional constraints:

```trace
Group PositiveHp is Hp and self > 0;
// PositiveHp inherits Hp's constraints and adds self > 0
```

If two bound Groups produce a logical contradiction, the behavior depends on detectability:

```trace
Group AlwaysZero is int and self == 0;
Group NonZero    is int and self != 0;

let x = 0;
x is AlwaysZero;
x is NonZero;   // ❌ Contradiction detected -> immediate hard crash
```

If the contradiction cannot be detected statically, it will surface as a runtime error.

### Groups on Enums

Groups can constrain Enum values:

```trace
Group LivingStatus is Status and self == Alive(_);
```

### Built-in Groups

Trace provides a set of built-in Groups that map directly to low-level memory representations. Unlike user-defined Groups (which perform runtime checks), built-in Groups allow the compiler to optimize the underlying memory layout — equivalent to Rust's `u8`, `u16`, etc.

| Group      | Description         |
|------------|---------------------|
| `8bit`     | 8-bit width         |
| `16bit`    | 16-bit width        |
| `32bit`    | 32-bit width        |
| `64bit`    | 64-bit width        |
| `unsigned` | unsigned constraint |

Built-in Groups are composed using the same `and` chaining syntax as regular Group declarations. When combined, the compiler optimizes the underlying memory layout accordingly:

```trace
// Composing built-in Groups to assemble concrete types
Group u8  is int and unsigned and 8bit;   // equivalent to Rust's u8
Group u16 is int and unsigned and 16bit;  // equivalent to Rust's u16
Group i8  is int and 8bit;                // equivalent to Rust's i8

// 2-byte character type (e.g., Korean characters)
Group korean_char is char and 16bit;

// Usage
let byte is u8 = 200;
let ch is korean_char = '한';
```

This is the one area where Trace provides compiler-level optimization guarantees. All other performance concerns are the developer's responsibility.

### Safe Pattern: `is Group ?`

The recommended idiom for guarding assignments is the `is Group ?` expression, which evaluates the constraint and returns a `bool` **without** triggering termination:

```trace
if predicted_hp is Hp ? {
    self.health = predicted_hp;
} else {
    because("hp out of range: predicted value exceeds bounds");
}
```

This pattern is strongly recommended because in release builds, a raw Group violation produces no diagnostic output. Pairing the guard with `because()` ensures a meaningful message is always available.

---

## Specs

Specs are behavior interfaces identical to Rust traits. They may define method signatures and default implementations, and can be implemented by Data, Enum, or Group types.

```trace
spec Damageable {
    fn take_damage(amount: int);

    fn is_alive() -> bool {
        self.health > 0
    }
}
```

**Method dispatch resolution order:** Group → Parent Group → Base Type.

---

## Actors

All mutation in Trace must occur through Actors. Reads are always unrestricted. There is no `&mut` — Actors are the sole mutation mechanism.

### Declaration

`impl Actor for T` is a special block (not a Spec) that grants exclusive write access to `T`'s fields. Code outside this block may not write to `T`'s fields.

```trace
impl Actor for Player {
    fn take_damage(amount: int) {
        let predicted_hp = self.health - amount;
        if predicted_hp is Hp ? {
            self.health = predicted_hp;
        } else {
            if predicted_hp < 0 {
                self.health = 0;
            } else {
                because("damage calculate error: unexpected bounds");
            }
        }
    }
}

// Outside the Actor block:
player.health = 200;  // ❌ Compile error: mutation requires Actor context
```

### Execution Model

Each Actor owns a **mailbox**. Calling an Actor function enqueues a message in that mailbox. Messages are processed in **FIFO order**. The caller **always blocks** until the Actor function completes, regardless of whether a return value is expected.

```trace
player.take_damage(10);       // Blocks until complete
let score = player.get_score(); // Also blocks until complete
// Execution continues here only after get_score() returns
```

### Actor-to-Actor Calls Are Forbidden

Actor functions may not call other Actor functions. Since all Actor calls are blocking, mutual calls between Actors would deadlock permanently. This is enforced at compile time.

```trace
impl Actor for Player {
    fn take_damage(amount: int) {
        enemy.notify();  // ❌ Compile error: Actor-to-Actor call forbidden
    }
}
```

### Orchestration Pattern

Multi-Actor workflows are coordinated from **ordinary (non-Actor) functions**. This is the canonical pattern in Trace, and it is what makes complex state changes readable at a single call site:

```trace
fn process_attack(attacker: Enemy, defender: Player) {
    let damage = attacker.calculate_damage();  // Actor call
    defender.take_damage(damage);              // Actor call
    ui.update_health_bar(defender.health);     // Actor call
}
```

### Concurrency

- **Reads** are unrestricted and always safe.
- **Writes** are always routed through Actor mailboxes.
- **Deadlocks** are structurally impossible: Actor-to-Actor calls are forbidden, and all callers are non-Actor code.

---

## Unsafe Blocks

Passing Actor functions as higher-order values is permitted only inside `unsafe` blocks. This restriction exists because higher-order Actor calls can break the ordering guarantees of the mailbox model.

```trace
unsafe {
    let actions = [player.take_damage, enemy.take_damage];
    actions.map(|f| f(10));
}
```

---

## Error Handling

`Result` and `Option` are identical to Rust:

```trace
fn find_player(id: int) -> Option<Player>
fn parse_damage(s: string) -> Result<int, string>
```

For unrecoverable logic errors, use `because()`:

```trace
because("damage calculate error: unexpected bounds");
```

`because()` immediately terminates the program. In debug builds, it emits a full CSI report. In release builds, only the message string is printed.

---

## Functions

Unlike Rust, Trace **requires explicit `return` statements**. Implicit returns via the last expression are not permitted.

```trace
// ✅ Explicit return required
fn get_health() -> int {
    return self.health;
}

// ❌ Implicit return is a compile error
fn get_health() -> int {
    self.health
}
```

This is consistent with the Readability First principle — the point at which a function returns is always visually explicit.

---

## Control Flow

Identical to Rust. Trace supports `if`, `else`, `loop`, `while`, `for`, `match`, and loop labels:

```trace
'outer: for i in collection {
    'inner: for j in other {
        if condition {
            break 'outer;
        }
    }
}
```

---

## Functional Features

Closures use identical syntax to Rust, with the exception that `&mut` captures do not exist:

```trace
let double = |x: int| x * 2;
let scores = values.map(|x| x * 10);
```

Supported iterator methods: `map`, `filter`, `fold`, `find`, `any`, `all`.

---

## CSI — Code State Investigation

CSI is Trace's built-in debugging subsystem. It is **active only in debug builds** and completely stripped in release builds. CSI data is stored **in memory for the duration of the current session only** — it is not written to disk. On crash, the in-memory CSI data is printed to the console at the moment of termination.

### What CSI Records

- Variable assignment history (value, location, call site)
- Actor call history (function name, arguments, call site)

### CLI Interface

CSI is exposed via the `trace` package manager CLI:

```bash
# Show current value of a variable
trace show hp

# Show full assignment history
trace show --history hp

# Show history grouped by Actor call
trace show --history --by-actor hp
```

Example output for `trace show --history --by-actor hp`:

```
[Actor: take_damage(20) @ main.trace:15]
  hp: 100 -> 80

[Actor: take_damage(20) @ main.trace:22]
  hp: 80 -> 60

[Actor: take_damage(100) @ main.trace:30]
  hp: 60 -> 0  ❌ GROUP VIOLATION: Hp
```

### Crash Behavior by Build Mode

| Scenario | Debug Build | Release Build |
|---|---|---|
| `because("msg")` | Hard crash + full CSI report | Hard crash + message only |
| Group violation | Hard crash + full CSI report | Hard crash, no output |

> **Note:** Because CSI data is not persisted to disk, crash reports are only visible at the moment of termination. Developers who need to retain crash output should redirect console output to a file externally.

---

## Build Modes

| Feature | Debug | Release |
|---|---|---|
| CSI recording | ✅ Active | ❌ Stripped |
| Group violation report | ✅ Full CSI report | ❌ No output |
| `because()` output | ✅ Full CSI report | Message string only |
| `trace show` CLI | ✅ Available | ❌ Unavailable |
| Built-in Group optimization | ✅ | ✅ |
| Performance | Standard | Optimized |

---

## Package Manager

Trace ships with a dedicated package manager named **`trace`**, inspired by Cargo. It handles dependency management, build configuration, and exposes the CSI debug CLI in debug builds.

---

## Performance Philosophy

Trace does not optimize away Group checks or Actor overhead. Expensive runtime verification is an intentional design choice. If a developer determines that a check is too costly, it is their responsibility to restructure the code.

The only compiler-level performance guarantees are those provided by the built-in Groups (`8bit`, `16bit`, `32bit`, `64bit`, `unsigned`), which map directly to low-level memory layouts.

---

## Summary

Trace is a systems programming language that treats **debuggability as a first-class feature**. For developers familiar with Rust, the learning surface is small — the key differences are:

| Feature | Rust | Trace |
|---|---|---|
| Mutation | `&mut`, interior mutability | Actors only |
| Semantic constraints | Type aliases, newtypes | Groups (runtime-verified tags) |
| Crash diagnostics | `panic!`, backtraces | `because()` + CSI report |
| Package manager | Cargo | trace |

Trace is not designed for developers who want the compiler to prove their code correct. It is designed for developers who accept that bugs happen — and want the fastest possible path from "something went wrong" to "here is exactly why."
