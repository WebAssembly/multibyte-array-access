# Multibyte Array Access

## Summary

Reuse the exiting linear memory load/store instructions to allow multibyte access to `(array i8)`.

## Motivation

A number of languages currently use `(array i8)` as a backing store for custom data types and/or
byte buffers style objects (e.g., custom structs, Dart typed arrays, JVM byte arrays). Reading and
writing to these custom data types requires performing sequences of single-byte operations that are
highly inefficient, and hinder performance.

## Proposal

### Semantics

The array versions of the load/store instructions will largely follow the same semantics as the
linear memory instructions. The main differences are as follows:
  - a type index immediate and array reference argument are required before the address argument
  - a memory index or align immediate are not allowed

Valid array types for the instructions:
  - load:`expand($t) = array i8`
  - store:`expand($t) = array mut i8`

### Encoding

Use a reserved bit (7) in the `memarg` field to signal that the load/store instructions will be
operating on an array. As mentioned above there will be a type index immediate and array reference
argument.

### Text Format Syntax

```
i32.load array $type_index (<array ref>) (<address>)

i32.store array $type_index (<array ref>) (<address>) (<value>)
```

## Alternatives

In the [original discussion](https://github.com/WebAssembly/gc/issues/395) a few alternatives were
discussed. A summary of the different options:

- Use new array instructions to mirror all the current load/store instructions. (The CG preferred
  reusing existing instructions in a straw poll)
- Reinterpret cast on array types.
- Pinning part of the array to a memory.
- Use the [slice proposal](https://github.com/WebAssembly/design/issues/1555).

The above options do have some advantages such as not introducing more instructions or reusing other
proposals. However, they are all more complicated than adding relatively straightforward array
instructions that would achieve the desired results.

## Previous Discussion

[Design Issue](https://github.com/WebAssembly/design/issues/1569). [GC Feature Request
Issue](https://github.com/WebAssembly/gc/issues/395).
