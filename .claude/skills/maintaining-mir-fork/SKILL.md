---
name: maintaining-mir-fork
description: Branch topology, upstream contribution workflow, test commands, and per-platform test baselines for this MIR fork (cyanogilvie/mir, upstream vnmakarov/mir). Use when fixing MIR bugs, adding regression tests, running the c-tests suite or bootstraps, rebasing the meson branch, validating on aarch64 or musl, or filing upstream PRs.
---

# Maintaining the MIR fork

This is a maintained fork of vnmakarov/mir (MIT JIT; upstream near-dormant since Aug 2024). It is the JIT substrate for libslimcc (`/home/cyan/git/slimcc`, branch `mir-backend` — see that repo's `developing-libslimcc` skill) and the future tclmir package. Production targets: aarch64+musl primary, x86_64+glibc secondary.

**For architecture and implementation internals, read `INTERNALS.md` at the repo root first** — 15 sections covering repo map, MIR_context, modules/items, W^X code memory, insns/operands, simplify+inlining, the generator pipeline, the register allocator, target backends, runtime stubs/FFI, error handling, serialization, testing, maintainer checklists, and fork status. Do not re-derive any of that.

## Branch topology

- `origin` = upstream vnmakarov/mir, `fork` = git@github.com:cyanogilvie/mir.
- **One topic branch per fix, branched off upstream `master`**, each independently PR-able with its own regression test: `fix-laddr-out-flag`, `fix-gvn-load-ext`, `fix-jump-opt-lref-labels`, `fix-aarch64-ld-stack-align`, `fix-aarch64-bb-thunk-clobber`, `support-musl-std-libs`, `pr-420` (cherry-picked foreign PR).
- **`meson` = the integration branch**: all topic branches merged + fork-only files (meson.build, meson.options, INTERNALS.md, this skill). Consumers (slimcc/tclmir wraps) pin this branch. When upstream merges a PR, rebase `meson` and update consumers' wrap revisions.

### Workflow for a new fix

1. Branch off upstream `master`; make the fix.
2. Add a regression test: `c-tests/mir/NAME.mir` (self-checking, `main`'s exit code; optional `.expect` stdout / `.expectrc` exit code / `.mach`/`.nomach` arch filters) or `c-tests/new/NAME.c`. Verify it fails before / passes after.
3. Validate (commands below), merge into `meson`, push both branches to `fork`.
4. File upstream: `gh pr create -R vnmakarov/mir --head cyanogilvie:BRANCH --body-file ...`.

Pulling someone else's upstream PR: `git fetch origin pull/N/head:pr-N` then `git cherry-pick -x` onto a local branch (preserves authorship), merge to `meson`.

## Build & test

```sh
make all -j$(nproc)      # GNUmakefile; binaries land in the repo root
make test -j$(nproc)     # full suite: c-tests in 6 modes + bootstraps; ~20-40 min

# One mode of c-tests directly (~1073 tests; modes: use-c2m-interp, use-c2m-gen,
# use-c2m-gen-bb, use-c2m-O0, use-c2m-O1, use-c2m-O3):
sh c-tests/runtests.sh c-tests/use-c2m-gen ./c2m
```

Meson (fork-only, for subproject-wrap consumption as a static lib): `meson setup build && meson compile -C build`. Option `c2mir` (default true) includes the C frontend; flags `-fsigned-char -fno-tree-sra -fno-ipa-cp-clone` are required (see INTERNALS.md §13).

## Per-platform baselines — expected failures, already root-caused

| Platform | Baseline |
|---|---|
| x86_64 glibc | Fully green incl. all bootstraps |
| aarch64 glibc | Green except 3 **pre-existing** gen-bb long-double c-tests: `regstack-1`, `20020413-1`, `20030914-1` (fail at upstream base commit too; gen-bb mode only) |
| aarch64 musl (Alpine) | Those 3 do NOT reproduce. Only remaining failure: `c-tests/new/jcall.c` segfaults after main returns in gen modes only (exotic `__builtin_jcall`/jret + global reg var; passes interp and glibc; we never emit JCALL). `c2mir-bb-bootstrap-test` passes (~7s, 535MB peak) WITH the fork's `fix-aarch64-bb-thunk-clobber`; without it, it deterministically OOMs (~8GB) — that was upstream #436, NOT a musl memory characteristic |

A regression is a change against these lists. BusyBox diff lacks `--strip-trailing-cr` — runtests.sh probes for it (was 30 phantom failures/mode before).

## Fork fixes and upstream filings (as of 2026-06)

Open PRs: **#430** (LADDR out-flag + simplified-RA jmpi), **#432** (GVN store-forwarding loses sign/zero extension, fixes #423), **#433** (jump_opt frees LADDR/lref labels → UAF, fixes #424), **#434** (aarch64 `% 16` where round-up meant — mir-aarch64.c va_arg `__stack` and ff_call sp_offset; fixes #431), **#435** (musl std-lib tables in the test drivers, fixes #307), **#437** (lazy-BB far-range fixes: aarch64 bb-thunk x9 clobber + 128MB code-space reservation; fixes #436). Cherry-picked: #420 (error-path null deref). Watchlist: #383 (`MIR_get_global_item` undefined — cherry-pick when needed), #426 (lref breaks .bmir round-trip — blocks caching computed-goto modules), #429 (did NOT reproduce), #411 (c2m compile-time memory generally — distinct from the fixed #436).

Bug-class lessons (details in `INTERNALS.md` §14-15 and the triage doc at `/home/cyan/git/slimcc/notes/mir-backend/upstream-triage.md`):
- **`% 16` vs `/16*16`**: grep target files for `% 16` when touching stack/alignment code.
- **jump_opt's do-not-remove label bitmap** must include every label referenced outside branch terminators: LADDR operands and `func->first_lref` chains (mirror `build_func_cfg`'s reachability handling).
- **GVN store-forwarding** must materialize the narrowing load's extension (EXT8…UEXT32) — a forwarded raw register is wider than the mem type.
- **Thunk far-redirect forms clobber a register** — the bb thunk passes its payload in a register (aarch64 x9, riscv t5, ppc64 r11, x86_64 r10), and the redirect's far form must use a *different* fixed temp (aarch64 now x10; x16/x17 are RA-allocatable hence unsafe across bb borders). All JIT code is assumed within direct-branch range of all other JIT code (`setup_rel` hard-exits otherwise) — guaranteed since #437 by the contiguous code-space reservation in mir.c; allocator mmap patterns (musl mallocng) otherwise scatter code holders.
- **Hand-building c2m for debugging MUST use the GNUmakefile flags** — `-fsigned-char -fno-tree-sra -fno-ipa-cp-clone` (`out_insn`'s template parser breaks with unsigned plain char on aarch64: `char d; (d = hex_value(*p)) >= 0` never terminates). A plain `gcc -O2` build produces convincing-but-bogus failures.
- **musl ≠ glibc test envs**: one DSO `/lib/ld-musl-<arch>.so.1` for libc/libm/libpthread/libdl (`__linux__ && !__GLIBC__`); `bits/alltypes.h` redeclares wchar_t per-arch — c2mir's aarch64 stddef/linux headers carry unsigned wchar_t for Linux (Apple keeps int).

## Remote validation boxes

Ephemeral EC2 instances, `ssh -i ~/.ssh/General.pem` — IPs change every start, ask the user for current ones. Ubuntu aarch64: user `ubuntu`. Alpine aarch64: user `alpine`; **login shell is fish** — wrap everything in `sh -c '...'` or `ssh ... 'sh -s' <<'EOF'`; privilege escalation is `doas`, not sudo. When polling a remote suite, `pgrep -f "make test"` matches its own ssh wrapper — use `pgrep -a make`.
