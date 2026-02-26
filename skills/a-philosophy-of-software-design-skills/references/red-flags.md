# Complete Red Flags and Design Principles Catalog

## Contents
- All red flags (signs of complexity)
- All design principles (summary)
- Quick-reference decision trees

---

## Red Flags: The Complete List

Each flag signals growing complexity. When you see one, stop and redesign.

### Shallow Module
**What**: Interface is nearly as complex as implementation. More to learn than the module hides.
**Test**: If the interface comment is as long as the implementation, the module is shallow.
**Fix**: Combine with related modules to create a deeper abstraction. Or simplify the interface by hiding more.

### Information Leakage
**What**: The same design decision (data format, protocol, algorithm) appears in multiple modules.
**Test**: If changing one detail requires changes in multiple modules, information has leaked.
**Fix**: Consolidate the shared knowledge into a single module that owns it completely.

### Temporal Decomposition
**What**: Code structure mirrors execution order rather than information structure.
**Test**: Do your "read" and "write" modules share format knowledge? Then they shouldn't be separate.
**Fix**: Group code by what information it uses, not when it runs.

### Overexposure (Too Many Configuration Parameters)
**What**: Module pushes complexity to callers through configuration that callers can't meaningfully set.
**Test**: Will users just copy default values from documentation? Then the parameter shouldn't be exposed.
**Fix**: Compute defaults internally. Only expose parameters users genuinely need to control.

### Pass-Through Method
**What**: Method whose implementation just calls another method with a similar signature.
**Test**: Does the method add any value beyond delegation?
**Fix**: Either merge the layers, move the functionality to where it's needed, or add real value at each layer.

### Pass-Through Variable
**What**: Variable passed through many layers, used only at the bottom.
**Test**: Does the variable appear in intermediate method signatures without being used by those methods?
**Fix**: Use a context object, dependency injection, or restructure to avoid threading.

### Repetitive Code
**What**: Same pattern appears in multiple places with slight variations.
**Test**: If you fix a bug in one copy, do you need to fix it in others?
**Fix**: Extract the common pattern into a shared method. If variations exist, parameterize.

### Special-General Mixture
**What**: General-purpose code contains knowledge of specific use cases.
**Test**: Does the general module need to change when a new use case is added?
**Fix**: Move special-case logic up to the caller. General code should be oblivious to specific uses.

### Conjoined Methods
**What**: Two methods that can't be understood independently — you must read both to understand either.
**Test**: Can you describe what method A does without mentioning method B?
**Fix**: Combine them if they share a single abstraction, or redesign so each is self-contained.

### Comment Repeats Code
**What**: Comment restates what the code already says, providing no new information.
**Test**: Delete the comment. Is any information lost?
**Fix**: Replace with a comment that explains WHY, or delete if the code is self-explanatory.

### Implementation Contaminates Interface
**What**: Interface documentation describes implementation details rather than the abstraction.
**Test**: If the implementation changed completely, would the interface comment need to change?
**Fix**: Rewrite the interface comment to describe WHAT, not HOW.

### Vague Name
**What**: Name that could refer to many different things, creating ambiguity.
**Test**: If someone sees only this name, can they guess what it refers to?
**Fix**: Use a more specific name that creates a precise mental image.

### Hard-to-Pick Name
**What**: Difficulty choosing a name, suggesting the design itself is unclear.
**Test**: If you can't name it, you can't describe it. If you can't describe it, you don't understand it.
**Fix**: Rethink the design. Clarify the purpose, then the name will follow.

### Hard-to-Describe Interface
**What**: Interface that's difficult to explain in a short comment.
**Test**: Can you describe the method's complete behavior in 1-2 sentences?
**Fix**: Simplify the interface. Split the method or redesign the abstraction.

### Nonobvious Code
**What**: Code whose behavior cannot be understood with a quick read.
**Test**: Does a competent developer's first guess about behavior match reality?
**Fix**: Rename, restructure, add strategic comments, or simplify the logic.

---

## Design Principles: Complete Summary

### Foundational
1. **Complexity is incremental** — It accumulates in small chunks. Zero tolerance.
2. **Working code isn't enough** — Design quality is the primary goal.
3. **Strategic > Tactical** — Invest 10-20% of time on design. It pays back within months.

### Module Design
4. **Modules should be deep** — Simple interfaces, powerful implementations.
5. **Information hiding is the key technique** — Each module embeds design decisions invisible to others.
6. **Different layer, different abstraction** — Adjacent layers should provide different abstractions.
7. **Pull complexity downward** — Better for a module to handle complexity than push it to callers.

### Generality
8. **Somewhat general-purpose modules are deeper** — Not fully general, not too specific.
9. **Separate general-purpose and special-purpose code** — Special code uses general code, never the reverse.
10. **Eliminate special cases** — Design abstractions that handle all cases uniformly.

### Error Handling
11. **Define errors out of existence** — Redefine semantics so errors can't occur.
12. **Mask exceptions** — Handle at low level so callers don't see them.
13. **Aggregate exceptions** — Handle related exceptions in one place.
14. **Just crash for unrecoverable errors** — Don't pretend you can handle the unhandleable.

### Clarity
15. **Design it twice** — Always consider alternatives before committing.
16. **Write comments first** — Comments are a design tool, not an afterthought.
17. **Comments describe what's not obvious** — WHY and WHAT, not HOW.
18. **Good names create images** — Precise, consistent, appropriate length.
19. **Code should be obvious** — Reader's first guess about behavior should be correct.
20. **Consistency reduces cognitive load** — Same thing done the same way everywhere.

### Maintenance
21. **Stay strategic when modifying code** — Don't just patch; improve the design.
22. **Keep comments near code and up to date** — Check diffs before committing.

### The Meta-Principle
23. **Decide what matters** — The most important (and hardest) skill. Minimize what matters: design so fewer things must go right. Emphasize what is important: make critical decisions visible and explicit. Treat unimportant things as unimportant: don't spend design energy on irrelevant details. Common failure mode: treating everything as equally important, which means nothing gets the attention it deserves.

### Software Trends (Ousterhout's Contrarian Positions)
24. **Implementation inheritance creates tight coupling** — Parent changes silently break children. Interface inheritance is fine; implementation inheritance should be rare. Prefer composition.
25. **Design patterns are starting points, not destinations** — Patterns that create deep modules are good. Patterns that create shallow boilerplate are complexity. Don't apply a pattern just because it has a name.
26. **TDD drives tactical thinking** — Writing tests first focuses on making tests pass, not on good design. Design the interface first (comments-first), then write tests to validate it. Tests should validate design, not drive it.
27. **Getters/setters are shallow by definition** — They expose the fact that a field exists while pretending to hide it. Only expose access when callers genuinely need it.

---

## Quick-Reference Decision Trees

### "Should I split this into two modules?"

```
Do they share information (format, protocol, state)?
  YES → Keep together (splitting causes information leakage)
  NO  → Continue

Does combining simplify the interface?
  YES → Keep together
  NO  → Continue

Is there duplication between them?
  YES → Keep together (extract shared logic)
  NO  → Continue

Is one general-purpose and one special-purpose?
  YES → Split them (special calls general)
  NO  → Keep together if simpler, split if each has clear independent purpose
```

### "How should I handle this error?"

```
Is this a normal/expected condition (empty input, missing optional)?
  YES → Define out of existence (return default, treat as success)
  NO  → Continue

Can the module recover without caller involvement?
  YES → Mask it (retry, fallback, degrade)
  NO  → Continue

Are there multiple similar errors that could be handled together?
  YES → Aggregate at a higher level
  NO  → Continue

Is this truly unrecoverable?
  YES → Just crash with clear diagnostic
  NO  → This is a genuinely exceptional condition. Throw a well-designed exception.
```

### "Is my interface deep enough?"

```
Can a caller use this without reading implementation? → If NO, too leaky
Is the interface comment shorter than the implementation? → If NO, too shallow
Can this handle multiple use cases without changes? → If NO, too specific
Does the caller need to configure things the module could figure out? → If YES, pull complexity down
```

### "Am I deciding what matters?"

```
How often is this code touched (tp in the complexity formula)?
  RARELY → Don't over-invest in design. Correctness is sufficient.
  FREQUENTLY → Invest heavily in design. This dominates total complexity.

Is this decision easy to change later?
  YES → Pick the simplest option and move on. Don't agonize.
  NO  → Design it twice. Consider radically different alternatives.

Am I treating this as equally important as everything else?
  YES → Stop. Identify what ACTUALLY matters in this change.
        Spend 80% of design effort on the 20% that matters.
  NO  → Good. Continue.
```

### "Should I use inheritance here?"

```
Is this a pure interface (no implementation to inherit)?
  YES → Interface inheritance is fine. Proceed.
  NO  → Continue.

Is the parent stable and unlikely to change?
  NO  → Don't inherit. Compose via interfaces instead.
  YES → Continue.

Do subclasses need the FULL parent interface?
  NO  → Don't inherit. You're coupling to things you don't need.
  YES → Inheritance is acceptable, but still consider composition.
```
