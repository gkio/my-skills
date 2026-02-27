# Code Clarity: Names, Comments, Obviousness, and Consistency

## Contents
- Naming: creating mental images
- The comment-first design process
- Three types of comments that matter
- Making code obvious
- Consistency as complexity reducer
- Modifying existing code strategically

---

## Naming: Creating Mental Images

A name should create a clear image in the reader's mind of what the entity represents and, equally important, what it does NOT represent.

### The precision test

For any name, ask: "If someone sees ONLY this name, without context, will they guess correctly what it refers to?" If the answer is no, the name is too vague.

### Names that signal design problems

If you can't find a good name, the design is probably wrong:

- **Hard to name → unclear purpose** — If a method does multiple unrelated things, no single name captures it. Split the method or rethink the design.
- **Name needs "And"** — `readAndValidate()`, `fetchAndCache()` suggest the method does too much.
- **Generic names** — `data`, `info`, `manager`, `handler`, `processor`, `result`, `value`, `status`, `flag` — these carry almost no information. What DATA? What RESULT?

### Consistency rules

- **Same name = same concept everywhere.** If `fileBlock` means a logical block in the file system, never use `fileBlock` to mean something else.
- **Different concepts = different names.** If you have both file-level blocks and disk-level blocks, use `fileBlock` and `diskBlock`, never just `block` for both.
- **Scope determines length.** `i` is fine for a 3-line loop. `userPreferences` is needed when the variable lives across 50 lines.

### Boolean naming

Booleans must be predicates — their name should read naturally in an `if` statement:
- `isActive`, `hasPermission`, `canExecute`, `shouldRetry` — these answer yes/no questions
- `status`, `flag`, `state`, `mode` — these don't tell you what true or false mean

---

## The Comment-First Design Process

Write interface comments BEFORE implementation. This forces you to think about the abstraction before getting lost in implementation details.

### Why this works

1. **Comments expose bad design early** — If the interface comment is long or confusing, the interface is too complex. Fix the design before writing code.
2. **Comments ARE the design tool** — Writing "this method does X" forces you to clearly define X. Vague thinking produces vague comments, which reveals vague design.
3. **Comments stay accurate** — Written alongside code, not retroactively. Retroactive comments are rushed and often wrong.

### The process

1. Write class-level comment: What is this class? What abstraction does it represent?
2. Write method signatures + interface comments: What does each method do? Parameters, return values, side effects, exceptions, preconditions.
3. Declare key instance variables with comments: What does each represent? Units? Boundaries? Null semantics?
4. Implement methods, adding implementation comments for non-obvious sections.
5. Review: Does the implementation match the comments? If not, fix the design (not the comments).

---

## Three Types of Comments That Matter

### 1. Interface comments — What, not How

**The rule**: Describe the abstraction. Do not mention implementation.

Bad: "This method walks the hash table using linear probing and compares using strcmp."
Good: "Returns the value associated with the given key, or null if not found."

**Complete interface comments include:**
- What the method/class does (high-level)
- All parameters with precise descriptions
- Return value with all cases (including null, empty, error)
- Side effects (modifies state, writes to disk, sends network requests)
- Exceptions and when they're thrown
- Preconditions the caller must satisfy

### 2. Data comments — Precise variable descriptions

Variables are the most under-documented elements. Each variable comment should answer:
- **What does it represent?** (nouns, not verbs — "the current user's permission level", not "used to check permissions")
- **Units?** — milliseconds vs seconds, bytes vs kilobytes, pixels vs points
- **Boundaries?** — Is this index inclusive or exclusive? Is this count zero-based?
- **Null semantics?** — What does null mean? "Not yet loaded"? "Not applicable"? "Error"?
- **Invariants?** — "Always positive." "Never empty." "Sorted in ascending order."
- **Ownership?** — Who is responsible for freeing/closing this resource?

### 3. Implementation comments — Why and strategy

**What to explain:**
- Overall strategy of a complex algorithm ("Process in three phases: validate, execute, collect")
- WHY this approach was chosen over alternatives
- Non-obvious rationale ("We check X before Y because Z can modify X's state")
- What each major code block does (a single sentence per block, not per line)

**What NOT to explain:**
- What each line does (`// increment counter` above `counter++`)
- Language features or standard library usage
- Anything already obvious from the code

---

## Making Code Obvious

Code is obvious when a reader's first quick guess about behavior is correct.

### Things that make code obvious

- **Good names** that create the right mental image
- **Consistency** — similar things done similarly
- **White space** separating logical blocks
- **Expected patterns** followed (readers have expectations from conventions)
- **Minimal nesting** — early returns instead of deep if/else

### Things that make code NON-obvious (expert knowledge)

- **Event-driven programming** — Handlers registered dynamically. You can't trace the flow by reading the code. Document each handler's trigger, thread context, and preconditions.
- **Generic containers** — `Pair<Integer, Boolean>` tells you nothing. Create a named type: `StatusResult { currentTerm: int, isLeader: bool }`.
- **Code that violates expectations** — If a constructor has a side effect (starts threads, connects to network), that's non-obvious. Document it prominently. Better: don't do it.
- **Type mismatches in declarations** — `List<T> x = new ArrayList<T>()` — the declared type differs from the allocated type. Readers must reconcile both.

---

## Consistency as Complexity Reducer

Consistency lets developers leverage existing knowledge. Once you learn how one thing works, consistent systems let you predict how similar things work.

### The five types of consistency that matter most

1. **Naming** — Same name for same concept. Different names for different concepts. Always.
2. **Interface patterns** — Similar operations have similar signatures. If `getUser(id)` returns a User, then `getOrder(id)` should return an Order, not a `Result<Order>`.
3. **Error handling** — One approach throughout. Either all file operations throw IOError, or all return Result. Never mix.
4. **Invariants** — Properties that are always true. "Every line in the buffer ends with `\n`." Invariants eliminate entire categories of special cases.
5. **Code style** — Indentation, braces, whitespace, organization. Automate with formatters. Never debate.

### The "don't change existing conventions" rule

The value of consistency outweighs the value of a marginally better convention. Only change an existing convention when:
- You have significant new information (not just preference)
- The new approach is substantially better (not marginally)
- You will update ALL existing code to match (no mixed styles)
- The team agrees
- The disruption is worth it

If you can't commit to all five, keep the existing convention.

---

## Modifying Existing Code Strategically

When modifying existing code, resist the tactical urge to make the minimum change. Think:

### "If I were designing this system from scratch, what would the ideal design be?"

Then move the current design a step toward that ideal. Every modification is an opportunity to improve.

### Comment maintenance rules

1. **Keep comments near the code they describe** — Not in commit messages, not in separate docs. Right next to the declaration or implementation.
2. **Avoid comment duplication** — If the same information exists in two comments, one will go stale. Pick one location.
3. **Check the diff before committing** — Scan every changed file. Ask: "Did I update all related comments?" This catches 90% of stale comment problems.
4. **Higher-level comments survive better** — Comments about overall strategy rarely go stale. Comments about specific implementation details go stale with every refactor.
