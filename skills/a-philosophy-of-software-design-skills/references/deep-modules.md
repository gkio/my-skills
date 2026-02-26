# Deep Modules, Information Hiding, and Layer Design

## Contents
- Module depth: the cost-benefit framework
- Information hiding vs information leakage
- Different layer, different abstraction
- Pull complexity downward
- Better together or better apart?
- Designing for performance without sacrificing simplicity

---

## Module Depth: The Cost-Benefit Framework

A module's **cost** is its interface (what callers must learn). Its **benefit** is its functionality (what it hides). Deep modules maximize benefit relative to cost.

**The rectangle visualization**: Interface = top edge width. Implementation = area. Deep modules are narrow on top, large in area. Shallow modules are wide on top, small in area.

### Decision framework: Is my module deep enough?

Ask: "Can someone use this module productively knowing ONLY the interface, without reading implementation?"

- **Yes, easily** → Deep. Good.
- **Yes, but they need to know a lot of interface details** → Shallow. Simplify the interface.
- **No, they need to understand internals** → Leaky abstraction. Redesign.

### The classitis disease

Conventional wisdom says "classes should be small." This is wrong when taken to extremes. Many small classes = many shallow modules = many interfaces = high cognitive load. The Java I/O system is the canonical example: basic file reading requires chaining FileInputStream, BufferedInputStream, and ObjectInputStream. Each layer is shallow — adding interface complexity without hiding much.

**The cure**: Combine related functionality into fewer, deeper classes. It's better to have one class with 10 methods that each hide significant complexity than 10 classes with 1 method each.

### When shallow is acceptable

- **Dispatchers** that route to implementations (e.g., HTTP router → handler)
- **Decorators** that add genuinely significant functionality (not just buffering)
- **Interface implementations** where the interface itself is the contract

---

## Information Hiding: The Core Technique

Each module should embed design decisions (data structures, algorithms, protocols, formats) that are invisible to the rest of the system.

### What to hide (expert knowledge)

The non-obvious things worth hiding:

- **Format decisions** — Wire protocols, file formats, serialization. If two modules both know that fields are separated by `\r\n`, that's leakage. One module should own the format.
- **Algorithm choices** — Whether you use quicksort or mergesort, B-tree or hash table. Callers shouldn't know or care.
- **Resource management** — Memory allocation, connection pooling, caching strategies. Handle internally.
- **Default values** — Don't make callers specify defaults. Compute sensible ones internally. Every configuration parameter is complexity pushed onto every user.
- **Error recovery** — Retry logic, fallback strategies, degraded modes. Mask these from callers when possible.

### The temporal decomposition trap

This is the most common information-hiding violation. Structure that mirrors execution order:

```
ReadFile module → ParseData module → WriteOutput module
```

If ReadFile and WriteOutput both know the file format, you've leaked information. **Fix**: Group by what information is shared, not by when things happen. A single FileFormat module owns all format knowledge.

### Information hiding within a class

Even within a single class, hide information between methods. Instance variables used by only a subset of methods should ideally be isolated. If method A's behavior depends on method B having been called first, that's a hidden dependency — document it or eliminate it.

---

## Different Layer, Different Abstraction

If adjacent layers have similar abstractions, one of them probably isn't adding value.

### Pass-through methods (the strongest signal)

A method that simply calls another method with the same or similar signature:

```
UserController.getUser(id) → UserService.getUser(id) → UserRepository.getUser(id)
```

Each layer should transform the abstraction. Controller handles HTTP. Service handles business logic + coordination. Repository handles persistence. If any layer is just passing through, merge it or add real value.

### Pass-through variables

Variables passed through many layers without being used. If `requestId` flows through 6 methods but only the bottom one needs it, use a context object or other mechanism to avoid threading it through every signature.

### When interface duplication IS acceptable

- **Dispatcher pattern**: A method that dispatches to different implementations based on a parameter. The signatures look similar but the behavior differs substantially.
- **Interface + implementation**: When a class implements an interface defined elsewhere. The similarity is the point.

### Decorators: usually shallow

Decorator classes (wrapping an object to add behavior) tend to be shallow. Before creating one, ask:
- Can this functionality be added directly to the underlying class?
- Can this be implemented as a standalone helper that doesn't wrap anything?
- Is this functionality used by most users of the underlying class? (If yes, put it in the class.)

---

## Pull Complexity Downward

When complexity exists, it's better for a module to handle it internally than to push it to callers. A module with a slightly more complex implementation but a simpler interface reduces total system complexity — because the implementation complexity exists once, but the interface complexity affects every caller.

### The configuration parameter test

Before adding a configuration parameter, ask:
- "Can the module compute a reasonable default?" → If yes, don't expose the parameter.
- "Will users know what value to provide?" → If not, they'll just use the default anyway, so hardcode it.
- "How many users will need to change this?" → If very few, consider not exposing it at all.

### Don't take it too far

Pulling complexity down makes sense when:
- The module can handle it better than callers (more context, more information)
- Many callers would otherwise each need to handle it

It doesn't make sense when:
- The module doesn't have enough context to make the right decision
- Different callers genuinely need different behavior
- It significantly complicates the module for marginal caller benefit

---

## Better Together or Better Apart?

When deciding whether to combine or separate code, use these indicators:

### Bring together when:
- **Information is shared** — Two pieces of code use the same knowledge (file format, protocol). Combining eliminates information leakage.
- **It simplifies the interface** — Combining two methods into one that handles both operations reduces what callers need to know.
- **It eliminates duplication** — Repeated code patterns suggest shared knowledge that should be centralized.

### Keep apart when:
- **General-purpose vs special-purpose** — General code should not contain special-case logic. Keep special-purpose code in a higher layer that calls the general code.
- **No information sharing** — If two pieces of code are truly independent with no shared knowledge, separation reduces cognitive load.
- **Each piece is independently useful** — If either half has no value alone, they probably belong together.

### Splitting methods: be careful

Splitting a method only makes sense if each piece has a clean, independent abstraction. A split that results in:
- Two methods that must be called in sequence → **bad split** (conjoined methods)
- Two methods where callers need both → **bad split** (didn't actually simplify)
- Two methods each with a clean interface → **good split**

The Clean Code advice to limit methods to small sizes can backfire. A 200-line method with a clear abstraction is better than 10 20-line methods where you need to read all 10 to understand the behavior.

---

## Designing for Performance

Design decisions that are hard to change later should consider performance from the start. But don't optimize prematurely.

### The fundamental approach:

Performance design has three levels. Most developers jump to level 3 (micro-optimize) and skip levels 1-2 where the real gains live:

1. **Design around the critical path** — Identify the common case. Make it fast and simple. Handle rare cases in separate, slower code paths. This is a design decision, not an optimization — and it's hard to change later.
2. **Minimize special cases on the hot path** — Every `if` on the critical path costs (branch prediction, cognitive load, code size). Design so the common case flows straight through with zero conditionals.
3. **Measure before micro-optimizing** — Never optimize based on intuition. Profile first. But steps 1-2 above are design decisions that should happen before any code is written.

### The critical path principle (from Ousterhout's RAMCloud example)

RAMCloud's buffer design illustrates the principle: instead of one general-purpose buffer class handling all cases, they designed a fast path for small buffers (the common case) and a separate path for large buffers (the rare case). The fast path avoided memory allocation entirely for small requests.

**The pattern**:
```
Is this the common case?
  YES → Optimize ruthlessly. Zero allocation. Straight-line code. No branches.
  NO  → Handle separately. Correctness over speed. Even if it's 10x slower,
        it happens rarely enough that it doesn't matter.
```

### Performance and simplicity are allies, not enemies

Ousterhout's key insight: the cleanest code is often the fastest code. Simple, straight-line code with minimal branching both runs faster AND is easier to understand. When you find yourself adding complexity for performance, you're probably optimizing the wrong thing.

**The "few things that matter" rule**: In any system, a small number of critical paths dominate performance. Find them. Make them clean and fast. Don't degrade the design of everything else for marginal gains.
