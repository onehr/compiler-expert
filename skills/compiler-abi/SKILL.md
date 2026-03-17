---
name: compiler-abi
description: ABI and calling convention expert — covers System V AMD64 ABI, Windows x64 calling convention, struct layout and alignment, register vs stack parameter passing, red zone, and WebAssembly ABI. Use when generating native code that interoperates with C, implementing FFI, or designing calling conventions for a new language.
---

# ABI and Calling Conventions

---

## Why ABI Matters

ABI violations produce **silent correctness bugs**, not crashes. The caller and callee independently agree on where to find arguments and return values. A mismatch means the callee reads garbage from the wrong register or stack offset — no fault, no signal, just wrong answers.

Common sources of ABI bugs:
- Forgetting to preserve callee-saved registers across a call
- Passing a struct by value when the ABI requires passing a pointer (and vice versa)
- Misunderstanding how multi-return values map to registers vs. memory
- Ignoring alignment requirements for stack-passed arguments
- Red zone violations in leaf functions that call signal handlers

---

## System V AMD64 ABI (Linux / macOS)

### Integer and Pointer Parameter Passing

Registers in order: `rdi`, `rsi`, `rdx`, `rcx`, `r8`, `r9`

- First 6 integer/pointer arguments go in registers (left to right)
- Arguments 7+ go on the stack, pushed right to left
- Return value: `rax` (64-bit or smaller), `rdx:rax` (128-bit)

### Floating-Point Parameters

Registers in order: `xmm0`–`xmm7`

- First 8 FP arguments go in XMM registers
- Integer and FP argument slots are counted independently: a function `f(int, double, int)` uses `rdi`, `xmm0`, `rsi`

### Caller-Saved vs. Callee-Saved

| Registers | Ownership |
|---|---|
| `rax`, `rcx`, `rdx`, `rsi`, `rdi`, `r8`–`r11`, `xmm0`–`xmm15` | Caller-saved (may be clobbered by callee) |
| `rbx`, `rbp`, `r12`–`r15` | Callee-saved (callee must preserve) |
| `rsp` | Stack pointer — callee must restore before return |

### Struct Passing Rules

Classification algorithm (each 8-byte "eightbyte" of the struct is classified independently):
1. If any eightbyte contains a non-trivial field (pointer, SSE, x87), classify as MEMORY
2. If all scalar: INTEGER (integer/pointer fields) or SSE (only FP fields)
3. If struct fits in 2 registers and both eightbytes classify as INTEGER or SSE: pass in registers
4. Otherwise: pass by hidden pointer (caller allocates space, passes pointer as first argument)

**Key rule**: A struct larger than 16 bytes is always passed in memory.

### Red Zone

The 128 bytes below `rsp` (the red zone) are not clobbered by signal handlers or interrupt handlers — on Linux/macOS. Leaf functions that do not call other functions can use the red zone without adjusting `rsp`, saving two instructions.

**Do not use the red zone in kernel code** — interrupts on x86 do not respect it in ring 0.

---

## Windows x64 ABI

### Differences from System V

| Aspect | System V AMD64 | Windows x64 |
|---|---|---|
| Integer arg registers | `rdi`, `rsi`, `rdx`, `rcx`, `r8`, `r9` (6) | `rcx`, `rdx`, `r8`, `r9` (4) |
| FP arg registers | `xmm0`–`xmm7` (8) | `xmm0`–`xmm3` (4) |
| Arg slot counting | Independent (int + FP counted separately) | Unified (slot 1 is either `rcx` or `xmm0`) |
| Shadow space | None | 32 bytes must be allocated by caller before every call |
| Callee-saved XMM | None | `xmm6`–`xmm15` are callee-saved |
| Red zone | 128 bytes | None |
| Stack alignment | 16-byte at call site | 16-byte at call site |

### Shadow Space

The caller must always allocate 32 bytes of "home space" (shadow space) before a call, even if the function takes fewer than 4 arguments. The callee is allowed (but not required) to spill its register arguments into this space.

Forgetting shadow space is a common portability bug when porting System V code to Windows.

---

## WebAssembly ABI

WebAssembly has a linear memory model rather than a native register file. There is no hardware stack or physical register convention at the Wasm bytecode level.

### Import/Export Interface

- Functions exported from a Wasm module or imported from the host are typed with Wasm value types: `i32`, `i64`, `f32`, `f64`, `v128`, `funcref`, `externref`
- Structs and complex types do not exist at the Wasm boundary — they must be manually serialized into linear memory

### Calling C from Wasm (wasm32 C ABI)

Emscripten and wasi-sdk use a C ABI layered on top of Wasm:
- Pointers are `i32` (wasm32) or `i64` (wasm64)
- Structs are passed by writing into linear memory and passing a pointer
- Return values larger than one Wasm value use an out-pointer passed as the first argument

### Key Design Implication

When generating Wasm from a higher-level language, any complex type crossing a module boundary must go through linear memory. Plan for a serialization layer at the FFI boundary.

---

## Struct Layout Rules

### Alignment

- Each field is aligned to its own size (or the target's ABI-specified alignment, whichever is smaller)
- `bool`/`char`: 1-byte aligned; `short`: 2; `int`/`float`: 4; `long`/`double`/pointers: 8
- Struct overall alignment = alignment of its most-aligned member
- Struct size is padded to a multiple of its alignment

### Padding Example

```c
struct S {
    char  a;    // offset 0, size 1
    // 3 bytes padding
    int   b;    // offset 4, size 4
    char  c;    // offset 8, size 1
    // 7 bytes padding
    double d;   // offset 16, size 8
};  // total size: 24
```

### Packed Structs

`__attribute__((packed))` (GCC/Clang) or `#pragma pack(1)` removes padding. Costs:
- Unaligned loads (slow or trapping on some architectures)
- Cannot pass packed struct fields by pointer to APIs expecting alignment
- Use only for wire formats and file formats, never for in-memory performance-sensitive data

---

## Varargs (C va_list) Implementation

Under System V AMD64:
- `va_list` is a struct tracking `gp_offset` (next GPR arg consumed), `fp_offset` (next FPR arg consumed), and overflow area pointer
- `va_arg` reads from the register save area (if slots remain) then falls back to the overflow area
- Variadic functions must always save all 6 GPR and 8 FPR argument registers to a save area in their prologue, regardless of how many are actually used

Under Windows x64:
- `va_list` is simply a pointer into the shadow space / stack area (simpler: all arguments occupy sequential slots)

---

## Planned Sources

- System V Application Binary Interface AMD64 Architecture Processor Supplement (psABI)
- chibicc ABI implementation (Rui Ueyama) — minimal C compiler, clear ABI encoding
- LLVM DataLayout and TargetLowering source
- Windows x64 ABI documentation (Microsoft Learn)
- WebAssembly component model spec

---

## Navigation

**Trace Up ↑** — skills that route here
- `compiler-codegen` — instruction emission for function calls/returns requires ABI decisions
- `compiler-expert` — routes here on calling convention / struct layout / FFI questions

**Trace Down ↓** — this is a leaf skill at the back-end layer
- (none — ABI knowledge is consumed by codegen directly)

**Related →** — closely related back-end skills
- `compiler-codegen` — tightly coupled: ABI determines how calls/returns are emitted
- `compiler-linker` — ABI affects symbol mangling and relocation types
- `compiler-runtime` — ABI is part of the runtime contract (e.g., stack layout for GC)
