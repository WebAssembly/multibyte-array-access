# Multibyte Array Access

## Summary

Reuse the existing linear memory load/store instructions to allow multibyte access to Wasm GC numeric, vector, and packed array types (e.g., `i8`, `i16`, `i32`, `i64`, `f32`, `f64`, `v128`).

## Motivation

A number of languages currently use GC arrays (such as `(array i8)` or arrays of other numeric types) as a backing store for custom data types and/or
byte buffers style objects (e.g., custom structs, Dart typed arrays, JVM byte arrays). Reading and
writing to these custom data types requires performing sequences of single-element operations that are
highly inefficient, and hinder performance.

## Proposal

### Semantics

The array versions of the load/store instructions follow the same data transformation semantics as the linear memory instructions, but they operate on GC arrays instead of linear memories.

#### Valid Array Types
- **Load instructions**: The array type `$t` must expand to `array (mut? t_elem)` where `t_elem` is a numeric type (`i8`, `i16`, `i32`, `i64`, `f32`, `f64`, `v128`).
- **Store instructions**: The array type `$t` must expand to `array (mut t_elem)` where `t_elem` is a numeric type (`i8`, `i16`, `i32`, `i64`, `f32`, `f64`, `v128`).

#### Validation

For type index `$t` and memory instruction `t_value.op`:
1. The type `$t` must be a valid type index in the module, and must expand to an array type.
2. The element type of the array must be a numeric type, vector type, or packed numeric type.
3. If it is a store instruction, the array type must be mutable.
4. The instruction has the following input and output types on the stack:
   - **Load** (`t_value.loadN_sx`):
     - Inputs: `[ref: (ref null $t), address: i32]`
     - Outputs: `[val: t_value]`
   - **Store** (`t_value.storeN`):
     - Inputs: `[ref: (ref null $t), address: i32, val: t_value]`
     - Outputs: `[]`
   - **Lane Load** (`v128.loadN_lane/spat/zero/xM_sx`):
     - Inputs: `[ref: (ref null $t), address: i32, vec: v128]`
     - Outputs: `[vec: v128]`
   - **Lane Store** (`v128.storeN_lane`):
     - Inputs: `[ref: (ref null $t), address: i32, vec: v128]`
     - Outputs: `[]`

#### Execution
1. **Null Check**:
   - If the array reference operand is null, execution traps.
2. **Effective Address Calculation**:
   - The effective address $ea$ is calculated as $address + offset$, where $address$ is the `i32` stack operand (interpreted as an unsigned 32-bit integer) and $offset$ is the static byte offset immediate from `memarg`.
3. **Bounds Check**:
   - Let $S$ be the size (in bytes) of the memory access (e.g., the byte size of type `t` for `t.load`/`t.store`, or $N/8$ for instructions with a size suffix $N$, such as `t.loadN_sx`, `t.storeN`, `v128.loadN_lane`, etc.).
   - Let $L$ be the length of the array (obtained via `array.len`).
   - Let $E$ be the element size in bytes of the array type `$t` (e.g., 1 for `i8`, 2 for `i16`, etc.).
   - The access is in bounds if $ea + S \le L \times E$.
   - If the access is out of bounds, execution traps.
4. **Operation**:
   - **Load**: Reads $S$ bytes from the array payload starting at byte offset $ea$, decodes them using little-endian byte order, possibly extends them to the result size, and pushes the resulting value onto the stack.
   - **Store**: Possibly wraps the value operand according to the target size, encodes it into $S$ bytes using little-endian byte order, and writes them to the array payload starting at byte offset $ea$.
   - **Lane Operations**: Loads/stores a single lane of the vector operand, possibly extended or wrapped, from/to the array payload at byte offset $ea$.
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
- The `memarg` is parsed normally for `flags` and `offset` (both `u32` in LEB128) as well as the `typeidx` (encoded as `u32` in LEB128). Bits 0-3 of `flags` represent the alignment exponent (expressed as `log_2(align)`).
- No memory index (`memidx`) is allowed. In a valid module, bit 6 of `flags` must be 0 when bit 4 is set.
- Immediately following the `memarg` fields (`flags` and `offset`), a `typeidx` (representing the type index of the array type `$t`) is encoded as a `u32` (LEB128).

Thus, the binary format of a multibyte array instruction is:
- `instr ::= op memarg` (for regular and SIMD load/store)
- `instr ::= op memarg laneidx` (for SIMD lane load/store)

Where `memarg` is defined as:
- `memarg ::= flags:u32 offset:u32 typeidx:u32` (where `flags & 0x50 = 0x10`)

### Text Format Syntax

In the text format, the instructions reuse the keyword names of the standard memory instructions. A type index immediate `(type $t)` is required.

Since these instructions operate on GC arrays and not linear memory, **a memory index is not allowed**. An offset immediate (`offset=N`) and an alignment immediate (`align=N`) are allowed (with `offset` defaulting to `0` and `align` defaulting to the instruction's natural alignment if omitted).

For lane instructions, a lane index immediate is required at the end.

#### Folded (S-Expression) Form
```wat
;; Load
(i32.load (type $t) [offset=N] [align=N] (local.get $array) (local.get $address))

;; Store
(i32.store (type $t) [offset=N] [align=N] (local.get $array) (local.get $address) (local.get $val))

;; Lane Load
(v128.load8_lane (type $t) [offset=N] [align=N] $lane (local.get $array) (local.get $address) (local.get $vec))

;; Lane Store
(v128.store8_lane (type $t) [offset=N] [align=N] $lane (local.get $array) (local.get $address) (local.get $vec))
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
