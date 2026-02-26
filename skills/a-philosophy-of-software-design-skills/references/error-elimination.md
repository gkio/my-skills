# Defining Errors Out of Existence

## Contents
- Why exceptions multiply complexity
- Four strategies for eliminating exceptions
- Decision tree: which strategy to use
- Special case elimination
- The "just crash" philosophy

---

## Why Exceptions Multiply Complexity

Each exception creates a new code path. Each code path must be designed, implemented, documented, and tested. Exception handlers are often:

- **Far from the source** — The handler may be 5 call-stack levels away from the throw
- **Incomplete** — Developers forget to handle certain exceptions
- **Duplicated** — The same exception handled differently in different callers
- **Poorly tested** — Exception paths are hard to trigger in testing

The fundamental insight: **most exceptions should not exist**. They represent a failure of API design, not a failure of the operation.

---

## Strategy 1: Define Errors Out of Existence

Redefine the operation's semantics so that the "error" condition is simply a normal case.

### The key insight

Change the specification, not the implementation. Instead of:
- `delete(file)` — "Remove this file. Error if it doesn't exist."

Redefine to:
- `delete(file)` — "Ensure this file does not exist after this call."

Now "file already gone" isn't an error — it's a successful no-op. The caller's intent (file should not exist) is satisfied regardless.

### When to apply

- The operation's goal can be redefined as an outcome, not an action
- The "error" case actually satisfies the caller's true intent
- There's a sensible definition that works for all inputs

### Classic examples from the book

**Substring method**: Java's `substring(start, end)` throws `StringIndexOutOfBoundsException` for out-of-range indices. Better design: clamp indices to valid range, return shorter string or empty string. The caller wants "text in this range" — returning what's available is more useful than crashing.

**File deletion on Windows**: Windows wouldn't let you delete an open file. Unix defines deletion as "remove the directory entry" — the file data persists until the last reference closes. This eliminates a complex class of errors.

---

## Strategy 2: Mask Exceptions

Handle exceptions internally so callers never see them. The module detects the problem and recovers without involving higher layers.

### When to apply

- The module can recover without caller involvement
- A reasonable fallback exists (retry, default value, cached data)
- The exception is transient or environmental, not a logic error

### Masking decision tree

```
Can the operation be retried?
  YES → Retry internally (with backoff). Only surface if all retries fail.
  NO  → Continue below.

Is there a cached/default value that's acceptable?
  YES → Return it. Log the exception for monitoring.
  NO  → Continue below.

Can the operation proceed with reduced functionality?
  YES → Degrade gracefully. Document the degradation.
  NO  → The exception cannot be masked. Let it propagate.
```

### Practical examples

- **Network timeout** → Retry 3 times internally, only throw if all fail
- **Config file missing** → Use hardcoded defaults, log warning
- **Cache miss** → Fall through to primary source (this is normal, not an error)
- **Disk full during write** → For non-critical data, silently skip. For critical data, must propagate.

---

## Strategy 3: Exception Aggregation

Handle many different exceptions in a single place instead of catching each one individually near its source.

### When to apply

- Multiple operations in a sequence that can all fail similarly
- A top-level handler has enough context to make a good recovery decision
- Individual fine-grained handling adds complexity without value

### The pattern

Instead of:
```
try: step1()
except E1: handle1()
try: step2()
except E2: handle2()
try: step3()
except E3: handle3()
```

Aggregate to:
```
try:
    step1()
    step2()
    step3()
except (E1, E2, E3) as e:
    handle_all(e)  # Single place for all import-related errors
```

### Web server example from the book

A web server's top-level dispatch handles all exceptions from all handlers. Individual request handlers don't need try-catch for most errors — they simply let exceptions propagate to the dispatcher, which returns an appropriate error response. This centralizes error handling instead of scattering it.

---

## Strategy 4: Just Crash

For truly unrecoverable errors, crash immediately with a clear diagnostic message.

### When to apply

- The error means the application cannot function correctly
- Continuing risks data corruption or silent incorrect behavior
- The error is very rare (not a normal operating condition)
- No meaningful recovery is possible

### Examples

- **Out of memory** during core operations — Can't meaningfully recover
- **Corrupt internal data structures** — Continuing produces wrong results
- **Missing critical configuration** at startup — Can't start without it
- **I/O errors on critical storage** — Partial operation is worse than stopping

### When NOT to just crash

- In a server handling multiple independent requests (one bad request shouldn't kill others)
- When partial results are useful
- When the error is expected under normal operation (network packet loss)
- When you can degrade gracefully

---

## Special Case Elimination

Special cases are a separate but related source of complexity. Each `if` statement for a special case:
- Doubles the code paths through a function
- Increases cognitive load
- Is a potential source of bugs
- Must be tested separately

### Elimination techniques

**Design invariants that prevent special cases**: If every line in a text buffer always ends with `\n`, you never need to special-case the last line. Establish the invariant once (in the insert method) and benefit everywhere.

**Normalize inputs early**: Convert all variations to a canonical form at the boundary. Guest checkout with no customer ID? Generate a temporary ID upfront. Now the entire pipeline handles all orders uniformly.

**Use the general case to handle the special case**: Free orders (total = 0) don't need a special payment path. Run them through the normal payment flow — a zero charge succeeds trivially.

### Decision framework for special cases

```
Can I change the semantics so this isn't special?
  YES → Redefine (Strategy 1 above)
  NO  → Continue

Can I normalize the input so the general case handles it?
  YES → Normalize at the boundary
  NO  → Continue

Can I establish an invariant that prevents this case?
  YES → Enforce the invariant at all entry points
  NO  → The special case may be genuinely necessary. Isolate it.
```
