# clef-lang-spec Revision Philosophy

## Core Principle: Clean Break, Not Parallel Commentary

**A spec describes what IS, not what ISN'T.**

### What We Do NOT Do

1. **NO "NOT APPLICABLE" markers** - If something doesn't apply, REMOVE it entirely
2. **NO parallel commentary** - Don't constantly reference "unlike .NET..." or "in managed F#..."
3. **NO trip wires** - Don't leave remnants that could confuse Clef implementation work
4. **NO exhaustive rewrites** - We don't need to draw a parallel to .NET in every instance

### What We DO

1. **REMOVE** sections that are entirely about managed runtime (null, GC heap, runtime types, CLI interop)
2. **KEEP** sections that describe language semantics that apply to native compilation
3. **SIMPLIFY** where native execution is simpler than managed (no runtime = less to specify)
4. **STAND ALONE** - Changes should stand on their own merit, not as "not .NET"

## The Balanced Position

We are threading a needle:

1. **Readers should follow along** without excessive "baggage from the old way"
2. **The spec is a North Star** for re-engineering Clef itself
3. **We won't solve every problem upfront** - we'll find things along the way
4. **No trip wires** - the spec shouldn't complicate the Clef migration
5. **Clean break** - changes stand on their own

## Runtime Considerations Section

The "Evaluation of Elaborated Forms" section (expressions.md lines 2997+) is fundamentally about managed runtime:
- Global heap of object values
- Runtime types and dispatch maps
- null values everywhere
- Exception handling
- GC considerations

**Decision**: This section should be either:
1. **REMOVED entirely** - Clef's execution model is much simpler
2. **REPLACED with brief native execution semantics** in program-structure-and-execution.md

Native execution is:
- Static initialization at program start (deterministic)
- Entry point execution
- Values on stack/arena (no GC heap)
- No null, no runtime types
- Memory regions enforced at compile time

## Platform-Specific Examples

**REMOVE entirely** - not just mark as N/A:
- Windows.Forms examples
- System.Windows.* references
- Desktop-only patterns
- Any single-platform perspective

Clef is **multi-platform and multi-hardware by default**. Examples should be:
- Platform-agnostic (console I/O, pure computation)
- Or use Platform.Bindings pattern (platform-neutral in syntax)

## Judgment Calls

When encountering managed runtime content, ask:
1. Does this describe language semantics that apply to native? → KEEP
2. Does this describe managed runtime behavior? → REMOVE
3. Can this be simplified for native? → SIMPLIFY
4. Is there a native equivalent that's simpler? → REPLACE with simpler version
