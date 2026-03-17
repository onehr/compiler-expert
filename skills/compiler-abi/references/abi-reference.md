# ABI Reference: Calling Conventions and Struct Layout

*Sources: chibicc compiler (Rui Iino), OSDev Wiki System V ABI, AMD64 psABI spec.*

---

## 1. Why ABI Matters

### Silent Correctness Bugs (Not Crashes)

ABI violations rarely produce immediate crashes. Instead they produce wrong values, corrupted state, or non-deterministic behavior. Examples:

- A function expects its first integer argument in `%rdi`, but your code-generator puts it in `%rax`. The callee reads garbage from `%rdi` and computes a wrong result. No segfault — just bad output.
- A struct returned from a function is expected in `%rax`/`%rdx`, but your compiler puts it somewhere else. The caller reads garbage registers. Undefined behavior that may only appear in release builds.
- Forgetting to align the stack to 16 bytes before a `call` instruction causes SSE/AVX instructions inside the callee to fault on misaligned memory — but only if those instructions happen to touch the stack, which depends on what the callee does.

The key insight: **ABI bugs are data-dependent and compiler/optimization-level-dependent.** They are among the hardest bugs to diagnose.

### When You Need Strict ABI Compliance

- Calling functions in system libraries (`libc`, `libm`, OS syscall wrappers).
- Generating code that will be linked with object files produced by GCC/Clang.
- Implementing a language runtime that the end user's C code will call into.
- Writing a compiler front-end whose output must link with arbitrary C code.

### When You Can Define Your Own ABI

- All callers and callees are under your control (e.g., a JIT where you compile all code).
- An embedded system with no pre-compiled libraries.
- A language that never interoperates with C at the binary level.

In these cases you can use a simpler calling convention — e.g., all arguments on the stack, no register assignments — at the cost of performance and interoperability.

---

## 2. System V AMD64 ABI (Linux/macOS) — The Complete Rules

[SysV-ABI]

### 2.1 Integer and Pointer Argument Registers (in order)

| Argument position | 64-bit register | 32-bit | 16-bit | 8-bit |
|---|---|---|---|---|
| 1st | `%rdi` | `%edi` | `%di` | `%dil` |
| 2nd | `%rsi` | `%esi` | `%si` | `%sil` |
| 3rd | `%rdx` | `%edx` | `%dx` | `%dl` |
| 4th | `%rcx` | `%ecx` | `%cx` | `%cl` |
| 5th | `%r8` | `%r8d` | `%r8w` | `%r8b` |
| 6th | `%r9` | `%r9d` | `%r9w` | `%r9b` |

Maximum 6 integer/pointer arguments in registers. Arguments 7+ go on the stack in **right-to-left order** (pushed such that argument 7 is at the lowest address, i.e., closest to the called function's frame).

[chibicc] Defined as:
```c
#define GP_MAX 6
static char *argreg64[] = {"%rdi", "%rsi", "%rdx", "%rcx", "%r8", "%r9"};
static char *argreg32[] = {"%edi", "%esi", "%edx", "%ecx", "%r8d", "%r9d"};
static char *argreg16[] = {"%di",  "%si",  "%dx",  "%cx",  "%r8w", "%r9w"};
static char *argreg8[]  = {"%dil", "%sil", "%dl",  "%cl",  "%r8b", "%r9b"};
```

### 2.2 Floating-Point Argument Registers

Arguments of type `float` or `double` are passed in `%xmm0` through `%xmm7` (8 registers maximum).

[chibicc] Defined as:
```c
#define FP_MAX 8
```

`long double` (80-bit x87 extended precision) is **always** passed on the stack, taking 16 bytes (aligned to 16).

### 2.3 Return Values

| Type | Register(s) |
|---|---|
| `int`, pointer, most scalars | `%rax` |
| 128-bit integer | `%rdx:%rax` (high 64 bits in `%rdx`) |
| `float` | `%xmm0` |
| `double` | `%xmm0` |
| Small struct (≤ 16 bytes, integer fields) | `%rax` (low), `%rdx` (high) |
| Small struct (≤ 16 bytes, float fields) | `%xmm0` (low), `%xmm1` (high) |
| Large struct (> 16 bytes) | Caller allocates buffer, passes pointer in `%rdi` as hidden first argument; callee returns that same pointer in `%rax` |

[SysV-ABI] "If the return value does not fit in rax and rdx, then the caller must reserve storage space and pass a pointer in rdi as-if it was the first argument. The callee must return the same pointer in rax."

### 2.4 Caller-Saved vs Callee-Saved Registers

**Caller-saved (scratch registers)** — the callee may overwrite these freely; the caller must save them before a call if their values are needed afterward:

```
%rax, %rcx, %rdx, %rsi, %rdi, %r8, %r9, %r10, %r11
%xmm0–%xmm15  (all XMM/YMM/ZMM registers are caller-saved)
```

**Callee-saved (preserved registers)** — the callee must restore these before returning:

```
%rbx, %rbp, %rsp, %r12, %r13, %r14, %r15
```

[SysV-ABI] "Functions preserve the registers rbx, rsp, rbp, r12, r13, r14, and r15; while rax, rdi, rsi, rdx, rcx, r8, r9, r10, r11 are scratch registers."

Note: `%rsp` is technically callee-saved in the sense that the stack pointer must be restored upon return, but it is restored implicitly by the `ret` instruction popping the return address.

### 2.5 The Red Zone

[SysV-ABI] The 128 bytes **below** `%rsp` (i.e., at addresses `%rsp - 128` through `%rsp - 1`) are the **red zone**. The ABI guarantees that asynchronous events (signals, interrupts in user space) will not clobber this area.

**When it matters:**
- Leaf functions (those that call no other functions) can use up to 128 bytes of local variables without adjusting `%rsp`. This saves two instructions (`sub`/`add`) in tight inner loops.
- Compilers like GCC exploit this automatically.

**When to disable it (`-mno-red-zone`):**
- Kernel code: hardware interrupts do not honor the red zone. The CPU pushes interrupt state to wherever `%rsp` points, potentially overwriting red-zone data. All x86-64 kernel code must be compiled with `-mno-red-zone` or handle interrupts on a separate stack.
- Signal-heavy programs that override signal handlers in assembly.

**Practical note for compiler writers:** If your generated code uses `sub $N, %rsp` for the function frame, you already avoid the red zone issue for non-leaf functions. The problem only arises if you assume the red zone is safe while also expecting signals or interrupts.

### 2.6 Stack Alignment

[SysV-ABI] The stack must be **16-byte aligned immediately before the `call` instruction executes**. The `call` instruction pushes an 8-byte return address, so at the entry to the callee, `%rsp` is 8-byte aligned (i.e., `%rsp % 16 == 8`). The callee's prologue (`push %rbp`) restores 16-byte alignment.

**The rule in practice:**
- Before any `call`, ensure `(%rsp) mod 16 == 0`.
- The standard prologue sequence:
  ```asm
  push %rbp          ; %rsp now misaligned by 8
  mov %rsp, %rbp
  sub $N, %rsp       ; N must be a multiple of 16 to maintain alignment
  ```
- If you push an odd number of values before a `call`, insert a padding `sub $8, %rsp`.

[chibicc] Alignment enforcement before calls:
```c
// In push_args():
if ((depth + stack) % 2 == 1) {
    println("  sub $8, %%rsp");
    depth++;
    stack++;
}
```
`depth` tracks how many 8-byte slots are currently on the stack (each `push`/`pushf` increments it). If the total would be odd (misaligned), chibicc inserts an 8-byte pad.

[chibicc] Stack frame size is always rounded to 16:
```c
fn->stack_size = align_to(bottom, 16);
```

### 2.7 Variadic Functions: The `%al` Convention

[SysV-ABI] For variadic functions (`printf`, etc.), before the `call` instruction, `%al` (the low byte of `%rax`) must be set to the **number of floating-point arguments passed in XMM registers**. This allows the callee's vararg implementation to know how many XMM registers to save when constructing the `va_list`.

[chibicc] Call emission:
```c
println("  mov %%rax, %%r10");   // save function pointer
println("  mov $%d, %%rax", fp); // fp = count of float args used
println("  call *%%r10");
```

For non-variadic functions, setting `%al` is not required (and chibicc skips it), but it is harmless.

---

## 3. Struct/Union Classification Algorithm

[SysV-ABI] Every struct is classified before a call to determine how it is passed.

### 3.1 The Eightbyte Classification

The struct is divided into **eightbyte chunks** (8-byte regions). Each chunk is assigned a class:

- **INTEGER**: the chunk contains an integer, pointer, or mixed-type data where any non-float byte exists.
- **SSE**: the chunk contains only `float` or `double` data.
- **MEMORY**: the struct is too large or has alignment requirements that prevent register passing.

**Rules (simplified):**
1. If the total size > 16 bytes → MEMORY (pass on stack).
2. If the struct has any unaligned field or `__attribute__((packed))` effectively causes misalignment → MEMORY.
3. Otherwise, classify each 8-byte chunk independently:
   - If all bytes in the chunk are floating-point types → SSE class.
   - Otherwise → INTEGER class.

### 3.2 Passing Decision

[chibicc] Uses the helper `has_flonum(ty, lo, hi, offset)` which recursively walks struct members and returns `true` only if every byte in the range `[lo, hi)` is covered by a `float` or `double` field:

```c
static bool has_flonum(Type *ty, int lo, int hi, int offset) {
  if (ty->kind == TY_STRUCT || ty->kind == TY_UNION) {
    for (Member *mem = ty->members; mem; mem = mem->next)
      if (!has_flonum(mem->ty, lo, hi, offset + mem->offset))
        return false;
    return true;
  }
  if (ty->kind == TY_ARRAY) {
    for (int i = 0; i < ty->array_len; i++)
      if (!has_flonum(ty->base, lo, hi, offset + ty->base->size * i))
        return false;
    return true;
  }
  return offset < lo || hi <= offset || ty->kind == TY_FLOAT || ty->kind == TY_DOUBLE;
}
```

Convenience wrappers:
```c
static bool has_flonum1(Type *ty) { return has_flonum(ty, 0, 8, 0); }  // first 8 bytes
static bool has_flonum2(Type *ty) { return has_flonum(ty, 8, 16, 0); } // second 8 bytes
```

### 3.3 Decision Table with Examples

| Struct layout | Size | First 8 bytes | Second 8 bytes | Passed in |
|---|---|---|---|---|
| `{int a; int b;}` | 8 | INTEGER | n/a | `%rdi` (both ints packed into one register) |
| `{long a; long b;}` | 16 | INTEGER | INTEGER | `%rdi`, `%rsi` |
| `{float a; float b;}` | 8 | SSE | n/a | `%xmm0` |
| `{double a; double b;}` | 16 | SSE | SSE | `%xmm0`, `%xmm1` |
| `{int a; double b;}` | 16 | INTEGER | SSE | `%rdi`, `%xmm0` |
| `{long a; long b; long c;}` | 24 | MEMORY | MEMORY | stack (size > 16) |
| `{char buf[17];}` | 17 | MEMORY | MEMORY | stack (size > 16) |

[chibicc] The size threshold check:
```c
case TY_STRUCT:
case TY_UNION:
  if (ty->size > 16) {
    arg->pass_by_stack = true;
    stack += align_to(ty->size, 8) / 8;
  } else {
    bool fp1 = has_flonum1(ty);
    bool fp2 = has_flonum2(ty);
    if (fp + fp1 + fp2 < FP_MAX && gp + !fp1 + !fp2 < GP_MAX) {
      fp = fp + fp1 + fp2;
      gp = gp + !fp1 + !fp2;
    } else {
      arg->pass_by_stack = true;
      stack += align_to(ty->size, 8) / 8;
    }
  }
```

Also: if there are not enough registers left for **all** chunks (both integer and float slots must be available simultaneously), the entire struct falls back to the stack.

### 3.4 Returning Structs: The Hidden First Argument

[SysV-ABI] If a function's return type classifies as MEMORY (struct > 16 bytes), the **caller**:
1. Allocates a buffer (typically on its stack frame) large enough to hold the return value.
2. Passes a pointer to that buffer in `%rdi` as a **hidden first argument** (shifting all other arguments by one register).

The **callee**:
1. Writes the return value into `*(%rdi)`.
2. Returns `%rdi` in `%rax`.

[chibicc] Hidden argument handling:
```c
// In push_args(): count the hidden pointer as using one GP register
if (node->ret_buffer && node->ty->size > 16)
    gp++;

// At call site: load the buffer address into the first GP register
if (node->ret_buffer && node->ty->size > 16) {
    println("  lea %d(%%rbp), %%rax", node->ret_buffer->offset);
    push();
}
// Then later: pop it into %rdi (argreg64[0])
if (node->ret_buffer && node->ty->size > 16)
    pop(argreg64[gp++]);
```

For small structs (≤ 16 bytes) that pass through registers, the callee writes the struct's bytes into the appropriate registers before returning, and the caller unpacks them. [chibicc] `copy_ret_buffer()` and `copy_struct_reg()` handle these paths.

---

## 4. Practical Implementation Pattern (from chibicc)

[chibicc] This section follows the complete code path for generating a function call.

### 4.1 Counting Integer vs Float Args

```c
int stack = 0, gp = 0, fp = 0;
// (see push_args above for the full loop)
```

`gp` counts how many integer/pointer registers have been consumed (max 6).
`fp` counts how many XMM registers have been consumed (max 8).
`stack` counts how many extra 8-byte slots need to be pushed onto the stack.

### 4.2 Stack Frame Setup

[chibicc] Prologue emitted by `emit_text()`:
```c
println("  push %%rbp");
println("  mov %%rsp, %%rbp");
println("  sub $%d, %%rsp", fn->stack_size);
```

`fn->stack_size` is computed by `assign_lvar_offsets()`:
```c
fn->stack_size = align_to(bottom, 16);
```

Local variables are assigned negative offsets from `%rbp`. Pass-by-stack parameters have positive offsets from `%rbp` (starting at `+16`, since `+8` is the saved return address and `+0` is the saved `%rbp`).

### 4.3 Handling Struct-By-Value Arguments

[chibicc] `push_struct()` copies a struct onto the stack when passing by stack:
```c
static void push_struct(Type *ty) {
  int sz = align_to(ty->size, 8);
  println("  sub $%d, %%rsp", sz);
  depth += sz / 8;
  for (int i = 0; i < ty->size; i++) {
    println("  mov %d(%%rax), %%r10b", i);
    println("  mov %%r10b, %d(%%rsp)", i);
  }
}
```

This is a byte-by-byte copy using `%r10b` as a scratch byte register. A production compiler would use wider moves (`movq`) for efficiency.

### 4.4 The Complete Function Call Emission Pattern

[chibicc] `gen_expr()` for `ND_FUNCALL`:

```c
// Step 1: Evaluate all arguments and push them to the stack as temporaries.
//         push_args() also computes which go to registers vs stack,
//         and inserts alignment padding if needed.
int stack_args = push_args(node);

// Step 2: Evaluate the function address/pointer into %rax.
gen_expr(node->lhs);

// Step 3: Pop register arguments from the stack into their registers.
int gp = 0, fp = 0;
if (node->ret_buffer && node->ty->size > 16)
    pop(argreg64[gp++]);  // hidden struct-return pointer goes first

for (Node *arg = node->args; arg; arg = arg->next) {
    // pop each register argument from the temporary stack into its register
    // (structs may need two pops, floats use popf into xmmN, etc.)
}

// Step 4: Set %al = number of float args (for variadic functions).
//         Save the function pointer (was in %rax) to %r10.
println("  mov %%rax, %%r10");
println("  mov $%d, %%rax", fp);
println("  call *%%r10");

// Step 5: Clean up stack arguments.
println("  add $%d, %%rsp", stack_args * 8);
depth -= stack_args;
```

**Why save to `%r10` before the call?** Because `%rax` is being repurposed to hold the float-argument count. `%r10` is a caller-saved scratch register with no special meaning in the calling convention.

### 4.5 Saving Incoming Arguments (Callee Side)

[chibicc] `emit_text()` saves register arguments to their stack slots immediately after the prologue:

```c
int gp = 0, fp = 0;
for (Obj *var = fn->params; var; var = var->next) {
    if (var->offset > 0) continue; // already on stack (pass-by-stack param)

    switch (ty->kind) {
    case TY_STRUCT:
    case TY_UNION:
        // Small struct: save each register chunk
        if (has_flonum(ty, 0, 8, 0))
            store_fp(fp++, var->offset, MIN(8, ty->size));
        else
            store_gp(gp++, var->offset, MIN(8, ty->size));
        if (ty->size > 8) {
            if (has_flonum(ty, 8, 16, 0))
                store_fp(fp++, var->offset + 8, ty->size - 8);
            else
                store_gp(gp++, var->offset + 8, ty->size - 8);
        }
        break;
    case TY_FLOAT:
    case TY_DOUBLE:
        store_fp(fp++, var->offset, ty->size);
        break;
    default:
        store_gp(gp++, var->offset, ty->size);
    }
}
```

[chibicc] `store_gp()` and `store_fp()` emit the appropriate `mov` variants:
```c
static void store_gp(int r, int offset, int sz) {
    switch (sz) {
    case 1: println("  mov %s, %d(%%rbp)", argreg8[r], offset); return;
    case 2: println("  mov %s, %d(%%rbp)", argreg16[r], offset); return;
    case 4: println("  mov %s, %d(%%rbp)", argreg32[r], offset); return;
    case 8: println("  mov %s, %d(%%rbp)", argreg64[r], offset); return;
    default:
        for (int i = 0; i < sz; i++) {
            println("  mov %s, %d(%%rbp)", argreg8[r], offset + i);
            println("  shr $8, %s", argreg64[r]);
        }
    }
}
```

---

## 5. Windows x64 ABI — Key Differences

The Windows x64 ABI (used on Windows only; macOS/Linux use System V) differs significantly.

### 5.1 Argument Registers (Only 4)

| Argument | Integer/Pointer | Float/Double |
|---|---|---|
| 1st | `%rcx` | `%xmm0` |
| 2nd | `%rdx` | `%xmm1` |
| 3rd | `%r8` | `%xmm2` |
| 4th | `%r9` | `%xmm3` |

**Critical difference**: Integer and float arguments share the same 4 slots. If argument 1 is `float` and argument 2 is `int`, they go in `%xmm0` and `%rdx` respectively — not `%rcx`. The position determines which register is used, not the running count by type.

Arguments 5+ go on the stack.

### 5.2 Shadow Space (Home Space) — 32 Bytes

The **caller** must allocate 32 bytes (4 × 8 bytes) on the stack **before** the `call` instruction, even if the function takes fewer than 4 arguments. This space is called the "shadow space" or "home space." The callee may (but is not required to) use it to spill its register arguments.

```asm
sub $32, %rsp     ; shadow space
sub $8, %rsp      ; optional: align to 16 bytes if needed
call foo
add $40, %rsp     ; clean up shadow space + alignment pad
```

**This is the most commonly forgotten Windows ABI rule.** Forgetting it causes the callee to potentially overwrite the caller's data if it uses the home space for spilling.

### 5.3 Callee-Saved XMM Registers

On Windows x64, `%xmm6` through `%xmm15` are **callee-saved**. On System V, all XMM registers are caller-saved. This means Windows code that modifies `%xmm6`–`%xmm15` must save and restore them.

**Callee-saved on Windows x64:**
- Integer: `%rbx`, `%rbp`, `%rdi`, `%rsi`, `%rsp`, `%r12`–`%r15`
- XMM: `%xmm6`–`%xmm15`

**Note**: `%rdi` and `%rsi` are callee-saved on Windows but caller-saved on System V.

### 5.4 No Red Zone

Windows x64 has **no red zone**. Code must not use memory below `%rsp` as scratch space without first decrementing `%rsp`.

### 5.5 Summary Comparison Table

| Feature | System V AMD64 | Windows x64 |
|---|---|---|
| Integer arg registers | rdi, rsi, rdx, rcx, r8, r9 | rcx, rdx, r8, r9 |
| Max integer args in regs | 6 | 4 |
| Float arg registers | xmm0–xmm7 | xmm0–xmm3 (same slots as int) |
| Max float args in regs | 8 | 4 |
| Shadow space | None | 32 bytes always |
| Red zone | 128 bytes | None |
| Callee-saved XMM | None | xmm6–xmm15 |
| rdi/rsi | Caller-saved | Callee-saved |

---

## 6. Struct Layout Rules

[chibicc] `struct_decl()` in `parse.c` implements the standard C struct layout algorithm.

### 6.1 Field Alignment

Each field is placed at the **next offset that is a multiple of the field's alignment requirement**. Alignment of a scalar type equals its size:
- `char`: align 1
- `short`: align 2
- `int`: align 4
- `long`, pointer: align 8
- `float`: align 4
- `double`: align 8
- `long double`: align 16

[chibicc] `align_to()` utility:
```c
int align_to(int n, int align) {
    return (n + align - 1) / align * align;
}
```

[chibicc] Struct member offset assignment:
```c
int bits = 0;  // current bit position within the struct
for (Member *mem = ty->members; mem; mem = mem->next) {
    if (!ty->is_packed)
        bits = align_to(bits, mem->align * 8);  // pad to alignment
    mem->offset = bits / 8;
    bits += mem->ty->size * 8;
    if (!ty->is_packed && ty->align < mem->align)
        ty->align = mem->align;  // struct alignment = max field alignment
}
ty->size = align_to(bits, ty->align * 8) / 8;  // round size up to alignment
```

### 6.2 Struct Alignment

The struct's own alignment equals the **maximum alignment of any of its fields**. This ensures that in an array of structs, every element is properly aligned.

### 6.3 Struct Size

The struct's total size is **rounded up to a multiple of the struct's alignment**. This is the trailing padding rule.

Example:
```c
struct { char a; int b; char c; };
// a: offset 0 (align 1)
// padding: 3 bytes (to align b to 4)
// b: offset 4 (align 4)
// c: offset 8 (align 1)
// trailing padding: 3 bytes (to round size up to multiple of 4)
// total size: 12 bytes, alignment: 4
```

### 6.4 Union Layout

[chibicc] `union_decl()`: all members have offset 0. The union's size is the maximum member size, rounded up to the union's alignment (which is the maximum member alignment).

```c
for (Member *mem = ty->members; mem; mem = mem->next) {
    if (ty->align < mem->align)
        ty->align = mem->align;
    if (ty->size < mem->ty->size)
        ty->size = mem->ty->size;
}
ty->size = align_to(ty->size, ty->align);
```

### 6.5 Packed Structs: `__attribute__((packed))`

[chibicc] `is_packed` flag disables alignment padding:
```c
if (consume(&tok, tok, "packed"))
    ty->is_packed = true;
```

With `is_packed`, the struct is laid out with no gaps; each field immediately follows the previous. Field offsets are no longer aligned.

**Dangers of packed structs:**
- Unaligned memory accesses. On x86-64 these work but are slower. On ARM/RISC-V strict-alignment platforms, unaligned accesses fault.
- Taking a pointer to a packed field (`&s.field`) produces a pointer that is not aligned to the field's natural alignment. Passing this to code that assumes alignment causes undefined behavior.
- SIMD/vectorized code may fault when operating on unaligned data.
- The compiler cannot reliably vectorize loops over packed structs.

### 6.6 Array Alignment (AMD64-Specific Rule)

[chibicc] Special handling for local arrays:
```c
// AMD64 System V ABI has a special alignment rule for an array of
// length at least 16 bytes. We need to align such array to at least
// 16-byte boundaries.
int align = (var->ty->kind == TY_ARRAY && var->ty->size >= 16)
    ? MAX(16, var->align) : var->align;
```

Arrays of 16 bytes or more on the stack are aligned to at least 16 bytes, enabling SSE instructions to access them safely.

---

## 7. Common ABI Mistakes

### 7.1 Forgetting Stack Alignment Before `call`

**Symptom:** Works most of the time, crashes or produces wrong results when the callee uses SSE instructions that require 16-byte alignment (`movaps`, `movdqa`, etc.).

**Diagnosis:** Add `-mstackrealign` to GCC options temporarily; if crashes disappear, you have an alignment bug.

**Fix:** Before every `call`, ensure `%rsp % 16 == 0`. Track the current depth (number of pushed 8-byte values) and insert a padding `sub $8, %rsp` when depth is odd.

[chibicc] Solution:
```c
if ((depth + stack) % 2 == 1) {
    println("  sub $8, %%rsp");
    depth++;
    stack++;
}
```

### 7.2 Wrong Register for Struct Return Pointer

**Symptom:** Large-struct return values are corrupted. Function returns a pointer but the caller reads garbage.

**Cause:** The hidden first argument (pointer to return buffer) must go in `%rdi` — the first integer argument register. If your codegen puts it anywhere else (e.g., it accidentally ends up as argument N instead of argument 0), the callee writes to the wrong location.

**Fix:** Ensure the hidden pointer is treated as consuming `gp[0]` (i.e., `%rdi`), shifting all other integer arguments to `%rsi`, `%rdx`, etc.

[chibicc] The hidden arg occupies `gp++` before any other args:
```c
if (node->ret_buffer && node->ty->size > 16)
    gp++;  // reserve %rdi for the hidden pointer
```

### 7.3 Missing Shadow Space on Windows

**Symptom:** Crashes or corrupted values in Windows builds. Linux build works fine.

**Cause:** On Windows x64, the caller must allocate 32 bytes of shadow space before every call, even if the function takes zero arguments. The callee may write its register arguments into this shadow space as part of its prologue.

**Fix:** Always emit `sub $32, %rsp` (plus alignment if needed) before calls on Windows, and `add $32, %rsp` (plus any other cleanup) after.

### 7.4 Treating All 6 Registers as Available When Some Are Used

**Symptom:** Functions with many arguments pass incorrect values; later arguments are corrupted.

**Cause:** If argument 1 is a large struct that takes the return-buffer slot (`%rdi`), effectively consuming `gp[0]`, then a subsequent integer argument should go in `%rsi` (gp[1]), not `%rdi` (gp[0]).

**Fix:** Track `gp` and `fp` as counters that increment as you assign arguments. Never reset them mid-function.

[chibicc] `gp` and `fp` are initialized once at the start of argument assignment and only ever increment.

### 7.5 Off-by-One in Stack Cleanup

**Symptom:** Stack grows unbounded; function returns to wrong address; `%rsp` points to garbage after return.

**Cause:** `stack_args * 8` bytes were pushed for stack arguments, but the `add $N, %rsp` after the call uses the wrong value of N.

**Fix:** Count `stack` carefully (including the alignment padding slot), and clean up exactly that many bytes.

[chibicc]:
```c
println("  add $%d, %%rsp", stack_args * 8);
depth -= stack_args;
```

### 7.6 Not Setting `%al` for Variadic Calls

**Symptom:** `printf`/`vprintf` crashes or produces garbage output, especially when floating-point format specifiers are used.

**Cause:** `%al` must hold the count of XMM registers used for floating-point arguments. If `%al` is zero but you passed floats in `%xmm0`–`%xmmN`, the callee's vararg implementation will not save those registers, reading garbage from the `va_list`.

**Fix:** Before every call to a potentially-variadic function, set `%al` to the number of float arguments actually passed in XMM registers.

### 7.7 Confusing System V and Windows ABI Register Assignments

**Symptom:** Code that calls a function expecting System V convention but emits Windows convention register assignments (or vice versa).

Most common mixup: forgetting that `%rdi` is the first argument on Linux but `%rcx` is the first argument on Windows.

**Fix:** Detect the target OS at compile time and emit the correct register sequence. Never hard-code `%rdi`/`%rsi` in code that must run on Windows.

---

## Quick Reference Cheatsheet

```
System V AMD64 (Linux/macOS):
  Integer args:  rdi rsi rdx rcx r8 r9  (max 6)
  Float args:    xmm0–xmm7              (max 8)
  Return int:    rax (or rdx:rax for 128-bit)
  Return float:  xmm0
  Return struct >16B: hidden ptr in rdi; callee returns ptr in rax
  Callee-saved:  rbx rbp rsp r12 r13 r14 r15
  Stack align:   16-byte before call (so rsp%16==8 at function entry)
  Red zone:      128 bytes below rsp (user space only)
  Variadic:      al = count of xmm args used

Windows x64:
  Integer args:  rcx rdx r8 r9          (max 4; shared slots with float)
  Float args:    xmm0–xmm3              (max 4; same position as int)
  Shadow space:  32 bytes always allocated by caller before call
  Callee-saved:  rbx rbp rdi rsi rsp r12–r15 xmm6–xmm15
  No red zone

Struct layout:
  field_offset = align_to(prev_end, field_align)
  struct_align = max(field_align for all fields)
  struct_size  = align_to(last_field_end, struct_align)

Struct passing (SysV):
  size > 16:            → stack (pass by reference for return)
  size ≤ 16, all-float: → xmm registers (one or two)
  size ≤ 16, mixed/int: → gp registers  (one or two)
  not enough regs left: → stack
```
