# Multibyte Array Access

## Summary

Reuse the existing linear memory load/store instructions to allow multibyte access to Wasm GC numeric (and packed) array types (e.g., `i8`, `i16`, `i32`, `i64`, `f32`, `f64`).

## Motivation

A number of languages currently use GC arrays (such as `(array i8)` or arrays of other numeric types) as a backing store for custom data types and/or
byte buffers style objects (e.g., custom structs, Dart typed arrays, JVM byte arrays). Reading and
writing to these custom data types requires performing sequences of single-element operations that are
highly inefficient, and hinder performance.

## Proposal

### Semantics

The array versions of the load/store instructions follow the same data transformation semantics as the linear memory instructions, but they operate on GC arrays instead of linear memories.

#### Valid Array Types
- **Load instructions**: The array type `$t` must expand to `array (mut? t_elem)` where `t_elem` is a numeric type (`i8`, `i16`, `i32`, `i64`, `f32`, `f64`).
- **Store instructions**: The array type `$t` must expand to `array (mut t_elem)` where `t_elem` is a numeric type (`i8`, `i16`, `i32`, `i64`, `f32`, `f64`).

#### Validation

For type index `$t` and memory instruction `t_value.op`:
1. The type `$t` must be a valid type index in the module, and must expand to an array type.
2. The element type of the array must be a numeric type or packed numeric type.
3. If it is a store instruction, the array type must be mutable.
4. The instruction has the following input and output types on the stack:
   - **Load** (`t_value.load`):
     - Inputs: `[ref: (ref null $t), index: i32]`
     - Outputs: `[val: t_value]`
   - **Store** (`t_value.store`):
     - Inputs: `[ref: (ref null $t), index: i32, val: t_value]`
     - Outputs: `[]`
   - **Lane Load** (`v128.loadN_lane`):
     - Inputs: `[ref: (ref null $t), index: i32, vec: v128]`
     - Outputs: `[vec: v128]`
   - **Lane Store** (`v128.storeN_lane`):
     - Inputs: `[ref: (ref null $t), index: i32, vec: v128]`
     - Outputs: `[]`

#### Execution
1. **Null Validation**:
   - If the array reference operand is null, execution traps with a null reference error.
2. **Effective Address Calculation**:
   - The effective address $ea$ is calculated as $index + offset$, where $index$ is the `i32` stack operand (interpreted as an unsigned 32-bit integer) and $offset$ is the static byte offset immediate from `memarg`.
3. **Bounds Check**:
   - Let $S$ be the size (in bytes) of the memory access (e.g., 4 for `i32.load`, 16 for `v128.load`).
   - Let $L$ be the length of the array (obtained via `array.len`).
   - Let $E$ be the element size in bytes of the array type `$t` (e.g., 1 for `i8`, 2 for `i16`, etc.).
   - The access is in bounds if $ea \ge 0$ and $ea + S \le L \times E$.
   - If the access is out of bounds, execution traps with an out-of-bounds array access error.
4. **Operation**:
   - **Load**: Reads $S$ bytes from the array payload starting at byte offset $ea$, decodes them using little-endian byte order, and pushes the resulting value onto the stack.
   - **Store**: Encodes the value operand into $S$ bytes using little-endian byte order, and writes them to the array payload starting at byte offset $ea$.
   - **Lane Operations**: Loads/stores a single lane of the vector operand from/to the array payload at byte offset $ea$.
5. **Alignment**:
   - The alignment value (expressed as power of 2 exponent in `memarg`) does not affect execution semantics, serving only as a hint for access alignment.

### Supported Instructions

The proposal supports the following memory load and store instructions:

| Instruction | Category | Operation |
| :--- | :--- | :--- |
| `i32.load` | Regular | Load |
| `i64.load` | Regular | Load |
| `f32.load` | Regular | Load |
| `f64.load` | Regular | Load |
| `i32.load8_s` | Regular | Load |
| `i32.load8_u` | Regular | Load |
| `i32.load16_s` | Regular | Load |
| `i32.load16_u` | Regular | Load |
| `i64.load8_s` | Regular | Load |
| `i64.load8_u` | Regular | Load |
| `i64.load16_s` | Regular | Load |
| `i64.load16_u` | Regular | Load |
| `i64.load32_s` | Regular | Load |
| `i64.load32_u` | Regular | Load |
| `i32.store` | Regular | Store |
| `i64.store` | Regular | Store |
| `f32.store` | Regular | Store |
| `f64.store` | Regular | Store |
| `i32.store8` | Regular | Store |
| `i32.store16` | Regular | Store |
| `i64.store8` | Regular | Store |
| `i64.store16` | Regular | Store |
| `i64.store32` | Regular | Store |
| `v128.load` | SIMD | Load |
| `v128.store` | SIMD | Store |
| `v128.load8x8_s` | SIMD | Load |
| `v128.load8x8_u` | SIMD | Load |
| `v128.load16x4_s` | SIMD | Load |
| `v128.load16x4_u` | SIMD | Load |
| `v128.load32x2_s` | SIMD | Load |
| `v128.load32x2_u` | SIMD | Load |
| `v128.load8_splat` | SIMD | Load (Splat) |
| `v128.load16_splat` | SIMD | Load (Splat) |
| `v128.load32_splat` | SIMD | Load (Splat) |
| `v128.load64_splat` | SIMD | Load (Splat) |
| `v128.load32_zero` | SIMD | Load (Zero) |
| `v128.load64_zero` | SIMD | Load (Zero) |
| `v128.load8_lane` | SIMD | Load (Lane) |
| `v128.load16_lane` | SIMD | Load (Lane) |
| `v128.load32_lane` | SIMD | Load (Lane) |
| `v128.load64_lane` | SIMD | Load (Lane) |
| `v128.store8_lane` | SIMD | Store (Lane) |
| `v128.store16_lane` | SIMD | Store (Lane) |
| `v128.store32_lane` | SIMD | Store (Lane) |
| `v128.store64_lane` | SIMD | Store (Lane) |

### Encoding

The array versions of the instructions reuse the existing opcodes for the standard memory instructions.

An instruction is determined to be an array access instruction if **bit 4** (value `0x10`) of the `flags` field in its `memarg` immediate is set.

When bit 4 of `flags` is set:
- The instruction operates on a GC array instead of linear memory.
- The `memarg` is parsed normally for `flags` and `offset` (both `u32` in LEB128). Bits 0-3 of `flags` represent the alignment exponent (expressed as `log_2(align)`).
- **No memory index (`mem_idx`) is read/parsed**, even if bit 6 of `flags` is set. In a valid module, bit 6 of `flags` MUST be 0 when bit 4 is set. This avoids conflicts where the type index would be misparsed as a memory index.
- Immediately following the `memarg` fields (`flags` and `offset`), a `typeidx` (representing the type index of the array type `$t`) is encoded as a `u32` (LEB128).
- For lane instructions (e.g., `v128.load8_lane`, `v128.store8_lane`), the 1-byte `laneidx` is encoded immediately *after* the `typeidx`.

Thus, the binary format of a multibyte array instruction is:
`instr ::= op memarg_array typeidx` (for regular and SIMD load/store)
`instr ::= op memarg_array typeidx laneidx` (for SIMD lane load/store)

Where:
- `memarg_array ::= flags:u32 offset:u32` (where `flags & 0x10 != 0`)

### Text Format Syntax

In the text format, the instructions reuse the keyword names of the standard memory instructions. A type index immediate `(type $t)` is required.

Since these instructions operate on GC arrays and not linear memory, **a memory index is not allowed**. An offset immediate (`offset=N`) and an alignment immediate (`align=N`) are allowed (with `offset` defaulting to `0` and `align` defaulting to the instruction's natural alignment if omitted).

For lane instructions, a lane index immediate is required at the end.

#### Folded (S-Expression) Form
```wat
;; Load
(i32.load (type $t) [offset=N] [align=N] (local.get $array) (local.get $index))

;; Store
(i32.store (type $t) [offset=N] [align=N] (local.get $array) (local.get $index) (local.get $val))

;; Lane Load
(v128.load8_lane (type $t) [offset=N] [align=N] $lane (local.get $array) (local.get $index) (local.get $vec))

;; Lane Store
(v128.store8_lane (type $t) [offset=N] [align=N] $lane (local.get $array) (local.get $index) (local.get $vec))
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
