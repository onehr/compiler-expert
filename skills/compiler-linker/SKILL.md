---
name: compiler-linker
description: Linker and object file expert — covers ELF/COFF/Mach-O format, symbol tables, relocations, section layout, dynamic linking, LTO, and the compiler-to-linker interface. Use when generating object files, implementing a linker, or debugging linking errors.
---

# Linker and Object File Formats

---

## Object File Formats

Three dominant formats, one per major OS family:

| Format | OS | File extension |
|---|---|---|
| ELF (Executable and Linkable Format) | Linux, BSD, most Unix | `.o`, `.so`, (no ext for executables) |
| COFF / PE (Portable Executable) | Windows | `.obj`, `.dll`, `.exe` |
| Mach-O | macOS, iOS | `.o`, `.dylib`, `.bundle` |

All three share the same conceptual structure: a header, a table of sections (named regions of bytes with associated metadata), a symbol table, and a relocation table. The differences are in encoding, alignment rules, and feature sets.

**ELF structure overview**:
- ELF header: magic bytes, architecture, entry point, offsets to section/program header tables
- Section header table: describes sections (name, type, flags, offset, size, alignment, link)
- Program header table: describes segments for the OS loader (load address, permissions, file offset)
- Sections: actual content bytes

---

## Section Layout

Standard sections and their purpose:

| Section | Contents | Writable? | Loaded? |
|---|---|---|---|
| `.text` | Executable machine code | No | Yes |
| `.rodata` | Read-only constants, string literals | No | Yes |
| `.data` | Initialized global/static variables | Yes | Yes |
| `.bss` | Uninitialized globals (zero-initialized) | Yes | Yes (no file bytes) |
| `.got` | Global Offset Table — runtime addresses of data symbols | Yes | Yes |
| `.plt` | Procedure Linkage Table — stubs for dynamic function calls | No | Yes |
| `.got.plt` | GOT entries reserved for PLT (lazy binding) | Yes | Yes |
| `.debug_*` | DWARF debug info | No | No (stripped in release) |
| `.eh_frame` | Exception handling / stack unwinding tables | No | Yes (if exceptions used) |

**`.bss` is not stored in the file** — it occupies no file bytes. The OS zero-initializes it at load time. Only its size is recorded.

---

## Symbol Tables

A symbol table entry records: name, section, offset within section, size, binding, type, and visibility.

### Symbol Binding

| Binding | Meaning |
|---|---|
| `STB_LOCAL` | Visible only within the object file (static functions/variables) |
| `STB_GLOBAL` | Visible across all object files — must be defined exactly once |
| `STB_WEAK` | Like global but definition can be overridden; if undefined, resolves to 0 |

### Symbol Visibility (ELF)

| Visibility | Meaning |
|---|---|
| `STV_DEFAULT` | Standard visibility — exported from shared library |
| `STV_HIDDEN` | Not exported; local to the shared library or executable |
| `STV_PROTECTED` | Exported but cannot be interposed (direct binding within DSO) |

**Use `STV_HIDDEN` by default** for internal symbols in shared libraries. Reduces dynamic symbol table size, speeds up dynamic linking, prevents accidental ABI surface.

### Strong vs. Weak Symbols

Strong symbols: exactly one definition must exist. Duplicate strong definitions are a linker error.

Weak symbols: multiple definitions allowed; the linker picks one (usually the strong one if both exist). Used for: default implementations that libraries can override, COMDAT folding of identical template instantiations.

---

## Relocations

A relocation tells the linker "at this offset in this section, patch the value using this formula referencing this symbol."

### Relocation Types

**Absolute relocation**: patch location gets the symbol's absolute virtual address.
- Used for: data pointers in `.data`, absolute jump targets
- Problem: incompatible with position-independent code (PIC) — the address is baked in at link time

**PC-relative relocation**: patch location gets `symbol_address - (patch_address + addend)`.
- Used for: `call` and `jmp` instructions (which encode a relative offset)
- Works in PIC because both the instruction and the target move together

**GOT-relative relocation**: patch location gets the offset to the symbol's GOT entry.
- Used in PIC to access data symbols whose address is unknown until runtime

### Dynamic Linking: GOT and PLT

**Problem**: In a shared library, the address of imported functions and data is not known at link time — it is fixed up by the dynamic linker at load time.

**GOT (Global Offset Table)**: A table of addresses in `.got`. The dynamic linker fills in the actual runtime addresses at load time. Code accesses symbols via the GOT with a PC-relative reference (which is itself position-independent).

**PLT (Procedure Linkage Table)**: Stubs that implement lazy binding for function calls:
1. First call to `foo`: PLT stub calls through GOT entry (initialized to point back into PLT resolver stub), which calls the dynamic linker to resolve `foo`'s address and patches the GOT entry.
2. Subsequent calls: GOT entry now points directly to `foo`. No dynamic linker involvement.

**Lazy vs. eager binding**:
- Lazy (default): functions resolved on first call — faster startup, harder to debug
- Eager (`LD_BIND_NOW=1` or `-z now`): all symbols resolved at startup — slower startup, detects missing symbols immediately, required for security hardening (RELRO)

---

## Link-Time Optimization (LTO)

### What LTO Does

Traditional compilation model: each translation unit compiled independently to machine code. The linker sees only object files with no semantic information — it cannot inline across translation units.

LTO stores the compiler's IR (instead of or alongside machine code) in object files. The linker invokes the compiler again with the combined IR from all translation units, enabling whole-program optimization:
- Cross-module inlining
- Dead code elimination across the whole program
- Interprocedural constant propagation
- Devirtualization

### Full LTO vs. Thin LTO

| | Full LTO | Thin LTO |
|---|---|---|
| IR loaded into memory | All at once | Per-module summaries only |
| Link time | Slow (large programs: minutes) | Fast (parallel, incremental) |
| Optimization scope | Global | Per-module with summary-driven decisions |
| Typical use | Release builds where link time acceptable | Debug/CI builds where speed matters |

---

## Compiler-to-Linker Interface

What the compiler emits, and what the linker expects:

| Compiler emits | Linker expects |
|---|---|
| `.o` object files with sections, symbol table, relocation table | Object files conforming to ELF/COFF/Mach-O spec for the target |
| External symbol references (undefined symbols) | All undefined symbols resolved by other object files or libraries |
| Relocation records for each fixup site | To patch those fixup sites with correct addresses |
| `.eh_frame` / `.pdata` for unwind info | For exception handling and stack unwinding support |
| COMDAT sections for template instantiations | For deduplication (linker picks one copy) |
| LTO IR sections (if LTO enabled) | For whole-program optimization pass |

**Key invariant**: The compiler assigns every global variable and function to a section, gives it a symbol, and emits relocations for every reference to an address not known at compile time. The linker resolves all references and produces a final binary.

---

## Deep Reference

See [`references/linker-reference.md`](references/linker-reference.md) for a comprehensive research-backed reference covering:

- ELF file format: header, section headers, program headers, all key sections
- Symbol table entries, binding (local/global/weak), visibility (default/hidden/protected)
- Strong vs. weak symbol resolution rules
- Two-pass symbol resolution algorithm; archive extraction semantics
- Relocations: RELA format, all key x86-64 types (PC32, PLT32, GOTPCREL, GLOB_DAT, JUMP_SLOT, RELATIVE)
- GOT (Global Offset Table): life of an entry, GOT optimization (GOTPCRELX relaxation), RELRO security
- PLT (Procedure Linkage Table): lazy vs. eager binding, step-by-step lazy binding sequence
- RPATH/RUNPATH library search, LD_PRELOAD
- mold linker design: parallel architecture, concurrent symbol table, key algorithmic choices vs. BFD/gold
- LTO: Full LTO vs. ThinLTO, compiler IR emission, linker plugin interface
- Linker scripts: SECTIONS command, VMA vs. LMA, embedded ROM/RAM layout
- Compiler-to-linker interface: COMDAT deduplication, .init_array/.fini_array, DWARF debug sections
- Common linker errors: undefined reference, multiple definition, circular archive dependencies, relocation overflow

*Sources: mold design doc (Rui Ueyama), MaskRay "All about Global Offset Table" (2021/2025), MaskRay "All about Procedure Linkage Table" (2021/2024), GNU ld manual (SECTIONS command), System V ABI*

---

## Navigation

**Trace Up ↑** — skills that route here
- `compiler-codegen` — generates the object files that the linker consumes
- `compiler-expert` — routes here on ELF/COFF/relocations/symbol table questions

**Trace Down ↓** — this is a leaf skill (end of the compilation pipeline)
- (none — the linker produces the final binary)

**Related →** — closely related back-end skills
- `compiler-abi` — symbol mangling, calling conventions, and section layout are ABI-defined
- `compiler-codegen` — upstream producer of object files
- `compiler-runtime` — runtime startup code and dynamic loader interact with the linker
