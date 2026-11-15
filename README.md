# WARNING in register_fprobe_ips (mm/slub.c:5459, v7.4-rc2)

Syzkaller-found bug on Linux v7.4-rc2, reproduced and verified on the same
kernel build. Externally reproduced **2/2 times** on fresh VMs using the
exact procedure described below.

## Summary

A warning fires during fprobe registration while the kernel is performing
a large kmalloc:

```
WARNING: mm/slub.c:5459 at ___kmalloc_large_node+0x13f/0x180 mm/slub.c:5459, CPU#2: syz.4.4204/72997
RIP: 0010:___kmalloc_large_node+0x13f/0x180 mm/slub.c:5459
```

The warning is triggered inside `___kmalloc_large_node()` (mm/slub.c:5459)
during a fprobe registration operation that requires a large kmalloc
allocation. The guest runs with `panic_on_warn=1`, which escalates this
warning to a kernel panic.

## How to reproduce

### Prerequisites

1. Boot a Debian trixie x86_64 VM (6 GiB RAM, 4 vCPU, KVM) using the QEMU
   command line in `environment.txt`, with the kernel built from v7.4-rc2
   (see `kernel.config`; CONFIG_KASAN=y).
2. Copy `syz-execprog` and `syz-executor` (built from syzkaller commit
   `853b104bb454141a5b0fc0e4218ec78e9f34ffb1`) into the guest.
3. Copy `reproducer/repro.kernsight.prog` into the guest.

### Run

In the guest, as root:

```sh
syz-execprog -procs=8 -threaded -repeat=0 -executor ./syz-executor repro.kernsight.prog
```

### What you will see

After **~6-13 minutes** the serial console (or `dmesg`) will show:

```
WARNING: mm/slub.c:5459 at ___kmalloc_large_node+0x13f/0x180 mm/slub.c:5459, CPU#N: syz.N.NNN/NNNNN
```

followed by a kernel panic (escalated by `panic_on_warn=1`).

### Verified runs

| Run | Date | WARNING at (after boot) | Serial log |
|---|---|---|---|
| 1 | 2026-11-14 | 384.3 s | `logs/external-verification/external-verification-serial.log` |
| 2 | 2026-11-15 | 774.4 s | `logs/external-verification/external-verification-serial-run2.log` |

Both runs used the exact same procedure: chunked single-pass replay of the
84-entry recipe (25 programs per chunk, sequential chunks on a fresh VM),
matching the fuzzing pipeline's verifyOnce semantics.

## Why the reproducer is a multi-program set

The WARNING depends on kernel state built up by preceding operations
(including `syz_mount_image$jfs` and fprobe registration). The 84-program
set constructs the required state before the triggering operation.

## Files

- `kernel.config` — kernel configuration used (also required to interpret addresses)
- `environment.txt` — kernel image info (sha256), QEMU command line, syzkaller commit, reproducer command
- `reproducer/repro.kernsight.prog` — pipeline-verified reproducer (84-program ordered set)
- `reproducer/repro.prog` — triggering program set from the pipeline
- `reproducer/repro.priming.prog` — state-priming program set (pipeline repro recipe)
- `logs/log-0.txt`, `logs/report-0.txt` — original syzkaller execution log and crash report
- `logs/machineInfo-*.txt`, `logs/title-stat.json` — guest environment info and title statistics
- `logs/external-verification/` — external verification serial consoles (2 runs)

## Notes for maintainers

- The warning is in the SLUB allocator's large allocation path during
  fprobe registration; the crafted workload triggers a large kmalloc
  that hits an unexpected condition in the allocator.
- We can provide the kernel image, additional crash instances or raw
  artifacts on request.
