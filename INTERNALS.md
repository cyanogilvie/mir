# MIR Internals — Maintainer's Architecture & Implementation Guide

This document is for people *maintaining* MIR (this fork in particular), not
for people using it. It complements the upstream docs: `MIR.md` (the IR
reference), `HOW-TO-PORT-MIR.md` (the porting contract), and
`CUSTOM-ALLOCATORS.md`. File:line references are against this fork's `meson`
branch (upstream master a6db87d4 + two RA fixes + meson build).

Production context for this fork: in-process JIT consuming `MIR_gen` with the
eager gen interface, modules built programmatically (slimcc frontend, later
direct API builders), aarch64+musl primary target, x86_64+glibc secondary,
Windows later. c2mir is not in the production path but drives most upstream
tests.

---

## 1. Repo map

| File | Role |
|---|---|
| `mir.h`, `mir.c` | Core: context, modules/items, insns/ops, simplify+inline, load/link, text+binary IO, code publishing. `mir.c` textually `#include`s the target runtime file and `mir-interp.c` |
| `mir-gen.h`, `mir-gen.c` | The generator (all optimization passes + RA); textually `#include`s `mir-gen-<arch>.c` at mir-gen.c:316-335 |
| `mir-<arch>.{h,c}` | Per-target: hard-reg model (.h), runtime stubs — thunks, FFI, interp shims, wrappers (.c, included by mir.c at mir.c:6900-6920) |
| `mir-gen-<arch>.c` | Per-target codegen: machinize (ABI), prolog/epilog, pattern table, encoder, relocation |
| `mir-interp.c` | Interpreter (included into mir.c; removable with `MIR_NO_INTERP`) |
| `mir-dlist.h` `mir-varr.h` `mir-htab.h` `mir-bitmap.h` `mir-hash.h` `mir-reduce.h` | Header-only ADTs (intrusive dlist, growable array, hash table, bitmap, hash fns, LZ4-ish compressor for .bmir) |
| `mir-alloc.h` + `mir-alloc-default.c`, `mir-code-alloc.h` + `mir-code-alloc-default.c` | Pluggable heap and executable-memory allocators |
| `c2mir/` | C11 frontend; not in our production path, but the test-suite driver |
| `c-tests/`, `mir-tests/`, `adt-tests/` | Test corpora (§13) |

Arch support: x86_64 (SysV + Win64), aarch64 (Linux + Apple), ppc64{,le},
s390x, riscv64. Anything else is a hard `#error` (mir.c:6918).

Terminology used throughout the generator (mir-gen.c:64-69):
**reg** = pseudo register (> MAX_HARD_REG), **hard reg** ≤ MAX_HARD_REG,
**var** = either, **loc** = hard reg *or* stack slot (slots number from
MAX_HARD_REG+1).

---

## 2. MIR_context

`struct MIR_context` (mir.c:31-62) is opaque and owns *everything*: there are
no process-global mutable structures apart from three benign target constants
in the interpreter (mir-interp.c:958, rewritten with identical values by every
init). Consequences:

- **Thread model: one thread per context.** Different threads may use
  different contexts with no synchronization (documented at MIR.md:15-16).
  There are no mutexes or atomics anywhere in mir.c/mir-gen.c. The
  "thread safe" comments on `_MIR_publish_code` etc. (mir.c:4427+) are
  aspirational; two threads publishing into the same context race on the
  code-holder VARR. Executing *already generated* code from any number of
  threads is fine.
- **Layout invariant**: `gen_ctx` must be the first member and `c2mir_ctx`
  the second (mir.c:32-33) — mir-gen.c and c2mir cast the context pointer.
  Never reorder.
- `MIR_finish` unmaps all code holders: **all generated machine code dies
  with the context**. There is no per-module or per-function code free
  (bump allocation, §4).
- Sub-contexts (string_ctx, simplify_ctx, machine_code_ctx, io_ctx, scan_ctx,
  interp_ctx, ...) are malloc'd structs reached through file-scope
  `#define field ctx->subctx->field` accessor macros. The same macro idiom
  pervades mir-gen.c (`#define optimize_level gen_ctx->optimize_level` etc.,
  mir-gen.c:253-291) — **bare identifiers are frequently macros needing a
  `ctx`/`gen_ctx` variable in scope**; grep before assuming a local.

Allocators: `MIR_init2(MIR_alloc_t, MIR_code_alloc_t)` (mir.h:477) accepts
vtables; `MIR_init()` uses libc/mmap defaults. Note `realloc` in
`MIR_alloc_t` receives the *old size* (mir-alloc.h:33) — deliberate, for
arena allocators. `MIR_init2` is a static-inline that checks
`MIR_API_VERSION` (0.2, mir.h:26) against the library and `exit(1)`s on
mismatch.

---

## 3. Modules and items

`MIR_module` = name + DLIST of `MIR_item`. Item kinds: `func`, `proto`,
`import`, `export`, `forward`, `data`, `ref_data` (pointer to another item ±
disp), `lref_data` (label address, for computed-goto tables), `expr_data`
(value computed at link time by *running the interpreter* on a nullary func),
`bss`.

Key mechanics:

- **Names are interned** per-context; the item hash table compares name
  *pointers* (mir.c:687-689). This is why `MIR_change_module_ctx` must
  re-intern every name (§3.1).
- **Item resolution**: export/forward/import chain to the definition through
  `ref_def`. Imports resolve at `MIR_link` time: first against the
  per-context *environment module* (populated by `MIR_load_external` and by
  exports of loaded modules), then the user `import_resolver` callback
  (mir.c:1995-2005).
- **Data section merging** (`load_bss_data_section`, mir.c:1800-1879): a
  *named* data/bss item plus all immediately following *anonymous* data items
  are sized together and placed in **one malloc allocation**, addresses
  assigned consecutively with **no alignment padding between items**. Every
  named object starts a fresh (malloc-aligned, i.e. 16-byte) allocation.
  Anything needing more than 16-byte alignment is the embedder's problem
  (slimcc handles it by over-allocating + rounding addresses).
- **Function loading**: every func item gets a *thunk* as its public address
  (`item->addr = _MIR_get_thunk`), initially pointing at an error stub.
  `MIR_set_{gen,lazy_gen,interp}_interface` work by re-pointing this thunk;
  this is the constant-time interp↔JIT switching mechanism.
- `MIR_load_external(ctx, "setjmp", addr)` is special-cased: the address is
  remembered (mir.c:1956-1963) and the interpreter calls setjmp *directly*
  rather than through an FFI stub (whose frame would be dead at longjmp time)
  (mir-interp.c:1541-1578).
- `item->data` is nominally a user field but `MIR_link` borrows it as an
  inlining flag (mir.c:1993, cleared after) and `remove_item` frees it if
  non-NULL (mir.c:860) — don't park non-heap pointers there.
- `module->data` is owned by the core (bitmap of hard regs used by tied
  globals, mir.c:596-598).
- `MIR_get_global_item` is declared (mir.h:621) but **not defined** —
  upstream PRs #383/#141 add the missing definition.

### 3.1 MIR_change_module_ctx (mir.c:2839-2897)

The scratch-context migration primitive (compile in throwaway ctx → migrate
finished module to the consumer ctx; this is how libslimcc contains compiler
errors). Hard constraints:

- Must happen **before `MIR_load_module`** in the old ctx (errors if any item
  has `addr != NULL`).
- Re-interns module/item/proto/reg/alias/string names into the new ctx; ref
  ops to items *within* the module stay valid; refs to other old-ctx modules
  would dangle — keep modules self-contained (imports are fine).
- **Allocator landmine**: the module's memory was allocated with the old
  ctx's `MIR_alloc_t` but will be freed with the new ctx's. Fine when both
  use the default allocator; broken for mismatched custom allocators
  (upstream issue #425 is the documentation gap).

---

## 4. Code memory & W^X

Code is bump-allocated from per-context "code holders" (mir.c:4353-4389):
mmap'd page runs, only the most recent holder is appended to, holders are
never individually freed. Page size from `sysconf(_SC_PAGE_SIZE)` /
`GetSystemInfo`.

The W^X model is **mprotect flipping over a single mapping**
(`_MIR_set_code`, mir.c:4398-4409): steady-state RX, briefly
PROT_WRITE|PROT_EXEC during each code write, back to RX. Implications:

- Kernels/policies that forbid *any* W+X mapping (PaX MPROTECT, SELinux
  `deny_execmem`, OpenBSD) fail at the mprotect — and **the return value of
  `MIR_mem_protect` is ignored** in `_MIR_set_code`, so the failure mode is
  a SIGSEGV on the next write, not a diagnostic. A hardened-environment
  embedder must supply a custom `MIR_code_alloc_t`; note that writes go
  through the *executable* address (mir.c:4403-4406), so a dual-mapping
  scheme must make that same address writable in `mem_protect(WRITE_EXEC)`.
- Linux default `mem_map` maps PROT_EXEC-only; the WRITE_EXEC flip omits
  PROT_READ except on riscv (mir-code-alloc-default.c:18-26) — unusual but
  tolerated by Linux.
- Apple aarch64: `MAP_JIT` + per-thread `pthread_jit_write_protect_np` +
  `sys_icache_invalidate` (mir-code-alloc-default.c:28-65).
- Windows: VirtualAlloc/VirtualProtect (mir-code-alloc-default.c:66-83).
- Every publish/change ends with `_MIR_flush_code_cache` =
  `__builtin___clear_cache` (mir.c:4391-4395) — essential on aarch64
  (non-coherent I/D caches), no-op on MSVC.
- **`MAP_FAILED` gotcha**: mir-code-alloc.h:14 defines `MAP_FAILED` as NULL,
  but mir.c includes `<sys/mman.h>` later, which redefines it to `(void*)-1`;
  the failure check at mir.c:4382 therefore tests -1 on POSIX. A custom
  `mem_map` returning NULL on failure (as the header implies) is **not
  detected**. Custom allocators should return `(void*)-1` on POSIX. Any
  embedder including both headers sees the redefinition warning.

There are **no stack probes** anywhere — large frames/allocas just `sub sp`.
Irrelevant on Linux main threads; a real concern for big frames on small
thread stacks (musl default thread stack is 128 KB) and for Windows guard
pages.

---

## 5. Functions, insns, operands

- `MIR_insn` is a var-length struct (`ops[]` tail). `MIR_op_t` has a
  mode tag (reg/int/float/ref/str/mem/label/var/var_mem; the var forms are
  generator-internal) plus an `op.data` aux pointer whose meaning is
  phase-dependent (§8 gotcha 5).
- **`insn_descs[]`** (mir.c:150-341) is the single source of truth for
  operand counts/modes. Output operands carry `OUT_FLAG` (bit 7) in their
  mode byte. *Everything* — SSA construction, liveness, RA, DCE, combine —
  derives def/use information from this table via `MIR_insn_op_mode` and the
  `FOREACH_IN/OUT_INSN_VAR` iterators (mir-gen.c:1226-1316). **A missing
  OUT_FLAG poisons every pass at once** and manifests as register corruption
  under pressure only; this was the LADDR bug fixed in this fork (54a82231).
  When touching the table, re-check every entry's out flags first.
- Type/mode validation is deferred to `MIR_finish_func` (mir.c:1556-1765),
  which also appends a default `ret 0` to functions that fall off the end and
  computes per-op `value_mode`.
- `MIR_T_LD` is silently canonicalized to `MIR_T_D` on Windows and any
  platform where `long double == double` (mir.c:1146-1151, 2173-2202) — the
  IR itself is platform-divergent there. On aarch64 (glibc *and* musl) LD is
  binary128 and stays distinct.
- Functions keep two insn lists: `original_insns` (pristine) and `insns`
  (what gen works on); `_MIR_duplicate_func_insns` / `_MIR_restore_func_insns`
  (mir.c:2753-2812) swap between them around code generation, so a function
  can be re-generated (e.g. after `MIR_set_func_redef_permission`).

---

## 6. Simplify and inlining (mir.c, runs inside MIR_link)

There is no public "simplify" entry point; `MIR_link` runs `simplify_func`
(mir.c:3681-3843) on every function — **linking mutates the IR**. What it
does: inserts arg-extension insns; splits mem-mem moves; consolidates
constant ALLOCAs; branch peepholes and jump-chain shortening; spills
float/string immediates to module-local `.lc<N>` data items; reduces every
memory operand to plain `[base_reg]` form (base/index/scale/disp lowered to
explicit arithmetic with local value numbering); funnels everything to a
single RET. The generator *requires* this shape (the interpreter asserts it,
mir-interp.c:169).

Inlining (`process_inlines`, mir.c:4008-4240) happens in the second link
pass, driven by `MIR_CALL`/`MIR_INLINE` (never JCALL), with size caps
(`MIR_MAX_INSNS_FOR_INLINE` 200 / `..CALL_INLINE` 50, overridable macros).
Callee regs are renamed `.c<n>_<name>`; top-level constant allocas of caller
and callees are coalesced. Functions with lrefs (computed-goto label tables)
are never inlined.

---

## 7. The generator pipeline (mir-gen.c)

`MIR_gen(ctx, item)` → `generate_func_code` (mir-gen.c:9284-9510). Pass
order, with the optimize-level gates (`MIR_gen_set_optimize_level`; default
2; **level 3 is currently identical to 2** — the ">=3" comment at
mir-gen.c:209 has no corresponding gate):

| Pass | Gate | Notes |
|---|---|---|
| `build_func_cfg` (1574) | always | entry/exit BBs + edges; detects `MIR_ADDR` (→`addr_insn_p`) and `MIR_JMPI` (→`jmpi_p`, our fix) |
| `clone_bbs` (2019) | ≥2 | superblock formation, growth cap 3× |
| `build_ssa` (2679) | ≥2 | RPO + on-demand phis (no dominance frontier); def-use chains as `ssa_edge` objects hung off `op.data` |
| `transform_addrs` (2900) | ≥2, ADDR present | reconcile address-taken pseudos, then SSA rebuilt |
| `gvn` (4921) | ≥2 | the heavyweight: value numbering + const folding/propagation + branch folding + redundant-load elimination (needs dominators + memory availability). **Upstream #423 lives here**: replacing a narrowing reload with the stored register value loses the load's implicit sign-extension |
| `copy_prop` (3217) | ≥2 | copy chains, mul/div→shift, redundant ext-pair removal |
| `dse` (5128) | ≥2 | backward mem-liveness over `nloc` numbers |
| `ssa_dead_code_elimination` (5225) | ≥2 | |
| `licm` (5526) + loop tree | ≥2 | only MUL/MULS considered expensive enough to hoist |
| `pressure_relief` (5540) | ≥2 | sink single-use const moves |
| out-of-SSA (2716/5843/2797) | ≥2 | conventional-SSA copies, SSA-level address fusion + cmp/branch fusion, phi removal |
| `jump_opt` (6646) | ≥2 | unreachable/empty BB removal, branch-over-jmp. **Upstream #424**: can delete labels still referenced by lref data → use-after-free in `gen_setup_lrefs` |
| `target_machinize` | always | ABI lowering (§9) |
| `make_io_dup_op_insns` (7117) | always | enforce two-address constraints |
| `coalesce` (6930) | ≥2 | pre-RA move coalescing, loop-weighted |
| `reg_alloc` (8527) | always | §8 |
| `combine` (9032) | ≥1 | post-RA forward selection within BBs, validated by `target_insn_ok_p`; folds loads into uses, fuses cmp+branch; deletes in-BB labels |
| `dead_code_elimination` (9194) | ≥1 | hard-reg liveness DCE |
| `target_make_prolog_epilog`, `target_translate`, `_MIR_publish_code`, `target_rebase`, thunk redirect | always | |

O0/O1 compile 2-3× faster, produce notably slower code. We ship O2.

Data structures to know: `bb` (in/out/gen/kill bitmaps **reused under
different aliases per phase** — live_*, dom_*, mem_live_*, then spill_gen/
spill_kill for RA; mir-gen.c:6268, 7270), `bb_insn` (wraps each insn at O≥1;
plain `insn_data` at O0 — `insn->data` is polymorphic), `ssa_edge`
(`op.data`), live ranges (`[start,finish]` point intervals per var, with
`lr_bb != NULL` marking *gap* segments — the unit of live-range splitting),
loop tree with `LOOP_COST_FACTOR^level` weighting (=5, mir-gen.c:293).

Debug: `MIR_gen_set_debug_file/level`. Level 0 = per-func summary line, 1 =
per-pass statistics, 2 = full IR dump after every pass + disassembly (via
`_MIR_dump_code`, which popens gcc/objdump — debug only), 4 = RA/edge-split
traces. Compiled out by `MIR_NO_GEN_DEBUG` (upstream PR #418 fixes a build
break in that config).

---

## 8. Register allocator (mir-gen.c:7180-8592)

Priority-based allocation over live ranges (not graph coloring, not classic
linear scan): pseudos sorted by (tied first, then frequency, then shorter
ranges); per-program-point occupancy bitmaps (`used_locs`[point] = set of
locs taken); each pseudo gets the first non-conflicting hard reg in
`TARGET_HARD_REG_ALLOC_ORDER` (preferring callee-saved when profitable), else
the full RA tries **gap splitting** (evict another pseudo's live-range *gap*
— BBs where it's live-across but unreferenced), else a first-fit stack slot.
`rewrite` then materializes assignments, generating loads/stores around insns
with spilled operands using reserved temp hard regs (2 int + 2 per FP type;
an insn needing more than `MAX_INSN_RELOADS` asserts). The full RA finally
places spill/restore code on edges (`split`, 8470), splitting critical edges
when needed.

**The simplified RA** (no gap splitting, no edge splitting, linear rewrite)
is used when `optimize_level < 2` **or the function contains `MIR_JMPI`**
(`jmpi_p`, this fork's a6db87d4): edges out of an indirect jump cannot be
split, and `split_edge_if_necessary` (915) otherwise rewrites the terminator's
label operand — which for JMPI is a register, corrupting it silently in
NDEBUG builds.

Invariants worth knowing:

- `used_locs` / `busy_used_locs` varrs **persist across functions** in
  ra_ctx and are length-synchronized lazily; anything that varies RA behavior
  per function must keep both grown in lockstep (mir-gen.c:7612-7623) — that
  was the second half of the jmpi fix.
- The `bitmap_equal_p(live, bb->live_in)` assert in rewrite (8266) is the
  canary that fires when liveness/out-flags are wrong anywhere upstream.
- Tied (hard-reg-named) global regs assert their reg is marked at every
  point (7649).

---

## 9. Target backends

Per-arch contract (HOW-TO-PORT-MIR.md): `mir-<arch>.h` (hard-reg model),
`mir-<arch>.c` (runtime stubs, §10), `mir-gen-<arch>.c` (codegen hooks §7
table + pattern-based encoder), `c2mir/<arch>/c<arch>-ABI-code.c` (C-level
aggregate classification — **ABI class for struct args is decided in the
frontend**, not in mir-gen; slimcc has the same responsibility).

### Pattern tables

Each backend has a `patterns[]` table mapping a MIR insn + operand-constraint
string to an encoding template string (constraint/replacement mini-languages
documented in the `struct pattern` comments: mir-gen-aarch64.c:1183-1262,
mir-gen-x86_64.c:1334-1398). `target_insn_ok_p` = "some pattern matches";
first match wins, so **specific patterns must precede general ones**. A MIR
insn reaching `target_translate` with no match is a fatal
`"fatal failure in matching insn"` exit. Synthetic codes past
`MIR_INSN_BOUND` exist (e.g. aarch64 SUB_UBO/MUL_BO for overflow branches,
mir-gen-aarch64.c:1264-1268).

### aarch64 specifics (production target)

- AAPCS64 subset: x0-x7/v0-v7 args, x8 hidden-result (RBLK), BLK ≤16 bytes
  fully in GPRs or stack, >16 bytes caller-copied + address passed.
  **No HFA support**: there is exactly one BLK class
  (caarch64-ABI-code.c:82-85), so `struct {float x, y;}` by value travels in
  integer regs. Internally consistent (MIR↔MIR calls fine), but **wrong for
  by-value HFA interop with host C functions** — both directions, gen and
  FFI. Known limitation to fix if host interop ever passes float-aggregates
  by value.
- long double = binary128 implemented via C helper builtins (`mir_ldadd`
  etc., mir-gen-aarch64.c:472-679) — every LD op except move is a call.
  LDMOV is real q-register code.
- va_list is the 5-field AAPCS64 struct (mir-aarch64.c:48-65); vararg
  prologue dumps x0-x7 + q0-q7 (192 bytes); `va_arg_builtin` /
  `va_block_arg_builtin` are host C functions called as builtins.
- **Branch ranges**: conditional branches ±1MB (19-bit), `b` ±128MB
  (26-bit), `MIR_LADDR` is a single `adr` = **±1MB**, with **no relaxation in
  normal translation** — a pathologically huge function would mis-encode.
  (BBV mode does relax via a reserved-NOP rewrite protocol.)
- SWITCH = `adr`+`ldr`+`br` over an inline table of absolute label addresses,
  relocated at `target_rebase`.
- Frame: FP always kept; positive imm12 offsets from FP; frame ≥4096 via
  temp reg; small-aggregate save area below FP for reg-passed structs.
- Known sloppy spots: two `% 16` where round-up was meant
  (mir-aarch64.c:94, :417), reachable only for binary128 long double passed
  *on the stack* through varargs/FFI (>8 FP args) — fix cheaply, treat
  LD-on-stack as undertested. The interp shim has magic frame constants
  (240/192/16) tied across `save_insns`/`prepare_pat` — recompute, don't
  guess, when touching.
- Apple aarch64 forks nearly every function in both files (`__APPLE__`):
  LD==double, x18 reserved, va_list = pointer, varargs on stack, MAP_JIT.

### x86_64 specifics

SysV + Win64 in the same file behind `_WIN32`. Conveniences aarch64 lacks:
FP omission (`keep_fp_p`), constant pool + rip-relative addressing (no
range limits), direct-call rewriting after lazy gen, loop-header alignment.
The generated code **uses the red zone** (`MIR_NO_RED_ZONE_ABI` opts out).
Win64 status: calling convention, shadow space, xmm6-15 saves, va_list as
pointer all present; **no SEH unwind info for generated code, no `__chkstk`
stack probes, single return value only, LD==double**; parallel c2m disabled.
CI builds/tests via CMake+MSVC. Treat as "runs, not unwindable".

---

## 10. Runtime stubs / FFI layer (mir-<arch>.c)

All synthesized as byte patterns and published into context code holders:

- **Thunk** per function item; `_MIR_redirect_thunk` re-points it (aarch64:
  single patched `b` if within ±128MB else `ldr x9; br x9; .quad`; x86: jmp
  rel32 or movabs r11).
- **`_MIR_get_ff_call`** (interp → native): per-*signature* stub (cached per
  context by signature hash, mir-interp.c:1742-1771) marshalling a
  `MIR_val_t` array into the native ABI. 16-byte slot stride (long double
  sized).
- **`_MIR_get_interp_shim`** (native → interp): per-function stub presenting
  the function's C ABI, saving arg regs, building a real va_list, calling the
  interpreter, moving results back. This is how `MIR_set_interp_interface`
  makes interpreted functions callable from C.
- **`_MIR_get_wrapper`** (lazy gen): per-function stub jumping to a shared
  `wrapper_end` blob that saves all arg regs, calls the generate-hook, and
  tail-jumps to the result. `ctx`/`func_item` are baked into the stub as
  immediates.
- bstart/bend builtins read/write SP for the VLA scope insns.

The interpreter (mir-interp.c) is direct-threaded (computed goto; plain
switch on MSVC), flattens each function to a `MIR_val_t` icode array on first
call, and calls *everything* (native or interpreted) through ff_call stubs.
Performance ~6-10× slower than generated code. `MIR_INTERP_TRACE` for
tracing. Expr-data items are evaluated by the interpreter at link time even
in pure-gen deployments, so `MIR_NO_INTERP` is only viable if no expr_data is
used.

Lazy modes: `MIR_set_lazy_gen_interface` (whole function on first call) and
`MIR_set_lazy_bb_gen_interface` (basic-block versioning: full pipeline+RA up
front, per-BB encoding on first execution, with branch back-patching and the
PRSET/PRBEQ property-speculation machinery, mir-gen.c:9547-10021). BBV is the
least mature subsystem (pre-existing aarch64 long-double failures in
`use-c2m-gen-bb` mode; upstream #308 was a bbv-branch DSE bug). We don't use
it in production.

---

## 11. Error handling

`MIR_error_func_t` is `noreturn`; the default prints and `exit(1)`s. Any API
misuse or internal failure calls it and never returns — embedders wanting
recovery must `longjmp` from their error func and accept leaked partial
state. The clean containment pattern (used by libslimcc) is: build in a
scratch context, longjmp on error, `MIR_finish(scratch)` to free everything
half-built, `MIR_change_module_ctx` on success. The text scanner is the only
subsystem with internal recovery (per-line setjmp, accumulated messages).

OOM inside VARR growth does not route through the error func
(mir-varr.h:134 doesn't check realloc) — it crashes.

---

## 12. Serialization

- Text: `MIR_output*` / `MIR_scan_string`. Round-trip caveat: output puts
  protos/forwards in an order the scanner may reject (#254) — emit forwards
  first if round-tripping.
- Binary: `MIR_write*` / `MIR_read*`, tagged byte stream, two-pass string
  pool, version-checked (`CURR_BIN_VERSION` mir.c:4578), wrapped in
  mir-reduce compression (~520 KB heap per stream object; output is
  byte-identical across architectures by using the strict hash). UNSPEC and
  the gen-internal insns are not serializable. **lref (label-address) data
  doesn't survive `MIR_write`/`MIR_read`** (upstream #426) — computed-goto
  modules can't be cached as .bmir until that's fixed.
- `MIR_NO_IO` / `MIR_NO_SCAN` / `MIR_NO_BIN_COMPRESSION` compile-time knobs
  exist for embedders that want a smaller library.

---

## 13. Testing & validation

What upstream CI runs per platform: `make test` = readme example +
mir-bin-run + c2mir suite (sieve + the ~1054-program c-tests corpus × modes
interp/gen/gen-bb/O0/O1/O3 + 6 self-bootstrap rounds comparing .bmir).
GitHub workflows cover ubuntu/macos/windows x86_64, Apple aarch64, and
self-hosted aarch64/ppc64le/riscv64/s390x. Other useful targets:
`make test-all` (adds ADT/io/scan/mir2c tests), `c2mir-parallel-gen-test`
(not in default `test`!), `make bench`, and the csmith fuzz scripts
(`csmith-c2m.sh`: 10k programs, interp vs lazy-gen diff).

For this fork, the pragmatic tiers:

1. **Smoke** (seconds): `make c2mir-simple-test` + the slimcc-side zero-fs
   smoke and `slimcc-mir-run` over slimcc's test corpus.
2. **Standard** (minutes): `make test` on x86_64 and aarch64 — this is what
   upstream CI gates on, and what we ran to validate the RA fixes.
3. **Deep**: csmith soak; slimcc test corpus under valgrind; the leak
   harnesses in slimcc's notes/mir-backend/.

Known pre-existing upstream failures (not caused by this fork's changes):
3 gen-bb long-double c-tests on aarch64 (20020413-1, 20030914-1, regstack-1)
fail at the upstream base commit too; x86_64 passes them. We don't use BBV.

musl note: the *library* is musl-clean (sysconf/mmap/mprotect only); what
breaks on Alpine is the **test drivers** — c2mir-driver.c and mir-bin-run.c
dlopen hardcoded glibc paths (`/lib/x86_64-linux-gnu/libc.so.6` etc.,
c2mir-driver.c:69-124), which is upstream #307. Patch those tables (musl is
`/lib/ld-musl-<arch>.so.1`, no separate libm/libpthread) to run the suite on
Alpine.

---

## 14. Maintainer checklists

**Adding/changing a MIR insn** touches, at minimum:
1. `insn_descs[]` (mir.c) — operand modes **with correct OUT_FLAGs**;
2. the interpreter switch (mir-interp.c);
3. `patterns[]` in *all five* mir-gen-<arch>.c (or machinize it to a
   builtin call);
4. machinize, if it has ABI implications;
5. the special-case lists: branch detection in `build_func_cfg`, `jump_opt`,
   DCE keep-list (mir-gen.c:9221-9229), combine substitution legality,
   BBV branch dispatch (mir-gen.c:9844-9986);
6. `target_io_dup_op_insn_codes` / early-clobber lists if the encoding
   needs them;
7. text scanner/binary IO get it for free from insn_descs, *unless* it has
   unusual operands.

**Recurring bug classes seen in this codebase**:
- def/use wrongness from insn_descs (→ §5);
- terminator rewriting that assumes label operands (→ §8);
- stale lengths in gen_ctx varrs/bitmaps reused across functions;
- `op.data`/`insn->data` phase polymorphism (ssa_edge vs bb_insn vs
  insn_data) — always pair `add_ssa_edge`/`remove_ssa_edge`;
- `VARR_ADDR` invalidated by push (classic dangling pointer);
- bb bitmap aliasing across phases (live vs dom vs spill);
- implicit-extension semantics of narrowing loads vs GVN substitution
  (upstream #423).

**Touch-with-care list**: interp shim frame constants (aarch64), pattern
table ordering, `wrapper_end` reachability (aarch64 `b`-range retry loop),
combine's label deletion (BBV depends on it), `simplified_p` consistency
across assign/rewrite/reg_alloc.

---

## 15. Fork status

Branches:
- `fix-laddr-out-flag` — the two RA fixes, submitted upstream as
  vnmakarov/mir#430 (LADDR OUT_FLAG; simplified RA for JMPI functions).
- `meson` — fix-laddr-out-flag + meson build (static `libmir`, `mir`
  dependency for subproject wraps) + this document.

Upstream issues we have verified do **not** reproduce on current master
(tested x86_64 + aarch64, O0-O2): #308 (DSE wrong-store, was bbv-branch),
#249 (>32-bit absolute displacement truncation). Upstream issues that **do**
affect master and matter to us: #423 (GVN sign-extension), #424 (jump_opt
lref use-after-free), #426 (lref binary IO) — see the triage document in the
slimcc repo (notes/mir-backend/upstream-triage.md) for the full review and
priorities.
