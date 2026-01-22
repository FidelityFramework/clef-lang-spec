# Discriminated Union Architecture (January 2026)

## Core Design Decision

DUs in F# Native are represented as **pointers to region-allocated storage** with **case eliminators** for type-safe payload extraction.

## Key Principles

1. **Pointer-based representation**: DU values are pointers, not inline structs
2. **Case eliminators**: Each case has a dedicated eliminator function that returns typed payload
3. **SRTP-driven layout**: Layouts are fully determined at compile time via SRTP resolution
4. **Heterogeneous storage**: Each case can have its own storage strategy
5. **BAREWire compatible**: Serialization follows BAREWire union conventions

## Why NOT inline structs with extractvalue

The naive `(tag, max-size-payload)` approach fails because:
- Different cases have different payload types (int vs float vs nested DU)
- MLIR values have single fixed types - can't use same SSA with different types
- Extractvalue requires struct type to match the value's declared type

## The Eliminator Pattern

```
func @Number.FloatVal.eliminate(%du_ptr: ptr) -> f64 {
    %typed_ptr = bitcast %du_ptr to ptr<struct<(i8, f64)>>
    %struct = load %typed_ptr
    %payload = extractvalue %struct[1]
    return %payload
}
```

The pointer bitcast allows case-specific struct types. This is **transliteration** (reinterpret same memory), not translation.

## PSG Changes Required

New SemanticKind nodes:
- `DUGetTag` - extract tag from DU pointer
- `DUEliminate` - extract typed payload using case eliminator
- `DUConstruct` - create DU value in arena

Baker should produce these instead of generic `FieldGet` nodes.

## Specification

Full spec at: `fsnative-spec/spec/discriminated-union-representation.md`
