# Audit record — axiom audit of the five public theorems

Machine-readable companion: `single-node-replay-2026-11-22.receipt.json` and the raw Lean diagnostics in `single-node-replay-2026-11-22.axioms.jsonl`.

## 1. Audit file

`AxiomCheck.lean` — SHA256 `9d6e2aae8260e4926914e3cd61d9117f5221f168de1fdc2c671d883abc25594f`

```lean
import Main

#print axioms riemannHypothesis
#print axioms riemannZeta_eq_zero_re
#print axioms LFunction_ne_zero_of_half_lt_re
#print axioms LFunction_eq_zero_re
#print axioms LFunction_eq_zero_re_of_isPrimitive
```

## 2. Command and exit status

```
lake env lean AxiomCheck.lean
```

Exit code **0**, elapsed ≈ 29 s.

## 3. Text output (verbatim)

```
'riemannHypothesis' depends on axioms: [propext, Classical.choice, Quot.sound]
'riemannZeta_eq_zero_re' depends on axioms: [propext, Classical.choice, Quot.sound]
'LFunction_ne_zero_of_half_lt_re' depends on axioms: [propext, Classical.choice, Quot.sound]
'LFunction_eq_zero_re' depends on axioms: [propext, Classical.choice, Quot.sound]
'LFunction_eq_zero_re_of_isPrimitive' depends on axioms: [propext, Classical.choice, Quot.sound]
```

SHA256 of the captured text: `2c7b168860c39a4b4afd417a327bbbd8a2a0615372655c5d4bd329fb06e7f3d5`.

## 4. Machine-readable output

`lake env lean --json AxiomCheck.lean` emits one JSON object per message. The raw file `single-node-replay-2026-11-22.axioms.jsonl` has **5** lines, SHA256 `f93e202c1a9f10c2ab5e41c209e85470ecf407f64f87042a7b1ecfe980e75eb2`. Each line carries `severity: "information"`, the position in `AxiomCheck.lean`, and the axiom list as `data`.

## 5. Verdict

| Check | Result |
|---|---|
| Axioms ⊆ {propext, Classical.choice, Quot.sound} | **PASS** |
| Custom axioms present | none |
| `sorryAx` present | none |

## 6. Kernel replay

`leanprover/lean4checker` is deprecated; the checker is now shipped inside Lean as `leanchecker` and is therefore the same implementation that any reader obtains by installing toolchain 4.33.1. Two scopes were run:

| Command | Scope | Result |
|---|---|---|
| `lake env leanchecker Main` | declarations **defined in** module `Main`, against the environment provided by its imports | **PASS**, exit 0, 42 s |
| `lake env leanchecker --fresh Main` | **every** declaration reachable from `Main`, replayed into a genuinely fresh environment | **in progress** — started 2026-11-22T17:05:25+08:00, still running; single-threaded by design |

The fresh replay is single-threaded (replaying into a fresh environment requires dependency order), so it cannot be parallelised. Its log (`leanchecker-fresh.log`, SHA256 `e8e0a0a8b5ce6a8db25241ed5f72a42b3b7c58d54e56d984eb75df2d5ca824fb`) is retained on the host.

## 7. What this does and does not establish

Establishes: the five propositions exported by `Main` are kernel-checked from the archive's own material, with no assumptions beyond Lean's three standard axioms.

Does **not** establish:
- that any load-bearing proposition (`Row0b`, `PropsB`, `GeoOne`, `TLiftK`) is **non-vacuously** satisfiable — a `∀ premises → ∃ conclusion` statement holds trivially if the premises are unsatisfiable, and neither the build nor the axiom audit can detect this;
- an independent fresh-kernel replay with `lean4checker` (not performed; `lean4checker` was not installed on the host);
- mathematical correctness in the human sense.
