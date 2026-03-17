# Linker and Object File Format — Deep Reference

*Sources: mold design doc (Rui Ueyama), MaskRay "All about Global Offset Table" (2021, updated 2025), MaskRay "All about Procedure Linkage Table" (2021, updated 2024), GNU ld manual (SECTIONS command), System V ABI [SysV-ABI]*

---

## 1. What a Linker Does

A linker performs two core jobs:

1. **Symbol resolution** — match every undefined symbol reference in every object file to exactly one definition. Definitions come from other object files, static archives (`.a`), or shared libraries.
2. **Relocation** — once all symbols have assigned addresses, patch every placeholder value in the code and data sections with the correct address or offset.

The output is a single executable or shared library containing the merged sections from all input files, with all relocations applied.

**The brief history (from mold design doc):** [mold]

- Original Unix: programs loaded to fixed addresses; linker applied absolute relocations. Static library support (`ar` archives) dates to this era.
- 1980s (SunOS): shared library support added. Required position-independent code (PIC), GOT/PLT, and a dynamic linker (`ld.so`).
- Each decade added features: COMDAT sections, weak symbols, LTO, debug sections, linker scripts, RELRO, now-binding, TLS, etc.

Modern linkers (GNU ld/BFD, gold, lld, mold) implement decades of accumulated features, which is why they appear complex.

---

## 2. Object File Format (ELF) [ELF]

### 2.1 ELF Header

The ELF header is the first 64 bytes of an `.o`, `.so`, or executable file. Key fields:

| Field | Meaning |
|---|---|
| `e_ident[0..4]` | Magic bytes: `\x7fELF` |
| `e_ident[EI_CLASS]` | 32-bit (ELFCLASS32) or 64-bit (ELFCLASS64) |
| `e_ident[EI_DATA]` | Endianness: ELFDATA2LSB (little) or ELFDATA2MSB (big) |
| `e_type` | ET_REL (relocatable .o), ET_EXEC (executable), ET_DYN (shared lib/PIE) |
| `e_machine` | Architecture: EM_X86_64, EM_AARCH64, EM_RISCV, etc. |
| `e_entry` | Virtual address of entry point (0 for .o files) |
| `e_phoff` | Offset to program header table |
| `e_shoff` | Offset to section header table |
| `e_shstrndx` | Section index of the section name string table |

### 2.2 Section Headers

Each section header entry (64 bytes on ELF64) describes one section:

| Field | Meaning |
|---|---|
| `sh_name` | Index into `.shstrtab` string table |
| `sh_type` | SHT_PROGBITS (code/data), SHT_NOBITS (.bss), SHT_SYMTAB, SHT_RELA, SHT_DYNSYM, etc. |
| `sh_flags` | SHF_ALLOC (loaded), SHF_EXECINSTR (executable), SHF_WRITE (writable), SHF_MERGE, SHF_GROUP |
| `sh_addr` | Virtual address (0 in .o files before linking) |
| `sh_offset` | File offset of section contents |
| `sh_size` | Size in bytes |
| `sh_link` | Section index of related section (e.g. symbol table for relocation sections) |
| `sh_info` | Extra info (e.g. first global symbol index for `.symtab`) |
| `sh_addralign` | Alignment requirement |
| `sh_entsize` | Entry size for sections with fixed-size entries (symbol/relocation tables) |

### 2.3 Program Headers (Segments)

Program headers are only meaningful in executables and shared libraries (not `.o` files). They describe **segments** — the loader's view of the file.

| Field | Meaning |
|---|---|
| `p_type` | PT_LOAD (load into memory), PT_DYNAMIC (dynamic linking info), PT_GNU_RELRO, PT_PHDR, PT_INTERP |
| `p_flags` | PF_R, PF_W, PF_X — read/write/execute permissions |
| `p_offset` | File offset |
| `p_vaddr` | Virtual address |
| `p_paddr` | Physical address (usually same as vaddr on modern systems) |
| `p_filesz` | Size in file |
| `p_memsz` | Size in memory (can be larger, e.g. `.bss`) |
| `p_align` | Alignment (typically page-aligned, 4096 or 65536) |

Multiple sections are packed into one PT_LOAD segment (e.g. `.text` + `.rodata` into read-execute segment).

### 2.4 Key Sections [ELF]

| Section | `sh_type` | Flags | Content |
|---|---|---|---|
| `.text` | SHT_PROGBITS | AX | Executable machine code |
| `.rodata` | SHT_PROGBITS | A | Read-only constants, string literals |
| `.data` | SHT_PROGBITS | AW | Initialized global/static variables |
| `.bss` | SHT_NOBITS | AW | Uninitialized globals (zero-init at load time, no file bytes) |
| `.got` | SHT_PROGBITS | AW | Global Offset Table — data symbol addresses filled by ld.so |
| `.got.plt` | SHT_PROGBITS | AW | GOT entries reserved for PLT (lazy binding targets) |
| `.plt` | SHT_PROGBITS | AX | Procedure Linkage Table — call stubs for dynamic functions |
| `.plt.got` | SHT_PROGBITS | AX | PLT stubs for symbols needing both JUMP_SLOT and GLOB_DAT (optimization) |
| `.rela.text` | SHT_RELA | — | Relocations for `.text` section (in .o files) |
| `.rela.dyn` | SHT_RELA | — | Dynamic relocations applied by ld.so at load time |
| `.rela.plt` | SHT_RELA | — | JUMP_SLOT relocations for PLT entries |
| `.dynsym` | SHT_DYNSYM | A | Dynamic symbol table (subset visible to ld.so) |
| `.symtab` | SHT_SYMTAB | — | Full symbol table (stripped in release binaries) |
| `.strtab` | SHT_STRTAB | — | Symbol name string table |
| `.shstrtab` | SHT_STRTAB | — | Section name string table |
| `.dynamic` | SHT_DYNAMIC | AW | Dynamic linking info: DT_NEEDED, DT_RELA, DT_SYMTAB, etc. |
| `.init_array` | SHT_INIT_ARRAY | AW | Array of constructor function pointers |
| `.fini_array` | SHT_FINI_ARRAY | AW | Array of destructor function pointers |
| `.eh_frame` | SHT_PROGBITS | A | DWARF call frame info for stack unwinding |
| `.debug_info` | SHT_PROGBITS | — | DWARF debug information |
| `.note.gnu.build-id` | SHT_NOTE | A | Unique build identifier (hash of output) |

**`.bss` rule**: The OS kernel zero-initializes `.bss` at load time. The section occupies zero file bytes (`sh_type = SHT_NOBITS`). Only its size is recorded. This is why global variables default to zero in C.

### 2.5 Symbol Table Entries [ELF] [SysV-ABI]

Each entry in `.symtab` or `.dynsym` is 24 bytes (ELF64):

| Field | Size | Meaning |
|---|---|---|
| `st_name` | 4 | Index into `.strtab` |
| `st_info` | 1 | High 4 bits: binding (STB_LOCAL/GLOBAL/WEAK); low 4 bits: type (STT_NOTYPE/FUNC/OBJECT/SECTION/FILE/TLS) |
| `st_other` | 1 | Visibility: STV_DEFAULT/HIDDEN/PROTECTED/INTERNAL |
| `st_shndx` | 2 | Section index (SHN_UNDEF=0 for undefined, SHN_ABS for absolute, SHN_COMMON) |
| `st_value` | 8 | Address or offset (0 for undefined) |
| `st_size` | 8 | Size of the symbol in bytes |

### 2.6 Symbol Binding [ELF] [SysV-ABI]

| Binding | `STB_*` | Meaning |
|---|---|---|
| Local | `STB_LOCAL` | Only visible within the defining object file. Compiler uses for `static` functions/variables and file-scoped symbols. Multiple objects can have `STB_LOCAL` symbols with the same name — no conflict. |
| Global | `STB_GLOBAL` | Visible across all object files. Standard C functions and non-`static` globals. **Must be defined exactly once** across the link. |
| Weak | `STB_WEAK` | Like global, but may be overridden. If a strong definition exists elsewhere, the strong one wins. If no definition exists, resolves to 0 (null) rather than being an error. |

**Strong vs. Weak resolution rules:**
- Strong + Strong → link error: "multiple definition"
- Strong + Weak → strong wins silently
- Weak + Weak → implementation-defined (usually the first one wins)
- Weak with no definition → value is 0; no error

### 2.7 Symbol Visibility [ELF]

| Visibility | `STV_*` | Meaning |
|---|---|---|
| Default | `STV_DEFAULT` | Exported from shared library; preemptible — another DSO can interpose it |
| Hidden | `STV_HIDDEN` | Not exported, not in `.dynsym`. Internal to the DSO. Compiler can use direct (non-GOT) addressing. |
| Protected | `STV_PROTECTED` | Exported (appears in `.dynsym`) but not preemptible within the defining DSO. Rare; has subtle issues with data copies. |
| Internal | `STV_INTERNAL` | Processor-specific reserved use, treated like hidden in most contexts. |

**Best practice**: Mark all internal symbols `__attribute__((visibility("hidden")))` in shared libraries. Reduces `.dynsym` size, speeds up `ld.so` symbol lookup, prevents accidental ABI exposure.

---

## 3. Symbol Resolution [ELF]

### 3.1 The Two-Pass Algorithm

The linker performs symbol resolution conceptually in two passes:

**Pass 1 — Collect definitions:**
- For each input object file and archive member, scan the symbol table.
- Insert all defined symbols (STB_GLOBAL, STB_WEAK) into a global hash table keyed by name.
- Record conflicts: if two strong definitions of the same name appear → error.

**Pass 2 — Resolve references:**
- For each undefined symbol reference (`st_shndx = SHN_UNDEF`) in each object file, look up the name in the global hash table.
- If found → record the binding (which object file + section + offset provides the definition).
- If not found and binding is STB_GLOBAL → "undefined reference" error.
- If not found and binding is STB_WEAK → resolve to address 0 (no error).

Modern linkers (including mold) interleave these passes or run them in parallel using concurrent hash maps. [mold]

### 3.2 Archive (`.a`) File Semantics

An archive is an `ar(1)` bundle of `.o` files. The linker treats archives differently from object files: **archive members are only extracted if they satisfy an undefined reference at extraction time**.

**Archive resolution algorithm:**
1. Maintain a set of "currently undefined" symbols.
2. When processing an archive, scan its symbol table (the `__.SYMDEF` or similar table embedded in the archive).
3. For each archive member that defines at least one currently-undefined symbol, extract and add it to the link.
4. Extracting a member may introduce new undefined symbols, triggering more archive member extraction.
5. Repeat until no more members satisfy undefined symbols.

**Order matters**: Archives must appear after the object files that reference them on the command line. `gcc -lfoo file.o` usually fails because `file.o` is processed after `libfoo.a`. Correct: `gcc file.o -lfoo`.

**Circular dependencies**: If `liba.a` and `libb.a` depend on each other, neither alone satisfies the other's undefined symbols. Solutions:
- `-Wl,--start-group -la -lb -Wl,--end-group` (GNU ld: repeatedly scan the group until stable)
- List the archives multiple times: `-la -lb -la`

### 3.3 Duplicate Symbol Handling

| Situation | Outcome |
|---|---|
| Two `STB_GLOBAL` definitions of same name | Link error: "multiple definition of `foo`" |
| `STB_GLOBAL` + `STB_WEAK` same name | Global wins; no error |
| Two `STB_WEAK` definitions | First-seen wins (linker-implementation-defined); no error |
| COMDAT group (see §9) | All but one copy silently discarded |

**Undefined reference errors**: Occur when a symbol is referenced (st_shndx = SHN_UNDEF, STB_GLOBAL) but no definition is found in any input file or library. Common causes: missing `-l` flag, wrong link order, missing library entirely.

---

## 4. Relocations [ELF] [SysV-ABI]

### 4.1 What a Relocation Is

A relocation record tells the linker: "at this byte offset in this section, compute a value using this symbol and this formula, and write it there."

**ELF relocation entry (RELA format, 24 bytes on ELF64):**

| Field | Size | Meaning |
|---|---|---|
| `r_offset` | 8 | Byte offset within the section where the patch goes |
| `r_info` | 8 | High 32 bits: symbol table index; low 32 bits: relocation type |
| `r_addend` | 8 | Constant addend to add to the computed value |

ELF also has REL format (no explicit addend — the addend is stored in the section content itself). RELA is preferred on modern architectures.

### 4.2 Key Relocation Types (x86-64) [ELF]

| Type | Formula | Use case |
|---|---|---|
| `R_X86_64_64` | `S + A` | Absolute 64-bit address (data pointer in `.data`) |
| `R_X86_64_32` | `S + A` (32-bit) | Absolute 32-bit address (position-dependent code) |
| `R_X86_64_PC32` | `S + A - P` | PC-relative 32-bit offset (call/jmp to local symbol) |
| `R_X86_64_PLT32` | `L + A - P` | Call via PLT (compiler emits for all extern function calls; linker may optimize away if local) |
| `R_X86_64_GOTPCREL` | `G + A - P` | Distance from PC to symbol's GOT entry (load address from GOT) |
| `R_X86_64_GOTPCRELX` | Like GOTPCREL | Relaxable version: linker can optimize away GOT indirection if symbol is non-preemptible |
| `R_X86_64_REX_GOTPCRELX` | Like GOTPCRELX | For instructions with REX prefix |
| `R_X86_64_GLOB_DAT` | `S` | Dynamic: ld.so fills GOT entry with symbol address (R_*_GLOB_DAT family) |
| `R_X86_64_JUMP_SLOT` | `S` | Dynamic: ld.so fills `.got.plt` entry for PLT lazy binding |
| `R_X86_64_RELATIVE` | `B + A` | Dynamic: position-independent base-relative relocation (no symbol lookup needed) |
| `R_X86_64_COPY` | — | Dynamic: copy symbol value from shared lib to executable's `.bss` |
| `R_X86_64_TPOFF32` | `S - TP` | Thread-local storage, initial-exec model |

Variables used: S = symbol value, A = addend, P = place (address of the relocation), L = PLT entry address, G = GOT entry address, B = base address, TP = thread pointer.

### 4.3 How the Linker Applies Relocations

For each relocation record in each input section:
1. Look up the referenced symbol in the global symbol table → get its final virtual address `S`.
2. Compute the formula (e.g. `S + A - P` for PC-relative).
3. Write the result to the output file at `r_offset` within the section's mapped location.

For dynamic relocations (`R_*_GLOB_DAT`, `R_*_JUMP_SLOT`, `R_*_RELATIVE`): the linker does **not** apply them. Instead it emits them into `.rela.dyn` or `.rela.plt` for `ld.so` to apply at load time.

### 4.4 PC-Relative vs. Absolute Relocations

**PC-relative** (`R_X86_64_PC32`, `R_X86_64_PLT32`): The encoded value is `target - (patch_location + 4)`. Works in position-independent code because both sides shift equally when the binary is loaded at a different base.

**Absolute** (`R_X86_64_64`): The encoded value is the symbol's virtual address. Requires the binary to load at a fixed address (position-dependent executable) or a dynamic relocation to fix it up at load time.

**GOT-indirect** (`R_X86_64_GOTPCRELX`): The instruction loads the address from a GOT slot. The GOT slot holds an absolute address (filled by ld.so or the linker). The instruction itself only contains a PC-relative offset to the GOT slot — so the instruction is position-independent, but the GOT entry contains an absolute address.

---

## 5. Dynamic Linking [ELF]

### 5.1 GOT — Global Offset Table [ELF]

The Global Offset Table is a runtime array of addresses located in `.got` and `.got.plt`. It solves the problem of position-independent code needing absolute addresses of external symbols.

**Why GOT is needed:**
- In a shared library compiled with `-fpic`, all code is position-independent: no absolute addresses baked into `.text`.
- But the code still needs to access global variables and call functions that may be defined in another DSO.
- Solution: emit code that loads an address from a table (`movq var@GOTPCREL(%rip), %rax`), and let `ld.so` fill in the table at load time.

**How `.got` entries work (from MaskRay GOT article):** [ELF]

```
Three cases for a symbol address:
1. Link-time constant  → position-dependent executable; direct or PC-relative addressing
2. Load base + constant → PIE or shared object; PC-relative addressing (no GOT needed)
3. Runtime computation (ld.so lookup) → GOT entry + dynamic relocation R_*_GLOB_DAT
```

When the linker encounters a GOT-generating relocation for a symbol for the first time, it allocates a GOT entry. For all subsequent GOT-generating relocations for the same symbol, it reuses that entry.

If the symbol is non-preemptible (cannot be overridden by another DSO), the linker may **optimize away** the GOT indirection, rewriting the instruction to use direct addressing:

```asm
; Input (GOT-indirect):
movq    var@GOTPCREL(%rip), %rax   ; R_X86_64_REX_GOTPCRELX
movl    (%rax), %eax

; Output after GOT optimization (non-preemptible symbol):
leaq    var(%rip), %rax
movl    (%rax), %eax
```

**`.got` structure:**
- `.got[0]`: Reserved (link-time address of `_DYNAMIC` on x86-32; architecture-dependent)
- Subsequent entries: one per distinct symbol, relocated by `R_*_GLOB_DAT`

**`.got.plt` structure:**
- `.got.plt[0]`: Link-time address of `_DYNAMIC`
- `.got.plt[1]`: ld.so fills with descriptor of the current component
- `.got.plt[2]`: ld.so fills with address of the lazy PLT resolver (`_rtld_bind_start`)
- `.got.plt[3..N]`: One entry per PLT stub; initially points to the `pushq` instruction in the PLT stub; ld.so overwrites with resolved function address

**RELRO security (PT_GNU_RELRO):** `.got` contains only eagerly-resolved relocations. With `-z relro`, the linker places `.got` in a PT_GNU_RELRO segment. After startup, ld.so calls `mprotect()` to make it read-only ("partial RELRO"). With `-z relro -z now`, `.got.plt` is also made read-only after all JUMP_SLOT relocations are eagerly resolved ("full RELRO").

### 5.2 PLT — Procedure Linkage Table [ELF]

The PLT provides a level of indirection for function calls to external (potentially preemptible) symbols, enabling lazy binding.

**x86-64 PLT structure (from MaskRay PLT article):**

```asm
; PLT header (resolver stub)
.plt:
    pushq   .got.plt+8(%rip)    ; push descriptor of this component (.got.plt[1])
    jmpq    *.got.plt+16(%rip)  ; jump to ld.so lazy resolver (.got.plt[2])
    nop

; Individual PLT stub for each external function symbol
foo@plt:
    jmpq    *.got.plt+24(%rip)  ; load address from .got.plt[3] and jump
    pushq   $0                  ; push JUMP_SLOT index (for lazy resolver)
    jmp     .plt                ; fall through to PLT header on first call
```

The `.got.plt` entries for `foo` are initially set to point at `foo@plt + 6` (the `pushq` instruction), so the first call falls through to the resolver.

### 5.3 Lazy Binding Sequence (Step by Step) [ELF]

1. Code executes `call foo`. The linker has rewritten this to `call foo@plt`.
2. `foo@plt` executes: `jmpq *.got.plt+24(%rip)`. The GOT entry still points to `pushq $0`.
3. `pushq $0` pushes the JUMP_SLOT descriptor (index 0 for the first entry).
4. `jmp .plt` jumps to the PLT header.
5. PLT header pushes `.got.plt[1]` (component descriptor) and jumps to `.got.plt[2]` (the lazy resolver in ld.so).
6. ld.so's `_dl_fixup` reads the relocation record, looks up `foo` in the dynamic symbol tables of loaded DSOs.
7. ld.so writes `foo`'s resolved address into `.got.plt[3]`.
8. ld.so jumps to `foo`. Execution proceeds.

On subsequent calls: step 2's `jmpq` now loads `foo`'s real address and jumps directly. The PLT resolver is never invoked again.

### 5.4 Eager Binding

With eager binding (ld.so processes all `R_*_JUMP_SLOT` relocations before transferring control to the program):
- All `.got.plt` entries are filled at startup.
- The `pushq` + `jmp .plt` instructions in each PLT stub are never executed.
- Detects missing symbols at startup instead of at first call.

Triggered by: `LD_BIND_NOW=1` environment variable, or linking with `-Wl,-z,now` (sets `DF_1_NOW` flag in `.dynamic`).

musl libc and Android bionic always use eager binding.

### 5.5 RPATH and RUNPATH — Library Search Path [ELF]

The dynamic linker searches for shared libraries at runtime. The search order (glibc `ld.so`):

1. `DT_RPATH` entries in the binary's `.dynamic` section (deprecated; overrides `LD_LIBRARY_PATH`)
2. `LD_LIBRARY_PATH` environment variable (colon-separated paths)
3. `DT_RUNPATH` entries in the binary's `.dynamic` section
4. `/etc/ld.so.cache` (built from `/etc/ld.so.conf`)
5. Default paths: `/lib`, `/usr/lib`

**Setting RPATH/RUNPATH at link time:**
```
gcc -Wl,-rpath,/opt/mylib/lib -o myapp myapp.o -L/opt/mylib/lib -lmylib
```

**`$ORIGIN` token**: Expands to the directory containing the binary. Use `-Wl,-rpath,'$ORIGIN/../lib'` to embed a relative path.

**RPATH vs. RUNPATH**: RPATH is searched before `LD_LIBRARY_PATH` (transitive, affects all DSOs loaded). RUNPATH is searched after `LD_LIBRARY_PATH` (only affects direct dependencies). Prefer RUNPATH.

### 5.6 LD_PRELOAD [ELF]

`LD_PRELOAD` is a colon-separated list of shared libraries to load before all others. Symbols defined in preloaded libraries take precedence over same-named symbols in all other libraries, including libc.

Common uses:
- Interposing `malloc`/`free` for debugging (e.g. `valgrind`, `jemalloc`)
- Security: injecting function wrappers
- Testing: replacing system calls with mocks

Example: `LD_PRELOAD=/usr/lib/libmymalloc.so ./myapp`

Works because `ld.so` performs symbol lookup in load order, and preloaded libraries are loaded first.

---

## 6. mold Linker Design [mold]

### 6.1 Why mold Is Fast

From the mold design document (Rui Ueyama):

> "In order to achieve a `cp`-like performance, the most important thing is to fix the layout of an output file as quickly as possible, so that we can start copying actual data from input object files to an output file as soon as possible."

Key performance insights:

1. **Copying is I/O-bound**: Once layout is fixed, copying section contents from input to output can proceed while computationally intensive tasks (relocation scanning, build-id computation) run concurrently.

2. **Fix layout first, then copy**: Applying relocations *while* copying is effectively free due to cache locality. Doing it separately (after copying) is expensive because section data has been evicted from CPU cache.

3. **Overwrite existing output file**: On Linux, overwriting an existing large file in the buffer cache is ~300ms faster than creating a new file (no need to allocate new disk blocks).

4. **Fast process exit**: Unmap all files before writing the result (use `_exit()` or a forked child process), since `close()`/`munmap()` of thousands of mmapped files takes hundreds of milliseconds.

5. **Parallel build-id computation**: Compute SHA-1 on 1 MiB chunks in parallel, then SHA-1 the concatenated hashes (Merkle tree of height 2). Achieves ~2 GiB/s on 16 cores.

### 6.2 Parallel Architecture [mold]

mold's concurrency model is **data parallelism** — large uniform datasets processed with parallel for-loops:

```
High-level flow (sequential passes, each pass parallelized internally):
1. Parse all input object files in parallel
2. Resolve symbols (concurrent hash map insertion)
3. Scan relocations in parallel (determine .got/.plt sizes)
4. Fix output layout (single thread — fast)
5. Copy section contents in parallel + apply relocations
6. Write output
```

**No channels, futures, or complex synchronization** — just parallel for-loops with atomic variables for shared flags.

**Map-reduce pattern**: Parallel for-loop produces small results, single thread aggregates. Used for: build-id computation, section size calculations.

### 6.3 Concurrent Symbol Table [mold]

Symbol resolution uses a **concurrent hash map** (`tbb::concurrent_hash_map` or equivalent):
- All input files are processed in a parallel for-loop
- Each file atomically inserts its defined symbols into the global hash table
- Symbol conflicts (two strong definitions) detected during insertion
- STB_WEAK vs. STB_GLOBAL resolution handled atomically via CAS operations

For scanning relocations, shared "requires GOT" and "requires PLT" flags on symbol objects are **atomic variables** to allow safe concurrent writes from multiple threads.

### 6.4 Key Algorithmic Choices vs. Traditional Linkers [mold]

| Feature | Traditional linkers (BFD, gold) | mold |
|---|---|---|
| Section GC | Single-threaded mark-sweep | Parallel concurrent mark |
| ICF (Identical Comdat Folding) | ~5 seconds for Chromium | ~1 second (5x faster; new isomorphic subgraph algorithm) |
| Symbol resolution | Sequential | Parallel concurrent hash map |
| Relocation scanning | Sequential | Parallel for-loop |
| Build-id | Sequential SHA-1 | Parallel Merkle-tree SHA-1 |
| String interning / merge sections | Post-load | Speculative during preload |
| Incremental linking | Supported (gold) | Explicitly rejected (too complex, too many edge cases) |
| Memory allocator | glibc malloc | jemalloc or mimalloc (better multi-thread scaling) |

**Rejected: incremental linking** — mold's position is that full-link speed should be fast enough to make incremental linking unnecessary. Incremental linking has fundamental correctness issues (weak symbols, archive member changes causing cascading effects) and significant overhead even for null (no-change) rebuilds.

### 6.5 Scale of the Problem (Chromium) [mold]

| Data item | Count |
|---|---|
| Object files | 30,723 |
| Public undefined symbols | 1,428,149 |
| Mergeable strings | 1,579,996 |
| Comdat groups | 9,914,510 |
| Regular sections | 10,345,314 |
| Public defined symbols | 10,512,135 |
| Relocations | 62,024,719 |
| Total input data | 3.4 GB |

---

## 7. LTO — Link-Time Optimization [ELF]

### 7.1 The Problem LTO Solves

Traditional compilation is **per-translation-unit**: the compiler sees one `.c` file at a time and produces machine code. The linker sees only final machine code in `.o` files — it cannot perform semantic optimizations like:
- Inlining functions across module boundaries
- Dead code elimination beyond simple garbage collection
- Interprocedural constant propagation

### 7.2 What the Compiler Emits for LTO [ELF]

With `-flto`, the compiler embeds the **intermediate representation (IR)** into the object file alongside (or instead of) machine code:

- **Full LTO** (GCC `-flto`, Clang `-flto=full`): emits LLVM bitcode (Clang) or GIMPLE/RTL (GCC) into `.o` files, replacing or augmenting machine code.
- **ThinLTO** (Clang `-flto=thin`): emits a compact **module summary** (function call graph, type hierarchy, inline hints) rather than full IR. Full IR is lazily loaded per-module only when needed.

The object files are valid ELF files with special sections (e.g. `.llvm_bitcode`, `.gnu.lto_*`) that the linker plugin reads.

### 7.3 Full LTO vs. ThinLTO

| | Full LTO | ThinLTO |
|---|---|---|
| Memory usage | All IR loaded at once | Summaries + on-demand per-module IR |
| Link time (Chromium) | Minutes | Comparable to non-LTO link |
| Optimization quality | Best (global view) | Near-Full-LTO quality |
| Incremental compilation | Poor (relinks everything) | Good (modules reused if unchanged) |
| Parallelism | Limited | Highly parallel (per-module optimization) |

### 7.4 How the Linker Invokes Optimization Passes

The linker uses a **plugin interface** (LLVM's `LLVMgold.so` plugin or GCC's `liblto_plugin.so`):

1. Linker reads `.o` files, finds LTO IR sections.
2. Linker passes all IR to the plugin.
3. Plugin (= the compiler backend) runs whole-program analysis and optimization passes.
4. Plugin emits final machine code (new `.o` files or in-memory objects).
5. Linker proceeds with normal symbol resolution and relocation.

With **ThinLTO**, step 3 is parallelized: each module's IR is optimized independently using the global summary for cross-module decisions (inlining, devirtualization).

---

## 8. Linker Scripts

### 8.1 What Linker Scripts Control

A linker script is a DSL (domain-specific language) interpreted by the linker. It controls:
- **Section placement**: which input sections go into which output sections, in what order
- **VMA (Virtual Memory Address)**: the address at which each section appears at runtime
- **LMA (Load Memory Address)**: the address at which each section is stored in the file (may differ from VMA for ROM/RAM systems)
- **Symbol definitions**: the script can define symbols (e.g. `__bss_start`, `__bss_end`) used by startup code
- **Memory regions**: named regions (e.g. `FLASH`, `RAM`) with address and size constraints

### 8.2 Common Uses

1. **Embedded systems**: Place code in flash ROM, data in RAM; define load and run addresses separately.
2. **Custom memory layouts**: Kernel code at a specific high address, user code at low address.
3. **Section merging**: Combine `.text` from multiple inputs into one output `.text`.
4. **Symbol injection**: Define `__stack_top` or `_end` for startup code to use.

**From mold design doc**: On Linux, `/usr/lib/x86_64-linux-gnu/libc.so` is (despite its `.so` extension) actually an ASCII linker script that loads the real `libc.so.6`. This is why all linkers must support a subset of linker script syntax. [mold]

### 8.3 SECTIONS Command

The `SECTIONS` command is the core of every linker script:

```ld
SECTIONS
{
  . = 0x10000;            /* Set VMA of next section to 0x10000 */

  .text : {               /* Output section .text */
    *(.text .text.*)      /* Collect all .text input sections from all files */
  }

  .rodata : {
    *(.rodata .rodata.*)
  }

  . = ALIGN(4096);        /* Align to page boundary */

  .data : {
    *(.data .data.*)
  }

  .bss : {
    __bss_start = .;      /* Define symbol at current address */
    *(.bss .bss.*)
    *(COMMON)             /* Legacy "common" symbols */
    __bss_end = .;
  }
}
```

**Location counter (`.`)**: Tracks current VMA. Can be assigned, read, and used in expressions.

**Input section wildcards**: `*(.text)` matches `.text` from all input files. `file.o(.text)` matches only `file.o`.

**`KEEP()`**: Prevents section garbage collection for specified sections (e.g. interrupt vectors that are not called directly).

### 8.4 VMA vs. LMA for Embedded Systems

```ld
MEMORY {
  FLASH (rx) : ORIGIN = 0x08000000, LENGTH = 512K
  RAM   (rw) : ORIGIN = 0x20000000, LENGTH = 128K
}

SECTIONS {
  .text : {
    *(.text*)
  } > FLASH

  .data : {
    *(.data*)
  } > RAM AT > FLASH  /* VMA in RAM, LMA in FLASH */
}
```

`> RAM AT > FLASH` means: at runtime this section lives in RAM (VMA), but it is stored in the file in FLASH (LMA). Startup code copies from LMA to VMA before `main()`.

### 8.5 Default Linker Script

If no linker script is specified, the linker uses a built-in default. View it with:
```
ld --verbose | grep -A1000 'SECTIONS'
```

---

## 9. Compiler-to-Linker Interface [ELF] [SysV-ABI]

### 9.1 What the Compiler Must Emit

The compiler must produce valid ELF object files conforming to the target ABI. Required:

| Compiler output | Linker expectation |
|---|---|
| Sections with correct names, types, flags | Linker uses flags to determine section placement and permissions |
| Symbol table with correct binding/visibility/type | Linker performs resolution and exports to `.dynsym` |
| Relocation records for every unresolved reference | Linker applies or emits dynamic relocs for each |
| `.eh_frame` for every function with stack frames | Stack unwinder and exception handling |
| `.debug_*` sections when `-g` | Debugger (stripped from release binaries) |
| COMDAT groups for templates/inline functions | Linker deduplicates |
| `.init_array`/`.fini_array` entries | Runtime startup/teardown |

### 9.2 COMDAT Sections — Template Deduplication [ELF]

C++ templates and inline functions may be instantiated in multiple translation units. Without deduplication, the final binary would contain N copies of `std::vector<int>::push_back`.

**ELF COMDAT groups** (`SHF_GROUP`): The compiler wraps each template instantiation in a group section (identified by a "group signature" = mangled symbol name). The linker discards all but one copy of each group (any copy; they should be identical by the one-definition rule).

Group sections are linked: `SHT_GROUP` section contains indices of all member sections. The linker either keeps all members or discards all members.

**GNU `.gnu.linkonce.*` (legacy)**: Older compilers used a convention where sections named `.gnu.linkonce.t.mangled_name` were deduplicated. Mostly superseded by COMDAT groups but still seen in old code.

### 9.3 `.init_array` / `.fini_array` [ELF] [SysV-ABI]

Global C++ constructors and destructors, and `__attribute__((constructor))`/`__attribute__((destructor))` functions, are registered via these sections:

**`.init_array`**: An array of function pointers. The C runtime (`__libc_csu_init` in glibc, or equivalent) iterates this array and calls each function before `main()`.

**`.fini_array`**: Same, called after `main()` returns (via `atexit()` mechanism).

The linker merges all `.init_array` sections from all input files into a single contiguous array. Order is determined by link order (not defined by C++ standard — avoid relying on it for cross-TU dependencies).

```c
// Compiler emits for each global constructor of class Foo:
static void __cxx_global_init_0() { /* call Foo's constructor */ }
// And in the same TU's .init_array:
__attribute__((section(".init_array")))
static void (*__init_ptr)() = __cxx_global_init_0;
```

**Legacy `.init` / `.fini`**: Single functions (not arrays). Called by the runtime as `(*_init)()`. Still supported but deprecated in favor of `_array` variants.

### 9.4 Debug Information — DWARF [ELF]

DWARF is the standard debug information format for ELF. Sections:

| Section | Content |
|---|---|
| `.debug_info` | Core DWARF Debugging Information Entries (DIEs): types, variables, functions |
| `.debug_abbrev` | Abbreviation table for compressed DIE encoding |
| `.debug_line` | Line number table (source file + line → machine address) |
| `.debug_str` | String pool for DWARF strings |
| `.debug_ranges` / `.debug_rnglists` | Address ranges for code with non-contiguous ranges |
| `.debug_loc` / `.debug_loclists` | Location lists for variables whose address changes at runtime |
| `.debug_frame` | Call frame information (used by unwinders) |
| `.debug_types` | Type unit info (DWARF v4+) |

**Important linker behavior**: The linker does **not** interpret DWARF. It merges `.debug_*` sections from all input files. However, many debug references use relocations (e.g. `.debug_info` references a variable's address in `.text`). The linker applies these relocations normally. For `--gc-sections`, the linker must skip updating dead section references in DWARF (or the debugger will see garbage).

---

## 10. Common Linker Errors and Causes [ELF]

### 10.1 Undefined Reference

**Error**: `undefined reference to 'foo'`

**Causes and diagnosis:**

| Cause | Diagnosis |
|---|---|
| Missing `-l` flag | Run `nm -u myapp.o \| grep foo` to confirm `foo` is needed; then `grep -r foo /usr/lib/*.so` to find the library |
| Wrong link order (archive before object) | Move `-lfoo` after the `.o` files that reference it |
| Circular archive dependency | Use `--start-group ... --end-group` |
| Name mangling mismatch | C++ calling a C function without `extern "C"` — check with `nm --demangle` |
| Missing `extern "C"` in C++ wrapping a C header | Add `extern "C" { #include "header.h" }` |
| Weak reference (intentional) | `__attribute__((weak))` — not an error; resolves to 0 |
| LTO visibility issue | Symbol has `__attribute__((visibility("hidden")))` but is referenced from outside its DSO |

### 10.2 Multiple Definition

**Error**: `multiple definition of 'foo'`

**Causes:**

| Cause | Solution |
|---|---|
| Function/variable defined in a header included in multiple TUs | Use `static` (C) or `inline` (C/C++) or put in `.c`/`.cpp` file |
| Duplicate `.o` or `.a` files on command line | Remove duplicates |
| LTO + non-LTO mixing with same symbol | Ensure consistent compilation flags |
| `__attribute__((weak))` missing on one copy | Make all but one copy `weak` |

**When it is NOT an error:**
- Two `STB_WEAK` definitions → linker picks one silently
- COMDAT groups → all duplicates except one are discarded silently
- `inline` functions in C99/C++ → COMDAT folding

### 10.3 Circular Dependencies Between Archives

**Symptom**: `undefined reference` errors even though the symbols are present in the archives.

**Root cause**: Archive `liba.a` defines symbols needed by `libb.a`, and vice versa. The linker processes archives left-to-right. When it first encounters `liba.a`, `libb.a`'s members haven't been pulled in yet, so `libb.a`'s symbols aren't undefined. After pulling in `liba.a` members, `libb.a` members are needed, but the linker has moved past `liba.a`.

**Solutions:**

```bash
# Solution 1: GNU ld group
gcc -o myapp -Wl,--start-group -la -lb -Wl,--end-group

# Solution 2: Repeat the archives
gcc -o myapp -la -lb -la

# Solution 3: Combine into one archive
ar r libab.a *.o
```

### 10.4 Relocation Overflow

**Error**: `relocation truncated to fit: R_X86_64_PC32 against symbol 'foo'`

**Cause**: A PC-relative 32-bit relocation target is more than ±2 GiB away from the reference. Common when:
- A very large binary exceeds the 32-bit displacement range
- Code references a large `.bss` or `.data` section far from `.text`

**Solutions:**
- Link with `-mcmodel=large` (uses 64-bit absolute addressing, slower)
- Reorganize sections with a linker script
- Use a thunk/trampoline for the distant reference

### 10.5 Linking Against Wrong Architecture

**Error**: `error adding symbols: file in wrong format` or `incompatible target`

**Cause**: Mixing 32-bit and 64-bit object files, or ARM vs. x86 objects.

**Diagnosis**: `file myobject.o` or `readelf -h myobject.o` to check `e_machine` field.

---

## Quick Reference: ELF Relocation Formula Cheat Sheet [ELF] [SysV-ABI]

```
Symbols:
  S  = symbol value (virtual address of the referenced symbol)
  A  = addend (from r_addend field in RELA, or from section content in REL)
  P  = place  (virtual address of the relocation site, i.e. where the patch goes)
  B  = base address (load address of the component; 0 for PIE at link time)
  G  = GOT entry offset (GOT entry address for this symbol)
  L  = PLT entry address for this symbol
  Z  = symbol size
  TP = thread pointer value

Common x86-64 relocation formulas:
  R_X86_64_64        = S + A           (absolute 64-bit)
  R_X86_64_PC32      = S + A - P       (PC-relative 32-bit)
  R_X86_64_PLT32     = L + A - P       (call via PLT, 32-bit offset)
  R_X86_64_GOTPCREL  = G + A - P       (PC-relative offset to GOT entry)
  R_X86_64_GLOB_DAT  = S               (dynamic: fill GOT entry with S)
  R_X86_64_JUMP_SLOT = S               (dynamic: fill .got.plt entry with S)
  R_X86_64_RELATIVE  = B + A           (dynamic: base-relative, no symbol lookup)
```

---

## Key Takeaways

1. **Linker = symbol resolver + relocator**. Everything else (LTO, linker scripts, COMDAT, GOT/PLT) is layered on top.

2. **ELF has two views**: sections (linker's view) and segments (loader's view). The linker operates on sections; the OS loader uses program headers.

3. **GOT enables PIC**: Code accesses external symbols via a pointer table filled by ld.so. The table itself is referenced with PC-relative addressing, which is position-independent.

4. **PLT enables lazy binding**: Functions are not resolved until first call. After resolution, the GOT entry is patched and the PLT stub is bypassed.

5. **Archives are selective**: Only members that satisfy undefined references are pulled in. Order matters.

6. **Strong beats weak, duplicate strong is an error** — the fundamental symbol resolution rule.

7. **mold's speed comes from parallelism + data locality**: Fix layout first, then copy+relocate in parallel with good cache behavior. [mold]

8. **Full RELRO + `-z now`** is the modern security baseline: eager binding + GOT made read-only.

9. **COMDAT deduplication** is how C++ avoids template bloat — the linker silently discards all but one copy of each instantiation group.

10. **LTO is a plugin**: The linker defers to the compiler backend for optimization passes; it just orchestrates the IR handoff.
