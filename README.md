# WARNING in register_fprobe_ips (mm/slub.c:5459, v7.4-rc2)

Syzkaller-found bug on Linux v7.4-rc2, reproduced and verified on the same
kernel build (fuzzing pipeline reproducer: confirmed=true, status=verified).

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

Environment: Debian trixie x86_64 guest (6 GiB RAM, 4 vCPU, KVM), kernel
built from v7.4-rc2 with CONFIG_KASAN=y, CONFIG_FPROBE=y, booted with
`oops=panic` and `panic_on_warn=1`. See `environment.txt` for the exact
QEMU command line and `kernel.config` for the configuration.

The crash depends on kernel state built up by the preceding programs. The
reproducer is a multi-program syzkaller execution log (83 programs):

```sh
# in the guest, as root:
syz-execprog -procs=8 -threaded -repeat=0 -executor ./syz-executor repro.kernsight.prog
```

Verified by the fuzzing pipeline's reproducer machinery on a fresh VM
(reproducer pipeline outcome: confirmed=true, status=verified). A C
reproducer has not been included yet.

## Files

- `kernel.config` — kernel configuration used (also required to interpret addresses)
- `environment.txt` — kernel image info (sha256), QEMU command line, syzkaller commit, reproducer command
- `reproducer/repro.kernsight.prog` — pipeline-verified reproducer (83-program ordered set)
- `reproducer/repro.prog` — minimal triggering testcase (state-dependent; does not reproduce alone)
- `reproducer/repro.priming.prog` — state-priming program set (pipeline repro recipe)
- `logs/log-0.txt`, `logs/report-0.txt` — original syzkaller execution log and crash report from the fuzzing pipeline
- `logs/machineInfo-*.txt`, `logs/title-stat.json` — guest environment info and title statistics

## Notes for maintainers

- The warning is in the SLUB allocator's large allocation path during
  fprobe registration; the crafted workload triggers a large kmalloc
  that hits an unexpected condition in the allocator.
- We can provide the kernel image, additional crash instances or raw
  artifacts on request.
