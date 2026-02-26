---
name: a-philosophy-of-software-design-skills
description: >-
  Apply John Ousterhout's software design philosophy to fight complexity through
  deep modules, information hiding, strategic programming, and error elimination.
  Use when designing modules, classes, interfaces, or APIs. Use when reviewing code
  for complexity, shallow abstractions, information leakage, or pass-through methods.
  Use when refactoring, decomposing systems, choosing between design alternatives,
  naming variables, writing comments, or deciding how to handle errors and exceptions.
  Use when someone asks about complexity management, modular design, or software
  design principles. Use when evaluating software trends (OOP, TDD, design patterns,
  getters/setters) through a complexity lens. Use when deciding what matters in a design.
  Triggers: code review, architecture, refactoring, API design, module design,
  interface design, abstraction, encapsulation, complexity reduction, software trends,
  design patterns critique, deep modules, information hiding, error elimination,
  boy scout rule, incremental improvement, cleanup, code quality, blast radius.
---

# A Philosophy of Software Design

The enemy is **complexity** — anything that makes code hard to understand or modify.

**Complexity = Σ(cp × tp)** — where cp is the complexity of each part and tp is the fraction of developer time spent on that part. A module that's complex but rarely touched contributes little. A slightly complex module touched constantly dominates total complexity. This means: **invest design effort proportional to how often code is touched, not how complex it looks.**

Complexity manifests as:
- **Change amplification** — A simple change requires modifying many places
- **Cognitive load** — How much a developer needs to know to make a change
- **Unknown unknowns** — Not knowing what you don't know (the worst of the three — it means there's something you need to know but have no way to discover)

Complexity accumulates incrementally from **dependencies** (code can't be understood in isolation) and **obscurity** (important information is not obvious). Note: what The Pragmatic Programmer calls "orthogonality" (independent components, no cascading changes) is the natural result of applying deep modules + information hiding. Orthogonality is the outcome; Ousterhout's principles are how you get there.

## The Strategic Mindset

Before touching code, ask yourself:

- **"Am I programming strategically or tactically?"** — If you're just making it work without thinking about design, stop. Working code isn't enough. Invest 10-20% of time on design. This investment pays for itself within months.
- **"Will this increase or decrease the system's total complexity?"** — Every change either fights complexity or feeds it. Zero tolerance for incremental complexity.
- **"Am I the tactical tornado?"** — Shipping fast but leaving destruction. The real heroes clean up after you. A 10% design investment turns you from a liability into a force multiplier.

## The Five Design Questions

Ask these for every module, class, method, or interface you write:

### 1. "Is this module deep or shallow?"

Deep = simple interface, powerful implementation. Shallow = complex interface, little functionality.

**The test**: Draw a rectangle. Width = interface complexity. Height = implementation power. If it's wider than tall, it's shallow — redesign it.

- Unix file I/O is deep: 5 calls (`open`, `read`, `write`, `lseek`, `close`) hide enormous complexity
- Java I/O is shallow: `FileInputStream` → `BufferedInputStream` → `ObjectInputStream` chains for basic reads

**NEVER** create a class or method whose interface is as complex as its implementation. That's a shallow module adding complexity, not hiding it.

### 2. "What information am I hiding — and leaking?"

Every module should embed design decisions that are hidden from the rest of the system.

**Information leakage red flag**: The same knowledge appears in multiple modules. If two classes both know a file format, wire protocol, or data structure — one of them shouldn't.

**Temporal decomposition trap**: Don't structure code by execution order (read module → process module → write module). Structure by information hiding. If reading and writing share format knowledge, they belong together.

### 3. "Is this somewhat general-purpose?"

Not fully general (too complex) — not too specific (leaks use-case details). The sweet spot:

- "What is the simplest interface that will cover all my current needs?"
- "In how many situations will this method be used?" (If only one, probably too specific)
- "Is this API easy to use for my current needs?" (If you have to write a lot of extra code, too general)

**Push specialization upward**: Lower layers general-purpose, higher layers special-purpose. Special code calls general code, never the reverse.

### 4. "Can I define this error out of existence?"

Exceptions are a major source of complexity. For each exception, ask:

| Strategy | When to Use |
|----------|-------------|
| **Define out of existence** | Redefine semantics so the error can't happen. `delete(file)` means "ensure file doesn't exist" — already-gone is success, not error. |
| **Mask exceptions** | Handle at low level so callers never see them. Retry network failures internally. Use defaults for missing config. |
| **Aggregate exceptions** | Handle many related errors in one place at a higher level instead of scattered try-catch. |
| **Just crash** | For truly unrecoverable errors (out of memory, corrupt state). Don't pretend you can handle the unhandleable. |

**NEVER** throw exceptions for expected conditions. Missing optional data, boundary conditions, empty collections — these aren't exceptional.

### 5. "Have I designed it twice?"

Never go with your first design. Always consider at least one radically different alternative. Compare them. The time investment is small, the quality improvement is large.

If you can't think of an alternative, you don't understand the problem well enough.

## Decide What Matters

The most important — and most difficult — skill: distinguish what matters from what doesn't.

- **Minimize what matters**: Find designs that reduce the number of things developers must get right. Fewer invariants, fewer constraints, fewer things to remember. Good design makes it hard to do wrong.
- **Emphasize the important**: When something IS important, make it visible and impossible to miss — naming, code structure, comments, all should scream "pay attention here."
- **Treat unimportant things as unimportant**: Don't waste design energy on irrelevant details. Don't treat coding style debates like architecture debates. Use formatters and move on.
- **Common mistake**: Treating everything as equally important. When everything matters, nothing matters. When developers sweat every detail equally, they miss the ones that actually compound.

## The Boy Scout Rule: Strategic Improvement in Practice

> "Leave the campground cleaner than you found it."

Ousterhout's principle #21: **"Stay strategic when modifying code — don't just patch; improve the design."** The Boy Scout Rule operationalizes this. Every file you touch is an opportunity to fight complexity — but with discipline.

### The blast radius constraint

Improvements must stay **within the blast radius** of your primary change. This is the bridge between Ousterhout's "zero tolerance for complexity" and practical shipping:

```
Is this improvement within files I'm already changing?
  NO  → Don't touch it. Note it for a future task.
  YES → Continue.

Does this improvement change behavior?
  YES → Only if covered by existing tests. Otherwise, note it.
  NO  → Safe to apply. Do it.

Can I verify this won't break anything?
  YES → Apply it in the same commit as your primary change.
  NO  → Don't risk it. Note it.
```

### What to improve when you touch a file

Apply the design philosophy red flags as a checklist within your blast radius:

- **Vague names** near your changes → rename to create mental images
- **Magic numbers** in code you're modifying → extract to named constants
- **Dead code** your changes orphaned → remove imports, variables, functions
- **Information leakage** you notice → consolidate if it's within scope
- **Shallow methods** you're calling → consider inlining or deepening
- **Repetitive patterns** you just duplicated → extract shared logic immediately
- **Missing type annotations** near your changes → add them
- **Outdated comments** near your changes → update or remove
- **Unused imports** your changes created → clean up

### What NOT to improve (discipline matters)

- Massive refactors unrelated to your task
- Code in files you're not touching
- Behavior changes without test coverage
- "Fix everything" in a file you touched one line of
- Style preferences in code that follows existing conventions

### The compound effect

Small improvements seem insignificant. But complexity accumulates incrementally — and so does quality. Three Boy Scout improvements per commit, five commits per day, 250 working days: **3,750 micro-improvements per year.** That's the difference between a codebase that decays and one that improves.

**Include Boy Scout improvements in your commit message** — make them visible so reviewers appreciate incremental stewardship.

## The NEVER List

These are hard-won anti-patterns. Each one adds complexity that compounds over time:

- **NEVER create pass-through methods** — Methods that just delegate to another method with a similar signature are shallow. They add interface without hiding anything. Merge layers or add real value.
- **NEVER let classitis win** — Don't create a class for every tiny thing. Java's I/O hierarchy (InputStream → BufferedInputStream → DataInputStream) is the canonical failure. Combine related functionality into deeper classes.
- **NEVER use temporal decomposition** — Don't create separate read/process/write modules when they share knowledge. Group by information, not by execution time.
- **NEVER expose configuration when you can compute defaults** — Every configuration parameter is complexity exported to every user. If you can compute a reasonable value, do it. Only expose what users genuinely need to control.
- **NEVER write a comment that repeats the code** — `// increment i` above `i++` adds noise. Comments must provide different information than the code: the WHY, the abstraction, the non-obvious constraints.
- **NEVER put implementation details in interface comments** — Interface says WHAT, not HOW. "Scans hash table using linear probing" belongs in implementation comments, not the interface.
- **NEVER use a vague name** — If you can't name it precisely, you don't understand what it does. `data`, `info`, `manager`, `handler`, `processor` are red flags. Use names that create a mental image.
- **NEVER use the same name for different things** — `block` meaning both "file block" and "disk block" creates unknown unknowns. Use `fileBlock` and `diskBlock`.
- **NEVER add special cases when you can generalize** — Each `if` for a special case multiplies code paths. Design abstractions that handle all cases uniformly.
- **NEVER let complexity accumulate** — "Just this once" repeated 100 times creates unmaintainable systems. Technical debt compounds. Fix it now or pay forever.
- **NEVER use getters/setters mechanically** — Ousterhout: getters and setters that merely expose instance variables are shallow by definition. They add interface without hiding anything. Only expose access when callers genuinely need it, and prefer methods that do something meaningful.
- **NEVER blindly follow "classes should be small"** — The obsession with small classes/methods is one of the biggest sources of classitis. A 200-line method with a clean abstraction beats 10 20-line methods that must be read together. Judge depth, not lines.
- **NEVER use inheritance when composition works** — Inheritance is the tightest coupling in OOP. Parent changes silently break children. Composition via interfaces preserves information hiding. Reserve inheritance for genuine is-a relationships with a deep, stable hierarchy.
- **NEVER skip "design it twice"** — Your first design is never the best. The cost of considering an alternative is minutes; the cost of a bad design is months. If you "don't have time" to consider alternatives, you don't have time not to.

## Ousterhout's Software Trends Critique

These are contrarian positions backed by deep reasoning. Apply the complexity lens:

- **OOP inheritance** — The most overused and harmful OOP feature. Creates tight coupling between parent and child. Parent implementation details leak into subclasses (violates information hiding). **Prefer composition via interfaces.** Interface inheritance (defining method signatures) is fine; implementation inheritance (sharing code via subclass) should be rare.
- **Design patterns** — Useful as a starting point but dangerous as an end point. The GoF elements that create deep modules (e.g., Strategy, Decorator adding real value) are good. Patterns that create shallow, boilerplate-heavy layers are complexity. Don't apply a pattern just because it has a name.
- **Test-driven development** — Writing tests first causes tactical thinking: focus shifts from good design to making tests pass. Write tests after designing the interface (comments-first), not before. Tests should validate design, not drive it. Code designed purely to satisfy tests tends toward shallow, procedure-oriented structure.
- **Getters/setters** — Mechanical getters and setters are shallow by definition. They expose implementation (the fact that a field exists) while pretending to hide it. Only expose access when callers genuinely need it, and prefer methods that embody meaningful operations.

## Writing Comments as Design Tool

Write interface comments BEFORE writing code. If the comment is hard to write, the design is wrong.

**Three comment types that matter**:

1. **Interface comments** — WHAT the module/method does. All parameters, return values, side effects, exceptions, preconditions. No implementation details.
2. **Data comments** — What each variable represents, units, boundaries (inclusive/exclusive), null semantics, invariants, ownership.
3. **Implementation comments** — WHY this approach. Overall strategy. Non-obvious rationale. NOT what each line does.

**Cross-module design decisions**: Put documentation where developers will naturally find it. When a change requires touching multiple files, document the connection at the most visible point.

## Red Flags Reference

When you see these, complexity is growing. See [references/red-flags.md](references/red-flags.md) for the complete catalog with examples.

| Red Flag | What It Means |
|----------|---------------|
| Shallow module | Interface nearly as complex as implementation |
| Information leakage | Same design decision in multiple modules |
| Temporal decomposition | Structure follows execution order, not information |
| Pass-through method | Method just delegates without adding value |
| Repetitive code | Same pattern copy-pasted with slight variations |
| Conjoined methods | Can't understand one method without reading another |
| Hard-to-pick name | Unclear design if you can't name it |
| Comment repeats code | Comment provides no new information |
| Implementation in interface | Interface describes HOW instead of WHAT |
| Special-case proliferation | Growing number of conditionals for edge cases |
| Nonobvious code | Reader can't quickly guess behavior |
| Vague name | `data`, `info`, `manager`, `handler`, `result` |

## Deep Reference Material

Load these ONLY when the specific topic is relevant to the current task:

- **[references/deep-modules.md](references/deep-modules.md)** — **MANDATORY** when designing modules, interfaces, APIs, or reviewing for information hiding. Covers module depth, classitis, abstraction quality, layer separation, pull-complexity-downward, performance design around the critical path, and the "better together or apart" decision framework. **Do NOT load** for simple naming or commenting tasks.

- **[references/error-elimination.md](references/error-elimination.md)** — **MANDATORY** when designing error handling, exception strategies, or eliminating special cases. Covers all four strategies (define out, mask, aggregate, crash) with decision trees and book examples. **Do NOT load** for module design or naming tasks.

- **[references/code-clarity.md](references/code-clarity.md)** — **MANDATORY** when naming things, writing comments, reviewing for obviousness, or establishing consistency conventions. Covers naming precision, comment-first design, three comment types, making code obvious, and consistency rules. **Do NOT load** for error handling or module depth tasks.

- **[references/red-flags.md](references/red-flags.md)** — **MANDATORY** when doing code review or evaluating design quality. Complete catalog of all 15 red flags (with What/Test/Fix), all 23 design principles, and 3 decision trees. **Do NOT load** for focused implementation tasks where a specific reference above is more targeted.

**Do NOT load ANY reference files** when just applying the core thinking frameworks above. The SKILL.md content is sufficient for most design decisions. Only load when you need the deep detail.
