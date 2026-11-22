# Build record — independent single-node replay

Machine-readable companion: `single-node-replay-2026-11-22.receipt.json`.

## 1. Command

```
lake build Main
```

Executed under a systemd unit (`rh-build.service`, `Restart=on-failure`) driving `/root/rh_build_loop.sh`, so the build resumes incrementally after host reboots. Each round removes zero-byte `.olean` files left by aborted compilations before starting.

## 2. Timeline (Asia/Shanghai)

| Time | Event |
|---|---|
| 2026-11-21 02:26:41 | build loop started on a clean tree (18 cores pinned, later 19) |
| 2026-11-21 01:34:49 | earlier attempt: host reboot #1 (VM lost) |
| 2026-11-21 ~02:21 | host reboot #2; SSH unreachable, loop resumed automatically on boot |
| **2026-11-22 08:04:12** | **`lake build Main` exits 0; `ALL_DONE`** |

The final `lake build Main` invocation ran uninterrupted for **29 h 37 min**. `systemd` reported `NRestarts=0` for that run.

Raw markers from the build log:

```
START 2026-11-21T02:26:41+08:00 host=xiaomeng nproc=19
=== ROUND 1 CPUS=0-17 2026-11-21T02:26:41+08:00 ===
START 2026-11-21T02:26:58+08:00 host=xiaomeng nproc=19
=== ROUND 1 CPUS=0-17 2026-11-21T02:26:58+08:00 ===
START 2026-11-21T02:30:22+08:00 host=xiaomeng nproc=19
=== ROUND 1 CPUS=0-17 2026-11-21T02:30:22+08:00 ===
START 2026-11-21T02:33:52+08:00 host=xiaomeng nproc=19
=== ROUND 1 CPUS=0-18 2026-11-21T02:33:52+08:00 ===
=== ROUND 1 EXIT=0 2026-11-22T08:04:12+08:00 ===
ALL_DONE 2026-11-22T08:04:12+08:00
```

## 3. Outcome

| Metric | Value |
|---|---|
| Modules built | **175,187 / 175,187** |
| `.olean` files | 175,187 |
| `.ilean` files | 175,187 |
| Errors | **0** |
| Exit code | **0** |
| `.lake` size | 1,294,423,698,907 bytes (1.29 TB) |

## 4. Build log

| Item | Value |
|---|---|
| File | `build7.log` (not committed — 171 MiB) |
| Bytes | 178,976,744 |
| Lines | 2,427,878 |
| SHA256 | `573b516655397a84746b90241bc373386642b1db663b0a6c02d5bdcb357b1786` |

The log is retained on the build host; the digest above pins it. Anyone reproducing the run can compare their own log size and line count as a coarse check.

## 5. Environment

| Item | Value |
|---|---|
| Host | `xiaomeng` |
| OS | Debian GNU/Linux 13 (trixie) |
| Kernel | `6.12.113+deb13-amd64` |
| CPU | Intel(R) Core(TM) i5-14600KF, 19 threads |
| RAM | 62 GiB |
| Data disk | 1.9T XFS |

## 6. Concurrency-induced native crashes

High-parallelism runs produced sporadic native crashes (`exit 139` SIGSEGV, `134` SIGABRT, `132` SIGILL) with `dmesg` reporting `segfault … in libleanshared.so`. Ruled out, with evidence:

| Hypothesis | Verdict | Evidence |
|---|---|---|
| OOM | rejected | 4–5 GiB used of 62 GiB at 6-way parallelism |
| Corrupted `.olean` from reboot | rejected | the same modules crashed at 01:29, before the 01:34 reboot; affected `.olean` files were intact |
| Stack overflow | rejected | single-process re-run of `Formalism.HermitianCurvature.HermitianCurvatureProofs.HermitianFrameCount` succeeded 4/4 with byte-identical 4,585,088-byte artifacts; raising `--tstack` did not help |
| Source defect | rejected | serial execution passes |

Approximately 0.2 % of targets under 12–19-way parallelism. Practical guidance: **low parallelism plus retry**; random failures under high parallelism do not indicate a broken archive.

## 7. Reproduction

```bash
sha256sum -c parts.sha256
cat part0{1..9} part1{0..4} > lean.tar.gz && gzip -t lean.tar.gz && tar -xzf lean.tar.gz
elan toolchain install leanprover/lean4:v4.33.1
lake update && lake exe cache get
lake build Main        # low parallelism + retry recommended
```
