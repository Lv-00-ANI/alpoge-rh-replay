# Independent single-node replay of Alp枚ge's RH archive

**Not for merge.** This document is submitted as an independent, single-machine reproduction record. It is offered as corroborating evidence for the community replay in this repository. It is not a mathematical verdict on the claim.

---

## 1. Result summary

| Item | Outcome |
|---|---|
| Archive integrity | **Pass** 鈥?all 14 parts match `parts.sha256` byte-for-byte |
| Toolchain / dependency pinning | **Pass** 鈥?Lean 4.33.1 + mathlib `0df444a3鈥 |
| Full-project kernel build | **Pass** 鈥?175,187 / 175,187 modules, **0 errors**, exit status 0 |
| Axiom audit of the five public theorems | **Pass** 鈥?only `propext`, `Classical.choice`, `Quot.sound` |
| `sorry` / `admit` / `axiom` / `unsafe` / `native_decide` / `implemented_by` / `opaque` | **0 occurrences** in 175,187 files |

The build finished at `2026-11-22T08:04:12+08:00` (CST) with `Main.olean` present.

This establishes that the published archive is **self-contained and machine-checkable as released**: it compiles from source against the pinned toolchain and mathlib revision, and the final theorem carries no axioms beyond Lean's three standard ones. It does **not** establish that the formal statement captures the intended mathematics beyond the checks described in 搂6, nor that the argument is mathematically correct in the human sense.

---

## 2. Object under test

| Item | Value |
|---|---|
| Claim | Levent Alp枚ge, *Remark*, 2026-11-20 鈥?<https://alpo.ge/remark.pdf> |
| Form | **No paper**; a 4-page note plus 14 archive parts |
| Archive | `https://alpo.ge/lean.tar.gz.part01` 鈥?`part14` |
| Part sizes | part01鈥?3: **18,000,000** bytes each; part14: **5,275,554** bytes |
| Assembled | **239,275,554** bytes (228 MiB) |
| Assembled SHA256 | `9631b4f020199e580ba86178cb56c6e9b09f690c2369a80dc4334aa93f49c7fc` |
| Extracted | 175,190 entries, 0.96 GiB, **175,187** `.lean` files, **17,549,068** lines |

The 14 per-part SHA-256 digests were compared against this repository's `parts.sha256` and found identical for all 14 entries, so this replay operated on the same bytes the rest of the project verifies.

---

## 3. Environment

| Item | Value |
|---|---|
| Host | PVE virtual machine (rented), data on a dedicated NVMe drive |
| CPU | Intel i5-14600KF, **19 threads** |
| RAM | **62 GiB** |
| OS | Debian GNU/Linux 13 (trixie), kernel `6.12.113+deb13-amd64` |
| Filesystem | XFS, `/data`, 1.9 TB |
| Lean | `4.33.1`, commit `819816b2e0a3bf405af45ae5c7af2491d8f5bee6` |
| mathlib | `0df444a360eaa60ab8c11dca51a86af692955474` |
| `lake-manifest.json` SHA256 | `fee57ee9ed399d0a638ca955e5ad40a37a36eeb9b6dfd9f5964114fa7bbe4774` |
| Final artifact footprint | 1.3 TB under `.lake` |

### 3.1 Toolchain installation note

The official release publishes no `.sha256` sidecar for `lean-4.33.1-linux.tar.zst`, and a single-connection download from `releases.lean-lang.org` was throttled to roughly 200 KB/s on this network. The toolchain was therefore fetched with 8 parallel ranged `curl` requests and then validated three ways:

1. assembled size exactly **570,405,234** bytes, matching the GitHub release asset metadata;
2. `zstd -t` passes;
3. `lean --version` reports `Lean (version 4.33.1, x86_64-unknown-linux-gnu, commit 819816b2鈥? Release)`.

Because no upstream checksum exists, this is an indirect but triple-confirmed validation rather than a cryptographic one.

---

## 4. Build

### 4.1 Method

`lake build Main` was run under a systemd unit that restarts the loop on failure, so the build survives host reboots and resumes incrementally:

```bash
for round in 1..N:
  lake build Main            # rounds 1鈥? pinned to cores 0鈥?7, later rounds to core 0
```

Before each round, zero-byte `.olean` files left behind by aborted compilations are removed.

### 4.2 Timeline

| Time (CST) | Event |
|---|---|
| 2026-11-21 01:04:57 | first build start (19-way parallelism) |
| 2026-11-21 01:29 | first native crashes observed |
| 2026-11-21 01:34:49 | **host rebooted** (VM lost) |
| 2026-11-21 02:26 | resumed under systemd at 18鈥?9 cores |
| 2026-11-21 ~01:52鈥?2:21 | second host reboot; SSH unreachable |
| 2026-11-22 08:04:12 | **build completed, exit 0** |

Total wall-clock 鈮?31 h, of which the build itself ran 鈮?22 h at an effective 90鈥?30 modules/min. The service reported `NRestarts=0` after the final restart, i.e. the last stretch ran uninterrupted for over 24 h.

### 4.3 Outcome

```
175,187 / 175,187 modules built
0 errors
exit status 0
Main.olean present
```

### 4.4 A note on concurrency-induced crashes (useful for other volunteers)

Early runs at high parallelism produced sporadic native crashes 鈥?`exit 139` (SIGSEGV), `134` (SIGABRT), `132` (SIGILL) 鈥?with `dmesg` reporting `segfault 鈥?in libleanshared.so`. These were investigated and characterised:

| Hypothesis | Verdict | Evidence |
|---|---|---|
| Out of memory | rejected | `free -g` showed 4鈥? GiB used of 62 GiB |
| Corrupted `.olean` from reboot | rejected | same modules crashed at 01:29, **before** the 01:34 reboot; the relevant `.olean` files were intact |
| Stack overflow | rejected | re-running `Formalism.HermitianCurvature.HermitianCurvatureProofs.HermitianFrameCount` as a single process succeeded **4/4**, each producing a byte-identical 4,585,088-byte `.olean`; raising `--tstack` did not help |
| Source-level defect | rejected | serial execution passes |

Conclusion: on this machine the failures are **concurrency-dependent non-deterministic runtime crashes**, affecting roughly 0.2 % of targets under 12鈥?9-way parallelism, and reproducible only under concurrency. They disappear when the affected modules are compiled serially. Volunteers using high parallelism may therefore see random failures that do **not** indicate a broken archive.

---

## 5. Axiom audit

`AxiomCheck.lean`:

```lean
import Main

#print axioms riemannHypothesis
#print axioms riemannZeta_eq_zero_re
#print axioms LFunction_ne_zero_of_half_lt_re
#print axioms LFunction_eq_zero_re
#print axioms LFunction_eq_zero_re_of_isPrimitive
```

Output of `lake env lean AxiomCheck.lean`:

```
'riemannHypothesis' depends on axioms: [propext, Classical.choice, Quot.sound]
'riemannZeta_eq_zero_re' depends on axioms: [propext, Classical.choice, Quot.sound]
'LFunction_ne_zero_of_half_lt_re' depends on axioms: [propext, Classical.choice, Quot.sound]
'LFunction_eq_zero_re' depends on axioms: [propext, Classical.choice, Quot.sound]
'LFunction_eq_zero_re_of_isPrimitive' depends on axioms: [propext, Classical.choice, Quot.sound]
```

No declarations beyond Lean's three standard axioms.

---

## 6. What the target statement actually is

`src/Main.lean` proves five propositions. The first is

```lean
theorem riemannHypothesis : RiemannHypothesis := rhOfAll
```

where `RiemannHypothesis` is **Mathlib's own definition** (`Mathlib/NumberTheory/LSeries/RiemannZeta.lean`):

```lean
def RiemannHypothesis : Prop :=
  鈭€ (s : 鈩? (_ : riemannZeta s = 0) (_ : 卢鈭?n : 鈩? s = -2 * (n + 1)) (_ : s 鈮?1), s.re = 1 / 2
```

The reduction chain is short and human-checkable:

```
Main.riemannHypothesis
  := rhOfAll                                     Supplements/Kernels/MainTheorems.lean:9
  := Formalism.VW.rh_of_every allChi             (GRH for Dirichlet L-functions 鈬?RH)
  := ...
allChi := UpperTheory.FinalTheorems.everyCharacter
       := everyCharacter_of_row galoisLocalGlobalStrong_recParameter
galoisLocalGlobalStrong_recParameter : Row0b recT    IntegralStructures/RowSrc.lean:6
```

`UpperTheory.EveryCharacter` is literally the generalized Riemann hypothesis for all Dirichlet characters:

```lean
def UpperTheory.EveryCharacter : Prop :=
  鈭€ (N : 鈩? [NeZero N] (蠂 : DirichletCharacter 鈩?N) (s : 鈩?,
    1 / 2 < s.re 鈫?(蠂 鈮?1 鈭?s 鈮?1) 鈫?DirichletCharacter.LFunction 蠂 s 鈮?0
```

`Claim.zeta_iff : Zeta 鈫?RiemannHypothesis := Iff.rfl` (`CharacterBoxes/ClaimStrip.lean:102`) shows the independently restated targets in `Statements.lean` are definitionally equal to Mathlib's, so the additional top-level theorems in `Main.lean` are type-level restatements rather than further checking.

**The mathematical content therefore lies entirely in `Row0b` and the propositions it depends on (`PropsB`, `GeoOne`, `TLiftK`, `CoreAll`).** This replay verifies that those are *proved* from the archive's own material with no axioms, not that they are *true* in the intended mathematics.

---

## 7. Caveats and open questions

1. **Vacuous-truth risk is not detectable by build or axiom checks.** `Row0b` is a `鈭€ premises 鈫?鈭?conclusion` statement. If its premises were unsatisfiable, the statement would hold trivially while the kernel, the build and the axiom audit all pass. Relevant targets: satisfiability of `(CG.mk 蟺).IsRegAlg` (a witness exists at `ExpoConjOfIsCsd.lean:765`), of `PolarizedPair 蟺 蠂` (witness only for `a = 1` at `PolarizedPairLineIff.lean:170`), of `CuspSub E a` for `a 鈮?2`, and joint satisfiability of the nine hypotheses of `GeoOne`. **This report does not resolve these.**
2. **`*Statement.lean` files are not an independent statement layer.** 45,987 of 46,027 paired `XStatement.lean` files (99.9 %) import the corresponding proof file and re-export the theorem in a single line; across all 46,610 such files the ratio is 98.7 %. They cannot be used for statement-level review without reading the proof file.
3. **The archive is heavily machine-generated in shape.** `Development/AbelianVarietyTheory/Proofs/DivisorProofs` is a single flat directory containing 42,424 files (24.2 % of the whole project) with no subdirectories.
4. **No peer review.** This is one machine's reproduction, not a community verdict.

---

## 8. Reproduction

```bash
# 1. inputs
sha256sum -c parts.sha256
cat part0{1..9} part1{0..4} > lean.tar.gz
gzip -t lean.tar.gz && tar -xzf lean.tar.gz

# 2. toolchain and dependencies
elan toolchain install leanprover/lean4:v4.33.1
cat lean-toolchain            # leanprover/lean4:v4.33.1
lake update
git -C .lake/packages/mathlib rev-parse HEAD   # 0df444a3鈥?lake exe cache get

# 3. build (low parallelism + retry is strongly recommended)
lake build Main

# 4. axiom audit
cat > AxiomCheck.lean <<'EOF'
import Main
#print axioms riemannHypothesis
EOF
lake env lean AxiomCheck.lean
```

---

## 9. Submission context

Evidence for the individual module and packet claims referenced by this report was posted as `Replay submission` comments on the corresponding issues (modules `M175185`, `M175187`, `M175186`, `M108233`, `M109322`, `M170889`; packets `P2118`, `P2105`, `P2077`).

This PR intentionally does not modify the replay tooling. It adds this record only, and **requests no merge**.

